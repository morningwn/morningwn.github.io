---
title: GTID、并行复制与复制延迟排查
summary: 深入讲解 GTID 模式的原理与配置、多线程并行复制的策略演进、复制延迟的系统排查方法，以及主从切换的数据一致性保障。
created: 2026-07-02
updated: 2026-07-16
tags: MySQL, GTID, 并行复制, 主从复制, 高可用
---

# GTID、并行复制与复制延迟排查

主从复制在 MySQL 高可用体系中承担读写分离、灾备与故障切换的基础能力。随着业务并发写入提升、拓扑从"一主一从"演化为多层级联与多源汇聚，传统 File/Position 复制在切换对位、级联管理与多源调度上运维成本高、易出错。MySQL 5.6 引入 GTID，并在 5.7/8.0 持续演进并行复制（MTS）策略，使自动定位位点、加速从库回放、系统排查延迟与安全切换成为可工程化落地的能力。

本文专章深入 GTID 原理与配置、并行复制版本演进、复制延迟排查，以及主从切换与多源复制实践。阅读前建议已了解 Binlog 格式与主从三线程模型；可与《MySQL 主从复制原理与实践》《Binlog：格式、写入机制与数据恢复》对照。

## 一、GTID 模式

### 1. GTID 的概念

**Global Transaction Identifier（全局事务标识符）** 是 MySQL 为每个已提交事务分配的唯一编号。开启 GTID 后，主库在 Binlog 中写入`GTID_LOG_EVENT`，从库维护已执行 GTID 集合，据此判断哪些事务尚未应用，而不再依赖"从哪个 Binlog 文件、哪个偏移开始读"。设计目标是将复制状态从"文件 + 字节偏移"的物理坐标，升级为"事务 ID 集合"的逻辑坐标，使故障切换、级联与多源拓扑中的位点对齐可自动化完成。

**格式：`source_uuid:transaction_id`**

```
source_uuid:transaction_id
```

- **source_uuid**：生成该 GTID 的实例 UUID（`SHOW VARIABLES LIKE 'server_uuid'`，或 `auto.cnf`），首次启动时生成，全局唯一。
- **transaction_id**：在该 UUID 下单调递增的正整数，从 1 开始，每提交一个事务加 1。

示例：`3E11FA47-71CA-11E1-9E33-C80AA9429562:154`。正常情况下同一实例 GTID 连续；若 `uuid:100` 已执行而 `uuid:99`未执行，说明复制异常或曾手工跳过事务。

**与传统 File/Position 的对比**

| 维度        | File/Position                        | GTID                                   |
|-----------|--------------------------------------|----------------------------------------|
| 定位方式      | `MASTER_LOG_FILE` + `MASTER_LOG_POS` | GTID 集合差集                              |
| 切换后从库指向新主 | 需人工计算 file/pos                       | `SOURCE_AUTO_POSITION = 1` 自动请求缺失 GTID |
| 防止重复执行    | 误设位点可能重复或跳过                          | `gtid_executed` 已执行 GTID 自动跳过          |
| 级联复制      | 各跳位点无直接映射                            | GTID 全局唯一，只需缺失集合                       |
| 多源复制      | 各源 file 名可能相同，位点混乱                   | 各源 UUID 独立，从库合并 `gtid_executed`        |
| 限制        | 相对较少                                 | 部分语句受限（见第五节）                           |

File/Position 在 `RESET MASTER` 或实例重建后 file 名重置、位点归零；GTID 只要 `source_uuid` 不变，事务 ID 持续递增，跨文件与跨切换语义一致。

### 2. GTID 的核心优势

**自动定位复制位点**：从库配置 `SOURCE_AUTO_POSITION = 1`（旧版 `MASTER_AUTO_POSITION = 1`）后，向主库发送`Retrieved_Gtid_Set` 与 `Executed_Gtid_Set`，主库 Dump 线程计算缺失 GTID 区间并从最小缺失处推送。网络中断重连、Failover 后改指向新主、从备份恢复后声明 `gtid_purged` 再 `START REPLICA`，均可自动续传，无需 DBA 手工查 `SHOW MASTER STATUS` 填位点。

**简化主从切换**：File/Position 切换需指定 `MASTER_LOG_FILE` 与 `MASTER_LOG_POS`；GTID 模式仅需：

```sql
STOP REPLICA;
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'new-master',
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = '***',
  SOURCE_AUTO_POSITION = 1;
START REPLICA;
```

Orchestrator、MHA 等 HA 工具在 GTID 下可更可靠地自动变更拓扑。

**防止重复执行**：从库在 `mysql.gtid_executed` 与 `Executed_Gtid_Set` 中持久化已重放 GTID；SQL 线程/worker 发现 GTID 已存在则跳过，避免重复应用。前提为复制链路正常、未手工篡改 `gtid_executed`——GTID 降低正常运维路径的错误率，不能替代规范操作。

