---
title: JOIN 的执行原理与优化策略
summary: 从 Nested-Loop Join 到 Hash Join，系统拆解 MySQL 多表关联的物理执行方式，理解驱动表选择、Join Buffer 与索引对 JOIN 性能的影响。
created: 2026-07-02
updated: 2026-07-16
tags: MySQL, JOIN, 嵌套循环, Hash Join, 性能优化
---

# JOIN 的执行原理与优化策略

多表关联是关系型数据库中最常见也最复杂的操作之一。业务 SQL 中 `JOIN` 的写法往往只有几行，但优化器在物理执行层可能选择 Simple Nested-Loop Join、Block Nested-Loop Join、Index Nested-Loop Join 或 Hash Join 等多种算法。不同算法在 I/O 次数、内存占用、索引依赖与 CPU 开销上差异显著，直接决定查询延迟与资源消耗。

本文从 SQL 标准中的 JOIN 逻辑语义出发，区分逻辑计划与物理计划，依次剖析 MySQL 中各类 JOIN 算法的实现机制、驱动表选择原则、多表 JOIN 顺序优化，以及子查询与 JOIN 的等价改写。文中示例基于如下测试表结构，数据量级假设为：`users` 约 10 万行，`orders` 约 500 万行，`departments` 约 100 行。

```sql
CREATE TABLE users (
    id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    dept_id    INT UNSIGNED    NOT NULL,
    name       VARCHAR(64)     NOT NULL,
    email      VARCHAR(128)    NOT NULL,
    status     TINYINT         NOT NULL DEFAULT 1,
    created_at DATETIME        NOT NULL,
    PRIMARY KEY (id),
    KEY idx_dept_status (dept_id, status),
    KEY idx_status_created (status, created_at)
) ENGINE=InnoDB;

CREATE TABLE orders (
    id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id    BIGINT UNSIGNED NOT NULL,
    amount     DECIMAL(12, 2)  NOT NULL,
    status     TINYINT         NOT NULL DEFAULT 0,
    created_at DATETIME        NOT NULL,
    remark     VARCHAR(255)    DEFAULT NULL,
    PRIMARY KEY (id),
    KEY idx_user_id (user_id),
    KEY idx_status_created (status, created_at)
) ENGINE=InnoDB;

CREATE TABLE departments (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(64)  NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB;
```

分析 JOIN 性能时，应结合 `EXPLAIN FORMAT=TRADITIONAL` 观察 `type`、`key`、`rows`、`filtered` 与 `Extra` 字段；MySQL 8.0.18+ 可使用 `EXPLAIN ANALYZE` 获取实际执行时间与循环次数。必要时开启 `optimizer_trace` 查看优化器对 JOIN 顺序与算法的选择依据。

## 一、JOIN 的逻辑语义与物理执行

### 1. SQL 标准中的 JOIN 语义

SQL 标准（SQL-92 及后续版本）将 JOIN 定义为两个关系（表）在连接谓词约束下的组合。连接谓词通常出现在 `ON` 子句中，过滤谓词出现在`WHERE` 子句中。标准 JOIN 类型及其逻辑语义如下：

| JOIN 类型          | 逻辑语义        | 结果集特征                       |
|------------------|-------------|-----------------------------|
| CROSS JOIN       | 笛卡尔积，无连接条件  | 行数 = 左表行数 × 右表行数            |
| INNER JOIN       | 仅保留连接条件匹配的行 | 两表键匹配的行对                    |
| LEFT OUTER JOIN  | 保留左表全部行     | 右表无匹配时右列填 NULL              |
| RIGHT OUTER JOIN | 保留右表全部行     | 左表无匹配时左列填 NULL              |
| FULL OUTER JOIN  | 保留两表全部行     | MySQL 不支持原生 FULL OUTER JOIN |

以下 INNER JOIN 在逻辑上描述：两表 `user_id` 相等且 `orders.status = 1` 的行构成结果集。

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.status = 1;
```

逻辑语义只回答"结果应该是什么"，并不规定"如何得到结果"。显式 JOIN、隐式 JOIN（逗号 + WHERE）优化器通常解析为相同结构；`IN`子查询属于半连接语义，去重行为与 INNER JOIN 略有差异。

LEFT JOIN 需特别注意：`ON` 条件不过滤左表行，`WHERE` 中对右表列的过滤会将外连接转化为内连接。应将过滤条件放入 `ON`：

```sql
SELECT u.name, o.amount FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.status = 1;  -- 正确
-- WHERE o.status = 1 会过滤掉 o 为 NULL 的行，等价于 INNER JOIN
```

MySQL 将 `RIGHT JOIN` 转换为等价的 `LEFT JOIN`。`NATURAL JOIN` 可读性差且易因 schema 变更引入隐患，实践中应避免。

### 2. 逻辑计划 vs 物理计划

查询优化分为两个层次：**逻辑优化**在关系代数层面做等价变换，**物理优化**为每个逻辑操作选择具体算法与访问路径。

**逻辑计划**描述"做什么"：Scan(users) → Scan(orders) → Filter(join_condition) → Filter(where_predicate) → Project(columns)。逻辑计划不关心数据在磁盘上的布局，也不指定使用哪种 JOIN 算法。

**物理计划**描述"怎么做"：选择驱动表与被驱动表、决定 JOIN 算法（NLJ / BNL / INLJ / Hash Join）、为每表选择访问方式（全表扫描 / 索引 range / ref / eq_ref）。物理计划直接对应 `EXPLAIN` 输出。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.dept_id = 10 AND o.status = 1;
```

