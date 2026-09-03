---
title: 排序、分组与分页查询的索引优化
summary: 深入讲解 ORDER BY 的文件排序与索引排序、GROUP BY 的松散扫描与紧凑扫描、深分页的延迟关联与游标方案、以及 COUNT 不同写法的性能差异。
created: 2026-07-02
updated: 2026-07-07
tags: MySQL, InnoDB, 索引, 排序, 分页, 性能优化
---

# 排序、分组与分页查询的索引优化

在 OLTP 与报表系统中，排序（ORDER BY）、分组（GROUP BY）、分页（LIMIT）与计数（COUNT）是最常见的 SQL 形态之一。它们与 WHERE 点查不同：优化器不仅要找到满足条件的行，还要保证输出顺序、完成聚合或统计，因此更容易触发文件排序（filesort）、临时表（temporary table）以及大 offset 扫描等额外成本。

本文从 InnoDB B+ 树索引的有序性出发，系统讲解上述四类查询如何利用索引、何时退化为额外排序与临时表，以及深分页与 COUNT 的工程化优化方案。文中示例基于如下测试表结构，分析时建议结合 `EXPLAIN`（MySQL 8.0.18+ 可用 `EXPLAIN ANALYZE`）与 `optimizer_trace` 交叉验证。

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
```

## 一、ORDER BY 与索引

### 1. 索引排序 vs 文件排序

InnoDB 的二级索引与聚簇索引均按 B+ 树键值有序存储。当 `ORDER BY` 的列顺序、排序方向与某条可用索引的前缀一致，且不需要额外排序步骤时，优化器可在索引扫描过程中直接得到有序结果，称为**索引排序**（index scan ordering）。此时 `EXPLAIN` 的 `Extra` 列通常**不出现** `Using filesort`。

索引排序成立的典型条件：

1. `ORDER BY` 中的列与索引列从左至右连续匹配（遵循最左前缀）。
2. 各列排序方向与索引定义方向兼容（MySQL 8.0 之前对混合 ASC/DESC 有限制，见后文）。
3. `WHERE` 中的等值条件已"消耗"索引前缀，剩余 `ORDER BY` 列仍落在同一索引的有序段内。
4. 未对 `ORDER BY` 列施加函数或表达式（如 `DATE(created_at)`）。

```sql
-- 索引排序：user_id 等值 + created_at 有序
EXPLAIN FORMAT=TRADITIONAL
SELECT *
FROM orders
WHERE user_id = 10086
ORDER BY created_at DESC
LIMIT 20;
```

典型输出：`type = ref`，`key = idx_user_created`，`rows ≈ 500`，`Extra = Using where`（**无** `Using filesort`）。MySQL 8.0 对降序索引支持更完善；5.7 中 InnoDB 可通过**反向索引扫描**满足纯 `DESC`，混合升降序时仍可能 filesort。

当 `ORDER BY` 列无法被任何可用索引的有序性覆盖时，优化器必须在获取结果集后执行额外排序，称为**文件排序**（filesort）。名称中的 "file" 指当内存缓冲区不足时，排序会落盘到临时文件，并非一定读写磁盘——小结果集可能完全在内存中完成。

```sql
-- 文件排序：ORDER BY amount 无匹配索引
EXPLAIN FORMAT=TRADITIONAL
SELECT *
FROM orders
WHERE user_id = 10086
ORDER BY amount DESC
LIMIT 20;
```

典型输出：`type = ref`，`key = idx_user_created`，`Extra = Using filesort`。

`Using filesort` 的含义：

- 优化器需要对中间结果集按 `ORDER BY` 键重新排序。
- 排序键可能包含 `ORDER BY` 列、SELECT 中的额外列（取决于排序算法），以及用于 tie-breaker 的主键。
- 排序发生在 server 层，与存储引擎返回行的顺序无关。
- 数据量较大或排序行较宽时，filesort 可能成为查询延迟的主要贡献者。

| 现象       | Extra                             | 含义                         |
|----------|-----------------------------------|----------------------------|
| 索引有序输出   | 无 `Using filesort`                | 扫描即有序，成本最低                 |
| 额外排序     | `Using filesort`                  | 需 sort buffer 或临时文件        |
| 排序 + 临时表 | `Using temporary; Using filesort` | 常见于 GROUP BY / DISTINCT 组合 |

### 2. 文件排序的两种算法

MySQL 的 filesort 在实现上分为**单路排序**（single-pass /全字段排序）与**双路排序**（two-pass /回表排序）两种策略。优化器根据排序行宽度、可用内存与版本差异选择具体算法。

**单路排序（全字段排序）**：将 SELECT 列表中需要的**全部列**与 `ORDER BY` 键一并放入 sort buffer，排序完成后直接输出，无需二次回表取列。

适用条件（简化理解）：

- 排序行总宽度不超过 `max_length_for_sort_data`（MySQL 5.7 默认 1024 字节；MySQL 8.0 该参数已弃用（deprecated），优化器内部策略调整）。
- SELECT 列较少，或宽列不在排序键中但可被一次性装入 sort buffer。

```sql
-- 较窄的结果集，倾向单路排序
SELECT id, user_id, created_at
FROM orders
WHERE status = 1
ORDER BY created_at
LIMIT 100;
```

单路排序的优点是一次性完成排序与投影，缺点是每个排序元素携带完整行宽，sort buffer 能容纳的行数变少，更容易触发外部排序（落盘）。

**双路排序（回表排序）**：在 sort buffer 中仅存放 **ORDER BY 键 + 主键**（InnoDB 下为聚簇索引键），先对键排序，再按主键顺序回表读取完整行。

适用条件：

- 排序行较宽（如 `SELECT *` 且表含 `VARCHAR(255)` 等大列）。
- 单路排序估算所需内存超过阈值，优化器判定双路更省内存。

```sql
-- 宽行 SELECT *，更可能触发双路排序
SELECT *
FROM orders
WHERE status = 1
ORDER BY created_at
LIMIT 100;
```

双路排序减少了 sort buffer 中每行的体积，可容纳更多排序行，但排序完成后需按主键顺序逐行回表，产生额外随机 IO。是否更优取决于回表行数、Buffer Pool 命中率与磁盘性能。

**sort_buffer_size**：每个排序操作可使用的内存上限（默认 256KB，可调整）。filesort 尝试在 sort buffer 内完成排序；若数据量超出，则采用
**外部归并排序**，分块排序后写入临时文件再归并。

```sql
SHOW VARIABLES LIKE 'sort_buffer_size';
-- 会话级临时调大（仅用于验证，生产需评估全局影响）
SET SESSION sort_buffer_size = 2097152;  -- 2MB
```

注意：

- `sort_buffer_size` 是**每个排序操作**分配的上限，不是全局共享池。并发大量 filesort 查询时，总内存消耗 = 连接数 × sort_buffer_size，可能引发 OOM。
- 盲目调大该参数不一定加速：超过实际需要时浪费内存，且现代 MySQL 版本对排序算法有多处优化，应通过 `EXPLAIN ANALYZE` 度量而非仅凭参数猜测。

**max_length_for_sort_data**（MySQL 5.7）：控制单路排序与双路排序的切换阈值；当排序行估算宽度超过该值时，优化器倾向双路排序。

```sql
SHOW VARIABLES LIKE 'max_length_for_sort_data';
-- MySQL 5.7 默认 1024
```

MySQL 8.0 已弃用该参数（deprecated），排序策略由优化器基于成本模型自动决定。升级版本后，同一 SQL 的 filesort 行为可能发生变化，应在回归测试中重新验证。

| 算法   | sort buffer 内容           | 优点           | 缺点           |
|------|--------------------------|--------------|--------------|
| 单路排序 | ORDER BY 键 + 全部 SELECT 列 | 排序后无需回表      | 单行宽，易触发外部排序  |
| 双路排序 | ORDER BY 键 + 主键          | buffer 容纳行数多 | 排序后需回表，随机 IO |

### 3. 临时文件排序

当待排序行数 × 行宽超过 `sort_buffer_size` 时，filesort 进入**外部排序**（external merge sort）：

1. 分批读取数据，每批在 sort buffer 内排序后写入临时文件（`tmpdir` 目录）。
2. 对所有临时文件做多路归并，得到最终有序输出。
3. 查询结束后删除临时文件。

可通过 `optimizer_trace` 观察排序细节（仅建议在测试环境开启）：

```sql
SET optimizer_trace = 'enabled=on', end_markers_in_json = on;

