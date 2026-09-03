---
title: MySQL 主从复制架构：异步复制与半同步复制
summary: 系统梳理 MySQL 主从复制的三线程架构、异步复制的工作流程与数据延迟风险、半同步复制的一致性保障机制，建立复制原理的基础认知。
created: 2026-07-02
updated: 2026-07-16
tags: MySQL, 主从复制, Binlog, 高可用
---

# MySQL 主从复制架构：异步复制与半同步复制

MySQL 主从复制（Replication）是构建读写分离、高可用与灾备体系的基础能力。其核心思路是：主库（Source / Master）将数据变更写入 Binlog，从库（Replica / Slave）拉取 Binlog 并在本地重放，使从库数据逐步追平主库。

复制机制横跨 Server 层的 Binlog、网络传输、Relay Log 与 SQL 重放线程，并与半同步等特性叠加，形成从"单主单从"到"多层级联"的复杂拓扑。理解复制，需要同时把握线程模型、一致性语义与 Binlog 格式选择三个维度。本文聚焦复制架构基础、异步复制与半同步复制；GTID、并行复制、延迟排查与主从切换将在后续专题中展开。

## 一、为什么需要主从复制

主从复制是将主库逻辑变更传播到一个或多个从库的基础设施，而非 MySQL 内置的高可用方案。围绕这一能力，生产环境中衍生出读写分离、故障转移、备份隔离、异地容灾等架构模式。先明确"复制能做什么、不能做什么"，有助于正确评估 RPO/RTO 与一致性风险。

### 1. 读写分离

OLTP 系统中读请求通常远多于写请求。将全部读写压力集中在单实例上，容易在 CPU、Buffer Pool 争用和磁盘 IO 上形成瓶颈。主从复制允许将只读查询路由到从库，主库专注处理写入，在不改动应用分片逻辑的前提下扩展读吞吐。

实现方式包括：应用层路由（主写从读）、中间件（ProxySQL、MySQL Router）、云厂商只读实例等。无论哪种方式，底层都依赖复制链路保证从库与主库最终一致。需要强调的是：异步复制下从库存在延迟窗口，读从库可能读到旧数据。对一致性敏感的业务（支付结果、库存确认）必须走主库或配合会话粘滞，不能假设"从库与主库实时等价"。

### 2. 高可用与故障转移

单点主库故障会导致写入路径中断。通过主从复制，可在主库不可用时将某个从库提升为新主库（Failover），配合 Orchestrator、MHA、Keepalived + VIP 等实现自动或半自动切换。

**复制本身并不等同于高可用。** 异步复制下，主库故障可能丢失尚未同步到从库的已提交事务；切换前必须评估 RPO，并选择半同步、组复制或 InnoDB Cluster 等更强一致性方案。半同步可在 commit 路径上缩小 RPO，但仍不保证从库 SQL 线程已重放——"提交成功"与"从库可读"之间仍可能存在差距。

### 3. 数据备份隔离

在主库执行全量备份（mysqldump、XtraBackup）或大型分析查询，会与在线业务争抢 IO 与锁。将从库专用于备份或离线报表，可将重负载从主库剥离。

从库备份需注意：备份是某一时刻的快照，复制存在延迟时内容与主库当前状态不等价；一致性要求高的备份应结合`SHOW REPLICA STATUS` 中的 `Seconds_Behind_Master` 与 Binlog 位点判断追平情况；从库备份不能替代 Binlog 归档——全量备份 + Binlog 增量才是完整 PITR 方案。

### 4. 异地容灾

将 Binlog 变更流复制到异地机房，是满足 RPO/RTO 与数据驻留合规的常见手段。跨地域复制面临网络延迟与带宽限制，复制延迟往往高于同城部署。

常见拓扑：一主一从（异地），RPO 取决于复制模式与网络质量；级联复制（主库 → 同城从库 → 异地从库），减轻主库跨地域推送压力，但级联会放大延迟。跨地域复制的带宽、延迟与`sync_binlog` 配置共同决定同步速度与故障可恢复点。

### 5. 复制不等于备份的区别

复制与备份目的不同，实践中常被混淆：

| 维度    | 主从复制                 | 备份                 |
|-------|----------------------|--------------------|
| 目的    | 近实时传播变更，扩展读能力或灾备     | 在特定时间点保存快照，用于恢复    |
| 数据形态  | 从库是在线可服务的 MySQL 实例   | 备份文件通常离线存储         |
| 误操作保护 | 不能阻止误删表、误 UPDATE 的传播 | 全量备份保留历史，可恢复到误操作之前 |
| 保留周期  | 从库随主库持续更新，不保留历史版本    | 按策略保留多天/多周         |
| 恢复粒度  | 无法"回到昨天 15:30"       | 结合 Binlog 可实现 PITR |