典型输出：

```
+----+-------------+-------+------+-----------------+-----------------+---------+-----------+------+----------+-------------+
| id | select_type | table | type | possible_keys   | key             | key_len | ref       | rows | filtered | Extra       |
+----+-------------+-------+------+-----------------+-----------------+---------+-----------+------+----------+-------------+
|  1 | SIMPLE      | u     | ref  | idx_dept_status | idx_dept_status | 4       | const     |  500 |   100.00 | Using where |
|  1 | SIMPLE      | o     | ref  | idx_user_id     | idx_user_id     | 8       | db.u.id   |   50 |    10.00 | Using where |
+----+-------------+-------+------+-----------------+-----------------+---------+-----------+------+----------+-------------+
```

解读要点：两行 `id` 均为 1，表示同一 JOIN 层级，**先读 `u`（驱动表），再读 `o`（被驱动表）**；`o.type=ref` 且 `ref=db.u.id` 表示**Index Nested-Loop Join**——对 `users` 每一行的 `id`，在 `orders.user_id` 索引上做等值查找。

逻辑等价的不同 SQL 写法，物理计划可能截然不同。连接列被函数包裹时，索引无法用于 JOIN 探测，几乎必然退化为全表扫描：

```sql
SELECT u.name, o.amount
FROM users u
JOIN orders o ON MD5(CAST(u.id AS CHAR)) = MD5(CAST(o.user_id AS CHAR));
```

### 3. MySQL 的 JOIN 执行框架

MySQL 8.0.20 起的服务端执行器采用 **迭代器模型（Volcano Iterator Model）**：每个物理操作符通过初始化与逐行读取接口向上层提供行；上层算子按需拉取下层结果。JOIN 算子位于表扫描算子之上，负责将两个输入流按连接条件合并。

InnoDB 存储引擎下，MySQL 优化器为 JOIN 节点可选择的物理算法包括：

| 算法                      | 引入版本    | 核心机制                | EXPLAIN Extra 标识                        |
|-------------------------|---------|---------------------|-----------------------------------------|
| Simple Nested-Loop Join | 早期      | 双层循环，内层全表扫描         | 无特殊标识，`type=ALL`                        |
| Block Nested-Loop Join  | 8.0.19 及以前 | Join Buffer 批量缓存驱动行 | `Using join buffer (Block Nested Loop)` |
| Index Nested-Loop Join  | 5.x     | 内层通过 B+ 树索引查找       | 被驱动表 `type=ref/eq_ref`                  |
| Hash Join               | 8.0.18+ | 构建哈希表 + 探测          | 8.0.20+ 可在传统格式中显示 `Using join buffer (hash join)` |

优化器在 **join_optimization** 阶段，基于统计信息、可用索引、`join_buffer_size`、连接类型与连接条件形态，估算各候选计划成本。简化成本模型：

```
总成本 ≈ 驱动表扫描成本 + 驱动表有效行数 × 被驱动表单次查找成本 + Join Buffer / Hash Table 内存成本
```

MySQL 8.0 的 `EXPLAIN FORMAT=TREE` 以树形结构展示 JOIN 算子嵌套：

```sql
EXPLAIN FORMAT=TREE
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id WHERE u.dept_id = 10;
```

```
-> Nested loop inner join
    -> Filter: (u.dept_id = 10)
        -> Index lookup on u using idx_dept_status
    -> Index lookup on o using idx_user_id
        -> Index condition: (o.user_id = u.id)
```

## 二、Simple Nested-Loop Join

### 1. 算法伪代码

Simple Nested-Loop Join（SNL）是最直观的 JOIN 实现，也是其他 NLJ 变体的基础：

```
algorithm SimpleNestedLoopJoin(outer_table, inner_table, join_condition):
    for each row r in outer_table:                // 驱动表，M 行
        for each row s in inner_table:            // 被驱动表，N 行
            if join_condition(r, s):
                emit (r, s)
```

SNL 不涉及任何内存结构或索引，内层循环始终对被驱动表做完整扫描（`type=ALL`）。

### 2. 时间复杂度分析

设驱动表行数为 M，被驱动表行数为 N，连接条件匹配率为 p。

| 指标       | Simple NLJ           |
|----------|----------------------|
| 内层扫描次数   | M 次（每次全表扫描 N 行）      |
| 比较次数（最坏） | M × N                |
| I/O（冷缓存） | M 次被驱动表全表读 + 1 次驱动表读 |
| 空间复杂度    | O(1)                 |

数值示例：M = 1,000，N = 5,000,000，无索引。

