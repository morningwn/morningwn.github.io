---
title: 慢查询分析与 SQL 性能调优方法论
summary: 从慢查询日志的定位工具到典型调优案例，建立一套可复用的 SQL 性能分析与优化方法论，串联索引设计与执行计划知识。
created: 2026-07-02
updated: 2026-07-16
tags: MySQL, 慢查询, 性能优化, SQL 调优
---

# 慢查询分析与 SQL 性能调优方法论

慢查询是 MySQL 生产环境中最常见的性能问题入口。一条 SQL 从"能跑"到"跑得慢"，往往涉及索引缺失、执行计划退化、数据分布变化、SQL 写法不当或参数配置不合理等多重因素。若缺乏系统化的分析流程，调优容易陷入"试索引、看 EXPLAIN、再试"的随机迭代。

本文以 InnoDB 为基准，从慢查询日志的采集与配置出发，介绍 `mysqldumpslow` 与 Percona Toolkit 的 `pt-query-digest`两类聚合工具，建立"减少扫描行数、减少回表、减少排序、减少临时表"的通用调优思路，并通过五个典型生产案例演示从定位到改写的完整路径。文中示例基于如下测试表结构：

```sql
CREATE TABLE orders (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id     BIGINT UNSIGNED NOT NULL,
    status      TINYINT         NOT NULL DEFAULT 0,
    amount      DECIMAL(12, 2)  NOT NULL,
    created_at  DATETIME        NOT NULL,
    updated_at  DATETIME        NOT NULL,
    remark      VARCHAR(255)    DEFAULT NULL,
    PRIMARY KEY (id),
    KEY idx_user_created (user_id, created_at),
    KEY idx_status_amount (status, amount),
    KEY idx_created (created_at)
) ENGINE=InnoDB;

CREATE TABLE users (
    id    BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    name  VARCHAR(64)     NOT NULL,
    city  VARCHAR(32)     NOT NULL,
    PRIMARY KEY (id),
    KEY idx_city (city)
) ENGINE=InnoDB;

CREATE TABLE order_items (
    id       BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    order_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    quantity INT             NOT NULL,
    PRIMARY KEY (id),
    KEY idx_order_id (order_id)
) ENGINE=InnoDB;
```

调优过程中，建议始终将慢查询日志、Performance Schema、`EXPLAIN`（MySQL 8.0.18+ 可用 `EXPLAIN ANALYZE`）三者交叉验证。慢查询日志告诉你"哪条 SQL 慢"，执行计划告诉你"为什么慢"，Performance Schema 告诉你"全局热点在哪里"。

## 一、慢查询日志

慢查询日志（Slow Query Log）是 MySQL 内置的性能诊断机制，记录执行时间超过指定阈值的 SQL 语句及其执行上下文。它是性能排查的第一手证据，也是建立 SQL 性能基线与持续改进的反馈入口。

### 1. 开启与配置

**slow_query_log 与 slow_query_log_file**

`slow_query_log` 控制是否开启慢查询日志，生产环境建议常开。`slow_query_log_file` 指定日志文件的绝对路径，建议放在独立磁盘或分区，避免与 InnoDB 数据文件、Redo Log、Binlog 争抢 IO。

MySQL 8.0 起支持两种输出目标：`FILE`（默认）与 `TABLE`（写入 `mysql.slow_log`）。生产环境通常使用 `FILE`，由日志采集组件转发至集中存储；开发环境可临时使用 `TABLE` 便于交互式查询。

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL log_output = 'FILE';