典型事故：DBA 在主库执行 `DROP TABLE`，若从库未延迟，误操作会迅速传播，复制无法提供回滚能力，只能依赖备份 + Binlog 做 PITR，或延迟从库（`SOURCE_DELAY`）预留的缓冲窗口。**复制解决数据扩散与读扩展，备份解决历史留存与恢复，二者互补不可替代。**

## 二、复制的基本架构

经典 MySQL 主从复制由主库 1 个线程、从库 2 个线程协作完成（并行复制下 SQL 线程扩展为多 worker，后续专题详述）。理解线程职责、Relay Log 定位与复制元数据存储，是排查复制异常与设计拓扑的前提。

### 1. 三个核心线程

#### Binlog Dump Thread（主库）

运行在主库，读取 Binlog 并通过复制连接发送给从库 IO Thread。

启动条件：主库 `log_bin = ON`；从库执行 `CHANGE REPLICATION SOURCE TO`（8.0.23+，旧版 `CHANGE MASTER TO`）并 `START REPLICA`后，主库为该从库创建 Dump 连接；复制账户需 `REPLICATION SLAVE`（8.0.23+ 亦称 `REPLICATION REPLICA`）权限。

工作流程：从库 IO Thread 连接并告知起始位点（File/Position 或 GTID）→ 主库 Dump Thread 从该位点读取 Binlog → 持续推送新事件。若主库 Binlog 被 purge 且位点已不存在，报错 `Got fatal error 1236`，需从备份重建。一主多从场景下，每个从库对应独立 Dump Thread。

#### IO Thread（从库）

连接主库、接收 Binlog 事件、写入 Relay Log。建立 TCP 连接（可配 SSL），顺序追加事件到 `relay-log.*`，更新 `Read_Master_Log_Pos`与 `Relay_Log_Pos` 并持久化元数据。IO Thread 只负责拉取落盘，不修改业务表。若 IO 停止而 SQL 仍在运行，从库继续消费已有 Relay Log 但不再接收新事件。

#### SQL Thread（从库）

读取 Relay Log、解析事件、在从库重放变更。按 `binlog_format` 重放：Statement 执行 SQL 文本，Row 应用行级 before/after 镜像；每重放一个事务更新 `Exec_Master_Log_Pos`。重放涉及 InnoDB 提交、索引维护、触发器，成本可能高于主库（额外索引、外键等）。IO 落后或 SQL 落后都会导致从库滞后，但原因与排查方向不同。

| 线程                 | 所在节点 | 职责              | 停止影响                       |
|--------------------|------|-----------------|----------------------------|
| Binlog Dump Thread | 主库   | 读 Binlog，网络发送   | 从库 IO 断开，无法拉取新事件           |
| IO Thread          | 从库   | 收事件，写 Relay Log | 不再接收；SQL 可继续消费已有 Relay Log |
| SQL Thread         | 从库   | 读 Relay Log，重放  | Relay Log 积压；IO 仍可写入       |

### 2. Relay Log 的作用

Relay Log 是 Binlog 在从库上的本地副本，格式几乎相同，由 IO Thread 写入、SQL Thread 读取。

存在 Relay Log 而非内存直传的原因：

1. **解耦速度与缓冲**：网络推送可能突发，SQL 重放可能因大事务、锁等待变慢，Relay Log 避免 IO 被 SQL 阻塞。
2. **崩溃恢复**：从库重启后 IO 从 `Read_Master_Log_Pos` 继续拉取，SQL 从 `Relay_Log_Pos` 继续重放，无需主库重发已持久化事件。
3. **独立控制**：可 `STOP REPLICA SQL_THREAD` 而保持 IO 运行，累积 Relay Log 用于延迟重放或比对；反之亦可仅停 IO。

| 维度      | Binlog                         | Relay Log                     |
|---------|--------------------------------|-------------------------------|
| 产生位置    | 主库 Server 层事务提交                | 从库 IO Thread 写入               |
| 内容      | 主库逻辑变更                         | 主库 Binlog 事件副本                |
| 参与 PITR | 是                              | 否（从库中间产物）                     |
| 清理      | `binlog_expire_logs_seconds` 等 | `relay_log_purge`，重放后自动 purge |

### 3. 复制的元数据

复制进程在从库持久化主库连接信息与位点，称复制元数据。MySQL 8.0 默认使用表存储；旧版本还可使用文件存储。

**基于文件**：`master.info`（主库连接 + IO 位点）、`relay-log.info`（SQL 位点 + Relay Log 文件名），位于 `datadir`，非事务性，崩溃可能半写损坏。

**基于表**：连接元数据存于 `mysql.slave_master_info`，应用元数据存于 `mysql.slave_relay_log_info`。表名为历史命名，MySQL 8.0 仍沿用；两表均为 InnoDB。

MySQL 8.0 默认使用 `TABLE`；`FILE` 存储及 `master_info_repository`、`relay_log_info_repository` 两个配置项均已弃用。

**推荐基于表**：InnoDB 事务与 redo 保证刷盘可靠性；物理/mysqldump 备份时位点一并纳入；可 SQL 查询接入监控。MySQL 8.0 无需显式配置；旧版 FILE 存储建议在维护窗口迁移至 TABLE。