**多源复制支持**：各主库 UUID 不同，GTID 天然全局唯一；从库 `gtid_executed` 为各源并集，分通道调度清晰（见第五节）。

### 3. GTID Set

GTID 以集合操作。GTID Set 字符串由若干 `uuid:interval` 组成，interval 为连续 transaction_id 范围：

```
3E11FA47-71CA-11E1-9E33-C80AA9429562:1-154,
a1b2c3d4-e5f6-7890-abcd-ef1234567890:1-500:502-510
```

**Executed_Gtid_Set**：本实例**已执行（已提交）**的 GTID 集合。主库等于 `@@GLOBAL.gtid_executed`；从库等于 SQL/worker 已重放完成的 GTID（含本实例曾作为主库产生的 GTID 及各复制源 GTID）。Failover 时，选 `Executed_Gtid_Set` 为其他从库超集的节点作为"数据最新"候选。

```sql
SELECT @@GLOBAL.gtid_executed;
SHOW MASTER STATUS;       -- 主库
SHOW REPLICA STATUS\G     -- 从库 Executed_Gtid_Set
```

**Retrieved_Gtid_Set**：仅出现在从库，表示 IO 线程已从当前主库接收并写入 Relay Log、但 SQL 尚未全部重放的 GTID 集合。它与 `Executed_Gtid_Set` 不能直接作整体包含关系比较：后者还可能包含本地写入、其他复制源或此前已执行的事务。应只针对当前源的 GTID 子集比较两者。

- `Retrieved` 远大于 `Executed`：瓶颈在**重放**（SQL/worker）。
- `Retrieved` 与主库 `Executed` 差距大，而 `Executed ≈ Retrieved`：瓶颈在 **IO**（网络或 Relay Log 写入）。

**gtid_purged**：记录已提交、但已不在任何本机 Binlog 中的 GTID，是 `gtid_executed` 的子集；它通常随 `PURGE BINARY LOGS`、日志自动过期或恢复时设置而变化。主库 `gtid_executed` 包含当前 Binlog 与已清理日志中的执行历史。若从库需要的事务已被主库清理且无法从其他来源补齐，自动定位会失败并报“主库已清理所需 GTID”一类错误：

```
The slave is connecting using a GTID set that includes transactions
that have been purged from the master's binary log.
```

处理：从可用备份重建从库（推荐）、延长主库 Binlog 保留期，或仅在**已通过校验确认数据确实包含这些事务**时补录 GTID（高风险，不能把 `gtid_purged` 当作跳过数据的常规手段）。

**GTID 集合运算在运维中的用法**：比较两实例是否一致时，可借助 MySQL 内置函数或脚本对 GTID Set 做差集运算——若从库`Executed_Gtid_Set` 缺少主库某 UUID 的区间，说明从库尚未追平；Failover 选举时，选"包含最多 GTID"的从库即可最大化数据完整性。Orchestrator 等工具内部即基于此类集合比较实现自动拓扑修复。注意 GTID Set 字符串中逗号分隔不同 UUID 区间，冒号后为 `start-end` 或单个 ID，手工拼接 `gtid_purged` 时须严格遵循格式，否则报错 `Malformed GTID set specification`。

### 4. GTID 模式的配置

**新集群完整配置**

主从 `my.cnf`：

```ini
[mysqld]
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = ON
binlog_format = ROW
```

启用顺序：① `enforce_gtid_consistency = ON` → ② `gtid_mode = OFF_PERMISSIVE` → ③ `ON_PERMISSIVE` → ④ `ON` → ⑤ 从库`SOURCE_AUTO_POSITION = 1` 并 `START REPLICA`。

主库创建复制用户：

```sql
CREATE USER 'repl'@'%' IDENTIFIED BY 'strong_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
```

从库建立复制：

```sql
-- 仅对新建或已清空执行历史的实例：先按备份元数据设置 gtid_purged
-- SET GLOBAL gtid_purged = 'uuid:1-N';
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'master-host',
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = 'strong_password',
  SOURCE_AUTO_POSITION = 1;
START REPLICA;
```

验证：`Replica_IO_Running = Yes`，`Replica_SQL_Running = Yes`，`Executed_Gtid_Set` 持续追平主库。

**从传统模式在线切换到 GTID**

MySQL 5.7.6+ 可利用过渡态在线迁移，无需停库重建；每一步必须在**全拓扑**完成后再进入下一步：