-- TABLE 模式查询示例
SELECT start_time, user_host, query_time, lock_time, rows_sent, rows_examined, sql_text
FROM mysql.slow_log
ORDER BY start_time DESC
LIMIT 20;
```

**long_query_time**

`long_query_time` 定义慢查询阈值，单位为秒，**默认值为 10 秒**——对大多数在线业务而言过高，会遗漏大量实际影响用户体验的慢
SQL。

```sql
SHOW VARIABLES LIKE 'long_query_time';
SET GLOBAL long_query_time = 0.5;   -- 生产常见：0.5 ~ 1 秒
SET GLOBAL long_query_time = 0;       -- 记录所有查询，仅短期诊断
```

`long_query_time` 同时具有 GLOBAL 和 SESSION 作用域：`SET GLOBAL` 只影响之后新建的会话；当前连接如需立即生效，还应执行对应的 `SET SESSION`。

| 业务类型     | 建议阈值        | 说明         |
|----------|-------------|------------|
| 在线交易/API | 0.5 ~ 1 秒   | 与接口 SLA 对齐 |
| 报表/批处理   | 2 ~ 5 秒     | 允许较长执行时间   |
| 开发/测试    | 0.1 ~ 0.5 秒 | 尽早发现潜在问题   |

**log_queries_not_using_indexes**

设为 `ON` 时，即使未超过 `long_query_time`，未使用索引的查询也会被记录。价值在于捕获"执行时间尚可但存在全表扫描风险"的 SQL。生产环境应配合 `min_examined_row_limit` 使用，排查完成后关闭，避免小表全表扫描产生大量噪声。

```sql
SET GLOBAL log_queries_not_using_indexes = ON;
```

**min_examined_row_limit**

定义扫描行数下限。无论语句是因超过 `long_query_time`，还是因启用 `log_queries_not_using_indexes` 而候选记录，均须至少扫描该行数才会写入慢日志。默认 0 表示不设下限。

```sql
SET GLOBAL min_examined_row_limit = 1000;
-- 记录未走索引且扫描超过 1000 行的查询
```

**完整配置示例**

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;
SET GLOBAL log_queries_not_using_indexes = OFF;
SET GLOBAL min_examined_row_limit = 1000;
SET GLOBAL log_slow_extra = ON;  -- MySQL 8.0+，输出扩展字段
```

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 0.5
log_queries_not_using_indexes = 0
min_examined_row_limit = 1000
log_slow_extra = 1
log_output = FILE
```

### 2. 慢查询日志的格式

以下是一条典型的慢查询日志条目（MySQL 8.0.14+，`log_slow_extra = ON`，且输出目标为 `FILE`）：

```
# Time: 2026-07-02T10:15:30.123456Z
# User@Host: app_user[app_user] @ 10.0.1.50 [10.0.1.50]  Id: 12345
# Query_time: 2.345678  Lock_time: 0.000123  Rows_sent: 20  Rows_examined: 150000
# Thread_id: 12345  Errno: 0  Killed: 0  Bytes_received: 96  Bytes_sent: 4096
# Read_key: 1  Read_next: 150000  Sort_rows: 0
# Created_tmp_tables: 0  Created_tmp_disk_tables: 0
# Start: 2026-07-02T10:15:30.123456Z  End: 2026-07-02T10:15:32.469134Z
SET timestamp=1719917730;
SELECT * FROM orders WHERE user_id = 10086 ORDER BY created_at DESC LIMIT 100000, 20;
```

**时间信息**：日志中的 `# Time:` 可用于关联业务高峰、批处理任务与发布变更；MySQL 8.0.14+ 的 `SET timestamp` 表示慢语句开始执行的时间，适合在 `pt-query-digest` 中按 `--since`/`--until` 过滤。

**Query_time**：语句执行耗时（秒），不包含获取初始锁所花的时间；初始锁等待由 `Lock_time` 单独记录。它是判断"慢"的直接指标。关注 avg vs max（长尾）、与 SLA 对比、同一 digest 的趋势变化。

**Lock_time**：等待表锁/行锁的累计时间。若占 `Query_time` 比例超过 50%，瓶颈在锁竞争（长事务、热点行、间隙锁、DDL 并发），而非索引。可用`performance_schema.data_lock_waits` 排查。

**Rows_sent**：返回给客户端的行数。与 `Rows_examined` 对比可判断扫描效率：`扫描效率 = Rows_examined / Rows_sent`。

| Rows_sent | Rows_examined | 比值   | 解读           |
|-----------|---------------|------|--------------|
| 1         | 1             | 1    | 主键点查，理想      |
| 20        | 25            | 1.25 | 索引精确匹配，良好    |
| 20        | 150000        | 7500 | 大量无效扫描，需优化   |
| 0         | 1000000       | ∞    | 全表扫描无结果，更需优化 |

**Rows_examined**：执行过程中检查的行数，是慢查询分析中最重要的字段之一。与 `EXPLAIN` 的 `rows` 估算可能不完全一致；MySQL 8.0 的 `EXPLAIN ANALYZE` 输出 `actual rows` 可交叉验证。即使 `Query_time` 不长，极高的 `Rows_examined` 也预示数据增长后将变成慢查询。

**扩展字段（`log_slow_extra = ON`）**

| 字段                                | 含义             | 分析价值                    |
|-----------------------------------|----------------|-------------------------|
| `Thread_id` / `Errno` / `Killed`  | 线程与语句结果        | 关联会话、识别失败或被终止的语句      |
| `Bytes_received` / `Bytes_sent`   | 收发字节数           | 识别大请求或大结果集              |
| `Read_*`                          | 本语句的 Handler 读取计数 | 判断索引查找、范围扫描与全扫描特征    |
| `Sort_*`                          | 本语句的排序计数       | 判断排序行数与归并次数              |
| `Created_tmp_*`                   | 本语句创建的内存/磁盘临时表数 | 判断临时表开销                  |
| `Start` / `End`                   | 语句开始与结束时间      | 精确关联业务事件与耗时              |

字段组合分析：`Rows_examined` 显著高于 `Rows_sent` 时检查 WHERE 条件与索引；`Sort_rows` 或 `Sort_merge_passes` 很高时检查 ORDER BY；`Lock_time` 接近 `Query_time` 时排查长事务；`Created_tmp_disk_tables` 持续增加时优化 GROUP BY、DISTINCT 或 UNION。`Full_scan`、`Filesort` 等字段不是 Oracle MySQL 8.0 的 `log_slow_extra` 输出，不能据此解析本文示例。