### 4. 架构图（ASCII）

```
                    主库 (Source)
  ┌─────────────────────────────────────────────┐
  │  客户端写入 → InnoDB 修改 → 两阶段提交写 Binlog │
  │                              │              │
  │                    ┌─────────▼─────────┐    │
  │                    │ Binlog Dump Thread│    │
  │                    └─────────┬─────────┘    │
  └──────────────────────────────┼──────────────┘
                                 │ TCP（Binlog 事件流）
                                 ▼
                    从库 (Replica)
  ┌─────────────────────────────────────────────┐
  │  IO Thread → Relay Log → SQL Thread → 数据  │
  │  元数据：slave_master_info / slave_relay_log_info │
  └─────────────────────────────────────────────┘
                                 │
                                 ▼
                         客户端只读（读写分离）
```

半同步在同一架构上增加 ACK 路径：从库 IO Thread 写入 Relay Log 后向主库发送 ACK，主库在 commit 返回客户端前等待（等待点见第四章）。

`SHOW REPLICA STATUS\G`（旧版 `SHOW SLAVE STATUS\G`）是观察复制状态的核心命令，关键字段：

| 字段                                          | 含义                   |
|---------------------------------------------|----------------------|
| `Replica_IO_Running` / `Slave_IO_Running`   | IO 线程是否运行            |
| `Replica_SQL_Running` / `Slave_SQL_Running` | SQL 线程是否运行           |
| `Read_Master_Log_Pos`                       | IO 已读取的主库 Binlog 位点  |
| `Exec_Master_Log_Pos`                       | SQL 已重放的主库 Binlog 位点 |
| `Relay_Log_File` / `Relay_Log_Pos`          | 当前 Relay Log 文件与偏移   |
| `Seconds_Behind_Master`                     | 从库相对主库的估计延迟（秒）       |

IO 与 SQL 线程可独立停止：仅停 SQL 时 Relay Log 仍累积，适用于从库数据比对；仅停 IO 时 SQL 继续消费已有 Relay Log 直至追平。

## 三、异步复制的工作流程

MySQL 默认采用异步复制。理解主库提交路径、Dump Thread 推送、从库 IO/SQL 协作，以及"提交成功"与"从库已应用"之间的语义鸿沟，是评估数据安全风险的基础。

### 1. 主库侧

#### 事务提交 → 写 Binlog → 唤醒 Dump Thread

InnoDB 事务提交路径（与 Binlog 专题两阶段提交一致）：

1. 执行阶段：修改 Buffer Pool，生成 Redo Log。
2. Prepare：InnoDB Prepare，Redo 刷盘（取决于 `innodb_flush_log_at_trx_commit`）。
3. Binlog 写入：Server 层写入 Binlog Cache 并追加到文件；`sync_binlog = 1` 时 fsync。
4. Commit：InnoDB 收到 Binlog 落盘确认后写入 Commit 标记。
5. 返回客户端 commit 成功。

**整个过程中主库不等待任何从库。** Binlog 落盘后，Dump Thread 阻塞读取 Binlog（类似 tail -f），新事件 flush 后被唤醒并推送；各从库 Dump Thread 独立维护发送位点。Dump Thread 读 Binlog 逻辑事件，不经过 Redo Log。

#### Dump Thread 读取 Binlog 发送给从库

复制协议流程：IO Thread 发起 `COM_BINLOG_DUMP`（GTID 模式下为 `COM_BINLOG_DUMP_GTID`）携带起始位点 → 主库验证权限与位点 → Dump Thread 逐个读取事件封装发送 → 无新事件时等待（heartbeat 防超时）→ 从库重连后从上次位点继续。

大事务（尤其 Row 格式）产生巨型事件，受 `max_allowed_packet` 限制，主从两端应一致且足够大。

### 2. 从库侧

#### IO Thread 接收 Binlog 事件

接收循环：读事件包 → 校验 checksum → 写入 Relay Log → 更新 `Read_Master_Log_Pos` 并持久化元数据 → 半同步模式下写 Relay Log 后发 ACK（见第四章）。IO 落后表现为 `Read_Master_Log_Pos` 落后于主库 `SHOW MASTER STATUS` 的 `Position`。

#### 写入 Relay Log

事件顺序与主库 Binlog 严格一致，格式相同（event header + body）。`Read_Master_Log_Pos > Exec_Master_Log_Pos` 时 Relay Log 积压，说明 SQL 重放慢于 IO 拉取。

#### SQL Thread 读取 Relay Log 并回放

从 `relay_log_pos` 读事件 → 按类型分发（`QUERY_EVENT`、`TABLE_MAP` + Row 事件、`XID_EVENT` 等）→ 在从库执行等价变更 → 更新`Exec_Master_Log_Pos` → 遇 `Rotate_Event` 切换文件。单线程模式下从库并发度为 1，是高并发写入场景延迟的重要来源。

