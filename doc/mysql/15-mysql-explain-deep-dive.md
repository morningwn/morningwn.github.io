---
title: EXPLAIN 执行计划全解析
summary: 逐列拆解 EXPLAIN 输出的含义，从访问类型、索引使用到 Extra 信息，建立通过执行计划定位 SQL 性能问题的分析能力。
created: 2026-07-02
updated: 2026-07-14
tags: MySQL, EXPLAIN, 执行计划, 性能优化
---

# EXPLAIN 执行计划全解析

优化 SQL 性能时，"感觉慢"与"知道慢在哪里"之间存在巨大鸿沟。`EXPLAIN` 是 MySQL 提供的执行计划查看工具：它不会真正执行查询（`EXPLAIN ANALYZE` 除外），而是让优化器输出对访问路径的估算——读哪些表、走哪条索引、预计扫描多少行、是否需要额外排序或临时表。

本文以 InnoDB 存储引擎为基准，逐列拆解 `EXPLAIN` 输出，建立从执行计划到性能诊断的完整分析框架。文中示例基于如下测试表结构；普通 `EXPLAIN` 默认输出经典表格格式（MySQL 8.0.32+ 可通过 `explain_format` 调整默认格式），本文统一显式使用 `FORMAT=TRADITIONAL`，便于逐列对照。

```sql
CREATE TABLE orders (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id     BIGINT UNSIGNED NOT NULL,
    status      TINYINT         NOT NULL DEFAULT 0,
    amount      DECIMAL(12, 2)  NOT NULL,
    phone       VARCHAR(20)     DEFAULT NULL,
    created_at  DATETIME        NOT NULL,
    updated_at  DATETIME        NOT NULL,
    remark      VARCHAR(255)    DEFAULT NULL,
    PRIMARY KEY (id),
    KEY idx_user_created (user_id, created_at),
    KEY idx_status_amount (status, amount),
    KEY idx_created (created_at),
    KEY idx_phone (phone)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE users (
    id       BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    name     VARCHAR(64)     NOT NULL,
    email    VARCHAR(128)    NOT NULL,
    city     VARCHAR(32)     NOT NULL,
    PRIMARY KEY (id),
    KEY idx_city (city),
    KEY idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

## 一、EXPLAIN 的基本用法

### 1. EXPLAIN 语法

`EXPLAIN` 用于查看 SELECT、DELETE、INSERT、UPDATE 等语句的执行计划。语法为`EXPLAIN [explain_type] {table_name | SELECT statement}`，`explain_type` 指定输出格式：

| 语法                                      | 说明                | 适用版本    |
|-----------------------------------------|-------------------|---------|
| `EXPLAIN SELECT ...`                    | 默认经典表格格式；8.0.32+ 可由 `explain_format` 调整 | 全版本     |
| `EXPLAIN FORMAT=TRADITIONAL SELECT ...` | 经典表格格式，逐列展示       | 5.6+    |
| `EXPLAIN FORMAT=JSON SELECT ...`        | JSON 结构化输出，便于程序解析 | 5.6+    |
| `EXPLAIN FORMAT=TREE SELECT ...`        | 树形文本，展示算子层级       | 8.0.16+ |
| `EXPLAIN ANALYZE SELECT ...`            | 真实执行并收集耗时与行数      | 8.0.18+ |

**经典表格格式：**

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders
WHERE user_id = 10086 AND created_at >= '2026-01-01';
```

**JSON 格式：**

```sql
EXPLAIN FORMAT=JSON
SELECT o.id, o.amount, u.name
FROM orders o JOIN users u ON o.user_id = u.id WHERE o.status = 1;
```

JSON 输出含 `query_block`、`access_type`、`key`、`key_length`、`rows_examined_per_scan`、`filtered`、`cost_info` 等字段，可通过`->>'$.query_block.cost_info.query_cost'` 提取成本。

**TREE 格式：**

```sql
EXPLAIN FORMAT=TREE
SELECT * FROM orders WHERE user_id = 10086 ORDER BY created_at DESC LIMIT 20;
```

TREE 以缩进展示 `-> Limit`、`-> Sort`、`-> Index lookup` 等算子，反映 MySQL 8.0 迭代器执行模型。

`EXPLAIN` 同样适用于 UPDATE/DELETE（不真正修改数据）。配合 `SHOW WARNINGS` 可查看优化器是否将子查询改写为 JOIN 或半连接（Semi-Join）。