SELECT *
FROM orders
WHERE status = 1
ORDER BY created_at
LIMIT 100000;

SELECT TRACE FROM information_schema.OPTIMIZER_TRACE\G
```

trace 中可能出现 `"sort_mode": "<sort_key, additional_fields>"` 或 `"<sort_key, rowid>"` 等字段，指示单路或双路排序；若出现`"filesort_priority_queue_optimization"` 则与 LIMIT 下的堆排序优化有关（见后文）。

外部排序的代价：

- 磁盘 IO 显著增加，`Created_tmp_disk_tables` 状态计数上升。
- 归并阶段可能占用大量 CPU 与 IO 带宽。
- 临时文件路径（`tmpdir`）若与数据目录同盘，可能与 InnoDB 刷脏竞争。

排查建议：

```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
SHOW GLOBAL STATUS LIKE 'Sort%';
```

`Sort_merge_passes` 持续偏高，说明外部归并频繁发生，应优先从索引消除 filesort，而非仅调大 sort buffer。

### 4. ORDER BY 与联合索引

联合索引 `(a, b, c)` 在 B+ 树上按 `(a, b, c)` 字典序排列。`ORDER BY` 能否利用该索引，取决于 `WHERE` 已匹配的 prefix 与 `ORDER BY` 剩余列的衔接关系。

当 `ORDER BY` 方向与索引方向一致时，`user_id` 等值确定后，同一 `user_id` 下 `created_at` 在 `idx_user_created` 中天然有序，ASC 无需 filesort；纯 DESC 时 InnoDB 可反向扫描索引。MySQL 8.0 起支持显式降序索引，使物理顺序与查询方向完全一致：

```sql
CREATE INDEX idx_user_created_desc
ON orders (user_id ASC, created_at DESC);
```

**ASC/DESC 混合排序**：MySQL 5.7 及之前，以下查询通常无法被 `(user_id, created_at)` 索引完全满足：

```sql
SELECT *
FROM orders
WHERE user_id = 10086
ORDER BY created_at ASC, id DESC;
```

若 `created_at` 相同行较多，需要 `id DESC` 作为 tie-breaker，但索引未按 `(created_at ASC, id DESC)` 定义，优化器可能 filesort。

MySQL 8.0 可创建混合方向索引：

```sql
CREATE INDEX idx_user_created_id
ON orders (user_id ASC, created_at ASC, id DESC);
```

设计原则：

- 将 `WHERE` 等值列放在索引最左。
- `ORDER BY` 列紧随其后，排序方向与索引定义一致。
- 若排序键不唯一，追加主键列作为 tie-breaker，保证顺序确定性。

```sql
-- 不唯一排序键需 tie-breaker
SELECT *
FROM orders
WHERE user_id = 10086
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