### 3. 异步复制的核心特点

**主库不等待从库确认，事务提交成功即向客户端返回。**

| 特点        | 说明                      |
|-----------|-------------------------|
| 提交与复制解耦   | "客户端收到成功"与"从库完成重放"无同步屏障 |
| 主库吞吐最高    | 写入延迟不受从库性能、网络 RTT 影响    |
| 从库故障不阻塞主库 | 从库宕机不影响主库新事务提交          |
| 最终一致      | 正常时从库最终追平，期间存在延迟窗口      |
| 一主多从独立    | 某从库落后不影响其他从库            |

异步复制优先保障主库可用性与写入性能，将一致性风险转移给故障切换与读从库场景。

### 4. 异步复制的数据安全风险

#### 主库崩溃时从库可能丢失事务

典型场景：主库事务 T1 已提交、Binlog 已写、客户端收到 ACK → Dump Thread 尚未发送或从库 IO 尚未写 Relay Log → 主库宕机 → 提升从库为新主 → **T1 永久丢失**。反之，若 T1 的 Binlog 至少到达一个从库 Relay Log（即使 SQL 未重放），提升该从库后可通过继续重放恢复 T1。

#### 丢失窗口的计算

**RPO（Recovery Point Objective）**：故障时可接受的最大数据丢失量。

```
RPO ≈ 主库最新已提交事务 − 至少一个从库 Relay Log 已持久化的事务
```

量化：对比主库 `SHOW MASTER STATUS` 位点与各从库 `Read_Master_Log_Pos`（或 GTID 集合），二者间 Binlog 区间即潜在丢失窗口。窗口大小取决于主库写入速率、Dump 推送速度、网络带宽、从库 IO 写 Relay Log 速度。示例：主库每秒 10MB Binlog，网络中断 30 秒后崩溃，理论最大丢失约 300MB 对应事务。

#### 适用场景

**适用**：读扩展为主、RPO 要求宽松、从库众多或跨地域、写入延迟敏感的非核心业务。

**不适用**：核心账务/支付（RPO ≈ 0，应半同步或组复制）；强依赖"写后读从库必可见"且无读主策略的业务。

读写分离下的读滞后：用户写主库后立即读从库可能查不到刚写入数据，根源是异步复制的延迟窗口。

#### 一次事务在复制链路上的完整路径

以 Row 格式下的一条 UPDATE 为例，串联三线程协作：

1. 主库执行 UPDATE，InnoDB 修改数据页，生成 Redo Log。
2. Server 层生成 Binlog 事件（`TABLE_MAP` + `UPDATE_ROWS`），两阶段提交写入 Binlog。
3. 客户端收到 commit 成功（**此时从库尚未收到任何变更**）。
4. Binlog Dump Thread 读取新事件，发送给从库 IO Thread。
5. IO Thread 写入 Relay Log，更新 `Read_Master_Log_Pos`。
6. SQL Thread 读取 Relay Log，在从库重放相同行变更。
7. 从库 InnoDB 提交，更新 `Exec_Master_Log_Pos`。

步骤 3 与 4–7 之间没有同步屏障——这正是异步复制的本质：**提交成功不等于从库已应用，甚至不等于从库已收到**。

## 四、半同步复制

半同步在异步基础上于 commit 路径插入"等待从库确认"，缩小 RPO。它是 MySQL 原生在性能与一致性间的折中，广泛应用于对数据丢失敏感但尚未引入 Group Replication 的系统。

### 1. 为什么需要半同步

#### 异步复制的 RPO > 0 问题

异步复制下 commit 成功不保证 Binlog 到达任何从库，主库故障时丢失窗口内已提交事务可能全部消失——订单已支付但切换后从库无记录、合规要求关键事务至少有一份异地副本等场景不可接受。

半同步目标：**commit 返回成功前，至少 N 个从库确认收到 Binlog（写入 Relay Log）**，将 RPO 从"任意大小"缩小到"至多丢失未被任何从库 ACK 的少量事务"（从库全部故障时 RPO 仍可能 > 0）。

### 2. 半同步的基本原理

由插件实现（8.0.26+：`rpl_semi_sync_source` / `rpl_semi_sync_replica`；8.0.25 及更早、5.7：`rpl_semi_sync_master` / `rpl_semi_sync_slave`）。

流程：主库到达等待点 → 暂停 commit 返回，等待 `rpl_semi_sync_source_wait_for_replica_count` 个从库 ACK → 从库 IO Thread 写 Relay Log 并 fsync 后发 ACK → 主库收到足够 ACK 后继续 commit → 超时则降级异步（见 4.5 节）。

**ACK 语义是"Binlog 已持久化到从库 Relay Log"，非"SQL Thread 已重放"。** 提交成功后从库数据可能仍不可读；若要求"提交成功则至少一个从库可读"，需监控 `Exec_Master_Log_Pos` 或使用组复制。