### 2. EXPLAIN 输出的列一览

`FORMAT=TRADITIONAL` 输出列如下（分区表另有 `partitions` 列）：

| 列名              | 含义                                   | 分析优先级  |
|-----------------|--------------------------------------|--------|
| `id`            | SELECT 标识符，反映查询块层级与执行顺序              | 中      |
| `select_type`   | SELECT 类型（SIMPLE、PRIMARY、SUBQUERY 等） | 中      |
| `table`         | 表名、派生表别名或 `<unionN>`                 | 高      |
| `partitions`    | 匹配的分区；未分区表为 NULL                     | 低      |
| `type`          | 访问类型，反映如何查找行                         | **最高** |
| `possible_keys` | 优化器评估过的候选索引                          | 中      |
| `key`           | 实际选定的索引                              | **高**  |
| `key_len`       | 使用的索引前缀长度（字节）                        | **高**  |
| `ref`           | 与索引列比较的值或列                           | 中      |
| `rows`          | 估算需要扫描的行数                            | **高**  |
| `filtered`      | 表级条件过滤后剩余行的百分比估算                     | 中      |
| `Extra`         | 附加信息（覆盖索引、排序、临时表等）                   | **高**  |

典型输出：

```
+----+-------------+--------+------+------------------+------------------+---------+-------+------+----------+-------------+
| id | select_type | table  | type | possible_keys    | key              | key_len | ref   | rows | filtered | Extra       |
+----+-------------+--------+------+------------------+------------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | orders | ref  | idx_user_created | idx_user_created | 8       | const |  120 |    33.33 | Using where |
+----+-------------+--------+------+------------------+------------------+---------+-------+------+----------+-------------+
```

解读：`idx_user_created` 做 `ref` 访问，`user_id = 10086`（`ref=const`），预计扫描 120 行，约 33% 满足`created_at >= '2026-01-01'`，其余条件在引擎层或 Server 层过滤（`Using where`）。

## 二、id 与 select_type

### 1. id 列

`id` 标识 SELECT 查询块，帮助理解多表 JOIN、子查询、UNION 的执行顺序。

**相同 `id` = 同一查询层级。** JOIN 中多行共享同一 `id`，`ref=db.u.id` 表示与另一表列关联：

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT o.*, u.name FROM orders o JOIN users u ON o.user_id = u.id WHERE o.status = 1;
```

```
| id | table | type | ref       | 说明                         |
|----|-------|------|-----------|------------------------------|
|  1 | u     | ALL  | NULL      | 同一 SIMPLE 查询             |
|  1 | o     | ref  | db.u.id   | orders.user_id 与 users.id JOIN |
```

**不同 `id`：通常表示不同查询块，传统表格中常先看数值更大的内层查询块。** 但优化器可能把子查询改写为物化表、半连接或 JOIN，`id` 大小不能机械等同于真实执行先后。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders
WHERE amount > (SELECT AVG(amount) FROM orders WHERE status = 1);
```

```
| id | select_type | table  | type | key               | rows |
|----|-------------|--------|------|-------------------|------|
|  1 | PRIMARY     | orders | ALL  | NULL              | 5000 |
|  2 | SUBQUERY    | orders | ref  | idx_status_amount | 1667 |
```

**UNION：** 各分支 `id` 递增，合并行 `id=NULL`、`select_type=UNION RESULT`；非 `UNION ALL` 常 `Using temporary` 去重。

**派生表：** `id=2` 的 DERIVED 先物化子查询为 `<derived2>`，外层 `id=1` 再扫描。

### 2. select_type 列

| select_type          | 含义                | 典型场景                  |
|----------------------|-------------------|-----------------------|
| `SIMPLE`             | 简单查询，不含子查询或 UNION | 单表或简单 JOIN            |
| `PRIMARY`            | 最外层 SELECT        | 含子查询的外层               |
| `SUBQUERY`           | 非关联子查询，独立执行一次     | 标量子查询                 |
| `DEPENDENT SUBQUERY` | 关联子查询，依赖外层当前行     | 外层每行执行一次              |
| `DERIVED`            | FROM 子句中的派生表      | `(SELECT ...) AS t`   |
| `UNION`              | UNION 第二及之后分支     | `UNION` / `UNION ALL` |
| `UNION RESULT`       | UNION 合并结果        | `<union1,2>`          |
| `MATERIALIZED`       | 子查询物化为临时表         | IN 子查询优化（5.6+）        |