```sql
-- 阶段一：各节点
SET GLOBAL enforce_gtid_consistency = WARN;  -- 观察后改 ON

-- 阶段二：各节点
SET GLOBAL gtid_mode = OFF_PERMISSIVE;

-- 阶段三：各节点
SET GLOBAL gtid_mode = ON_PERMISSIVE;

-- 阶段四：确认匿名事务已清空并已复制到所有节点后，再在各节点执行
SHOW STATUS LIKE 'ONGOING_ANONYMOUS_TRANSACTION_COUNT';  -- 观察到 0
SET GLOBAL gtid_mode = ON;

-- 阶段五：各从库
STOP REPLICA;
CHANGE REPLICATION SOURCE TO SOURCE_AUTO_POSITION = 1;
START REPLICA;
```

`ON_PERMISSIVE` 期间新事务分配 GTID，仍兼容匿名事务。观察到匿名事务计数为 0 后，还必须确认此前匿名事务已复制到所有节点；此后旧的含匿名事务 Binlog 不能再用于恢复。迁移前应先以 `enforce_gtid_consistency = WARN` 观察业务告警、修正不兼容语句，再改为 `ON`；级联拓扑须全节点同步切换。完成后持久化 `my.cnf` 并重启验证。

### 5. GTID 的限制与注意事项

**不支持的语句**（`enforce_gtid_consistency = ON` 时）：

| 语句/场景                                  | 原因                       |
|----------------------------------------|--------------------------|
| `CREATE TABLE ... SELECT`（8.0.21 前） | ROW 格式下曾拆为两个事务，无法保持一条语句对应一个 GTID |
| STATEMENT 格式下事务/存储程序中的临时表 DDL | 临时表 Binlog 语义特殊            |
| 非事务引擎与 InnoDB 混在同一事务                   | 跨引擎无法统一 GTID 语义          |

在 MySQL 8.0.21+，支持原子 DDL 的引擎可使用 `CREATE TABLE ... SELECT`；从 8.0.13 起，ROW/MIXED 格式下临时表 DDL 可在事务、存储程序、函数和触发器中使用。面向 5.7 或混合版本拓扑时，仍宜拆分 `CREATE TABLE` 与 `INSERT INTO ... SELECT`，并避免在同一事务中混用事务型和非事务型表。

**enforce_gtid_consistency**：`OFF` 不检查；`WARN` 允许但记日志；`ON` 拒绝并报 `ER_GTID_UNSAFE_*`。生产 `gtid_mode = ON` 时必须`ON`。

**与 binlog_format 的关系**：GTID 不强制格式，但生产强烈推荐 **ROW**——重放语义确定、与 GTID 跳过及 WRITESET 并行配合最好；STATEMENT 下非确定性函数可能导致主从不一致；WRITESET 依赖 Row 格式提取主键/唯一键写集。8.0 默认 ROW；迁移时可配合`binlog_row_image` 与 `binlog_transaction_compression`（8.0.20+）控制体积。

## 二、并行复制

### 1. 为什么需要并行复制

经典复制中，从库仅一个 SQL 线程串行重放 Relay Log。主库 InnoDB 多线程并发写入、组提交批量刷盘，吞吐随 CPU/SSD 扩展；从库所有变更排队进单线程，**重放能力存在硬上限**。

典型症状：`Seconds_Behind_Master` 持续增大；`Read_Master_Log_Pos` 接近主库但 `Exec_Master_Log_Pos` 明显落后；从库仅单核打满；主库写入 QPS 上升时从库延迟同比恶化。

```
主库（组提交，多核并行）          从库（MTS 关闭，单 SQL 线程）
Session 1~4 ──► 提交  ──Binlog──►  T1 → T2 → T3 → T4 …（串行）
   高吞吐                              低吞吐，Relay Log 积压
```

当主库 TPS 超过从库单线程重放 TPS，延迟不可收敛。MTS 通过多个 **worker** 并行重放**无冲突**事务，将从库能力提升至接近主库（受冲突率、硬件、worker 数限制）。

### 2. MySQL 5.6：基于 Schema 的并行

5.6 引入 MTS，粒度为 **Database**：

```sql
SET GLOBAL slave_parallel_workers = 4;
```

不同 database 的事务可并行；同一库内仍串行。多租户分库（`db_order`、`db_user`）有效；**单库 OLTP**（最常见）几乎无效。5.6 普及有限，为后续策略奠基。

### 3. MySQL 5.7：基于组提交（LOGICAL_CLOCK）

5.7 引入 `slave_parallel_type = LOGICAL_CLOCK`（8.0 为 `replica_parallel_type`），利用主库**组提交**的逻辑时钟实现同库内并行。

主库提交时，Binlog 携带 **last_committed** 与 **sequence_number**。对事务 B，`last_committed` 表示 B 提交前必须已提交的最大逻辑时钟；从库只有在不晚于该时钟的事务完成后才可提交 B。具有相同 `last_committed`、且彼此不在该依赖边界内的事务可以由不同 worker 并行执行。