推荐索引：`(user_id, created_at DESC, id DESC)`。

### 5. ORDER BY + WHERE 的索引利用

经典模式 `WHERE a = 1 ORDER BY b` 与联合索引 `(a, b)` 构成完美匹配：

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT id, user_id, created_at
FROM orders
WHERE user_id = 10086
ORDER BY created_at DESC
LIMIT 20;
-- type=ref, key=idx_user_created, Extra=Using where（无 filesort）
-- 若仅 SELECT user_id, created_at，Extra 追加 Using index（覆盖索引）
```

**范围条件截断 ORDER BY**：

```sql
SELECT *
FROM orders
WHERE user_id = 10086
  AND created_at >= '2026-01-01'
ORDER BY created_at
LIMIT 20;
```

`user_id` 等值 + `created_at` 范围：`created_at` 在索引中仍有序，`ORDER BY created_at` 通常仍可被索引满足。

但若范围列与 ORDER BY 列不一致：

```sql
SELECT *
FROM orders
WHERE user_id = 10086
  AND created_at >= '2026-01-01'
ORDER BY amount;
```

`created_at` 范围扫描后，同一 `user_id` 内行不再按 `amount` 有序，必然 filesort。

| WHERE 模式          | ORDER BY | 索引 (a, b, c) | 能否索引排序      |
|-------------------|----------|--------------|-------------|
| `a = ?`           | `b`      | `(a, b)`     | 可以          |
| `a = ? AND b > ?` | `b`      | `(a, b)`     | 可以          |
| `a = ? AND b > ?` | `c`      | `(a, b, c)`  | 不可以（b 范围截断） |
| `a > ?`           | `b`      | `(a, b)`     | 不可以         |

### 6. ORDER BY + LIMIT 的优化

当存在 `ORDER BY ... LIMIT N` 且 N 较小时，优化器可能选择**优先队列堆排序**（priority queue optimization），而非对全部满足 WHERE 的行做完整排序。

```sql
SELECT *
FROM orders
WHERE status = 1
ORDER BY created_at DESC
LIMIT 10;
```

trace 中可能出现 `"filesort_priority_queue_optimization": true`，表示仅维护大小为 N 的堆，扫描过程中保留 Top N，复杂度约 O(rows × log N)，而非 O(rows × log rows)。

这对无索引排序的查询是显著优化：即使 `status = 1` 匹配百万行，也只需堆维护 10 行。

堆排序触发条件：存在 `ORDER BY` + `LIMIT`、走 filesort 路径、LIMIT 较小。但也可能引入问题：

1. **与覆盖索引的交互**：若存在覆盖索引 `(status, created_at, id)`，优化器可能选择索引扫描 + 早停（遇到 LIMIT 满足即停止），而非全量 filesort。不同版本成本估算可能导致计划波动。

2. **深 LIMIT offset 不适用堆优化**：`LIMIT 100000, 10` 仍需跳过 100000 行，堆排序无法消除 offset 扫描成本。

3. **不稳定顺序**：若 `ORDER BY` 键不唯一且缺少 tie-breaker，Top N 结果可能在重复键边界上不确定；生产环境应追加主键保证确定性。

```sql
-- 推荐：追加主键 tie-breaker
SELECT *
FROM orders
WHERE status = 1
ORDER BY created_at DESC, id DESC
LIMIT 10;
```

4. **EXPLAIN 仍显示 Using filesort**：堆优化是 filesort 的一种实现策略，`Extra` 仍可能显示 `Using filesort`，需通过 `EXPLAIN ANALYZE` 或 trace 区分全量排序与堆优化。

## 二、GROUP BY 与索引

### 1. 松散索引扫描（Loose Index Scan）

松散索引扫描（Loose Index Scan）是 MySQL 针对 GROUP BY 的一种高效执行方式：优化器在索引上**跳跃式**读取每个分组的第一条记录，而非扫描组内全部行，从而以 O(分组数) 而非 O(总行数) 的代价完成分组。

```sql
SELECT status, COUNT(*)
FROM orders
GROUP BY status;
```

若走 Loose Index Scan，`EXPLAIN` 的 `Extra` 出现：

```
Using index for group-by
```

含义：利用索引有序性，每次跳到下一个 distinct 分组键值，再对该组做聚合（如 COUNT）。

Loose Index Scan **适用条件**（需同时满足）：

1. 查询仅涉及单表（无 JOIN）。
2. GROUP BY 列构成某索引的**最左前缀**，且顺序一致。
3. 聚合函数为 `MIN()`、`MAX()`、`COUNT(*)`（无 DISTINCT）、`SUM()`（特定条件）等可增量计算的形式。
4. WHERE 条件为常量等值，且仅涉及索引前缀列。
5. GROUP BY 列之间无表达式（如 `DATE(created_at)` 会破坏条件）。

```sql
-- 可能适用：GROUP BY 首列是 idx_status_amount 前缀
EXPLAIN FORMAT=TRADITIONAL
SELECT status, MIN(amount), MAX(amount)
FROM orders
GROUP BY status;
```

```
Extra: Using index for group-by
```

```sql
-- 不适用：GROUP BY 含表达式
EXPLAIN FORMAT=TRADITIONAL
SELECT DATE(created_at) AS d, COUNT(*)
FROM orders
GROUP BY DATE(created_at);
```

```
Extra: Using temporary; Using filesort
```

Loose Index Scan 是 GROUP BY 优化中的"理想路径"，但适用面较窄；多数复杂 GROUP BY 会退化为紧凑扫描或临时表。

### 2. 紧凑索引扫描（Tight Index Scan）

紧凑索引扫描（Tight Index Scan）是更常见的 GROUP BY 执行方式：优化器按索引顺序**逐行扫描**，在扫描过程中识别分组边界并累积聚合值，无需为每组随机跳转。

```sql
SELECT user_id, COUNT(*)
FROM orders
GROUP BY user_id;
```

若存在 `idx_user_created (user_id, created_at)`，相同 `user_id` 的行在索引中相邻，优化器可沿索引顺序扫描，在 `user_id` 变化处提交上一组聚合结果。

与 Loose Index Scan 的区别：

| 维度    | Loose Index Scan           | Tight Index Scan        |
|-------|----------------------------|-------------------------|
| 扫描方式  | 跳跃，每组读首行                   | 顺序，读组内所有行               |
| 适用聚合  | MIN/MAX/COUNT(*) 等         | 更广泛，含 SUM、AVG 等         |
| Extra | `Using index for group-by` | 可能无特殊标记，或 `Using index` |
| 成本    | 接近分组数                      | 接近满足条件的行数               |

对 `COUNT(*)` 而言，Tight Scan 仍需扫描组内每一行以计数；Loose Scan 若适用，每组仅读一条。因此 MIN/MAX 类聚合更常从 Loose Scan 受益。

```sql
-- Tight Scan + 覆盖索引
EXPLAIN FORMAT=TRADITIONAL
SELECT user_id, SUM(amount)
FROM orders
WHERE user_id > 10000
GROUP BY user_id;
```

优化器沿 `idx_user_created` 扫描 `user_id > 10000` 的区间，边扫描边按 `user_id` 分组累加。若 `amount` 不在索引中，需回表取 `amount`，`Extra` 可能出现 `Using where; Using temporary` 或回表相关提示。

### 3. GROUP BY 的临时表与排序

当无法利用索引有序性完成分组时，MySQL 采用以下策略之一或组合：

1. **哈希聚合**：建临时哈希表，按 GROUP BY 键分组（内存不足则落盘）。
2. **排序聚合**：先对 GROUP BY 键 filesort，再顺序扫描分组（内存不足同样落盘）。

```sql
SELECT user_id, status, COUNT(*)
FROM orders
GROUP BY user_id, status;
```

若无 `(user_id, status)` 联合索引，`EXPLAIN` 常见：

```
Extra: Using temporary; Using filesort
```

`Using temporary` 表示创建临时表（内存或磁盘）；`Using filesort` 表示对输入流或临时表按 GROUP BY 键排序。两者**同时出现**的典型原因：

1. **无合适索引**：分组键与任何可用索引前缀不对齐，必须构建临时结构。
2. **GROUP BY 与 ORDER BY 不一致**：

```sql
SELECT status, COUNT(*) AS cnt
FROM orders
GROUP BY status
ORDER BY cnt DESC;
```

分组可能走 `idx_status_amount`，但 `ORDER BY cnt` 需对聚合结果再排序，出现 filesort。若分组本身也需临时表，则两者并存。

3. **DISTINCT 与 GROUP BY 混用**或 SELECT 列不满足 ONLY_FULL_GROUP_BY 时的隐式行为（MySQL 8.0 默认开启 ONLY_FULL_GROUP_BY，不规范写法直接报错）。

4. **UNION / 子查询** 内层 GROUP BY 各自产生临时表，外层再排序。

```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
```

`Created_tmp_tables` 与 `Created_tmp_disk_tables` 比例升高，说明 GROUP BY 频繁落盘，应评估索引与 SQL 改写。

### 4. GROUP BY 优化策略

**索引设计原则**：让 GROUP BY 列顺序与联合索引前缀一致，并尽量覆盖 WHERE 过滤列。

```sql
-- 查询模式
SELECT user_id, status, COUNT(*), SUM(amount)
FROM orders
WHERE created_at >= '2026-01-01'
GROUP BY user_id, status;