### 3. Performance Schema 中的语句统计

慢查询日志只记录超过阈值的个体事件，存在采样盲区。Performance Schema（PFS）的 `events_statements_summary_by_digest` 按 SQL 指纹聚合已启用并保留的语句统计，提供全局视角；生产环境应确认相应 consumer 已开启且汇总表未因容量上限过早淘汰热点。

```sql
SELECT
    SCHEMA_NAME,
    DIGEST_TEXT,
    COUNT_STAR                    AS exec_count,
    ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
    ROUND(AVG_TIMER_WAIT / 1e12, 4) AS avg_sec,
    ROUND(MAX_TIMER_WAIT / 1e12, 4) AS max_sec,
    SUM_ROWS_SENT,
    SUM_ROWS_EXAMINED,
    ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT, 0), 1) AS scan_ratio,
    SUM_NO_INDEX_USED,
    FIRST_SEEN,
    LAST_SEEN
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'shop'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

关键字段：`DIGEST_TEXT`（归一化 SQL 模板）、`COUNT_STAR`（执行次数）、`SUM_TIMER_WAIT`（累计耗时，皮秒 ÷ 1e12 = 秒）、`SUM_ROWS_EXAMINED`/`SUM_ROWS_SENT`（扫描与返回行数）、`SUM_NO_INDEX_USED`（未使用索引次数）。

按扫描效率找问题 SQL：

```sql
SELECT DIGEST_TEXT, COUNT_STAR,
       ROUND(AVG_TIMER_WAIT / 1e12, 4) AS avg_sec,
       SUM_ROWS_EXAMINED, SUM_ROWS_SENT,
       ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT, 0), 0) AS scan_ratio
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ROWS_SENT > 0 AND SUM_ROWS_EXAMINED / SUM_ROWS_SENT > 1000
ORDER BY SUM_ROWS_EXAMINED DESC LIMIT 20;
```

**与慢查询日志的互补**

| 维度     | 慢查询日志        | Performance Schema |
|--------|--------------|--------------------|
| 数据范围   | 仅超过阈值的 SQL   | 所有 SQL             |
| 时间精度   | 精确到每条的时间戳    | 聚合统计               |
| SQL 原文 | 完整 SQL（含参数值） | 仅 digest 模板        |
| 适用场景   | 复现具体慢 SQL    | 发现全局热点             |

推荐工作方式：用 PFS 找出总耗时 Top N 的 digest → 在慢日志中搜索完整 SQL → 优化后对比同一 digest 的 `AVG_TIMER_WAIT` 与`SUM_ROWS_EXAMINED`。

```sql
-- 优化前后对比时重置统计
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
```

## 二、慢查询分析工具

单条慢查询日志只能反映一次执行；生产环境每天可能产生 GB 级慢日志，必须借助聚合工具提取规律。

### 1. mysqldumpslow

`mysqldumpslow` 随 MySQL 客户端工具包分发，无需额外安装。

```bash
mysqldumpslow /var/log/mysql/slow.log
mysqldumpslow -t 20 /var/log/mysql/slow.log
mysqldumpslow /var/log/mysql/slow.log /var/log/mysql/slow.log.1
```

`-s` 参数控制排序：

| 选项      | 含义             | 适用场景    |
|---------|----------------|---------|
| `-s t`  | 按累计 Query_time | 总负载贡献最大 |
| `-s at` | 按平均 Query_time | 单次最慢    |
| `-s c`  | 按出现次数          | 高频 SQL  |
| `-s l`  | 按累计 Lock_time  | 锁竞争     |
| `-s r`  | 按累计 Rows_sent  | 返回大量数据  |

```bash
mysqldumpslow -s t -t 20 /var/log/mysql/slow.log
mysqldumpslow -s c -t 20 /var/log/mysql/slow.log
mysqldumpslow -g 'orders' -s at -t 20 /var/log/mysql/slow.log
mysqldumpslow -g '!^SELECT' -s at -t 10 /var/log/mysql/slow.log
```

输出示例：

```
Count: 1523  Time=2.35s (3579s)  Lock=0.00s (0s)  Rows=20.0 (30460), app_user@10.0.1.50
  SELECT * FROM orders WHERE user_id = N ORDER BY created_at DESC LIMIT N, N
```

解读：`Count` 为出现次数；`Time=2.35s (3579s)` 为平均/累计耗时；最后一行为归一化 SQL 模板。

**局限性**：无法按数据库/用户分组；无 P95/P99 百分位；分组粗糙；无历史对比。适合快速浏览，生产系统化分析应使用`pt-query-digest`。

### 2. pt-query-digest

Percona Toolkit 的 `pt-query-digest` 是业界标准的慢查询日志分析工具。

```bash
pt-query-digest /var/log/mysql/slow.log > digest_report.txt
pt-query-digest --since '1h' /var/log/mysql/slow.log
pt-query-digest --filter '$event->{db} eq "shop"' /var/log/mysql/slow.log
pt-query-digest --limit 10 /var/log/mysql/slow.log
pt-query-digest --review h=127.0.0.1,D=percona,t=query_review \
  --review-history h=127.0.0.1,D=percona,t=query_review_history \
  /var/log/mysql/slow.log