```
总比较次数 ≈ 1,000 × 5,000,000 = 50 亿次
被驱动表全表扫描次数 = 1,000 次
```

对比 INLJ：M = 1,000，若索引树高约为 3～4 层，每次查找还需读取匹配记录，总页访问量通常远低于反复全表扫描；具体次数受缓存命中率和每个键的匹配行数影响。

### 3. 为什么在实践中很少使用

SNL 在工业级数据库中几乎不会作为大表 JOIN 的首选，原因如下：

- **I/O 放大**：外层每行触发内层一次全表扫描，M 次完整遍历造成严重的缓存污染。
- **无法利用索引**：内层循环是顺序扫描，不会自动转为 B+ 树点查。
- **并发恶化**：长时间占用 CPU 与 Buffer Pool，挤压其他会话。
- **优化器默认规避**：对无索引的连接，旧版本可能使用 BNL；MySQL 8.0.20+ 会在原本使用 BNL 的位置改用 Hash Join。SNL 更适合作为理解嵌套循环代价的概念模型，而非现代版本中可稳定复现的独立执行标识。

### 4. MySQL 什么时候退化为 Simple NLJ

现代 MySQL 中不应仅凭 `type=ALL`、也不应通过把 `join_buffer_size` 设为 0 来判断或复现 SNL：该参数有最小值，且 MySQL 8.0.20+ 已不再使用 BNL。连接列发生隐式类型转换、缺少可用索引或谓词无法形成索引访问路径时，仍可能产生代价很高的全表扫描；应以 `EXPLAIN ANALYZE` 的实际循环次数和读取行数判断问题，而不是给它贴上 SNL 的标签。

## 三、Block Nested-Loop Join（BNL）

### 1. Join Buffer 的引入

Simple NLJ 的核心缺陷在于被驱动表被 **重复全表扫描 M 次**。Block Nested-Loop Join（BNL）引入 **Join Buffer**——一块在 JOIN 执行期间临时分配的内存区域，用于批量缓存驱动表的行，从而将被驱动表扫描次数从 M 次降低至 ⌈M / batch_size⌉ 次。

Join Buffer 通常缓存 **参与 JOIN 的列**（而非整行），以在有限内存内容纳更多行组合。每个可缓冲的 JOIN 都可能分配一个 Buffer，查询结束后释放；因此并发环境中调大该参数需评估总内存消耗。

### 2. BNL 的工作流程

BNL 分两个阶段执行：

**阶段一：将驱动表批量读入 Join Buffer**

```
load batch of rows from outer_table into join_buffer
// 重复直到驱动表全部处理完毕
```

**阶段二：一次扫描被驱动表，与 Buffer 中所有行比较**

```
for each row s in inner_table:
    for each row r in join_buffer:
        if join_condition(r, s): emit (r, s)
```

ASCII 示意（驱动表 1000 行，Join Buffer 容纳 250 行，被驱动表 500 万行）：

```
Batch 1: users[1..250]   → Join Buffer → 全表扫描 orders 500 万行
Batch 2: users[251..500] → Join Buffer → 再次全表扫描 orders
...
被驱动表总扫描次数 = 4 次（而非 SNL 的 1000 次）
```