-- 推荐索引
CREATE INDEX idx_created_user_status
ON orders (created_at, user_id, status);
```

注意：`created_at` 为范围条件时，其右侧 `user_id, status` 可能无法用于索引排序分组，但仍可用于过滤。更精确的设计需结合 EXPLAIN 验证：有时 `(user_id, status, created_at)` 更优，取决于选择性。

**SQL_BIG_RESULT / SQL_SMALL_RESULT** 提示（传统写法，现代版本应谨慎使用）：

```sql
SELECT SQL_BIG_RESULT user_id, COUNT(*)
FROM orders
GROUP BY user_id;

SELECT SQL_SMALL_RESULT status, COUNT(*)
FROM orders
GROUP BY status;
```

- `SQL_BIG_RESULT`：暗示结果集大，优化器倾向使用磁盘临时表与排序聚合。
- `SQL_SMALL_RESULT`：暗示结果集小，倾向内存临时表。

现代版本中优化器成本模型已较智能，盲目使用提示可能适得其反。更可靠的做法是强制索引：

```sql
SELECT user_id, COUNT(*)
FROM orders FORCE INDEX (idx_user_created)
GROUP BY user_id;
```

## 三、分页查询优化

### 1. LIMIT offset, count 的问题

传统分页写法：

```sql
SELECT *
FROM orders
WHERE user_id = 10086
ORDER BY created_at DESC
LIMIT 100000, 10;
```

语义：跳过前 100000 行，返回后续 10 行。

InnoDB 在索引有序扫描下仍需：

1. 从索引起点（或 WHERE 确定的范围起点）开始顺序读取。
2. 计数已扫描行数，跳过前 100000 行。
3. 输出第 100001 至 100010 行。

因此**至少需要扫描 100010 行**，其中 100000 行被读取后丢弃。延迟随 offset 线性增长，与是否走索引无关——索引只能保证有序，不能跳过中间段。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders
WHERE user_id = 10086 ORDER BY created_at DESC LIMIT 100000, 10;
-- type=ref, key=idx_user_created, rows≈100010, Extra=Using where
```