```

**指纹（Fingerprint）**：对 SQL 做归一化处理后的模板——字面量替换为 `?`，IN 列表替换为 `?+`，去除注释。结构相同、参数不同的 SQL 归为同一类，反映"SQL 模式"级别的性能。

**报告解读**

Profile 部分按响应时间贡献排序：

```
# Rank Query ID           Response time  Calls  R/Call  V/M
#    1 0xABC123...         3579.0000 65.0%   1523  2.3500  0.01
#    2 0xDEF456...          890.5000 16.2%    456  1.9520  0.02
```

- `Response time`：对总响应时间的贡献（绝对值与百分比）。
- `R/Call`：每次调用平均响应时间。
- `V/M`：方差/均值比，越大说明长尾越严重。

Query Analysis 部分逐条展开：

```
# Query 1: 65.0% (3579s) SELECT * FROM orders WHERE user_id = ? ORDER BY ...
# Attribute    pct   total     min     max     avg     95%  stddev  median
# Exec time     65   3579s      1s      5s      2s      3s      1s      2s
# Rows examine  99  228M   100020  150020  149800  150020   10000  150020
```

关注 `Exec time` 的 95% 列（P95）、`Rows examine` 与 `Rows sent` 对比、Query_time 分布直方图。

**按响应时间贡献排序**：生产优化优先级应基于"对总负载的贡献"——高贡献 + 高频率优先优化；高贡献 + 低频率评估是否可异步；低贡献 + 高频率累积影响不可忽视。

### 3. 分析工作流

从日志收集到问题定位的完整流程：

1. **确认采集**：检查 `slow_query_log`、`long_query_time` 与 `min_examined_row_limit`，使阈值与 SLA 对齐。
2. **收集日志**：`scp db-server:/var/log/mysql/slow.log ./slow_20260702.log`。
3. **聚合分析**：`mysqldumpslow -s t -t 20` 查看累计耗时 Top SQL；`pt-query-digest --limit 30` 产出详细报告。
4. **问题分类**：按 Rows_examined/Rows_sent（扫描过多）、`Sort_*`（排序）、`Created_tmp_*`（临时表）、Lock_time（锁）以及 EXPLAIN 的 JOIN 访问方式归类。
5. **复现**：从库执行 `EXPLAIN FORMAT=TRADITIONAL` 与 `EXPLAIN ANALYZE`（8.0.18+）。
6. **改写验证**：对比优化前后 `Rows_examined`、Query_time、Buffer Pool 命中率。
7. **基线存档**：优化前后各跑一次 `pt-query-digest`，或写入 review 表做历史对比。

## 三、SQL 调优的通用思路

SQL 性能调优是有层次、有优先级的系统化过程：先减少扫描行数，再减少回表，再减少排序，最后减少临时表。

### 1. 减少扫描行数

扫描行数是大多数慢查询的根因。`EXPLAIN` 的 `type` 字段反映扫描规模：`ALL`（全表）> `index`（全索引）> `range` > `ref` >`eq_ref` > `const`。

**加索引**

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 10086 AND status = 1;
-- type = ALL, rows = 1000000

CREATE INDEX idx_user_status ON orders (user_id, status);
EXPLAIN SELECT * FROM orders WHERE user_id = 10086 AND status = 1;
-- type = ref, key = idx_user_status, rows = 120
```

索引设计：WHERE 等值列在前、范围列在后；遵循最左前缀；选择性高的列优先。

**改写查询条件**

```sql
-- 差：函数导致索引失效
SELECT * FROM orders WHERE YEAR(created_at) = 2026;
-- 好：范围条件
SELECT * FROM orders WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';

-- 差：OR 导致索引合并或全表扫描
SELECT * FROM orders WHERE user_id = 10086 OR status = 1;
-- 好：UNION ALL
SELECT * FROM orders WHERE user_id = 10086
UNION ALL
SELECT * FROM orders WHERE status = 1 AND user_id <> 10086;

-- 差：前缀通配
SELECT * FROM users WHERE name LIKE '%张三%';
-- 好：前缀匹配或外部检索
SELECT * FROM users WHERE name LIKE '张三%';
```

目标：`Rows_examined / Rows_sent` 比值在个位数以内。

### 2. 减少回表

InnoDB 二级索引不包含完整行，定位主键后需回表读取聚簇索引页。

**覆盖索引**：索引包含查询所需的全部列，`Extra` 出现 `Using index`。

```sql
EXPLAIN SELECT user_id, created_at FROM orders
WHERE user_id = 10086 ORDER BY created_at DESC LIMIT 20;
-- Extra: Using index
```

**延迟关联**：先通过覆盖索引获取主键，再回表取宽行，回表次数从 O(n) 降为 O(limit)。

```sql
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders WHERE user_id = 10086
    ORDER BY created_at DESC LIMIT 100000, 20
) t ON o.id = t.id;
```

### 3. 减少排序