```sql
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 8;
SET GLOBAL slave_preserve_commit_order = ON;  -- 从库 commit 顺序与主库一致
```

| 参数                                 | 含义                                  |
|------------------------------------|-------------------------------------|
| `LOGICAL_CLOCK`                    | 按 last_committed/sequence_number 调度 |
| `slave_parallel_workers`           | worker 数，建议 `min(32, 2×CPU核数)` 起步压测 |
| `slave_preserve_commit_order = ON` | 并行执行但 commit 顺序一致，避免从库读乱序           |

**局限**：并行度受主库组提交批次大小约束——主库低并发、每批仅 1～2 事务时，从库并行收益有限。这是 LOGICAL_CLOCK 依赖主库提交模式的根本约束。

可通过 `mysqlbinlog -vv` 查看事件中 `last_committed` 与 `sequence_number` 字段，评估主库组提交批次大小。若长期每批仅 1 事务，应优先考虑升级至 8.0 WRITESET，而非单纯增大 `slave_parallel_workers`。

**5.7 时代参数名对照**（升级 8.0 时需注意）：

| 5.7                           | 8.0                             |
|-------------------------------|---------------------------------|
| `slave_parallel_workers`      | `replica_parallel_workers`      |
| `slave_parallel_type`         | `replica_parallel_type`         |
| `slave_preserve_commit_order` | `replica_preserve_commit_order` |
| `SHOW SLAVE STATUS`           | `SHOW REPLICA STATUS`           |
| `CHANGE MASTER TO`            | `CHANGE REPLICATION SOURCE TO`  |

8.0 仍识别旧参数名但会记 deprecation 警告，新部署应统一使用 `replica_*` 前缀。

### 4. MySQL 8.0：基于写集（WRITESET）

8.0 引入 `binlog_transaction_dependency_tracking = WRITESET`，基于**事务修改的行集合**判断冲突，不再依赖主库组提交窗口。

```sql
-- 主库
SET GLOBAL binlog_transaction_dependency_tracking = WRITESET;

-- 从库
SET GLOBAL replica_parallel_type = LOGICAL_CLOCK;
SET GLOBAL replica_parallel_workers = 16;
SET GLOBAL replica_preserve_commit_order = ON;
```

| 值                | 说明                |
|------------------|-------------------|
| COMMIT_ORDER     | 仅组提交顺序（5.7 默认）    |
| WRITESET         | 写集无冲突则并行          |
| WRITESET_SESSION | WRITESET + 同连接内串行 |

主库根据事务修改行的主键和唯一键计算写集，并把依赖信息写入 Binlog；从库 Coordinator 再按此依赖调度。写集无冲突时可分配给不同 worker，有冲突时需保持依赖顺序。即使主库每批组提交仅 1 个事务，并发修改不同行时从库仍可能获得较高并行度。

```
T1: UPDATE orders SET status=1 WHERE id=100;  -- 与 T2 可并行
T2: UPDATE orders SET status=2 WHERE id=200;
T3: UPDATE orders SET status=3 WHERE id=100;  -- 与 T1 冲突，须等待 T1
```

**transaction_write_set_extraction**：启用 WRITESET 前应确认该变量使用 `XXHASH64`，并在变更前阅读目标小版本的默认值与兼容性说明。写集依赖提取依赖表上的主键或唯一键；缺少可用键的事务无法获得细粒度写集依赖，通常会显著限制并行度。热点行（库存扣减、计数器）写集高度重叠，需业务拆热点或接受延迟。

### 5. 并行复制的配置最佳实践

**MySQL 8.0 生产配置示例（主从参数分开）：**

```ini
[mysqld]
replica_parallel_workers = 16
replica_parallel_type = LOGICAL_CLOCK
replica_preserve_commit_order = ON
binlog_format = ROW
innodb_buffer_pool_size = 物理内存 60%~70%
read_only = ON

# 仅主库：生成 WRITESET 依赖信息
binlog_transaction_dependency_tracking = WRITESET
# 仅主库：8.0.20+，可选，跨机房时评估 CPU 与网络后启用
binlog_transaction_compression = ON
```

**worker 数量选择**：基线记录延迟与 `replication_applier_status_by_worker`；将 workers 设为 4/8/16/32 分别压测主库写入峰值；观察延迟是否收敛、CPU 各核利用、锁等待；取"延迟可收敛且未过载"的最小值。worker 已足但延迟仍高，瓶颈可能在 IO、大事务或 Write Set 冲突。

### 6. 并行复制的监控

```sql
SELECT WORKER_ID, SERVICE_STATE, LAST_ERROR_NUMBER,
       LAST_APPLIED_TRANSACTION, APPLYING_TRANSACTION
FROM performance_schema.replication_applier_status_by_worker;

SELECT * FROM performance_schema.replication_applier_status_by_coordinator\G
```