示例 EXPLAIN（假设 `orders.user_id` 无索引）：

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id WHERE u.dept_id = 10;
```

```
| table | type | key             | rows    | Extra                                 |
|-------|------|-----------------|---------|---------------------------------------|
| u     | ref  | idx_dept_status | 500     | NULL                                  |
| o     | ALL  | NULL            | 5000000 | Using join buffer (Block Nested Loop) |
```

在仍支持 BNL 的版本中，`Using join buffer (Block Nested Loop)` 是其明确标识；被缓冲表的 `type` 可以是 `ALL`、`index` 或 `range`，不局限于 `ALL`。

### 3. join_buffer_size 参数

`join_buffer_size` 控制单次 Join Buffer 的最大字节数，会话级与全局级均可设置：

```sql
SHOW VARIABLES LIKE 'join_buffer_size';   -- 典型默认 262144（256 KB）
SET SESSION join_buffer_size = 8 * 1024 * 1024;  -- 8 MB
```

| 场景       | 建议值           | 说明                     |
|----------|---------------|------------------------|
| OLTP 默认  | 256 KB ~ 1 MB | 大多数 INLJ 场景不需要大 Buffer |
| 报表 / 批处理 | 4 MB ~ 32 MB  | 无索引大表 JOIN 时可减少扫描轮次    |
| 内存紧张     | 保持默认          | 过大 Buffer 可能导致 OOM     |

估算：`batch_size ≈ join_buffer_size / Buffer 中行组合的宽度`。行组合约 100 字节、Buffer 256 KB 时，batch_size 约为 2621。`join_buffer_size` 同时也是 Hash Join 可使用内存的上限；Hash Join 的 Buffer 会按需增量分配。

### 4. BNL 的性能特点

**优势**：将被驱动表扫描次数从 M 降至 ⌈M/B⌉。M = 10,000、B = 2,500 时，扫描次数从 10,000 降至 4。

**劣势**：

- 被驱动表 **仍然是全表扫描**，每次扫描仍需读取全部页。
- Buffer 中逐行比较，CPU 消耗显著；Hash Join 通过 O(1) 探测通常更优。
- BNL 面向没有可用索引访问路径的场景；即使被缓冲表是索引扫描或范围扫描，也不等同于通过连接键进行 INLJ 探测。

BNL 是历史版本中的算法：在 MySQL 8.0.17 及以前，它是无可用连接索引的等值 JOIN 的主要备选；8.0.18、8.0.19 中同类等值 JOIN 可使用 Hash Join。

### 5. MySQL 8.0.20 起 BNL 被 Hash Join 替代

MySQL 8.0.20 起，服务器**移除了 BNL**：凡是旧版本会使用 BNL 的位置，均改由 Hash Join 处理。Hash Join 的覆盖范围也扩展到非等值 INNER JOIN、半连接、反连接及左右外连接；非等值条件可在 Hash Join 后作为过滤条件执行。

```sql
-- MySQL 8.0.20+，无索引等值 INNER JOIN
EXPLAIN SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id;
-- 8.0.17 及以前：Extra: Using join buffer (Block Nested Loop)
-- 8.0.20 之后：Extra: Using join buffer (hash join)
```

生产环境中若在 8.0.20+ 看到 `Using join buffer (hash join)`，应优先检查是否缺少更合适的连接索引、驱动侧过滤是否不足以及成本估算是否准确，而非单纯增大 `join_buffer_size`。

## 四、Index Nested-Loop Join（INLJ）

### 1. 被驱动表上有合适索引

Index Nested-Loop Join（INLJ）在 SNL 结构上增加关键优化：**对驱动表每一行，利用被驱动表连接列上的 B+ 树索引做等值查找**，而非全表扫描。INLJ 是 OLTP 场景下最高效的 JOIN 形态。

前提条件：被驱动表连接列有可用索引；连接条件为等值（`=` / `<=>`）；连接列类型一致、无隐式转换；优化器估算索引查找成本更低。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT u.name, o.id, o.amount FROM users u JOIN orders o ON u.id = o.user_id WHERE u.status = 1;
```

```
| table | type | key               | ref       | rows  | Extra |
|-------|------|-------------------|-----------|-------|-------|
| u     | ref  | idx_status_created| const     | 33333 | Using where |
| o     | ref  | idx_user_id       | db.u.id   | 50    | NULL  |
```

`o.type=ref`，`ref=db.u.id`——INLJ 的标志性输出，`Extra` 中 **不出现** Join Buffer 字样。

### 2. 每次通过索引查找而非全表扫描

INLJ 伪代码：

```
algorithm IndexNestedLoopJoin(outer, inner, join_key):
    for each row r in outer:                              // M 行
        matches = BTreeLookup(inner, join_key, r.key)     // O(log N)
        for each row s in matches:
            if remaining_conditions(r, s): emit (r, s)
```

联合索引可同时服务连接条件与过滤条件：

```sql
CREATE INDEX idx_user_status ON orders (user_id, status);
EXPLAIN
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id AND o.status = 1;
-- o.key=idx_user_status, key_len=9, Extra: Using index condition
```

`Using index condition` 表示 Index Condition Pushdown（ICP）在存储引擎层过滤 `status = 1`，减少回表行数。

### 3. 性能分析

设驱动表有效行数 M，被驱动表行数 N，B+ 树高度为 h（通常接近 `log_B(N)`，其中 B 为页扇出）。

| 指标       | INLJ            |
|----------|-----------------|
| 被驱动表访问   | M 次索引查找         |
| 单次查找复杂度  | O(h) ≈ O(log N) |
| 总页访问（估算） | 约 M ×（索引树高 + 匹配记录访问成本） |

数值对比（M = 500，N = 5,000,000，h ≈ 4，每用户平均 5 笔订单）：

```
INLJ：索引查找 500 次，页访问 ≈ 2000 次，输出 ≈ 2500 行
SNL：比较次数 ≈ 500 × 5,000,000 = 25 亿
```

当被驱动表有选择性良好的连接索引、且驱动侧结果集较小时，INLJ 通常优于 Hash Join——无需额外构建哈希表，也可利用索引范围访问。若驱动侧行数很大或索引回表代价很高，仍应以实际计划和执行数据为准。连接键为唯一索引或主键时，`type=eq_ref`，每驱动行最多匹配一行。

### 4. 什么条件下优化器会选择 INLJ

优化器选择 INLJ 需同时满足：

1. 被驱动表存在连接键索引，且 `possible_keys` 包含该索引。
2. 等值连接条件可直接映射到索引列。
3. 成本估算：`M × index_lookup_cost < full_scan_cost + join_buffer_cost`。
4. 连接类型允许：INNER / LEFT / SEMI JOIN 均支持。
5. 若连接索引的估算成本低于扫描并构建哈希表的成本，优化器倾向于选择 INLJ；Hash Join 常见于没有可用连接索引的场景，但并非由“有索引”这一条件单独决定。

常见导致退化的因素：类型不一致、OR 连接条件、统计信息过时。定期 `ANALYZE TABLE orders` 更新索引基数。