### 3. AFTER_SYNC vs AFTER_COMMIT

`rpl_semi_sync_source_wait_point` 决定等待 ACK 的时机，是最关键的一致性参数。

#### AFTER_SYNC（增强半同步 / Loss-Less）

**等待 ACK 在 Engine Commit 之前**（Binlog 已刷盘，InnoDB 尚未 Commit）。

```
主库：Prepare → 写 Binlog 刷盘 → 【等待 ACK】→ InnoDB Commit → 返回客户端
从库：IO Thread 收事件 → 写 Relay Log 刷盘 → 发送 ACK
```

特性：客户端收到成功时，至少一个从库 Relay Log 已有该事务 Binlog，主库随后崩溃提升从库不丢已 ACK 事务（Loss-Less）。5.7+ 默认且强烈推荐。

故障语义：等待 ACK 时客户端尚未收到成功；ACK 后才提交存储引擎，因此源库已提交的事务均已写入至少一个从库 Relay Log。源库异常退出并提升从库后，可实现无损切换；旧源库不能直接重启后重新加入拓扑，因为其 Binlog 可能包含恢复后会外化、但与新源库冲突的未提交事务，应丢弃或按新源库重建。

#### AFTER_COMMIT

**等待 ACK 在 Engine Commit 之后**，InnoDB 已 Commit，主库读者可见。

特性：Commit 后、ACK 前主库 session 已能读到新数据；若此时崩溃且 Binlog 未到达从库，事务丢失。**幽灵读**：其他连接读到尚未复制到从库的数据，切换后消失。5.6 默认，5.7 起不推荐。

| 维度    | AFTER_SYNC          | AFTER_COMMIT        |
|-------|---------------------|---------------------|
| 等待时机  | Binlog 刷盘后、Commit 前 | Commit 后            |
| 崩溃丢数据 | 已 ACK 事务不丢          | 已 Commit 未 ACK 可能丢失 |
| 生产推荐  | 是                   | 否                   |

**生产必须显式设置 `rpl_semi_sync_source_wait_point = AFTER_SYNC`**，升级后验证未被还原。

半同步与异步对比：

| 维度           | 异步复制         | 半同步（AFTER_SYNC）                |
|--------------|--------------|--------------------------------|
| 主库 commit 延迟 | 最低           | +网络 RTT + Relay Log fsync      |
| 数据丢失风险       | 主库故障可能丢已提交数据 | 至少一个从库持有 Relay Log 副本，RPO 接近 0 |
| 从库故障影响       | 无            | 超时后降级异步，或 commit 阻塞直至超时        |
| ACK 语义       | 无            | 从库 Relay Log 已持久化，非 SQL 已重放    |

### 4. 半同步的配置

#### 插件安装

MySQL 8.0.26+：

```sql
-- 主库
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
-- 从库
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';
```

8.0.25 及更早（包括 5.7）使用 `rpl_semi_sync_master` / `rpl_semi_sync_slave` 及对应的 `semisync_master.so` / `semisync_slave.so`。对于 8.0.26+，可将插件加载项写入 `my.cnf`：

```ini
plugin-load-add = rpl_semi_sync_source=semisync_source.so
plugin-load-add = rpl_semi_sync_replica=semisync_replica.so
```

#### rpl_semi_sync_master_enabled

8.0.26+ 为 `rpl_semi_sync_source_enabled`：

```sql
SET GLOBAL rpl_semi_sync_source_enabled = ON;
SET GLOBAL rpl_semi_sync_replica_enabled = ON;  -- 从库
```

主库开关，开启后新事务走半同步（需至少一个从库启用 replica 插件且连接正常）。关闭立即退化为异步。建议写入配置文件持久化。

#### rpl_semi_sync_master_timeout

8.0.26+ 为 `rpl_semi_sync_source_timeout`，单位毫秒，默认 10000：

```sql
SET GLOBAL rpl_semi_sync_source_timeout = 10000;
```

超时后：当前事务异步提交；主库关闭半同步，后续均为异步直至从库恢复；错误日志记录降级。同城 3000–10000 ms 常见；跨地域需权衡 RPO 与可用性；不应盲目设极大值以免从库长期故障阻塞 commit。

#### rpl_semi_sync_master_wait_point

8.0.26+ 为 `rpl_semi_sync_source_wait_point`：

```sql
SET GLOBAL rpl_semi_sync_source_wait_point = AFTER_SYNC;
```

建议部署阶段写入配置文件。其他参数：

```sql
SET GLOBAL rpl_semi_sync_source_wait_for_replica_count = 1;  -- 至少几个从库 ACK
```

验证：

```sql
SHOW STATUS LIKE 'Rpl_semi_sync_source%';
-- Rpl_semi_sync_source_status: ON/OFF
-- Rpl_semi_sync_source_yes_tx / no_tx: 半同步 vs 降级异步事务数
-- Rpl_semi_sync_source_clients: 半同步从库连接数
```

### 5. 超时降级