`ORDER BY` 列无法被索引有序访问时，优化器执行 filesort。filesort 开销与需排序的行数成正比，而非 LIMIT 的值——即使 `LIMIT 20`，对 50 万行排序后取 20 行成本仍然极高。

```sql
EXPLAIN SELECT * FROM orders WHERE status = 1 ORDER BY amount DESC LIMIT 50;
-- Extra: Using filesort

CREATE INDEX idx_status_amount ON orders (status, amount);
-- 沿索引有序扫描，遇到 50 行即可终止，无 filesort
```

MySQL 8.0 支持索引降序：`CREATE INDEX idx_status_created ON orders (status, created_at DESC);`

减少不必要的 ORDER BY：主键点查无需排序；结果集小可下推应用层。

### 4. 减少临时表

GROUP BY、DISTINCT、UNION 可能触发内部临时表。在 MySQL 8.0 中，默认的 TempTable 内存引擎受 `tmp_table_size` 等限制，超出限制或遇到不支持的列类型时可能转为磁盘存储；应以 `Created_tmp_disk_tables` 和执行计划验证，不能只凭参数判断。

```sql
EXPLAIN SELECT status, COUNT(*), AVG(amount) FROM orders GROUP BY status;
-- Extra: Using temporary; Using filesort

CREATE INDEX idx_status ON orders (status);
-- Extra: Using index，无需临时表
```

减少 DISTINCT：若只需判断存在性，用 `SELECT 1 ... LIMIT 1` 替代 `SELECT DISTINCT ...`。

### 5. 调优决策树

根据 EXPLAIN 输出选择优化方向：

```
EXPLAIN 分析
├─ type = ALL？
│   ├─ 是 → WHERE 列有索引？无则加索引；有则查隐式转换/函数/最左前缀
│   └─ 否 → 继续
├─ Rows_examined >> Rows_sent？
│   ├─ 是 → 优化索引列顺序 / 改写缩小范围 / 覆盖索引减少回表
│   └─ 否 → 继续
├─ Extra 含 Using filesort？
│   ├─ 是 → 建立 (WHERE列, ORDER BY列) 联合索引
│   └─ 否 → 继续
├─ Extra 含 Using temporary？
│   ├─ 是 → GROUP BY/DISTINCT 列加索引或改写 SQL
│   └─ 否 → 继续
├─ Lock_time 高？→ 排查锁竞争（非索引问题）
└─ 以上均否 → 检查 Buffer Pool 命中率、参数配置、硬件 IO
```

自上而下排查；未解决全表扫描就调 `sort_buffer_size` 是本末倒置；MySQL 8.0.18+ 用 `EXPLAIN ANALYZE` 验证。

## 四、典型案例一：深分页优化

### 1. 问题描述与 EXPLAIN 分析

业务订单列表采用 OFFSET 分页，翻到第 5000 页时接口超时：

```sql
SELECT * FROM orders WHERE user_id = 10086
ORDER BY created_at DESC LIMIT 100000, 20;
```

`pt-query-digest` 显示累计耗时占 Top 10 的 35%，平均 `Rows_examined` 约 100020，`Query_time` 2.35 秒。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders WHERE user_id = 10086 ORDER BY created_at DESC LIMIT 100000, 20;
```

```
| type | key              | rows   | Extra       |
| ref  | idx_user_created | 100020 | Using where |
```

走了索引，但 OFFSET 语义要求扫描并丢弃前 100000 行；`SELECT *` 导致 100020 次回表随机 IO。MySQL 8.0.18+ 的`EXPLAIN ANALYZE` 可验证 `actual rows=100020`。

### 2. 方案一：延迟关联

```sql
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders WHERE user_id = 10086
    ORDER BY created_at DESC LIMIT 100000, 20
) t ON o.id = t.id;
```

子查询 `Using index`（覆盖索引扫描 id），外层仅 20 次主键回表。`Query_time` 从 2.35 秒降至约 0.8 秒。局限：索引扫描行数仍为 O(offset + limit)。

### 3. 方案二：游标分页

用上一页最后一条记录的排序键边界替代 OFFSET：

```sql
-- 第一页
SELECT * FROM orders WHERE user_id = 10086
ORDER BY created_at DESC, id DESC LIMIT 20;

-- 下一页（上一页最后一条 created_at='2026-03-15 10:00:00', id=987654321）
SELECT * FROM orders WHERE user_id = 10086
  AND (created_at, id) < ('2026-03-15 10:00:00', 987654321)