**SIMPLE：**

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE id = 1;
-- select_type=SIMPLE
```

**SUBQUERY：** 标量子查询内层 `select_type=SUBQUERY`，通常只执行一次，结果作为 `const` 传给外层。

**DEPENDENT SUBQUERY（性能敏感，应优先改写为 JOIN）：**

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id AND u.city = 'Shanghai');
-- 若未优化为 Semi-Join：select_type=DEPENDENT SUBQUERY，外层每行触发内层

-- 改写：
SELECT o.* FROM orders o INNER JOIN users u ON o.user_id = u.id AND u.city = 'Shanghai';
```

**MATERIALIZED（优于 DEPENDENT SUBQUERY）：**

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders o WHERE o.user_id IN (SELECT id FROM users WHERE city = 'Beijing');
-- 优化器可能将 IN 子查询物化，内层 select_type=MATERIALIZED
```

## 三、type 列：访问类型详解

`type` 是执行计划分析中**最核心**的指标。一般排序（最优 → 最差）：

```
system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > range > index > ALL
```

### 1. system 与 const

**system**：表仅一行，是 `const` 的特殊情况。

**const**：通过主键或唯一索引一次定位**至多一行**，条件为常量等值且索引列未被函数包裹。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE id = 12345;
-- type=const, key=PRIMARY, key_len=8, ref=const, rows=1
```

**失效示例**——对列运算：

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE id + 0 = 12345;
-- type=ALL, key=NULL
```

### 2. eq_ref

JOIN 时，对驱动表每一行，在被驱动表上通过**主键或唯一索引**精确匹配**一行**。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT o.*, u.name FROM orders o JOIN users u ON o.user_id = u.id WHERE o.id = 100;
```

```
| table | type   | key     | ref          | 说明                    |
|-------|--------|---------|--------------|-------------------------|
| o     | const  | PRIMARY | const        | 主键定位一行            |
| u     | eq_ref | PRIMARY | db.o.user_id | 每行精确匹配 users 一行 |
```

被驱动表 JOIN 列无唯一索引时降为 `ref`（一个 user 可能有多条 order）。

### 3. ref

使用**非唯一索引**等值匹配，可能返回多行。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE user_id = 10086;
-- type=ref, key=idx_user_created, key_len=8, ref=const, rows=150
```

联合索引最左前缀等值也属 `ref`：`WHERE user_id = 10086 AND status = 1` 若索引为 `(user_id, created_at)`，则 `key_len=8`（仅 user_id）。

### 4. range

索引**范围扫描**：`>`、`<`、`BETWEEN`、`IN`、前缀 `LIKE 'abc%'` 等。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders WHERE created_at BETWEEN '2026-01-01' AND '2026-01-31';
-- type=range, key=idx_created, key_len=5, rows=850
```

`IN` 列表、前缀 `LIKE 'VIP%'` 均属 `range`；`LIKE '%VIP%'` 通常 `ALL`。`range` 是 OLTP 中最常见且通常可接受的访问类型。

### 5. index

**全索引扫描**，遍历整棵二级索引树。比 `ALL` 略好（索引更窄），但仍读大量索引页。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT user_id FROM orders;
-- type=index, key=idx_user_created, rows=500000, Extra=Using index
```

`Using index` 表示覆盖索引，无需回表；但 `type=index` 仍扫描全表行数的索引项。与 `range` 区别：`range` 只扫满足范围的部分，`index` 扫整棵索引。

### 6. ALL

**全表扫描**，逐行读聚簇索引叶子节点，大数据量下通常最差。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE remark LIKE '%urgent%';
-- type=ALL, key=NULL, rows=500000, filtered=11.11, Extra=Using where
```

小表 `ALL` 可能被优化器主动选择（随机索引查找成本高于顺序全表读）。

### 7. ref_or_null、index_merge、fulltext 等

**ref_or_null**：类似 `ref`，额外包含 `IS NULL` 查找。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE remark IS NULL OR remark = 'VIP';
-- remark 有索引时可能 type=ref_or_null
```

**index_merge**：合并多个索引（Intersection / Union / Sort-Union）。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE user_id = 10086 OR status = 1;
-- type=index_merge, Extra=Using union(idx_user_created,idx_status_amount); Using where
```