`rows` 估算约 100010：需扫描 100010 行并丢弃前 100000 行。若 `SELECT *` 还需回表，宽表 IO 压力显著。这是 OFFSET 的结构性缺陷，浅分页可接受，深分页必须换方案。

### 2. 延迟关联方案

延迟关联（Deferred Join / Late Row Lookup）的核心思想：**先在索引上完成过滤、排序与分页定位，仅对最终需要的少量行回表取宽列**。

```sql
-- 延迟关联：子查询在索引上定位 id，外层仅回表 10 行
SELECT o.*
FROM orders o
INNER JOIN (
    SELECT id FROM orders
    WHERE user_id = 10086
    ORDER BY created_at DESC
    LIMIT 100000, 10
) AS t ON o.id = t.id;
```

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders WHERE user_id = 10086
    ORDER BY created_at DESC LIMIT 100000, 10
) AS t ON o.id = t.id;
/*
  DERIVED: type=ref, key=idx_user_created, rows≈100010
  PRIMARY o: type=eq_ref, key=PRIMARY, rows=1（每匹配行回表一次，共 10 次）
*/
```

子查询涉及索引列 `user_id, created_at` 与主键 `id`（InnoDB 二级索引叶子必含主键）。**延迟关联不消除 offset 扫描**，仅将回表从约 100010 次降为 10 次；对宽表收益明显，对窄表或已覆盖索引的查询收益有限。

### 3. 游标分页（Keyset Pagination）

游标分页（也称 Seek Method / Keyset Pagination）用**上一页最后一条记录的排序键**作为下页起点，替代 OFFSET。

```sql
-- 第一页
SELECT *
FROM orders
WHERE user_id = 10086
ORDER BY created_at DESC, id DESC
LIMIT 10;