### 5. EXPLAIN 中的表现

INLJ 识别 checklist：

| 特征          | 说明                        |
|-------------|---------------------------|
| 被驱动表 `type` | `ref`、`eq_ref` 或 `range`  |
| 被驱动表 `key`  | 非 NULL，为连接键相关索引           |
| 被驱动表 `ref`  | 指向驱动表列，如 `db.u.id`        |
| `Extra`     | **无** `Using join buffer` |

三表 JOIN 示例：

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT d.name, u.name, o.amount FROM departments d
JOIN users u ON d.id = u.dept_id JOIN orders o ON u.id = o.user_id WHERE d.id = 10;
```

```
| table | type  | key               | ref       |
|-------|-------|-------------------|-----------|
| d     | const | PRIMARY           | const     |
| u     | ref   | idx_dept_status   | const     |
| o     | ref   | idx_user_id       | db.u.id   |
```

`EXPLAIN ANALYZE` 可验证实际循环次数：

```sql
EXPLAIN ANALYZE
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id WHERE u.dept_id = 10;
-- 内层 Index lookup loops=500（驱动表 500 行）
```

## 五、Hash Join（MySQL 8.0.18+）

### 1. Hash Join 的工作流程

MySQL 8.0.18 增加 Hash Join，最初主要用于每个连接对都含等值条件且无可用连接索引的 JOIN。MySQL 8.0.20 起，它还可用于非等值 INNER JOIN、外连接、半连接与反连接。通过哈希表实现 O(1) 平均探测，避免 BNL 的 O(batch_size × N) 逐行比较。

**Build 阶段**：优化器根据成本选择一个输入作为 Build 侧，通常会倾向于估计结果较小的一侧；按可哈希的连接键构建哈希表（键 → 行链表，处理冲突）。

**Probe 阶段**：顺序扫描 Probe 侧（较大输入），计算哈希值在哈希表中查找匹配项。

```
algorithm HashJoin(build_table, probe_table, join_key):
    for each row r in build_table: hash_table.insert(hash(r.key), r)
    for each row s in probe_table:
        for each row r in hash_table.lookup(hash(s.key)):
            if r.key == s.key: emit (r, s)
```

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id;
```

MySQL 8.0.20+ 典型输出：

```
| table | type | rows    | Extra                           |
|-------|------|---------|---------------------------------|
| u     | ALL  | 100000  | NULL                            |
| o     | ALL  | 5000000 | Using join buffer (hash join)   |
```

`EXPLAIN FORMAT=TREE` 输出：

```
-> Inner hash join (o.user_id = u.id)
    -> Table scan on o
    -> Hash
        -> Table scan on u
```

Build 侧为 `users`（10 万行），Probe 侧为 `orders`（500 万行）。

### 2. 内存中的 Hash Join

当 Build 侧所需哈希表可放入 `join_buffer_size` 限制的内存时，Hash Join 完全在内存中完成：

```
内存需求 ≈ Build 侧行数 × 每行哈希表占用 + 哈希表 overhead
```

Build 侧 10 万行、每行约 50 字节 → 内存约 5 MB。若 `join_buffer_size = 8 MB`，内存 Hash Join 可行。Probe 阶段顺序扫描大表，对 Buffer Pool 友好，CPU 远低于 BNL。

### 3. 磁盘溢出的 Hash Join

所需内存超过上限时，MySQL 会使用磁盘文件完成 Hash Join，性能可能明显下降；若打开的文件数超过 `open_files_limit`，查询甚至可能失败。`Created_tmp_disk_tables` 不能作为 Hash Join 溢出的专用指标，应结合 `EXPLAIN ANALYZE`、慢查询与服务器资源指标诊断。优先评估连接索引、过滤条件和 `join_buffer_size`，而不是只依赖增大内存。

### 4. Hash Join 的适用场景

| 场景                      | 是否适合 Hash Join |
|-------------------------|----------------|
| INNER JOIN + 等值条件       | 8.0.18+ 适合      |
| 大表关联，至少一侧无合适索引          | 适合             |
| 被驱动表有高效索引               | 通常优先比较 INLJ 成本 |
| LEFT / RIGHT OUTER JOIN | 8.0.20+ 可适用，受外连接语义与成本约束 |
| 非等值连接                   | 8.0.20+ 可适用；条件作为连接后的过滤执行 |
| 极小表                     | 优化器可能选 INLJ    |

```sql
-- 两表均无连接列索引
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id
WHERE u.created_at >= '2025-01-01' AND o.created_at >= '2025-01-01';

-- 为 orders.user_id 加索引后通常切换为 INLJ
CREATE INDEX idx_user_id ON orders (user_id);
```

### 5. Hash Join 的限制