关注：各 worker `SERVICE_STATE` 是否为 `ON`；比较 `LAST_APPLIED_TRANSACTION` 与 `APPLYING_TRANSACTION` 的推进情况和 worker 间积压是否均衡；`LAST_ERROR_NUMBER != 0` 为 worker 级错误。`LAST_APPLIED_TRANSACTION` 是 GTID，不是时间戳。

| 现象                | 判断                   |
|-------------------|----------------------|
| 所有 worker 忙碌，延迟仍增 | 主库写入超从库总重放能力         |
| 仅 1～2 worker 忙碌   | Write Set 冲突高或组提交批次小 |
| worker 均衡，延迟趋近 0  | 并行度足够                |
| IO 位点落后大，SQL 追平   | 网络/Relay Log 瓶颈      |

配合 `SHOW REPLICA STATUS` 的 `Relay_Log_Space`、`Exec_Master_Log_Pos` 与主库 `SHOW MASTER STATUS` 量化积压。

**Coordinator 与 Worker 的分工**：MTS 开启后，原 SQL 线程角色拆为 **Coordinator**（读取 Relay Log、按依赖关系分发事务给 worker）与多个 **Worker**（各自重放分配的事务）。Coordinator 维护依赖图：WRITESET 模式下，若新事务与某 worker 正在执行的事务 Write Set 冲突，则须等待该 worker 完成后再分发；无冲突则分配给空闲 worker。因此 worker 均衡度直接反映业务写冲突率——电商大促期间若大量更新同一 SKU 库存，会观察到个别 worker 长期忙碌，此时应从业务层拆热点而非无限加 worker。

## 三、复制延迟

### 1. 如何度量延迟

**Seconds_Behind_Master**：从库 SQL 重放事件的时间戳与从库当前时间的估算差值（秒）。`0` 通常表示 SQL 已追上已接收的事件；`NULL` 常见于复制线程未正常运行或尚未收到事件，必须结合 `Replica_IO_Running`、`Replica_SQL_Running` 与错误字段判断。

**局限性**（估算值，不可全信）：

| 场景           | 表现                      |
|--------------|-------------------------|
| 大事务重放中       | 可能显示 0，实际仍有大量 Relay Log |
| 主从时钟不同步      | 失真，可能被显示为 0 而掩盖差异      |
| SOURCE_DELAY | 含人为延迟                   |
| MTS 多 worker | 不能完全代表各 worker 进度       |
| IO 线程落后      | 可能未反映网络延迟               |

不能仅凭 `Seconds_Behind_Master = 0` 断定完全一致；还要确认 IO 线程已追上主库，并用当前源的 GTID 差集或可比的日志位点判断。

**pt-heartbeat（更精确）**：主库每秒更新心跳表一行；从库复制重放；监控比较主从 `updated_at` 差值，得真实延迟（不受大事务时间戳影响）。

```bash
# 主库：写入心跳
pt-heartbeat --update --database=percona --create-table --host=master-host ...

# 从库：监控心跳
pt-heartbeat --monitor --database=percona --host=slave-host ...
# 示例输出：0.05s、2.30s
```

秒级以下精度，适合 SLA 告警。心跳表用 InnoDB、主键单行更新，避免锁争用。

### 2. 延迟的常见原因

| 原因        | 说明                                                |
|-----------|---------------------------------------------------|
| 单线程/并行度不足 | SQL 串行或 WRITESET 冲突高，见第二节                         |
| 从库硬件弱     | HDD、小 Buffer Pool、少 CPU，重放 Row 事件慢                |
| 大事务       | `ALTER TABLE`、千万行 `INSERT ... SELECT`，长时间占 worker |
| 主库写入超从库能力 | 促销/导入尖峰，延迟积分式增长                                   |
| 从库额外负载    | 报表、备份、慢查询争抢 IO/锁，阻塞 DDL 重放                        |
| 网络带宽不足    | Row Binlog 体积大，IO 线程落后（`Read_Master_Log_Pos` 小）   |
| 缺主键       | Row 定位慢，WRITESET 无法并行                             |

报表/备份应迁离核心从库；跨机房可开 `binlog_transaction_compression`；**所有复制表必须有主键**。

### 3. 排查步骤

**第一步：确认延迟是否增长**

1 分钟间隔采样 10～30 分钟，记录 `Seconds_Behind_Master`、`Retrieved/Executed_Gtid_Set`、主库 `Position`。单调递增则不可自愈；pt-heartbeat 曲线更直观。

**第二步：定位 IO 还是 SQL 瓶颈**