**fulltext**：全文索引，`MATCH ... AGAINST` 语法。

```sql
-- CREATE FULLTEXT INDEX ft_remark ON orders(remark);
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE MATCH(remark) AGAINST('urgent payment');
-- type=fulltext
```

### 8. 各类型的性能排序与优化方向

| type          | 扫描特征           | 相对成本 | 优化方向              |
|---------------|----------------|------|-------------------|
| `system`      | 单行表            | 极低   | 无需优化              |
| `const`       | 主键/唯一等值，一行     | 极低   | 确保 WHERE 列有唯一索引   |
| `eq_ref`      | JOIN 被驱动表主键/唯一 | 低    | JOIN 列建主键或唯一索引    |
| `ref`         | 非唯一索引等值        | 低～中  | 提高选择性、优化联合索引前缀    |
| `ref_or_null` | ref + NULL     | 低～中  | 控制 NULL 行比例       |
| `index_merge` | 多索引合并          | 中    | 评估改写 UNION 或建联合索引 |
| `range`       | 索引范围           | 中    | 缩小范围、消除函数、调整列顺序   |
| `index`       | 全索引扫描          | 中～高  | 加过滤条件；覆盖索引仍扫全索引   |
| `ALL`         | 全表扫描           | 高    | 建索引、改写 SQL、更新统计   |

性能差距可达数量级，但不应唯 `type` 论：小表 `ALL` 可能快于错误 `range`；覆盖索引的 `index` 也可能优于回表很多的 `ref`。

## 四、possible_keys、key、key_len

### 1. possible_keys

列出优化器**可能**用到的索引，**不代表一定使用**。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE user_id = 10086 AND status = 1;
-- possible_keys: idx_user_created, idx_status_amount, ...
-- key: idx_user_created（优化器基于选择性/成本选择）
```

**有候选但 `key=NULL` 的常见原因：**

| 原因          | 示例                                | EXPLAIN 特征                                    |
|-------------|-----------------------------------|-----------------------------------------------|
| 对索引列使用函数    | `DATE(created_at) = '2026-01-01'` | possible_keys 可能为 NULL，key=NULL，type=ALL |
| 隐式类型转换      | `phone = 13800138000`（VARCHAR 列与数字常量比较） | possible_keys 含 idx_phone，可能无法快速查找 |
| 前缀 LIKE 不匹配 | `LIKE '%abc'`                     | key=NULL                                      |
| 优化器判定全表更优   | 小表或高比例扫描                          | type=ALL，possible_keys 非空                     |
| 统计信息过时      | Cardinality 偏差                    | rows 不准，选错索引                                  |

排查：确认谓词与最左前缀；检查类型一致；`ANALYZE TABLE`；`EXPLAIN FORMAT=JSON` 查看 `cost_info`。

### 2. key

显示优化器**实际选择**的索引；`NULL` 表示未使用索引。

**选了"意料之外"索引的原因：**

1. **Cardinality / 选择性**：优化器估算另一索引扫描行更少。
2. **覆盖索引**：能覆盖 SELECT 列，减少回表。
3. **排序需求**：`ORDER BY` 列在某索引有序，避免 filesort。
4. **index_merge**：`key` 可能显示多个索引名。

索引提示 `FORCE INDEX` / `USE INDEX` / `IGNORE INDEX` 可用于对比验证，生产环境应优先修正索引与 SQL，而非长期依赖 Hint。

### 3. key_len

索引被使用的**前缀字节长度**，判断联合索引利用了**几列**的关键依据。

**计算规则：**

1. 联合索引从左到右累加，遇范围条件列后右侧通常不计入（范围列本身计入）。
2. 定长按实际存储长度；**可空列额外 +1 字节**（NULL 标记）。
3. **变长类型（VARCHAR 等）额外 +1 或 +2 字节**（长度前缀）：最大字节数不超过 255 时为 +1，超过 255 时为 +2；utf8mb4 每字符最多 4 字节。

**各类型占用字节数：**

| 类型                   | NOT NULL | 允许 NULL | 说明           |
|----------------------|----------|---------|--------------|
| `TINYINT`            | 1        | 2       |              |
| `SMALLINT`           | 2        | 3       |              |
| `INT`                | 4        | 5       |              |
| `BIGINT`             | 8        | 9       |              |
| `DATE`               | 3        | 4       |              |
| `DATETIME`           | 5        | 6       |              |
| `TIMESTAMP`          | 4        | 5       |              |
| `VARCHAR(N)` utf8mb4 | N×4+L    | N×4+L+1 | L 为长度前缀：最大字节数 ≤255 时 L=1，否则 L=2；+1 若可空 |
| `CHAR(N)` utf8mb4    | N×4      | N×4+1   | 定长按最大长       |

**联合索引 `(user_id, created_at)` 示例：**

- `user_id` BIGINT NOT NULL → 8；`created_at` DATETIME NOT NULL → 5；合计 **13**。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE user_id = 10086;
-- key_len=8（仅 user_id）

EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders WHERE user_id = 10086 AND created_at >= '2026-01-01';
-- key_len=13（user_id 等值 + created_at 范围）

EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders WHERE user_id = 10086 AND created_at = '2026-01-15 10:00:00';
-- key_len=13（两列等值，完整前缀）

EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE created_at >= '2026-01-01';
-- 违反最左前缀：key=idx_created 或 NULL；若 key=idx_created 则 key_len=5
```