-- 后续页：假设上一页最后一条 created_at = '2026-03-15 10:00:00', id = 987654321
SELECT *
FROM orders
WHERE user_id = 10086
  AND (created_at, id) < ('2026-03-15 10:00:00', 987654321)
ORDER BY created_at DESC, id DESC
LIMIT 10;
```

MySQL 行构造比较 `(created_at, id) < (?, ?)` 在 8.0 中可正确用于复合排序的 seek 条件。

每页查询通过索引 seek 到起点，仅扫描 LIMIT 行，复杂度 O(LIMIT)，与页码无关（`rows ≈ 10`，`Extra = Using where`）。

**限制**：不支持随机跳页，仅适合无限滚动、时间线等顺序浏览；须保证排序键唯一（或接受 tie-breaker 边界风险）；翻页参数由客户端持久化。

### 4. 覆盖索引 + 延迟关联

若分页排序列与主键均在覆盖索引中，子查询可完全在索引层完成；外层仅需列也在索引内时可退化为单表覆盖查询：

```sql
-- 索引 idx_user_created_id (user_id, created_at, id)
SELECT id, user_id, created_at
FROM orders
WHERE user_id = 10086
ORDER BY created_at DESC
LIMIT 100000, 10;
-- Extra: Using where; Using index（仍扫描 offset+limit 行，但无回表）

-- 游标 + 覆盖：深分页最优组合
SELECT id, user_id, created_at, amount
FROM orders
WHERE user_id = 10086
  AND (created_at, id) < ('2026-03-15 10:00:00', 987654321)
ORDER BY created_at DESC, id DESC
LIMIT 10;
```

### 5. 各方案性能对比

设 `user_id = 10086` 匹配 50 万行，每页 20 条，不同 offset 下的扫描行数：

| offset  | OFFSET 分页扫描行数 | 延迟关联索引扫描 | 游标分页扫描 |
|---------|---------------|----------|--------|
| 0       | 20            | 20       | 20     |
| 1,000   | 1,020         | 1,020    | 20     |
| 100,000 | 100,020       | 100,020  | 20     |
| 500,000 | 500,020       | 500,020  | 20     |

延迟关联与 OFFSET 在**索引扫描行数**上等价，差异在回表次数。游标分页在扫描行数上与 offset 无关。

| 方案           | 扫描行数（深 offset）    | 回表次数                         | 跳页  | 实现复杂度 |
|--------------|-------------------|------------------------------|-----|-------|
| LIMIT offset | O(offset + limit) | O(offset + limit) 或 O(limit) | 支持  | 低     |
| 延迟关联         | O(offset + limit) | O(limit)                     | 支持  | 中     |
| 游标分页         | O(limit)          | O(limit)                     | 不支持 | 中     |
| 覆盖 + 游标      | O(limit)          | 0                            | 不支持 | 中     |

**方案选择决策树**：需随机跳页 → offset 较小用 `LIMIT offset` + 索引，offset 大用延迟关联或禁止深跳页；仅顺序翻页 → 游标分页 + 覆盖索引 + 主键 tie-breaker。

## 四、COUNT 的不同写法与性能差异

### 1. COUNT(*) vs COUNT(1) vs COUNT(column) vs COUNT(主键)

| 写法              | 语义                         |
|-----------------|----------------------------|
| `COUNT(*)`      | 统计结果集行数，含 NULL 列的行         |
| `COUNT(1)`      | 与 COUNT(*) 等价，统计行数         |
| `COUNT(column)` | 统计 column **非 NULL** 的行数   |
| `COUNT(主键)`     | 主键通常 NOT NULL，等价于 COUNT(*) |

```sql
SELECT
    COUNT(*)      AS c_star,
    COUNT(1)      AS c_one,
    COUNT(remark) AS c_remark  -- remark 可为 NULL
FROM orders;
```

若 10% 行的 `remark` 为 NULL，`c_remark` 约为 `c_star` 的 90%。

`COUNT(*)` 与 `COUNT(1)` 在 InnoDB 中**等价**：优化器对二者采用相同策略：选择**最小的可用二级索引**或聚簇索引进行扫描计数，不会逐行读取完整宽行（除非无二级索引，则扫描聚簇索引叶子）。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT COUNT(*) FROM orders;
EXPLAIN FORMAT=TRADITIONAL SELECT COUNT(1) FROM orders;
```