| 对比项                                        | IO 瓶颈 | SQL 瓶颈 |
|--------------------------------------------|-------|--------|
| 主库 Position vs Read_Master_Log_Pos         | 差距大   | 接近     |
| Read_Master_Log_Pos vs Exec_Master_Log_Pos | 接近    | 差距大    |

IO 瓶颈查带宽、Relay Log 磁盘、MTU；SQL 瓶颈查 MTS、worker、大事务、从库慢查询与锁。

**第三步：检查大事务**

```sql
-- 主库活跃长事务
SELECT * FROM performance_schema.events_transactions_current
WHERE STATE = 'ACTIVE' ORDER BY TIMER_WAIT DESC;
SHOW BINARY LOGS;  -- 文件大小突增
```

```bash
mysqlbinlog --base64-output=DECODE-ROWS -v relay-log.N | less
```

元凶常为 `ALTER TABLE`、无 LIMIT 批量 DML、迁移脚本。

**第四步：检查从库负载**

```sql
SHOW FULL PROCESSLIST;
SELECT * FROM performance_schema.data_lock_waits;
SHOW ENGINE INNODB STATUS\G
SELECT * FROM performance_schema.replication_applier_status_by_worker\G
```

结合 `iostat`、`top` 看 CPU/磁盘/网络。

### 4. 延迟的解决方案

| 方案                | 适用场景                             |
|-------------------|----------------------------------|
| 启用 WRITESET + MTS | SQL 瓶颈，冲突可接受                     |
| 升级从库硬件            | 配置明显低于主库                         |
| 拆分大事务             | 应用小批次 commit；DDL 用 gh-ost/pt-osc |
| 复制过滤              | `replicate_wild_do_table` 减少重放   |
| 增加从库分担读           | 报表/备份迁离核心从库                      |
| Binlog 压缩         | 跨机房网络瓶颈                          |
| 全表主键              | Row 重放与 WRITESET 基线              |

原则：**先度量 → 定位 IO/SQL/网络/负载 → 针对性优化**，避免未验证瓶颈就盲目加 worker。

**延迟从库（SOURCE_DELAY）与性能延迟的区别**：`CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 3600` 使从库故意晚 1 小时重放，用于误操作恢复窗口（如误删后立刻停止延迟从库，保留尚未重放的事件）。这与性能导致的复制延迟性质不同——`Seconds_Behind_Master` 在 SOURCE_DELAY 场景下会包含人为延迟，排查性能问题时应排除配置了 DELAY 的从库，或从当前源的 GTID 差集判断。

**复制过滤对延迟的副作用**：从库配置 `replicate_ignore_table` 或 `replicate_do_db` 后，IO 线程仍拉取完整 Binlog（除非 8.0 某些过滤优化），SQL 线程跳过不匹配事件。过滤本身不减少 IO 压力，但减少 SQL 重放量。若仅需部分表，可考虑在从库侧用 Parallel Applier 配合过滤，或改用多源复制分通道拉取，避免从库重放全库变更。

## 四、主从切换

### 1. 计划内切换（Switchover）

用于主库升级、机房迁移等，目标 **RPO = 0**。流程：**停写 → 等同步 → 切换角色**。

```sql
-- 1. 切换前：从库 S1 确认追平
SHOW REPLICA STATUS\G;  -- Seconds_Behind_Master = 0，Executed_Gtid_Set 与主库 M 一致

-- 2. 主库 M 停写
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
-- 确认 innodb_trx、PROCESSLIST 无活跃写

-- 3. S1 追平（可用 WAIT 函数）
SELECT WAIT_FOR_EXECUTED_GTID_SET('M的Executed_Gtid_Set', 300);

-- 4. 提升 S1 为新主
STOP REPLICA;
RESET REPLICA ALL;
SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;

-- 5. 其他从库 S2 指向 S1
STOP REPLICA;
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 's1-host', SOURCE_USER = 'repl', SOURCE_PASSWORD = '***',
  SOURCE_AUTO_POSITION = 1;
START REPLICA;

-- 6. 旧主 M 恢复后降级为从库（新主已接管后）
-- 不要执行 RESET MASTER：它会删除 Binlog 与 GTID 执行历史
STOP REPLICA;
RESET REPLICA ALL;
CHANGE REPLICATION SOURCE TO ... SOURCE_AUTO_POSITION = 1;
START REPLICA;
```

**7. 应用层**：切换 VIP/DNS/连接串至 S1，验证读写。GTID 模式无需 file/pos，但**不可跳过 WAIT 追平**。

**Switchover 常见失误与规避**：