**VARCHAR 示例：** `email VARCHAR(128) NOT NULL utf8mb4` 最大字节数为 512，长度前缀为 2 字节 → key_len = 128×4+2 = **514**。

## 五、ref、rows、filtered

### 1. ref

显示与**索引列比较**的对象。

| ref 值             | 含义       | 示例                      |
|-------------------|----------|-------------------------|
| `const`           | 与常量比较    | `WHERE user_id = 10086` |
| `db.table.column` | 与另一表列比较  | JOIN：`ref=db.u.id`      |
| `func`            | 与函数结果比较  | 可能限制索引利用                |
| `const,const`     | 多列索引与多常量 | `(a,b) = (1,2)`         |

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE user_id = 10086;
-- ref=const

EXPLAIN FORMAT=TRADITIONAL
SELECT o.*, u.name FROM orders o JOIN users u ON o.user_id = u.id WHERE u.city = 'Shanghai';
-- u: ref=const（city）；o: ref=db.u.id
```

### 2. rows

优化器**估算**的扫描行数，**非精确值**，也非最终结果行数。

- 单表：预计从存储引擎读取的行数。
- JOIN：各表 `rows` 乘积近似中间结果规模。
- `const` 通常为 1；`ALL` 接近表总行数。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE status = 1;
-- type=ref, key=idx_status_amount, rows=1667
```

**rows 不准的原因：** 统计信息过时（`ANALYZE TABLE orders`）；数据倾斜；多列条件独立估算忽略相关性。

统计信息过时可通过 `ANALYZE TABLE` 或查询 `mysql.innodb_index_stats` 修正。`rows` 宜横向对比改写前后，并与`EXPLAIN ANALYZE` 的 `actual rows` 交叉验证。

### 3. filtered

MySQL 5.7+，表示扫描行中预计满足表级 WHERE 的**百分比**。估算返回行数 ≈ `rows × filtered / 100`。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders WHERE user_id = 10086 AND amount > 1000;
-- rows=150, filtered=33.33, Extra=Using where
```

解读：索引扫 150 行，约 33% 满足 `amount > 1000`（`amount` 不在索引前缀，需额外过滤）。

**对 JOIN 的影响：** 优化器用 `filtered` 估算中间结果，选择驱动表顺序。某表 `filtered` 极低时可能优先作为驱动表。

**filtered 较低时：** 将高过滤性条件纳入联合索引；确认 ICP（`Using index condition`）是否生效。

## 六、Extra 列详解

`Extra` 承载排序、临时表、回表、ICP 等关键信号，一行可含多个标记（分号分隔）。

### 1. Using index

查询列全部可从当前索引获得，**无需回表**（覆盖索引）。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT user_id, created_at FROM orders WHERE user_id = 10086;
-- type=ref, Extra=Using index

EXPLAIN FORMAT=TRADITIONAL SELECT user_id FROM orders;
-- type=index, Extra=Using index（全索引扫描但覆盖）
```

`Using index` 只说明覆盖，不保证 `type` 最优；理想是 `type=ref/range` + `Using index`。

### 2. Using index condition

**索引下推（ICP）**：部分 WHERE 在 InnoDB 引擎层、**回表前**过滤二级索引项，减少回表。