半同步哲学是"尽力保证一致性，但不牺牲主库无限期可用性"。

降级流程：等待 ACK 超时 → 当前事务异步提交 → `Rpl_semi_sync_source_status` 置 OFF，后续均为异步 → 错误日志记录。从库 IO 重连并成功 ACK 后自动恢复半同步，无需人工干预，但应监控 `no_tx` 是否在恢复后停止增长。

频繁降级说明从库性能不足、网络抖动或超时过短，半同步名存实亡；降级期间等同异步，故障切换仍可能丢数据。半同步非"绝对 RPO = 0"——主从全部故障、磁盘损坏等极端场景仍需备份与跨机房容灾。

### 6. 半同步的性能影响

#### 额外的网络 RTT

每笔事务在 AFTER_SYNC 等待点至少增加：

```
额外延迟 ≈ 网络 RTT（主→从→主）+ 从库 Relay Log fsync
```

同城 0.1–1 ms，跨 AZ 1–5 ms，跨地域可达数十至上百 ms。组提交可部分摊薄 RTT，但等待仍存在。

#### 优化方向

| 方向               | 说明                  |
|------------------|---------------------|
| 同城 ACK 从库        | 跨地域从库用异步，半同步从库放同 AZ |
| Relay Log 独立 SSD | 减少从库 fsync 耗时       |
| 避免从库过载           | 报表/备份拖慢 ACK，导致超时降级  |
| 监控 yes_tx 占比     | no_tx 持续增长则半同步未生效   |
| ACK 与只读从库分离      | 专用轻量从库负责 ACK        |

| 场景         | 异步 commit 延迟 | 半同步（AFTER_SYNC，同城） |
|------------|--------------|--------------------|
| 低 QPS OLTP | 基准           | +0.5–2 ms          |
| 高 QPS 写入   | 基准           | +1–5 ms            |
| 跨地域 ACK    | 基准           | +20–100 ms，通常不可接受  |

## 五、Binlog 格式对复制的影响

`binlog_format` 决定源库写入 Binlog 的事件形态，直接影响从库重放确定性、日志体积与延迟。复制事件按源库写入时的格式执行；级联复制场景中，启用二进制日志的从库还应避免配置为无法记录上游 Row 事件的 `STATEMENT`。

### 1. Statement 格式的复制风险

记录原始 SQL，日志体积小，但存在复制隐患：

**非确定性函数**：`NOW()`、`UUID()`、`RAND()` 在主从重放时结果可能不同；无 `ORDER BY` 的 `INSERT ... SELECT` 行顺序不确定。

**UDF 与存储过程**：依赖 session 变量或外部状态时重放可能不一致。

**replicate-do-db 与 USE db**：Statement 格式下按默认数据库（`USE db`）过滤，非 SQL 中 `db.table` 限定名，易误过滤（详见第六章）。

**锁粒度差异**：从库重放无索引 UPDATE 可能锁全表，与主库锁行为不一致。

MySQL 对部分非确定性语句在 Mixed 模式下自动改写为 Row，纯 Statement 需人工规避。

### 2. Row 格式的精确性

记录行级 before/after 镜像，重放语义确定，不受非确定性函数影响，是 WRITESET 并行复制的基础。缺点是日志体积大（宽表、无索引 UPDATE 可能产生大量 Row 事件），网络与 IO 压力上升。

`binlog_row_image` 控制镜像完整度：`FULL`（最安全，体积最大）、`MINIMAL`（仅记录定位行及重放所需列）、`NOBLOB`。`MINIMAL` 并不要求表必须有主键或唯一键，但缺少有效唯一标识时可能需要记录更多列，且排查与恢复的可观测性较弱。生产推荐 `FULL` 或 `MINIMAL`。

### 3. 实际选择建议

| 格式        | 推荐度      | 场景                     |
|-----------|----------|------------------------|
| ROW       | **生产首选** | 一致性要求高、ORM 复杂 SQL、金融核心 |
| STATEMENT | 谨慎       | SQL 完全确定性、延迟不敏感        |
| MIXED     | 一般       | 自动切换，边界排查困难，不如统一 ROW   |

```ini
binlog_format = ROW
binlog_row_image = FULL
```

新建复制拓扑建议源库统一使用 Row。从 Statement 迁移到 Row 应在低峰期进行；若从库也开启 Binlog 供级联使用，先确认其格式兼容上游事件，再观察 Binlog 体积与复制延迟变化。

复制格式对延迟与一致性的综合影响：

| 格式        | 日志体积 | 重放确定性       | 并行复制支持      | 典型风险               |
|-----------|------|-------------|-------------|--------------------|
| STATEMENT | 小    | 低（非确定性 SQL） | 弱           | 主从数据不一致            |
| ROW       | 大    | 高           | 强（WRITESET） | 网络/Relay Log IO 压力 |
| MIXED     | 中等   | 中           | 中           | 边界 case 排查困难       |