ORDER BY created_at DESC, id DESC LIMIT 20;
```

每次只扫描 20 行，性能与页码无关。要求排序键追加 `id` 作 tie-breaker；前端传递游标而非页码；不支持跳转任意页。

### 4. 方案三：覆盖索引 + 子查询

先用覆盖索引取 id（`SELECT id FROM orders ... LIMIT offset, 20`），再 `SELECT * FROM orders WHERE id IN (...)`回表取宽行。效果与延迟关联类似；注意 MySQL 对 IN 子查询的优化限制，推荐 JOIN 写法。

### 5. 各方案的适用场景

| 方案           | 扫描行数              | 回表次数              | 支持跳页 | 适用场景           |
|--------------|-------------------|-------------------|------|----------------|
| LIMIT offset | O(offset + limit) | O(offset + limit) | 是    | 仅前几页           |
| 延迟关联         | O(offset + limit) | O(limit)          | 是    | 宽表 + 中等 OFFSET |
| 游标分页         | O(limit)          | O(limit)          | 否    | 无限滚动、深分页       |

推荐：OFFSET < 1000 可直接 LIMIT；中等深度用延迟关联；Feed 流用游标分页；强需求跳页则前几页 OFFSET + 深页游标。

## 五、典型案例二：大 IN 列表

### 1. 问题描述

批量查询接口传入数千个 `user_id`：

```sql
SELECT * FROM orders
WHERE user_id IN (1001, 1002, /* ... 5000 个 ID ... */)
  AND status = 1
ORDER BY created_at DESC;
```

`Rows_examined` 可达数百万，`Query_time` 随 IN 列表长度线性增长。

```sql
EXPLAIN SELECT * FROM orders
WHERE user_id IN (1001, 1002, /* ... */) AND status = 1 ORDER BY created_at DESC;
-- type = range, rows = 500000
-- Extra: Using index condition; Using where; Using filesort
```

### 2. 为什么大 IN 列表性能差

1. **范围优化器开销**：IN 转为多个范围条件，列表超过 `range_optimizer_max_mem_size`（默认 8MB）时，优化器可能放弃范围优化并选择代价更高的访问路径，须以实际 EXPLAIN 为准。
2. **索引扫描开销**：5000 个值的范围扫描会产生大量索引查找，且命中行数可能很大。
3. **filesort 叠加**：IN 打断索引有序性，ORDER BY 无法利用索引。
4. **SQL 解析开销**：5000 个字面量的 SQL 文本本身很大。

```sql
SHOW VARIABLES LIKE 'range_optimizer_max_mem_size';
```

### 3. 方案一：临时表 JOIN

```sql
CREATE TEMPORARY TABLE tmp_user_ids (
    user_id BIGINT UNSIGNED NOT NULL PRIMARY KEY
) ENGINE=Memory;

INSERT INTO tmp_user_ids (user_id) VALUES (1001), (1002), (1003);  -- ... 5000 行

SELECT o.* FROM orders o
INNER JOIN tmp_user_ids t ON o.user_id = t.user_id
WHERE o.status = 1
ORDER BY o.created_at DESC;
-- orders 作为被驱动表通常为 type = ref；实际次数和行数取决于每个 user_id 的订单数
```

跨会话批量可用带 `batch_id` 的辅助表 `batch_user_ids`，查询完成后按 batch 清理。

### 4. 方案二：分批查询

应用层将 5000 个 ID 拆为 10 批，每批 500，分别执行 `WHERE user_id IN (...)`后合并结果并在应用层排序。优点：简单，无需临时表。缺点：多次数据库往返、应用层归并。适用：ID 来自外部且无法写入临时表、批次数可控（< 20）的场景。

### 5. 方案三：子查询改写

若 ID 来自同库另一张表，用 JOIN 替代 IN：

```sql
-- 差
SELECT * FROM orders WHERE user_id IN (SELECT user_id FROM vip_users WHERE level >= 3);
-- 好
SELECT o.* FROM orders o
INNER JOIN vip_users v ON o.user_id = v.user_id WHERE v.level >= 3;
```

ID 来自应用层时，结合临时表 JOIN 是最佳实践。

## 六、典型案例三：ORDER BY + LIMIT 陷阱

### 1. 问题描述

首页"最新已支付订单"：

```sql
SELECT * FROM orders WHERE status = 1 ORDER BY created_at DESC LIMIT 20;
```

存在 `idx_status_amount (status, amount)` 和 `idx_created (created_at)`，P99 延迟达 3 秒。

```sql
EXPLAIN SELECT * FROM orders WHERE status = 1 ORDER BY created_at DESC LIMIT 20;
-- 路径 A：走 idx_created，按时间有序扫描，每行回表检查 status，大量丢弃
-- 路径 B：走 idx_status_amount，过滤 10 万行后 filesort，O(n log n)
-- Extra: Using where; Using filesort
```

### 2. 优化器的行为

优化器面临两难：`idx_created` 可按时间有序但需回表过滤 status（10% 选择性，平均扫 200 行找 20 行）；`idx_status_amount` 先过滤 10 万行再 filesort。成本估算可能摇摆，导致计划不稳定。核心矛盾：现有索引无法同时满足 `WHERE status=1`（等值）和`ORDER BY created_at`（排序）。

### 3. 解决方案

建立联合索引，WHERE 等值列在前、ORDER BY 列在后：

```sql
CREATE INDEX idx_status_created ON orders (status, created_at DESC);