```sql
-- 索引 (user_id, amount)
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE user_id = 10086 AND amount > 500;
-- Extra=Using index condition
```

无 ICP 时，每个 `user_id=10086` 的索引项都需回表再过滤 `amount`。ICP 默认开启（`index_condition_pushdown=on`）。

### 3. Using where

Server 层（或引擎返回行后）需应用 WHERE 过滤。

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE user_id = 10086 AND remark LIKE '%VIP%';
-- type=ref, Extra=Using where（remark 无法索引）
```

需警惕：`type=ALL` + `Using where` + 大 `rows` → 首要优化对象。有 ICP 时可能同时出现 `Using index condition` 与
`Using where`。

### 4. Using filesort

无法利用索引有序性完成 `ORDER BY`，需额外排序（内存或磁盘）。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders WHERE status = 1 ORDER BY created_at DESC LIMIT 20;
-- key=idx_status_amount, Extra=Using where; Using filesort
```

`(status, amount)` 无法提供 `created_at` 排序。优化：建 `(status, created_at)` 索引；MySQL 8.0 可用降序索引，可能出现`Backward index scan`。

### 5. Using temporary

需创建**临时表**，常见于 `GROUP BY`、`DISTINCT`、`UNION`（非 ALL）。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT status, COUNT(*) FROM orders GROUP BY status, DATE(created_at);
-- Extra=Using temporary; Using filesort
```

优化：让 GROUP BY/ORDER BY 与索引前缀一致；用生成列替代函数；`UNION ALL` 替代 `UNION` 若允许重复。

### 6. Using join buffer

JOIN 被驱动表**无合适索引**时，可能使用 Join Buffer；MySQL 8.0.18 引入 Hash Join，8.0.20 起不再使用传统 Block Nested-Loop（BNL）。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT o.*, u.name FROM orders o JOIN users u ON o.remark = u.name;
-- u: type=ALL, Extra=Using join buffer
```

优化：为 JOIN 列建索引，使被驱动表升级为 `ref`/`eq_ref`。

### 7. 其他重要标记

| Extra                          | 含义               | 说明                           |
|--------------------------------|------------------|------------------------------|
| `Impossible WHERE`             | WHERE 恒假         | 如 `id=1 AND id=2`，可能直接返回空    |
| `Select tables optimized away` | MIN/MAX 直接从索引获取  | `SELECT MIN(id) FROM orders` |
| `Using MRR`                    | Multi-Range Read | range 场景下随机主键转顺序读            |
| `No matching min/max row`      | MIN/MAX 无匹配      | 空表或条件无匹配                     |
| `Backward index scan`          | 反向索引扫描           | 8.0 降序索引 / ORDER BY DESC     |
| `Rematerialize`                | 重新物化             | 8.0 子查询相关                    |

`SELECT MIN(id)` 等聚合可能 `Select tables optimized away`；`WHERE id=1 AND id=2` 可能 `Impossible WHERE`。

## 七、EXPLAIN ANALYZE（MySQL 8.0）

### 1. 与 EXPLAIN 的区别

| 对比项      | EXPLAIN | EXPLAIN ANALYZE  |
|----------|---------|------------------|
| 是否执行 SQL | 否       | **是**            |
| 输出       | 优化器估算   | 估算 + **实际耗时与行数** |
| 最低版本     | 全版本     | **8.0.18+**      |
| 生产风险     | 低       | 大查询/写操作占资源       |

```sql
EXPLAIN ANALYZE
SELECT o.* FROM orders o
WHERE o.user_id = 10086 AND o.created_at >= '2026-01-01'
ORDER BY o.created_at DESC LIMIT 20;
```

传统 `EXPLAIN` 可能 `rows=120`；ANALYZE 报告实际扫描行数与每步耗时，揭示估算偏差。

### 2. 输出格式解读

默认 **TREE** 格式，含 `actual time`、`rows`、`loops`：

```
-> Limit: 20 row(s)  (actual time=0.052..0.061 rows=20 loops=1)
    -> Sort: o.created_at DESC  (actual time=0.049..0.056 rows=20 loops=1)
        -> Index range scan on o using idx_user_created
           (user_id=10086 AND '2026-01-01' <= created_at)
           (actual time=0.035..0.042 rows=118 loops=1)
```