Row 格式下批量 UPDATE 产生大量 Row 事件，可能放大 IO 与 SQL 线程处理时间，间接推高复制延迟——格式选择应在架构设计阶段与主从统一配置。

## 六、复制过滤

复制过滤允许主库或从库有选择地复制部分库表，配置不当是主从不一致的高频原因，生产应谨慎使用。

### 1. 主库侧过滤

控制哪些库变更写入 Binlog：

```ini
binlog-do-db = db1
binlog-ignore-db = mysql
```

仅 `binlog-do-db` 时只有列出库写入 Binlog；仅 `binlog-ignore-db` 时列出库不写入。主库过滤影响所有从库，被过滤库无法通过 Binlog 做 PITR。**生产更推荐主库记录全量 Binlog，从库侧精细过滤。**

### 2. 从库侧过滤

控制 SQL Thread 重放哪些事件（IO Thread 仍拉取全量写入 Relay Log）：

```ini
replicate-do-db = db1
replicate-ignore-db = db2
replicate-do-table = db1.users
replicate-wild-do-table = db1.order_%
replicate-wild-ignore-table = db1.tmp_%
```

8.0.31+ 可用 `CHANGE REPLICATION FILTER`：

```sql
CHANGE REPLICATION FILTER
  REPLICATE_DO_TABLE = (db1.users, db1.orders),
  REPLICATE_WILD_IGNORE_TABLE = ('db1.tmp_%');
```

表级过滤在 Statement 与 Row 下行为相对一致，**推荐优先表级规则**。

### 3. 过滤规则的坑

#### Statement 格式下的 USE db 影响

`replicate-do-db` 在 Statement 格式下按事件**默认数据库**（Binlog 中 `USE db` 或连接默认库）判断，**非** SQL 中 `db.table`限定名。

```sql
-- 主库 default db = mysql
USE mysql;
UPDATE mydb.users SET name = 'x' WHERE id = 1;
```

从库 `replicate-do-db = mydb`：Statement 格式下默认库是 mysql，**UPDATE 被跳过**；Row 格式按变更行库 mydb 判断，**会重放**。

**使用库级过滤时应统一 Row 格式**，或确保应用默认库与操作库一致。

#### 其他常见坑

跨库 `UPDATE db1.t1 JOIN db2.t2` 在库级过滤下可能部分重放导致不一致；从库触发器写入被过滤库时行为与主库不同；IO 仍接收全量事件，Relay Log 占磁盘；多源复制下各 channel 独立过滤。最佳实践：尽量避免过滤；若必须，用 **Row + 表级白名单**，搭建后做 checksum 校验。

## 七、搭建主从复制的完整步骤

以下从零搭建一主一从异步复制，使用 MySQL 8.0 语法（`CHANGE REPLICATION SOURCE TO` / `START REPLICA`），标注 5.7 等价命令。半同步在基础复制稳定后再启用。

### 环境规划

| 角色 | 主机           | server_id |
|----|--------------|-----------|
| 主库 | 192.168.1.10 | 1         |
| 从库 | 192.168.1.11 | 2         |

前提：主从大版本一致；时钟 NTP 同步；网络互通；主库 `log_bin = ON`，`server_id` 唯一。

### 主库配置

```ini
[mysqld]
server_id = 1
log_bin = mysql-bin
binlog_format = ROW
binlog_row_image = FULL
gtid_mode = OFF
binlog_expire_logs_seconds = 604800
```

```sql
CREATE USER 'repl'@'192.168.1.11' IDENTIFIED BY 'Repl@SecurePass123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.1.11';
FLUSH PRIVILEGES;
```

### 从库配置

```ini
[mysqld]
server_id = 2
log_bin = mysql-bin
binlog_format = ROW
relay_log = relay-log
read_only = ON
super_read_only = ON
```

### 基于 mysqldump 的初始化

主库导出：

```bash
mysqldump -h 192.168.1.10 -u root -p \
  --single-transaction \
  --master-data=2 \
  --set-gtid-purged=OFF \
  --all-databases \
  --triggers --routines --events \
  > /backup/full_dump.sql
```

- `--single-transaction`：InnoDB 一致性快照。
- `--master-data=2`：以注释记录 `CHANGE MASTER TO` 所需 File/Position。
- `--set-gtid-purged=OFF`：未启用 GTID 时必须关闭。

查看位点：

```bash
grep -m1 '^-- CHANGE MASTER TO' /backup/full_dump.sql
```

从库导入：

```bash
mysql -h 192.168.1.11 -u root -p < /backup/full_dump.sql
```

### CHANGE MASTER TO 语句

从库执行：

```sql
STOP REPLICA;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '192.168.1.10',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = 'Repl@SecurePass123',
  SOURCE_LOG_FILE = 'mysql-bin.000003',
  SOURCE_LOG_POS = 157;

-- 5.7 等价：CHANGE MASTER TO MASTER_HOST=..., MASTER_LOG_FILE=..., MASTER_LOG_POS=...
```