两者 `key` 通常相同（如 `idx_created` 或最窄二级索引），`Extra: Using index`。官方 SQL 标准推荐写 `COUNT(*)`，语义最清晰。

### 2. InnoDB 中 COUNT(*) 的实现

MyISAM 将总行数维护在表元数据中，`COUNT(*)` 为 O(1)。InnoDB 支持 MVCC，不同事务可见行集不同，无法在表级维护精确行数，因此 `COUNT(*)` 必须扫描索引叶子节点，本质 O(N)。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT COUNT(*) FROM orders;
-- type=index, key=idx_created（最小索引）, Extra=Using index
```

优化器选择**最短 key_len** 的二级索引；避免 `COUNT(big_varchar_column)`，必要时 `USE INDEX` 指定。

### 3. 大表 COUNT 的优化方案

亿级表上 `COUNT(*)` 可能耗时数秒至数十秒，阻塞或占用 IO。工程上常用以下方案：

**计数表**：

```sql
CREATE TABLE table_counters (
    table_name  VARCHAR(64) PRIMARY KEY,
    row_count   BIGINT UNSIGNED NOT NULL DEFAULT 0
);

-- 插入时 +1，删除时 -1，事务内与业务同事务提交
```

精确、O(1) 读取，但需在应用层或触发器维护，且带条件计数需分维度维护（如按 status 分别计数）。

**SHOW TABLE STATUS / information_schema.TABLES**：读取 `Rows` / `table_rows` 估算值，误差可达 40%～50%，极快但不精确。**Redis 缓存计数**：适合展示型场景，需处理一致性与穿透问题。

| 方案                | 精确度 | 读成本  | 写成本       |
|-------------------|-----|------|-----------|
| COUNT(*)          | 精确  | O(N) | 无         |
| 计数表               | 精确  | O(1) | 每次 DML 维护 |
| SHOW TABLE STATUS | 估算  | O(1) | 无         |
| Redis             | 可配置 | O(1) | 维护逻辑      |

### 4. COUNT 与条件查询

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT COUNT(*) FROM orders WHERE user_id = 10086;
-- type=ref, key=idx_user_created, Extra=Using where; Using index
```

条件 COUNT 应将 WHERE 列纳入索引前缀：

```sql
-- 高频：按状态计数
SELECT COUNT(*) FROM orders WHERE status = 1;

-- 索引 idx_status_amount (status, amount) 可用于 ref 访问 status = 1 的全部行
```

若仅需判断存在性，避免 COUNT：

```sql
-- 低效
SELECT COUNT(*) FROM orders WHERE user_id = 10086 AND status = 1;

-- 高效
SELECT 1 FROM orders WHERE user_id = 10086 AND status = 1 LIMIT 1;
```

`EXISTS` 子查询在找到首行后即可停止，适合"是否有记录"场景。

复合条件 COUNT 的索引选择取决于选择性：优化器选择估算扫描行数最少的索引。可通过 `EXPLAIN` 对比 `(status, user_id)` 与`(user_id, status)` 的计划。

## 五、DISTINCT 与索引

### 1. DISTINCT 的执行方式

`SELECT DISTINCT col` 需在结果集中去除重复值，常见实现路径：

**基于排序去重**：对 DISTINCT 列排序后合并相邻重复项（`Extra: Using filesort`），或利用索引有序性跳过排序。**基于临时表去重**：建哈希表按 DISTINCT 键插入（`Extra: Using temporary`）。优化器根据列宽、基数与内存选择路径。

### 2. DISTINCT 与 INDEX

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT DISTINCT status FROM orders;
-- 有 idx_status_amount 时可能 Extra=Using index for group-by

EXPLAIN FORMAT=TRADITIONAL SELECT DISTINCT user_id, status FROM orders;
-- 无 (user_id, status) 索引时常见 Using temporary; Using filesort
```

若 DISTINCT 列构成索引**最左前缀**，可沿索引有序扫描跳过重复值。`SELECT DISTINCT a, b` 与索引 `(a, b)` 完美匹配。注意 `DISTINCT ... ORDER BY other_col` 若 `other_col` 不在 distinct 键中，可能引入 filesort。

### 3. DISTINCT vs GROUP BY

无聚合时，`SELECT DISTINCT col` 与 `GROUP BY col` **结果等价**：

```sql
SELECT DISTINCT status FROM orders;
-- 等价于
SELECT status FROM orders GROUP BY status;
```

MySQL 优化器对二者可能生成相似计划。带聚合时只能用 GROUP BY：

```sql
SELECT status, COUNT(*) FROM orders GROUP BY status;
-- 不能用 DISTINCT 直接表达
```

**不等价**的情况：

1. **ONLY_FULL_GROUP_BY 模式**：`GROUP BY` 对 SELECT 列有严格约束；`DISTINCT` 作用于整体 SELECT 列表。

```sql
SELECT DISTINCT status, amount FROM orders;
-- 与 GROUP BY status, amount 结果相同