| 字段                 | 含义                                       |
|--------------------|------------------------------------------|
| `actual time=A..B` | 首行..末行耗时（毫秒）                             |
| `rows=N`           | 该算子**实际**处理行数                            |
| `loops=N`          | 执行次数；Nested Loop 内层 JOIN 时 loops = 驱动表行数 |

```sql
EXPLAIN ANALYZE
SELECT o.*, u.name FROM users u JOIN orders o ON o.user_id = u.id WHERE u.city = 'Shanghai';
-- 观察 orders 索引节点的 loops 是否接近 Shanghai 用户数，而非 orders 总行数
```

注意：MySQL 8.0 的 `EXPLAIN ANALYZE` 只支持 TREE 输出；若指定 `FORMAT=JSON` 或 `FORMAT=TRADITIONAL` 会报错。因此需要程序化分析时，通常解析 TREE 文本或结合 Performance Schema、慢日志等来源补充。

### 3. 什么时候使用

**适合：** 估算与实际差异大（慢日志 `Rows_examined` >> EXPLAIN `rows`）；优化前后验证；JOIN `loops` 过高诊断；复杂子查询确认物化/Semi-Join。

**谨慎：** 生产主库大结果集 DML；无法限流的 ad-hoc 查询（加 `LIMIT` 或在从库执行）。

推荐工作流：`EXPLAIN FORMAT=TRADITIONAL` 快速看计划 → `EXPLAIN ANALYZE` 验证 → `ANALYZE TABLE` 更新统计；生产环境配合`max_execution_time` 限制超时。

## 八、实战案例

### 1. 案例一：函数导致索引失效

**SQL：**

```sql
SELECT id, user_id, amount FROM orders WHERE DATE(created_at) = '2026-01-15';
```

**EXPLAIN：**

```
| type | possible_keys | key  | rows   | Extra       |
|------|---------------|------|--------|-------------|
| ALL  | NULL          | NULL | 500000 | Using where |
```

`possible_keys` 与 `key` 均为 `NULL`：`DATE(created_at)` 对列施加函数，B+ 树无法按 `created_at` 的有序值定位。

**改写：**

```sql
SELECT id, user_id, amount FROM orders
WHERE created_at >= '2026-01-15 00:00:00' AND created_at < '2026-01-16 00:00:00';
-- type=range, key=idx_created, key_len=5, rows=85
```

**进阶（频繁按日期查）：**

```sql
ALTER TABLE orders
ADD COLUMN order_date DATE AS (DATE(created_at)) STORED,
ADD INDEX idx_order_date (order_date);
-- WHERE order_date = '2026-01-15' → type=ref
```

### 2. 案例二：联合索引未完全使用

**SQL：**

```sql
SELECT id, user_id, amount FROM orders WHERE status = 1 ORDER BY created_at DESC LIMIT 50;
```

**EXPLAIN（索引 `idx_status_amount(status, amount)`）：**

```
| key               | key_len | rows | Extra                     |
|-------------------|---------|------|---------------------------|
| idx_status_amount | 1       | 1667 | Using where; Using filesort |
```

`key_len=1`：仅用 `status`（TINYINT 1 字节）；`created_at` 不在索引中，触发 filesort。

**优化：**

```sql
CREATE INDEX idx_status_created ON orders(status, created_at);

EXPLAIN FORMAT=TRADITIONAL
SELECT id, user_id, amount FROM orders WHERE status = 1 ORDER BY created_at DESC LIMIT 50;
-- key=idx_status_created, key_len=1, 可能消除 filesort
```

原则：等值列在前，排序/范围列在后；用 `key_len` 验证使用了哪些列。

### 3. 案例三：驱动表选择错误

**SQL：**

```sql
SELECT o.id, o.amount, u.name FROM orders o
JOIN users u ON o.user_id = u.id WHERE u.city = 'Shanghai';
```

**数据：** orders 500 万行；users 10 万行，`city='Shanghai'` 仅 200 人。

**问题计划（示例）：**

```
| table | type | key  | rows   | filtered | 说明               |
|-------|------|------|--------|----------|--------------------|
| o     | ALL  | NULL | 5000000 | 100.00  | 驱动表全表扫       |
| u     | eq_ref| PRIMARY | 1 | 0.20     | 每行 JOIN 再过滤 city |
```

**优化改写：**