- **版本边界**：8.0.18、8.0.19 要求每个连接对至少有一个等值条件；8.0.20+ 放宽此限制并支持外连接、半连接与反连接。
- **索引不一定无效**：Hash Join 更常见于没有可用连接索引时；单表过滤仍可使用索引，且有高效连接索引时优化器可能改选嵌套循环。
- **内存依赖**：Build 侧过大 + 小 `join_buffer_size` 导致磁盘溢出，性能可能差于索引优化后的 INLJ。
- **表达式与类型兼容性**：不兼容的字符集、排序规则或隐式转换可能改变可用访问路径；8.0.20+ 不会因此退回 BNL，而应由 `EXPLAIN` 确认最终计划。

```sql
EXPLAIN FORMAT=TREE SELECT u.name, o.amount FROM users u JOIN orders o ON u.id > o.user_id;
-- MySQL 8.0.20+ 可以显示无等值条件的 Inner hash join，并在其上过滤 u.id > o.user_id
```

## 六、驱动表的选择

### 1. 优化器如何决定驱动表

对于两表 JOIN，优化器可在 A JOIN B 与 B JOIN A 之间选择；N 表 JOIN 时顺序数量阶乘增长。优化器通过 **基于成本的搜索**选择驱动表顺序，决策依据包括：

- 各表过滤后的有效行数：`rows × filtered / 100`
- 被驱动表能否 INLJ
- INLJ、BNL、Hash Join 的相对代价
- 连接类型约束：LEFT JOIN 左表必须是驱动表

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id WHERE u.dept_id = 10;
-- u 排在 o 之上，o.ref=db.u.id → u 为驱动表
```

### 2. 小表驱动大表的原理

"小表驱动大表"的准确含义：**让过滤后行数较少的一侧作为驱动表，让行数较多的一侧作为被驱动表，并在内层尽可能走索引**。"小"与"大"指 **有效行数**，而非物理存储大小。

INLJ 成本对比（M = 500 驱动小表，N = 5,000,000 被驱动大表有索引）：

```
成本(A) ≈ 500 × (index_lookup + 5 × row_cost) ≈ 8 ms
成本(B) ≈ 5,000,000 × (index_lookup + row_cost) ≈ 60,000 ms
```

反例：大表有强过滤时，优化器可能让 filtered 后行数少的大表驱动：

```sql
EXPLAIN SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id WHERE o.id = 12345;
-- o 主键点查仅 1 行，o 可能作为驱动表
```

### 3. STRAIGHT_JOIN 强制指定驱动表

确认优化器选错 JOIN 顺序时，可使用 `STRAIGHT_JOIN` 强制按 `FROM` 子句顺序确定驱动关系：

```sql
EXPLAIN
SELECT u.name, o.amount FROM users u
STRAIGHT_JOIN orders o ON u.id = o.user_id WHERE u.dept_id = 10;
```

使用注意：仅在有充分证据（`EXPLAIN ANALYZE`、`optimizer_trace`）表明优化器选错时使用；数据分布变化后需定期复查；不可滥用替代索引优化。

### 4. LEFT JOIN 对驱动表的约束

LEFT OUTER JOIN 要求 **左表必须作为驱动表（外层）**，优化器不会交换 LEFT JOIN 左右表（除非转为 RIGHT JOIN 等价形式）。

```sql
EXPLAIN SELECT u.name, o.amount FROM users u LEFT JOIN orders o ON u.id = o.user_id;
-- u 始终先扫描（驱动表）
```

若误写为 `orders LEFT JOIN users`，驱动表变为 500 万行的 `orders`——通常极慢。INNER JOIN 无此约束，优化器可自由交换顺序。

## 七、多表 JOIN 的优化

### 1. JOIN 顺序的优化

N 个表 INNER JOIN 在不考虑等价剪枝时，可能的表顺序有 **N!** 种，N = 5 时已有 120 种排列。优化器采用成本搜索，受`optimizer_search_depth`（默认 62）限制：

```sql
SHOW VARIABLES LIKE 'optimizer_search_depth';
```

表数量 ≥ 5 时，应警惕优化器得不到全局最优顺序。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT d.name, u.name, o.amount FROM departments d
JOIN users u ON d.id = u.dept_id JOIN orders o ON u.id = o.user_id
WHERE d.id = 10 AND u.status = 1 AND o.status = 1;
-- 理想顺序：d(1行) → u(500行) → o(索引查找)
```

### 2. 贪心算法与动态规划

MySQL 优化器会在搜索深度允许时评估更多候选顺序；当候选空间过大时，会使用基于 `optimizer_prune_level` 等配置的启发式剪枝。具体是否穷举及搜索范围受表数、连接约束和优化器参数共同影响，不宜把固定表数阈值或“每次选两表”的过程当作保证。

人工干预手段：调整 `FROM` 子句表顺序、`STRAIGHT_JOIN`、派生表预过滤。

```sql
SELECT d.name, t.user_name, t.total_amount FROM departments d
JOIN (
    SELECT u.dept_id, u.name AS user_name, SUM(o.amount) AS total_amount
    FROM users u JOIN orders o ON u.id = o.user_id
    WHERE u.status = 1 AND o.status = 1 GROUP BY u.id, u.dept_id, u.name
) t ON d.id = t.dept_id WHERE d.id = 10;
```

### 3. 减少 JOIN 表数量