SELECT status, COUNT(*) FROM orders GROUP BY status;
-- DISTINCT 无法替代
```

2. **NULL 处理**：两者均将多个 NULL 视为一个 distinct 值。
3. **优化路径差异**：同一语义下计划可能不同，应用 EXPLAIN 对比，勿假设性能等价。

## 六、索引维护成本

排序、分组、分页与 COUNT 的优化往往依赖**更多或更宽的索引**。索引改善读路径的同时，引入写放大与空间成本，必须在设计时纳入评估。

### 1. 写放大

InnoDB 除聚簇索引外，每个二级索引均为独立 B+ 树。INSERT 需更新全部索引（本例 4 棵 B+ 树），可能触发页分裂与 Redo/Undo 写入。UPDATE 若变更索引列键值，需在相关二级索引中 delete-mark 旧项并插入新项。为排序/分组新增的联合索引若含高变更列，写放大显著。

```sql
INSERT INTO orders (user_id, status, amount, created_at, updated_at)
VALUES (10086, 1, 99.00, NOW(), NOW());

UPDATE orders SET status = 2, amount = amount + 10 WHERE id = 12345;
```

### 2. 空间开销

宽联合索引显著增加 `Index_length`（可通过 `SHOW TABLE STATUS` 对比 `Index_length` 与 `Data_length`）。索引过多时，热点索引页挤占 Buffer Pool，降低数据页命中率。经验：单表二级索引宜控制在 5～6 个以内，超过时需审计冗余。

### 3. 如何评估索引的价值

**sys 库**（MySQL 5.7+）：

```sql
-- 从未使用的索引（自上次重启起）
SELECT *
FROM sys.schema_unused_indexes
WHERE object_schema = 'mydb'
  AND object_name = 'orders';

-- 索引统计
SELECT *
FROM sys.schema_index_statistics
WHERE table_schema = 'mydb'
  AND table_name = 'orders';
```

`schema_unused_indexes` 需结合业务周期解读：报表索引可能仅在月批 job 中使用，重启后未命中不代表无用。

删除前先 `ALTER TABLE ... ALTER INDEX idx_candidate INVISIBLE`（MySQL 8.0）观察，确认无影响后再 `DROP INDEX`。**pt-index-usage** 从慢查询日志分析索引使用频率：

```bash
pt-index-usage /var/log/mysql/slow.log --host localhost
```

输出建议删除的索引及支持证据。应在代表性负载下采集足够长的日志，避免误删低频但关键的索引（如备份校验、监管报表）。

**索引价值评估框架**：

1. 该索引服务哪些高频 SQL（ORDER BY / GROUP BY / 分页 / COUNT）？
2. 优化器是否**稳定**选择该索引（非 occasional force）？
3. 写 QPS 与读 QPS 比值如何？写多读少场景应更保守。
4. 能否与现有索引合并为联合索引，减少冗余？

为深分页单独建 `(user_id, created_at DESC, id DESC)` 若同时覆盖 80% 列表查询，价值高；若仅服务于极低频后台导出，应权衡。

## 总结

排序、分组、分页与 COUNT 的优化，本质是让查询在 B+ 树有序结构上完成工作，避免 filesort、临时表与大 offset 扫描。

- **ORDER BY**：列顺序与索引前缀对齐且方向兼容时可索引排序；filesort 分单路/双路，超出 `sort_buffer_size` 则外部归并。`WHERE a=? ORDER BY b` 匹配 `(a,b)`；`LIMIT N` 可能堆优化但不解决深 OFFSET。
- **GROUP BY**：Loose Scan（`Using index for group-by`）最优但条件严；否则 Tight Scan 或 `Using temporary; Using filesort`。分组键与 ORDER BY 不一致时常再排序。
- **分页**：OFFSET 必扫 offset+n 行；延迟关联降回表不降扫描；游标分页性能恒定，需 tie-breaker 主键，适合顺序翻页。
- **COUNT / DISTINCT**：InnoDB 中 `COUNT(*)`≈`COUNT(1)`，扫最小索引；大表用计数表或估算。DISTINCT 可利用索引有序去重。
- **维护成本**：索引带来写放大与 Buffer Pool 压力；用 sys、`pt-index-usage` 评估 ROI，少而精。

优化应建立在 EXPLAIN / EXPLAIN ANALYZE 度量之上，在延迟、吞吐量与索引维护成本间取得平衡。