```sql
SELECT o.id, o.amount, u.name
FROM users u STRAIGHT_JOIN orders o ON o.user_id = u.id
WHERE u.city = 'Shanghai';
-- 期望：u type=ref key=idx_city rows=200；o type=ref key=idx_user_created ref=db.u.id
```

或用 `EXPLAIN ANALYZE` 确认 `orders` 索引节点 `loops ≈ 200` 而非 500000。

### 4. 案例四：隐式类型转换

**SQL：**

```sql
SELECT * FROM orders WHERE phone = 13800138000;  -- phone 为 VARCHAR(20)，有索引 idx_phone
```

**EXPLAIN：**

```
| type | possible_keys | key  | rows   | Extra       |
|------|---------------|------|--------|-------------|
| ALL  | idx_phone     | NULL | 500000 | Using where |
```

VARCHAR 列与数字常量比较时，MySQL 将 **phone 列**的每行值逐行转为数字（等价于 `CAST(phone AS DOUBLE)`），B+ 树字典序被破坏，索引失效。 注意反过来不成立：`INT 列 = '字符串常量'` 时转换发生在常量侧，索引不受影响。

**修正：**

```sql
SELECT * FROM orders WHERE phone = '13800138000';
-- type=ref, key=idx_phone, ref=const, rows=1
```

排查习惯：比较两侧类型一致；见 `ALL` + 有 `possible_keys` 时优先怀疑转换或函数。

### 5. 案例五：深分页

**SQL：**

```sql
SELECT id, user_id, amount, remark FROM orders ORDER BY id LIMIT 100000, 20;
```

**EXPLAIN：**

```
| type  | key     | rows   | 说明                          |
|-------|---------|--------|-------------------------------|
| index | PRIMARY | 100020 | 扫 100020 行，丢弃前 100000 行 |
```

**优化一：Seek 分页（推荐）**

```sql
SELECT id, user_id, amount, remark FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;
-- type=range, key=PRIMARY, rows=20
```

**优化二：延迟关联（按二级索引排序时减少宽行回表）**

```sql
SELECT o.id, o.user_id, o.amount, o.remark
FROM orders o INNER JOIN (
    SELECT id FROM orders FORCE INDEX (idx_created)
    ORDER BY created_at, id
    LIMIT 100000, 20
) t ON o.id = t.id;
-- 内层扫 100020 个 idx_created 索引项，外层 20 次主键 lookup；根本解决仍是 Seek 分页
```

**EXPLAIN ANALYZE 对比：**

```sql
EXPLAIN ANALYZE SELECT id FROM orders ORDER BY id LIMIT 100000, 20;
-- actual rows ≈ 100020，耗时远高于 Seek 方案
```

## 总结

`EXPLAIN` 是 SQL 性能分析的基础工具。其价值在于建立 **访问路径 → 估算成本 → Extra 信号 → 优化手段**的推理链，并与慢查询日志、Performance Schema、`EXPLAIN ANALYZE` 交叉验证。

**分析顺序：** `type` → `key` / `key_len` → `rows` / `filtered` → `Extra` →（8.0）`EXPLAIN ANALYZE` 验证。

**type 列：** 从 `system/const/eq_ref/ref/range` 到 `index/ALL` 成本递增；`ALL` 与大数据量 `index` 重点排查；`index_merge`、`ref_or_null` 等需结合 `Extra` 理解。

**索引列：** `possible_keys` 是候选，`key` 是最终决定；有候选无使用 → 查函数、隐式转换、最左前缀。`key_len` 精确反映联合索引使用深度（可空 +1，变长 +2）。Hint 仅用于验证。

**ref / rows / filtered：** `ref` 标明索引与谁比较；`rows` 是估算，用 `ANALYZE TABLE` 与 ANALYZE 校正；`filtered` 影响 JOIN 顺序与返回行估算。

**Extra：** `Using index`（覆盖）、`Using index condition`（ICP）、`Using filesort`、`Using temporary`、`Using join buffer`是常见优化切入点。

**EXPLAIN ANALYZE：** 8.0.18+ 提供 `actual time`、`rows`、`loops`；估算偏差大、JOIN loops 过高、优化验证时优先使用。

**实战高频根因：** 函数包裹列、隐式类型转换、联合索引列顺序错误、驱动表不当、深分页 `OFFSET`。优化是迭代过程：改写 SQL → 调整索引 → 更新统计 → ANALYZE 验证，直至成本可接受。执行计划随数据分布变化，应定期复查核心 SQL。