- 未等活跃事务结束即提升从库：可能丢失最后几笔写入。务必 `read_only` 后检查 `innodb_trx` 为空。
- 旧主恢复后直接启动为独立主库：造成双主。旧主必须完成 fencing，并在确认无分叉写入后以从库身份接入新主；不可把 `RESET MASTER` 当作常规切换步骤。
- 跳过 GTID 校验：即使 `Seconds_Behind_Master = 0`，仍应用 `WAIT_FOR_EXECUTED_GTID_SET` 或对比 `Executed_Gtid_Set` 字符串。
- 半同步从库未同步完即切换：异步复制下 RPO 可能 > 0；若要求 RPO = 0，切换前须确认半同步 ACK 从库已包含全部已提交 GTID。

### 2. 非计划切换（Failover）

主库崩溃/隔离时，最短时间恢复写入，异步下可能 **RPO > 0**，半同步 **RPO ≈ 0**。

**紧急步骤**：

1. **Fencing** 旧主，防脑裂（见第四节第 4 点）。
2. **选举**数据最新从库（非延迟最大者）。
3. 候选库：`STOP REPLICA; RESET REPLICA ALL; SET GLOBAL read_only = OFF;`
4. 其余从库 `CHANGE REPLICATION SOURCE TO ... SOURCE_AUTO_POSITION = 1`
5. 应用切换写入。
6. 旧主恢复后**必须以从库身份**重新接入，禁止直接恢复为主。

**如何选择最新从库**

- **GTID（推荐）**：比较各从库 `Executed_Gtid_Set`，选为其他从库超集的节点；可用 Orchestrator 自动选举。
- **File/Position**：比较 `Exec_Master_Log_Pos` 与 `Relay_Master_Log_File`（主库 Binlog 仍可读时）。
- **半同步**：`AFTER_SYNC` 下优先选收到 ACK 的从库，RPO 更接近 0。

**GTID 自动定位**：Failover 后 `SOURCE_AUTO_POSITION = 1` 即可。若新主缺少某 GTID（旧主未同步即崩溃），该 GTID 对应数据**永久丢失**，需业务对账。

### 3. 数据一致性校验

**pt-table-checksum**：主库分 chunk 计算 checksum，经复制到从库对比。

```bash
pt-table-checksum --host=master-host --databases=app_db \
  --replicate=percona.checksums --recursion-method=hosts --no-check-binlog-format
```

大表对主库有负载，低峰执行；表须有主键；checksum 表须被复制。

**GTID Set 对比**：

```sql
SELECT @@GLOBAL.gtid_executed;           -- 新主
SHOW REPLICA STATUS\G                  -- 从库 Executed 应为新主子集（追平后相等）
```

GTID 一致是**必要非充分**——手工改从库、复制过滤错误仍可能导致行级不一致，关键表需 checksum 或业务对账。

### 4. 防止脑裂

**脑裂**：旧主仍在线写入，新主也被提升，从库只跟新主，旧主写入分叉。

**Fencing 手段**：

| 手段      | 说明                          |
|---------|-----------------------------|
| STONITH | 强制关机/断电旧主                   |
| 网络隔离    | 撤销 VIP、禁止应用访问旧主 IP          |
| 存储隔离    | 撤销共享存储写权限                   |
| 云 API   | stop instance、detach volume |

Orchestrator/MHA 会尝试 SSH 将旧主设为 `read_only` 或停止 mysqld；网络分区时仍须依赖外部 fencing。**旧主恢复后**应先排查并处理可能的分叉事务，再以从库身份接入，禁止直接恢复为主。

### 5. 读一致性保证

**半同步与 RPO**：`rpl_semi_sync_source` + `AFTER_SYNC` 保证 commit 前至少一从库收到 Binlog（Relay Log），Failover 到该从库**RPO ≈ 0**。但半同步不保证 SQL 已重放，读从库仍可能延迟。

**从库读延迟处理**：

1. **WAIT_FOR_EXECUTED_GTID_SET**（5.7+）：写入返回 GTID 后，读从库前等待该 GTID 已重放。

```sql
SELECT WAIT_FOR_EXECUTED_GTID_SET('uuid:155', 10);  -- 0 成功，1 超时
```

需驱动/中间件传递 GTID；高并发下 WAIT 增加从库负载。

2. **强制读主**：支付结果、库存确认等敏感读走主库，实现简单。

3. **会话粘滞**：写后 N 秒内读主库，N 取复制延迟 P99（2～5 秒）。

4. **中间件 lag 检测**：ProxySQL `max_replication_lag`，超限路由主库或剔除从库。

```sql
UPDATE mysql_servers SET max_replication_lag = 5 WHERE hostgroup_id = 20;
LOAD MYSQL SERVERS TO RUNTIME;
```

建议：金融核心 **半同步 + 关键读主**；一般业务 **WRITESET + 会话粘滞**；极敏感且可接受复杂度时用 **GTID WAIT**。