`SOURCE_LOG_FILE` / `SOURCE_LOG_POS` 必须与 dump 中 `--master-data=2` 完全一致。若有过旧配置，先 `RESET REPLICA ALL;`。

### START SLAVE

```sql
START REPLICA;   -- 5.7: START SLAVE;
```

同时启动 IO Thread 与 SQL Thread。IO 从指定位点拉取 Binlog，SQL 重放 Relay Log 直至追平。

### SHOW SLAVE STATUS 检查

```sql
SHOW REPLICA STATUS\G
```

| 字段                                 | 期望值           | 含义       |
|------------------------------------|---------------|----------|
| `Replica_IO_Running`               | Yes           | IO 线程正常  |
| `Replica_SQL_Running`              | Yes           | SQL 线程正常 |
| `Last_IO_Error` / `Last_SQL_Error` | 空             | 错误信息     |
| `Read_Master_Log_Pos`              | 接近主库 Position | IO 进度    |
| `Exec_Master_Log_Pos`              | 接近 Read 位点    | SQL 进度   |
| `Seconds_Behind_Master`            | 0 或很小         | 估算延迟     |

主库对照 `SHOW MASTER STATUS`：当从库已执行的 Source Log File/Position 与主库当前 File/Position 对齐时可视为追平。`Seconds_Behind_Master` 仅为估算值，不能单独作为追平或一致性的判据。

**常见错误**：`error connecting to master` → 查网络、权限、bind_address；`fatal error 1236` → 位点已被 purge，需重新 dump；`Duplicate entry` → 位点错误或手动改从库；`Connecting` 持续 → 查 `Last_IO_Error`。

验证：

```sql
-- 主库：先创建测试对象，DDL 会通过复制同步到从库
CREATE DATABASE IF NOT EXISTS test;
CREATE TABLE IF NOT EXISTS test.repl_check (
  id INT PRIMARY KEY,
  message VARCHAR(32) NOT NULL
);
INSERT INTO test.repl_check VALUES (1, 'hello');
-- 从库
SELECT * FROM test.repl_check;
```

半同步插件（复制稳定后可选，以下为 MySQL 8.0.26+ 语法）：

```sql
-- 主库
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
SET GLOBAL rpl_semi_sync_source_enabled = ON;
SET GLOBAL rpl_semi_sync_source_timeout = 10000;
SET GLOBAL rpl_semi_sync_source_wait_point = AFTER_SYNC;

-- 从库
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';
SET GLOBAL rpl_semi_sync_replica_enabled = ON;

STOP REPLICA;
START REPLICA;
```

主库验证：`SHOW STATUS LIKE 'Rpl_semi_sync_source_clients';` 应 >= 1；`Rpl_semi_sync_source_status` 应为 ON。

#### 搭建检查清单

1. 主库 `log_bin = ON`，`server_id` 与从库不同。
2. 复制账户 `REPLICATION SLAVE` 权限，从库 IP 可连主库 3306。
3. mysqldump 位点与 `CHANGE REPLICATION SOURCE TO` 完全一致。
4. `SHOW REPLICA STATUS` 中 IO/SQL 均为 Yes，无 Last_*_Error。
5. 源库使用 ROW；若从库开启 Binlog 用于级联，确认其格式能记录上游 Row 事件。
6. 从库 `read_only = ON`，防止误写入。
7. 半同步（若启用）确认 `wait_point = AFTER_SYNC`，监控降级指标。

## 总结

MySQL 主从复制通过 **Binlog Dump Thread、IO Thread、SQL Thread** 协作，将主库 Binlog 传播到从库 Relay Log 并重放，是读写分离、备份隔离与异地容灾的底层基础设施。Relay Log 解耦网络拉取与 SQL 重放；复制元数据推荐存于`mysql.slave_master_info` 与 `mysql.slave_relay_log_info`。

**异步复制**是默认模式：主库不等待从库，写入延迟最低，但主库崩溃存在 RPO > 0 丢失窗口，读从库可能滞后。适用于读扩展为主、RPO 宽松的场景。

**半同步复制**在 commit 路径增加从库 ACK，推荐 **AFTER_SYNC** 等待点，保证返回成功的事务至少存在于一个从库 Relay Log，显著缩小 RPO。需权衡网络 RTT 与 fsync 开销，监控超时降级——降级期间等同异步。

**Binlog 格式**建议生产统一 **ROW**；**复制过滤**谨慎配置，优先表级规则配合 Row 格式。搭建流程：`mysqldump --master-data=2` → 从库导入 → `CHANGE REPLICATION SOURCE TO` → `START REPLICA` → `SHOW REPLICA STATUS` 验证。

GTID 拓扑管理、并行复制、延迟排查、故障切换与防脑裂将在后续专题深入。掌握复制原理后，可与 Binlog 专题及 Redo Log 崩溃恢复等知识对照，建立完整的 MySQL 数据流动与高可用知识图谱。