**业务层拆分**：超过 3~4 张表的 JOIN 在 OLTP 中应引起警觉，拆为 ID 列表查询 + 批量 IN 查询。**反范式设计**：高频读场景冗余关联列（如`orders.user_name`），避免 JOIN，但需维护一致性。

## 八、子查询 vs JOIN

### 1. 相关子查询的性能陷阱

**相关子查询**引用外层列，语义上对外层 **每一行** 执行一次内层查询。若未改写为 Semi-Join 或 Materialization，复杂度类似 SNL：

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT u.name FROM users u
WHERE (SELECT MAX(o.amount) FROM orders o WHERE o.user_id = u.id) > 1000;
```

```
| id | select_type        | table | type | rows   |
|----|--------------------|-------|------|--------|
|  1 | PRIMARY            | u     | ALL  | 100000 |
|  2 | DEPENDENT SUBQUERY | o     | ref  | 50     |
```

`DEPENDENT SUBQUERY` 意味着 10 万次子查询执行，即使每次 `ref` 索引查找仅 0.1ms，总耗时也达秒级。

### 2. 优化器的子查询优化

MySQL 5.6+ 对 IN / EXISTS 提供 **Semi-Join** 优化，常见策略：

| 策略               | 说明               | 适用场景          |
|------------------|------------------|---------------|
| FirstMatch       | 找到首个匹配即停止        | IN / EXISTS   |
| Materialization  | 子查询物化到临时表再 JOIN  | 非相关子查询        |
| DuplicateWeedout | JOIN 后去重         | 半连接语义         |
| LooseScan        | 索引松散扫描           | DISTINCT 列有索引 |
| Hash Join        | 物化表与外表 Hash Join | 8.0.18+ 大结果集  |

```sql
EXPLAIN SELECT u.name FROM users u WHERE u.id IN (SELECT o.user_id FROM orders o WHERE o.status = 1);
-- Extra 可能出现 Start temporary; End temporary 或 Semi-join 相关标识
```

```sql
SHOW VARIABLES LIKE 'optimizer_switch';  -- semi_join=on, materialization=on
```

### 3. 什么时候用子查询更好

| 需求       | 推荐写法          | 原因                               |
|----------|---------------|----------------------------------|
| 仅判断存在性   | `EXISTS`      | Semi-Join 友好，找到即停                |
| 集合成员检测   | `IN (非相关子查询)` | Materialization 或 Hash Semi-Join |
| 多列投影、聚合  | `JOIN`        | 避免标量子查询逐行执行                      |
| 每用户聚合后过滤 | 派生表 `JOIN`    | 一次 GROUP BY                      |
| 反连接（不存在） | `NOT EXISTS`  | 优于 `NOT IN`（NULL 安全）             |

`NOT IN` 在子查询列含 NULL 时结果可能为空集（三值逻辑陷阱），应使用 `NOT EXISTS`：

```sql
-- 危险
SELECT u.name FROM users u WHERE u.id NOT IN (SELECT user_id FROM orders);
-- 安全
SELECT u.name FROM users u WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

### 4. 具体改写示例

**相关标量子查询 → JOIN + 派生表：**

```sql
-- 改写前
SELECT u.name FROM users u
WHERE (SELECT MAX(o.amount) FROM orders o WHERE o.user_id = u.id) > 1000;

-- 改写后
SELECT u.name FROM users u
JOIN (SELECT user_id, MAX(amount) AS max_amount FROM orders GROUP BY user_id) t
  ON u.id = t.user_id WHERE t.max_amount > 1000;
```

**IN 子查询 → INNER JOIN：**

```sql
-- 改写前
SELECT u.name FROM users u
WHERE u.dept_id IN (SELECT id FROM departments WHERE name LIKE 'Engineering%');

-- 改写后
SELECT u.name FROM users u
JOIN departments d ON u.dept_id = d.id WHERE d.name LIKE 'Engineering%';
```

**EXISTS → JOIN（半连接等价）：** 若确实需要改写，可使用 `SELECT DISTINCT u.name FROM users u INNER JOIN orders o ON u.id = o.user_id WHERE ...` 保持“存在即可”的去重语义。但 MySQL 已能将许多 `EXISTS` 转换为半连接，改写并不必然更快；应比较两种写法的 `EXPLAIN ANALYZE`，并为 `orders` 的连接键和过滤列设计联合索引。

无论哪种写法，都应以 `EXPLAIN` / `EXPLAIN ANALYZE` 验证是否消除 `DEPENDENT SUBQUERY`。

## 九、JOIN 优化的实战建议

### 1. 优先为被驱动表的连接键建立索引

在典型 OLTP JOIN 中，INLJ 常是高效路径；被驱动表连接列上的索引通常是第一优先级。对于低选择性、分析型或外连接查询，仍需以成本和实际执行情况决定是否使用该索引。

```sql
CREATE INDEX idx_user_id ON orders (user_id);
CREATE INDEX idx_user_status_created ON orders (user_id, status, created_at);

EXPLAIN
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id
WHERE o.status = 1 AND o.created_at >= '2026-06-01';
```