EXPLAIN SELECT * FROM orders WHERE status = 1 ORDER BY created_at DESC LIMIT 20;
-- type = ref, key = idx_status_created, rows = 20, 无 Using filesort
```

沿 `created_at DESC` 有序扫描，遇到 20 行即可终止（Early Termination）。

无法对齐时的降级：建立 `(created_at, amount)` 若范围选择性足够；使用延迟关联；评估业务是否可改排序字段。

**核心原则**：`ORDER BY col LIMIT N` 依赖索引有序扫描的提前终止；filesort + LIMIT 无法替代。

## 七、典型案例四：隐式类型转换

### 1. 问题描述与 EXPLAIN 分析

```sql
CREATE TABLE user_contacts (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    phone VARCHAR(20) NOT NULL,
    PRIMARY KEY (id),
    KEY idx_phone (phone)
);

EXPLAIN SELECT * FROM user_contacts WHERE phone = 13800138000;
-- type = ALL, key = NULL, rows = 1000000

EXPLAIN SELECT * FROM user_contacts WHERE phone = '13800138000';
-- type = ref, key = idx_phone, rows = 1
```

### 2. 根因分析

MySQL 对字符串列与数值常量比较时，会按数值规则解释字符串列。许多不同字符串都可能转换为同一数值，因此无法通过字符串索引快速定位。反过来，数值列与可转换的字符串常量比较通常会将常量转换为数值，不能笼统地认定索引失效；应以 `EXPLAIN` 验证。

常见场景：

| 场景         | 示例                              | 后果        |
|------------|---------------------------------|-----------|
| 数字列 = 字符串  | `user_id = '10086'`             | 常量通常可转换，仍应以 EXPLAIN 验证 |
| 字符串列 = 数字  | `phone = 13800138000`           | 字符串列按数值比较，索引不能快速定位 |
| 字符集不一致     | utf8mb4 列 = latin1 常量           | 转换，索引可能失效 |
| JOIN 类型不匹配 | `orders.user_id = users.id_str` | JOIN 索引失效 |

```sql
EXPLAIN FORMAT=TREE SELECT * FROM user_contacts WHERE phone = 13800138000;
```

### 3. 修复方案

应用层确保绑定变量类型与列类型一致（例如 phone 传字符串 `'13800138000'`，而非数值）。SQL 规范：避免让索引列参与隐式转换；若必须转换，应显式转换常量或参数侧，并用 `EXPLAIN` 验证。Code Review 时，`type = ALL` 且表行数 > 10000，优先检查类型、函数与索引前缀。

修复后可将扫描范围从全表收敛为索引定位；实际 `Rows_examined` 与耗时取决于数据分布和返回行数，应以 `EXPLAIN ANALYZE` 或慢日志复测为准。

## 八、典型案例五：不合理的 JOIN

### 1. 问题描述

订单详情页，多表 JOIN 且被驱动表无合适索引或驱动表选择不当：

```sql
SELECT o.id, o.amount, o.created_at, u.name, u.city, oi.product_id, oi.quantity
FROM orders o
INNER JOIN users u ON o.user_id = u.id
INNER JOIN order_items oi ON o.id = oi.order_id
WHERE o.status = 1 AND u.city = 'Shanghai'
ORDER BY o.created_at DESC LIMIT 50;
```

P99 延迟 5 秒，`Rows_examined` 达数百万。

### 2. EXPLAIN 分析

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT o.id, o.amount, ... ;
```

优化器从 `users` 驱动（city='Shanghai' 过滤 5 万用户），对每个用户查订单（平均 12 条），再查明细（平均 3 条）。总扫描：50000 × 12 × 3 ≈ 180 万行。`Using temporary; Using filesort`。

若从 `orders` 驱动：10 万 status=1 的订单，回表查 users 过滤 city，99% 被丢弃。

### 3. 优化方案

**方案 A：在确定的排序路径上逐条过滤，再 JOIN**

```sql
CREATE INDEX idx_status_created ON orders (status, created_at DESC);

SELECT o.id, o.amount, o.created_at, u.name, u.city, oi.product_id, oi.quantity
FROM (
    SELECT id, user_id, amount, created_at FROM orders
    WHERE status = 1
      AND EXISTS (
          SELECT 1 FROM users u1
          WHERE u1.id = orders.user_id AND u1.city = 'Shanghai'
      )
    ORDER BY created_at DESC LIMIT 50
) o
INNER JOIN users u ON o.user_id = u.id
INNER JOIN order_items oi ON o.id = oi.order_id;
```

子查询以 `(status, created_at)` 索引提供排序路径，并在 LIMIT 前完成城市过滤，避免先取 50 条订单再过滤城市而导致结果不足 50 条或遗漏后续符合条件的订单。外层仅对最多 50 个订单做 JOIN；若城市选择性极低，仍可能需要扫描较多候选订单，应以 `EXPLAIN ANALYZE` 评估。

**方案 B：确保被驱动表有索引并调整驱动顺序**

`order_items.order_id` 需有 `idx_order_id`；`users.id` 是主键。本例瓶颈在驱动表选择——可用 `STRAIGHT_JOIN` 强制从 orders 驱动（慎用），或拆分为"先 LIMIT 取订单 ID，再批量 JOIN"的业务层查询。