**读写分离中间件的延迟感知**：ProxySQL 可基于 `Seconds_Behind_Master` 的阈值摘除延迟从库。这类机制依赖复制状态的准确性——MTS 场景下若该指标失真，中间件仍可能将读路由到实际落后的从库。部署 pt-heartbeat 并将 lag 写入可供健康检查读取的自定义表，是更可靠的补充方案；具体接入方式应以所用中间件版本的官方文档为准。

## 五、多源复制

### 1. 什么是多源复制

**多源复制（Multi-Source Replication）**：一个从库同时从**多个主库**拉取 Binlog，各通道独立 IO/SQL 线程，在从库本地重放汇聚。

```
Master A (db_order) ──channel order──► ┐
                                       ├──► Replica R
Master B (db_user)  ──channel user───► ┘
```

与级联（A→B→C）不同，这是多主到单从；与 Group Replication 不同，各主库仍独立写入，从库通常为只读汇聚节点。

### 2. 配置方法

MySQL 5.7+ 每源一个 **channel**：

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'master-a', SOURCE_USER = 'repl', SOURCE_PASSWORD = '***',
  SOURCE_AUTO_POSITION = 1
FOR CHANNEL 'order';
START REPLICA FOR CHANNEL 'order';

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'master-b', SOURCE_USER = 'repl', SOURCE_PASSWORD = '***',
  SOURCE_AUTO_POSITION = 1
FOR CHANNEL 'user';
START REPLICA FOR CHANNEL 'user';
```

```sql
SHOW REPLICA STATUS FOR CHANNEL 'order'\G
STOP REPLICA FOR CHANNEL 'order';
```

`performance_schema.replication_connection_status` / `replication_applier_status` 按 `CHANNEL_NAME` 过滤，分通道监控延迟。

### 3. 适用场景

**数据汇聚**：订单、用户、商品等各自主库，汇聚到分析从库做跨域 JOIN、报表、数仓 ODS，避免直连多个生产主库。

**多机房同步**：地域分主，汇聚到中心从库做全局查询或灾备。设计阶段须规划库表命名空间。

### 4. 注意事项

**表名冲突**：各主库复制到从库的表名空间不可冲突——通常用不同 database（`order`、`user`）或 `replicate_wild_do_table`过滤。两主库同表 `app.users` 且结构不同会导致数据混乱。

**GTID 冲突**：各主库 UUID 不同，GTID 全局唯一，不存在"两主产生相同 GTID"。需注意：

- 从库 `gtid_executed` 为各通道并集，随源数量增长。
- 各通道延迟独立，汇聚查询须感知通道间延迟差。
- Failover 针对**单个源**，不能将多源从库整体提升为多主。
- 备份恢复须正确设置各源 `gtid_purged`。

```sql
CHANGE REPLICATION FILTER REPLICATE_DO_DB = (order_db) FOR CHANNEL 'order';
```

各通道可独立复制过滤，减少不必要重放。

## 总结

GTID 将复制定位从 File/Position 升级为全局事务 ID 集合，`SOURCE_AUTO_POSITION` 实现自动对齐，简化 Switchover/Failover、级联与多源运维；`gtid_executed`、`gtid_purged`、`Retrieved_Gtid_Set` 构成完整复制状态语义。启用须配合`enforce_gtid_consistency = ON` 与 `binlog_format = ROW`，规避 `CREATE TABLE ... SELECT` 等语句；在线迁移经`OFF_PERMISSIVE → ON_PERMISSIVE → ON` 平滑过渡。

并行复制历经 5.6 Schema 级、5.7 LOGICAL_CLOCK、8.0 WRITESET 三阶段。生产推荐`binlog_transaction_dependency_tracking = WRITESET`、`replica_parallel_workers` 压测调优、`replica_preserve_commit_order = ON`，并以 `replication_applier_status_by_worker` 评估并行度。热点行、无主键或唯一键、大事务是并行与延迟的关键制约。

复制延迟排查应超越 `Seconds_Behind_Master`，结合 pt-heartbeat、IO/SQL 位点差、GTID 趋势定位瓶颈；方案从 MTS、硬件、事务拆分、读负载隔离到 Binlog 压缩需对症下药。主从切换区分计划内（停写、追平、提升）与非计划（选举最新 GTID 从库、fencing 防脑裂）；切换后以 pt-table-checksum 与 GTID 对比校验；读一致性靠半同步缩小 RPO，靠 `WAIT_FOR_EXECUTED_GTID_SET`、会话粘滞或读主弥补从库滞后。

多源复制以 FOR CHANNEL 实现多主到单从汇聚，须在设计阶段解决库表命名隔离。与 Orchestrator、MHA、ProxySQL 配合，方能在生产构建可观测、可切换、可收敛延迟的高可用复制架构。更基础的复制线程模型与半同步原理，请参阅《MySQL 主从复制原理与实践》与 Binlog 专题。