验证 checklist：被驱动表 `type` 为 `ref`/`eq_ref`；`key` 为预期索引；`SHOW WARNINGS` 检查隐式转换；`ANALYZE TABLE orders`保持统计信息新鲜。

### 2. 控制驱动表的结果集大小

驱动表有效行数直接乘以被驱动表查找次数，通过 WHERE 谓词 **尽早过滤** 驱动表：

```sql
-- 优：多条件缩小 users 参与 JOIN 的行数
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id
WHERE u.dept_id = 10 AND u.status = 1 AND o.status = 1;

-- 劣：users 全表驱动
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id WHERE o.status = 1;
```

分页场景避免大 OFFSET 驱动大 JOIN，采用延迟关联：

```sql
-- 优：先查 orders ID 再 JOIN users
SELECT u.name, t.amount FROM (
    SELECT user_id, amount FROM orders WHERE status = 1
    ORDER BY created_at DESC LIMIT 100000, 20
) t JOIN users u ON u.id = t.user_id;
```

### 3. 避免 SELECT * 减少 Join Buffer 压力

宽表 JOIN 会放大 Join Buffer 占用、回表与网络传输。对于历史 BNL，Buffer 可容纳的行组合数与行宽成反比；对于 Hash Join，行宽也会增加构建哈希表所需内存：

```sql
-- 优：仅必要列
SELECT u.id, u.name, o.id, o.amount, o.created_at FROM users u
JOIN orders o ON u.id = o.user_id WHERE u.dept_id = 10;
```

覆盖索引进一步减少回表：

```sql
CREATE INDEX idx_user_amount_created ON orders (user_id, amount, created_at);
-- Extra 可能出现 Using index
```

### 4. 监控全表扫描与 Hash Join

生产环境应监控全表扫描与 Hash Join 出现频率——它们可能意味着缺少适合的连接索引、过滤不足或统计信息失真；分析型查询中出现 Hash Join 不必然是问题。

关注慢查询日志中 `Rows_examined` 与 `Query_time` 比值异常的 JOIN；通过 Performance Schema 的`events_statements_summary_by_digest` 定位高频慢 JOIN；监控 `Select_scan` 与 `Created_tmp_disk_tables` 状态变量。

在 MySQL 8.0.20+ 中，若 `EXPLAIN` 出现 `Using join buffer (hash join)`，先评估连接列索引和驱动侧过滤能否降低成本；旧版本的 `Block Nested Loop` 也应按同一方向排查。出现`DEPENDENT SUBQUERY` 时，先确认优化器不能改写的原因，再考虑 JOIN 或派生表；驱动表 `rows` 过大则加强 WHERE 过滤并 `ANALYZE TABLE`。

## 总结

JOIN 的性能并非由 SQL 表面结构单独决定，而是逻辑语义、优化器代价模型与物理算法（SNL、BNL、INLJ、Hash Join）共同作用的结果。

**逻辑与物理的分工。** JOIN 的逻辑语义由 SQL 标准定义结果集；优化器将其映射为 Nested-Loop 系列或 Hash Join 等物理路径。同一 SQL 的多种写法可能逻辑等价但物理计划截然不同，必须以 `EXPLAIN`、`EXPLAIN ANALYZE` 与 `optimizer_trace` 验证实际路径。

**四类物理算法的选择逻辑。**

- **Simple NLJ**：内外双层循环，被驱动表重复全表扫描，大表场景应极力避免。
- **Block Nested-Loop Join**：Join Buffer 批量缓存驱动行，扫描次数从 M 降至 ⌈M/batch_size⌉；仅适用于 8.0.20 之前的历史版本。
- **Index Nested-Loop Join**：驱动表每行在被驱动表连接列索引上查找，典型查找成本接近 O(M × log_B N)，是常见的 OLTP 高效路径。
- **Hash Join**（8.0.18+）：构建侧哈希表 + 探测侧扫描，常用于无可用连接索引的等值 JOIN；8.0.20+ 还可处理非等值 INNER JOIN、外连接、半连接和反连接。

**驱动表与多表顺序。** 让过滤后有效行数少的一侧驱动行数多且有索引的一侧。LEFT JOIN 左表必须是驱动表；多表 JOIN 受`optimizer_search_depth` 限制，表过多时应拆分查询或反范式。

**子查询与 JOIN 的取舍。** 优化器可将 IN / EXISTS 转为 Semi-Join；相关子查询若未改写则退化为 DEPENDENT SUBQUERY。多列投影与聚合优先 JOIN；存在性检测用 EXISTS；`NOT IN` 注意 NULL 陷阱。

**实战要点：** 优先评估被驱动表连接列索引；通过 WHERE 缩小驱动表结果集；避免 `SELECT *`；结合全表扫描与 Hash Join 分析成本；定期`ANALYZE TABLE`；仅在证据充分时使用 `STRAIGHT_JOIN`。

JOIN 优化是一项需要结合存储结构、统计信息与业务访问模式的持续工作。建立"理解物理路径、度量行数与 I/O、迭代索引与 SQL"的分析习惯，比死记"小表驱动大表"等口号更能应对生产环境中的复杂关联查询。