JOIN 优化原则：小表驱动大表；被驱动表 JOIN 列必须有索引；先过滤后 JOIN；避免 SELECT *。

优化后，外层 JOIN 的行数可显著收敛；整体收益取决于城市选择性、每单明细数和统计信息，不能脱离实际数据承诺固定的扫描行数或耗时。

## 九、数据库参数调优

SQL 改写与索引设计是主路径；参数调整是补充。SQL 未优化前盲目调参数，往往事倍功半。

### 1. Buffer Pool 相关

`innodb_buffer_pool_size` 决定 InnoDB 缓存数据页与索引页的大小，通常设为物理内存的 50%~70%。

```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
-- 命中率 = 1 - Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests
```

命中率 > 99% 为充足；< 95% 可能不足或存在大量随机 IO。增大 Buffer Pool 无法替代索引优化——全表扫描会将热数据挤出缓存。

```ini
[mysqld]
innodb_buffer_pool_size = 32G
innodb_buffer_pool_instances = 8
```

### 2. 排序相关（sort_buffer_size）

每个会话的排序缓冲区，仅在 filesort 时使用。默认 256KB。

```sql
SET SESSION sort_buffer_size = 2097152;  -- 2MB
```

filesort 落盘时可适当增大，但**优先消除 filesort**。全局过大 × 高并发 = 内存爆炸。

### 3. JOIN 相关（join_buffer_size）

`join_buffer_size` 是 JOIN 执行中可能使用的每个连接缓冲区。`Extra: Using join buffer` 常提示可进一步检查 JOIN 条件、驱动顺序与被驱动表索引，但不等同于“必然缺索引”；MySQL 8.0 的哈希 JOIN 等执行方式也会使用它。正确做法是先用 EXPLAIN ANALYZE 确认访问路径，再决定是否加索引或调整该参数。

### 4. 临时表相关（tmp_table_size）

对于用户显式创建的 `MEMORY` 临时表，大小受 `tmp_table_size` 和 `max_heap_table_size` 中较小者限制（默认各 16MB）。MySQL 8.0 的内部临时表默认使用 TempTable 引擎，其内存与落盘行为还受 `temptable_max_ram` 等参数影响。

```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
-- Created_tmp_tables vs Created_tmp_disk_tables
```

磁盘临时表比例高时，优先优化 SQL 消除临时表，再考虑增大参数。

### 5. 连接相关（max_connections、wait_timeout）

```sql
SHOW VARIABLES LIKE 'max_connections';       -- 默认 151
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'wait_timeout';          -- 默认 28800 (8 小时)
```

`max_connections` 过高导致 per-session 内存 × 连接数膨胀与线程切换开销。根本解决：应用层连接池、优化慢 SQL 减少连接占用、读写分离。

```ini
[mysqld]
max_connections = 500
wait_timeout = 600
interactive_timeout = 600
```

### 6. 参数调优的原则：先优化 SQL，再调参数

```
1. 慢查询日志 + pt-query-digest 定位 Top SQL
2. EXPLAIN / EXPLAIN ANALYZE 分析执行计划
3. 索引优化 + SQL 改写
4. 验证 Rows_examined、Query_time 下降
5. 评估 Buffer Pool 命中率
6. 针对性调参数（filesort/临时表/JOIN buffer）
7. 硬件扩容作为最后手段
```

其他参数：`range_optimizer_max_mem_size`（大 IN 列表）、`innodb_io_capacity`（SSD 刷脏）、`innodb_flush_log_at_trx_commit` 与`sync_binlog`（持久化策略）。配合 `processlist` 与 PFS 等待事件、主机 iostat 做全链路排查。

## 总结

慢查询调优是需要工具、方法论与领域知识共同支撑的工程实践。

**采集与定位**：慢查询日志应常开，`long_query_time` 与 SLA 对齐（通常 0.5~1 秒）；关注 `Query_time`、`Lock_time`、`Rows_examined`、`Rows_sent` 及其比值。`mysqldumpslow` 快速浏览，`pt-query-digest` 生产级聚合；PFS 的`events_statements_summary_by_digest` 与慢日志互补。

**通用思路**：按"减少扫描行数 → 减少回表 → 减少排序 → 减少临时表"顺序排查；索引应同时服务 WHERE、ORDER BY 与 LIMIT 的早期终止；用调优决策树根据 EXPLAIN 选择方向。

**案例回顾**：深分页用延迟关联或游标分页；大 IN 用临时表 JOIN；ORDER BY + LIMIT 需联合索引对齐；隐式类型转换使索引失效；不合理 JOIN 需调整驱动表与被驱动表索引。

**参数与工程**：先优化 SQL 再调参数；建立"聚合报告 → EXPLAIN ANALYZE → 改写验证 → 基线对比"闭环；优化前后存档 digest，关注`ANALYZE TABLE` 与计划退化。以度量驱动迭代，比零散"加索引"更能应对生产环境的性能挑战。
