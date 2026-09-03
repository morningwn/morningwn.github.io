---
title: 索引失效场景分析：为什么你的索引没有生效
summary: 系统梳理导致 InnoDB 索引失效的常见场景，包括函数操作、隐式类型转换、OR 条件、LIKE 前缀通配、索引合并等，结合 EXPLAIN 输出深入分析每种失效的根因。
created: 2026-07-02
updated: 2026-07-07
tags: MySQL, InnoDB, 索引, 性能优化, EXPLAIN
---

# 索引失效场景分析：为什么你的索引没有生效

"明明建了索引，为什么还是全表扫描？"——这是 DBA 与后端工程师在性能排查中最常遇到的问题之一。索引失效并非指索引损坏或不可用，而是优化器在成本估算后，判定走索引的代价高于全表扫描、索引合并或其他访问路径，从而放弃使用开发者预期的索引。

理解索引失效，需要同时掌握三个层面的知识：B+ 树索引的物理有序性（索引按什么键排序、支持怎样的查找模式）、MySQL 优化器的成本模型（何时选择 ref、range、index_merge 或 ALL）、以及 SQL 写法与索引结构的匹配关系（谓词形态是否允许在 B+ 树上定位起点）。本文从索引设计原则出发，逐类深入分析导致索引无法生效的常见场景，并结合 `EXPLAIN` 输出说明根因与排查方法。排序、分组、分页与 COUNT 的索引优化将在独立文章中讲解。

文中示例基于如下测试表结构。分析时请结合 `EXPLAIN FORMAT=TRADITIONAL`（或 MySQL 8.0 的 `EXPLAIN ANALYZE`）观察 `type`、`key`、`key_len`、`rows`、`filtered` 与 `Extra` 字段；必要时配合 `SHOW WARNINGS`、`SHOW INDEX FROM` 与 `optimizer_trace`交叉验证。

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
    id       BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    name     VARCHAR(64)     NOT NULL,
    email    VARCHAR(128)    NOT NULL,
    phone    VARCHAR(11)     NOT NULL,
    city     VARCHAR(32)     NOT NULL,
    PRIMARY KEY (id),
    KEY idx_name (name),
    KEY idx_email (email),
    KEY idx_phone (phone),
    KEY idx_city (city)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 一、索引设计的核心原则

在分析"为什么索引没生效"之前，需要先明确"什么样的索引才值得建、怎样建才能被优化器选中"。索引设计的三条核心原则——选择性、覆盖性、有序性——既是建索引的出发点，也是理解失效场景的理论基础。

### 1. 选择性（Selectivity）

**什么是索引选择性。** 索引选择性（Selectivity），也称区分度或基数（Cardinality），衡量索引列中不同取值数量相对于表总行数的比例：`选择性 = COUNT(DISTINCT column) / COUNT(*)`。取值范围 0～1；为 1 表示列值全部唯一（如主键），接近 0 表示高度重复（如性别字段 M/F）。

选择性决定索引过滤效率：优化器通过索引定位候选行后，仍需回表或索引条件下推验证其他谓词。若索引只能把 100 万行过滤到 50 万行，收益可能不足以抵消随机 IO 与 B+ 树遍历，优化器会倾向全表扫描。

**如何计算选择性。** 对单列索引：

```sql
SELECT
    COUNT(*) AS total_rows,
    COUNT(DISTINCT user_id) AS distinct_user_id,
    COUNT(DISTINCT status) AS distinct_status,
    ROUND(COUNT(DISTINCT user_id) / COUNT(*), 4) AS user_id_selectivity,
    ROUND(COUNT(DISTINCT status) / COUNT(*), 4) AS status_selectivity
FROM orders;
```

典型输出（100 万行）：

```
+------------+------------------+-----------------+---------------------+--------------------+
| total_rows | distinct_user_id | distinct_status | user_id_selectivity | status_selectivity |
+------------+------------------+-----------------+---------------------+--------------------+
|    1000000 |           850000 |               3 |              0.8500 |             0.0000 |
+------------+------------------+-----------------+---------------------+--------------------+
```

联合索引需评估前缀组合：`COUNT(DISTINCT user_id, created_at) / COUNT(*)`。也可查 `information_schema.statistics` 的`cardinality` 字段——注意这是采样估算，DDL 或大批量写入后应 `ANALYZE TABLE orders`刷新，否则优化器可能基于过时基数做出错误决策，本身也会导致"有索引却不走"。

**选择性多高才值得建索引。** 无绝对阈值，参考：

| 选择性范围     | 单列索引价值 | 典型处理        |
|-----------|--------|-------------|
| > 0.9     | 极高     | 独立索引或联合索引前缀 |
| 0.1 ~ 0.9 | 中等     | 结合查询频率联合设计  |
| < 0.01    | 极低     | 通常不单独建索引    |

`status` 仅 0、1、2 三值，选择性约 0.000003，单独索引最多缩小到约三分之一扫描量：

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE status = 1;
-- type=ref, key=idx_status_amount, rows≈333333
```

表较宽且 `status=1` 占比超 30%～50% 时，优化器可能改选 `type=ALL`——这不是失效，而是正确成本判断。更合理做法：低选择性列作联合索引后续列，如`(user_id, status, created_at)`，先以高选择性 `user_id` 精确定位再过滤 `status`。

### 2. 覆盖性

**尽可能让索引满足查询需求。** 覆盖索引（Covering Index）指查询所需列全部在某条索引中，优化器在索引层完成过滤与投影，无需回表。InnoDB 二级索引叶子节点存索引列值与主键值。

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT user_id, created_at
FROM orders
WHERE user_id = 10086 AND created_at >= '2026-01-01'
ORDER BY created_at DESC LIMIT 20;
-- type=ref, key=idx_user_created, key_len=8, Extra=Using where
```

若扩展为 `SELECT user_id, created_at, amount`，每匹配行需回表读聚簇索引。将 `amount` 纳入索引后：

```sql
CREATE INDEX idx_user_created_amount ON orders (user_id, created_at, amount);

EXPLAIN FORMAT=TRADITIONAL
SELECT user_id, created_at, amount
FROM orders WHERE user_id = 10086 AND created_at >= '2026-01-01';
-- Extra=Using index  → Index Only Scan，零回表
```

**减少回表。** 回表是二级索引固有成本：先在二级索引 B+ 树定位，取主键，再到聚簇索引读完整行，每次回表是一次随机 IO。高 QPS 点查场景下回表次数直接决定延迟。覆盖索引以空间换时间，但宽索引降低扇出、增加写放大，应针对高频延迟敏感查询专门设计，低频报表不必强求。

### 3. 有序性

**索引天然有序的利用。** B+ 树按索引列定义顺序物理有序。联合索引 `(a, b, c)` 先按 `a`、再按 `b`、再按 `c`排序。等值条件从左连续匹配时，可在索引中精确定位子范围：

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders
WHERE user_id = 10086
  AND created_at >= '2026-01-01' AND created_at < '2026-02-01';
-- type=ref/range, key=idx_user_created, user_id 等值后 created_at 在索引中连续
```

**排序和分组。** 索引有序性也影响 `ORDER BY`/`GROUP BY` 能否避免额外排序（专项优化见独立文章）。核心规则：**WHERE 中某列出现范围谓词，其右侧索引列通常无法继续用于排序或分组**（范围截断）。

```sql
-- created_at 范围截断后，amount 在索引中不再全局有序
SELECT * FROM orders
WHERE user_id = 10086 AND created_at >= '2026-01-01'
ORDER BY amount;
-- Extra 可能出现 Using filesort
```

设计法则：等值列在前、范围列在后；排序列尽量纳入索引；避免 `(user_id)` 与 `(user_id, created_at)` 重复建设。

## 二、函数操作导致索引失效

对索引列施加函数，是最常见也最易避免的失效原因之一。根因：B+ 树按列原始值排序，而非函数计算结果的排序。

### 1. 对索引列使用函数

**WHERE YEAR(created_at) = 2026**

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders WHERE YEAR(created_at) = 2026;
```

```
+------+---------------+------+---------+----------+-------------+
| type | possible_keys | key  | rows    | filtered | Extra       |
+------+---------------+------+---------+----------+-------------+
| ALL  | idx_created   | NULL | 1000000 |    10.00 | Using where |
+------+---------------+------+---------+----------+-------------+
```

`possible_keys` 有 `idx_created` 但 `key=NULL`：`YEAR(created_at)` 需对每行计算后再比较，B+ 树存的是原始 DATETIME 排序，无法定位"YEAR 值为 2026"的起点。同类失效：`DATE(created_at)`、`MONTH()`、`DATE_FORMAT()`、`UNIX_TIMESTAMP(created_at)`。

**WHERE UPPER(name) = 'JOHN'**

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM users WHERE UPPER(name) = 'JOHN';
-- type=ALL, possible_keys=idx_name, key=NULL
```

`UPPER(name)` 的字典序与 `name` 原始序不同，B+ 树无法有效定位。其他失效写法：`amount + 10 > 1000`（列侧算术）、`SUBSTRING(phone,1,3)='138'`、`JSON_EXTRACT(meta,'$.type')='vip'`。

**为什么函数会让索引失效：B+ 树按原始值排序。** 查找流程要求查找键与索引项直接比较。谓词变为 `f(column)=constant` 时，索引存的是`column` 而非 `f(column)`，除非有函数索引，否则只能全表扫描逐行计算。示意：

```
B+ 树（created_at 原始值）: 2026-01-01 ... 2026-12-31  ← 可范围定位
YEAR(created_at) 的值序:    2026 对应全年多行，无法在树上连续
```

### 2. 解决方案

**改写为范围查询**（首选）：

```sql
-- 失效
SELECT * FROM orders WHERE YEAR(created_at) = 2026;
SELECT * FROM orders WHERE DATE(created_at) = '2026-01-15';

-- 改写
SELECT * FROM orders
WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';

SELECT * FROM orders
WHERE created_at >= '2026-01-15 00:00:00' AND created_at < '2026-01-16 00:00:00';
-- 改写后: type=range, key=idx_created
```

算术同理：`amount + 10 > 1000` → `amount > 990`。

**函数索引（MySQL 8.0 Generated Column / Functional Index）**：

```sql
-- 生成列方案
ALTER TABLE orders
ADD COLUMN created_year INT GENERATED ALWAYS AS (YEAR(created_at)) STORED,
ADD INDEX idx_created_year (created_year);

-- 或直接函数索引（8.0.13+）
CREATE INDEX idx_year ON orders ((YEAR(created_at)));
```

函数索引有额外存储与维护成本，应作为无法改写的兜底。

**具体 SQL 改写示例：**

| 场景       | 失效写法                                     | 改写方案                                           |
|----------|------------------------------------------|------------------------------------------------|
| 手机号前四位   | `SUBSTRING(phone,1,4)='1380'`            | `phone LIKE '1380%'` 或冗余 `phone_prefix` 列      |
| Unix 时间戳 | `UNIX_TIMESTAMP(created_at) BETWEEN ...` | 应用层转边界，`created_at >= FROM_UNIXTIME(...)`      |
| 大小写不敏感   | `UPPER(name)='JOHN'`                     | 使用 `utf8mb4_*_ci` collation 或 `name_lower` 生成列 |

## 三、隐式类型转换

隐式类型转换是函数失效的特殊 case：开发者未显式写函数，MySQL 自动对索引列做类型转换，等效于列侧 `CAST()`，同样破坏 B+ 树有序访问。

### 1. 什么是隐式类型转换

**VARCHAR 列用数字查询：**

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM users WHERE phone = 13800138000;
-- type=ALL, possible_keys=idx_phone, key=NULL
```

`phone` 是 VARCHAR，字面量 `13800138000` 是数字。MySQL 将 `phone` 转为数字再比较，等效 `CAST(phone AS SIGNED)=13800138000`，字符串索引有序性被破坏。正确写法：`WHERE phone = '13800138000'`。

**INT 列用字符串查询：**

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE user_id = '10086';
-- type=ref, key=idx_user_created（索引仍然有效）
```

`BIGINT` 列与字符串 `'10086'` 比较时，MySQL 将**字符串常量**转为数字，等效 `user_id = 10086`，转换发生在常量侧，索引不受影响。 不过仍建议保持类型一致 `WHERE user_id = 10086`，避免阅读歧义和边界情况（如字符串含非数字字符时会被转为 0）。

### 2. MySQL 的类型转换规则

核心原则：**字符串与数字比较时，字符串转数字**。索引是否失效取决于转换是否发生在**索引列**上——只有对列值逐行做转换才会破坏
B+ 树的有序性。

| 比较场景                            | 转换方向           | 索引受影响    |
|---------------------------------|----------------|----------|
| VARCHAR 列 = 数字常量                | 列 → 数字         | 是（列侧转换）  |
| INT 列 = 字符串常量                   | 常量 → 数字        | 否（常量侧转换） |
| DATETIME 列 = 字符串 `'2026-01-01'` | 字符串 → DATETIME | 否（常量侧）   |
| VARCHAR 列 = 字符串                 | 无              | 否        |

**只要转换发生在索引列上，索引就可能失效。** 典型场景是 VARCHAR 列与数字比较——VARCHAR 列值需逐行转为数字，B+ 树字典序失效。

### 3. 为什么会导致索引失效

B+ 树按列存储类型和 collation 排序。`VARCHAR` 字典序（'2' > '13800138000'）与 `CAST AS SIGNED` 数值序（2 < 13800138000）完全不同；'abc' 转数字为 0，'010' 转为 10——转换后的值序与 B+ 树存储序不一致，优化器无法定位，只能全表扫描逐行转换比较。

### 4. 常见踩坑场景

**phone VARCHAR(11) WHERE phone = 13800138000** — 最高频陷阱。Java/MyBatis 中参数定义为 `Long` 会以数字绑定：

```java
// 错误：触发隐式转换
jdbcTemplate.query("SELECT * FROM users WHERE phone = ?", 13800138000L);
// 正确
jdbcTemplate.query("SELECT * FROM users WHERE phone = ?", "13800138000");
```

ORM 层 `user_id` 传字符串到 `BIGINT` 列虽不会破坏索引（转换发生在常量侧），但也应保持类型一致——排查慢查询时可一并检查参数绑定类型。

其他隐蔽场景：`ENUM` 与数字比较、JOIN 条件两侧类型不一致（`BIGINT user_id` JOIN `VARCHAR user_id_str`）、`CHAR` 尾部空格与`VARCHAR` 比较。

### 5. 排查方法

**EXPLAIN + SHOW WARNINGS：**

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM users WHERE phone = 13800138000;
SHOW WARNINGS;
-- Message: ... where (cast(`users`.`phone` as signed) = 13800138000)
```

`cast(phone as signed)` 是索引失效的直接证据。正确查询的 Warnings 无 cast，`type=ref, key=idx_phone, rows=1`。

排查清单：`DESC`/`SHOW CREATE TABLE` 确认列类型 → `EXPLAIN` 看 type/key/rows → `SHOW WARNINGS` 查 cast/convert → 核对应用层参数绑定类型 → 必要时 `optimizer_trace` 看成本估算。

## 四、OR 条件

`OR` 要求满足左条件**或**右条件的行均返回，与 B+ 树"在有序结构中定位连续范围"的访问模式存在张力。

### 1. OR 与索引的关系

**两个条件都有索引 → 可能 Index Merge：**

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders WHERE user_id = 10086 OR status = 1;
```

```
+-------------+------+--------+----------------------------------------------------------+
| type        | key  | rows   | Extra                                                    |
+-------------+------+--------+----------------------------------------------------------+
| index_merge | ...  | 333453 | Using union(idx_user_created,idx_status_amount); Using where |
+-------------+------+--------+----------------------------------------------------------+
```

单条索引无法覆盖 OR 两侧，优化器分别扫描两索引，Union 合并去重后回表。

**有一个条件没索引 → 全表扫描：**

```sql
EXPLAIN FORMAT=TRADITIONAL
SELECT * FROM orders WHERE user_id = 10086 OR remark = 'VIP客户';
-- type=ALL, key=NULL  （remark 无索引，必须扫全表找匹配行）
```

### 2. 什么时候 OR 会导致全表扫描

| 场景          | 原因                          |
|-------------|-----------------------------|
| OR 一侧无索引    | 必须全表扫描覆盖无索引侧                |
| 两侧选择性均低     | Index Merge Union 成本 > 全表扫描 |
| OR 条件过多     | 多路 Merge 成本指数增长             |
| OR 含函数/隐式转换 | 两侧均无法有效走索引                  |

```sql
SELECT * FROM orders WHERE user_id = 10086 OR amount > 5000;     -- 右侧无索引
SELECT * FROM orders WHERE user_id = 10086 OR YEAR(created_at)=2026; -- 函数破坏
```

Index Merge Union 的 `rows` 约等于两路匹配行数之和，接近全表时性能与 ALL 无异。

### 3. 解决方案

**UNION ALL 改写：**

```sql
SELECT * FROM orders WHERE user_id = 10086
UNION ALL
SELECT * FROM orders WHERE status = 1 AND user_id <> 10086;
-- 分支一: type=ref, key=idx_user_created, rows≈120
-- 分支二: type=ref, key=idx_status_amount, rows≈333333
```

若第二分支匹配行数过大，UNION 无法解决根本问题，需重新审视需求或建联合索引。

**添加联合索引 / 改写为 IN：**

```sql
-- 同列 OR → IN
SELECT * FROM orders WHERE status IN (1, 2, 3);       -- 可走 idx_status_amount
SELECT * FROM orders WHERE user_id IN (10086, 10087);   -- 可走 idx_user_created

-- 跨列 OR 无法单一联合索引覆盖，高频场景建 (user_id, status) + UNION ALL 拆分
```

## 五、LIKE 前缀通配

B+ 树索引只支持**前缀固定**的匹配；通配符 `%`/`_` 出现在开头时，前缀未知，无法定位起点。

### 1. LIKE 'abc%' → 可以用索引

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM users WHERE name LIKE 'John%';
-- type=range, key=idx_name, rows≈15
```

等价于 `name >= 'John' AND name < 'Johp'`，优化器定位 'John' 起点后沿叶子链表顺序扫描。

### 2. LIKE '%abc' → 不能用索引

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM users WHERE email LIKE '%@gmail.com';
-- type=ALL, possible_keys=idx_email, key=NULL
```

后缀匹配前缀未知，只能全表扫描。

### 3. LIKE '%abc%' → 不能用索引

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE remark LIKE '%退款%';
-- type=ALL, possible_keys=NULL
```

前后均有通配符是最常见的 LIKE 失效模式。`LIKE '订单%退款%'` 仅前缀段可用 range，中间 `%` 之后无法索引过滤。

### 4. 原理分析：B+ 树只能做前缀匹配

B+ 树按完整字符串字典序排列。`'abc%'` 可二分定位起点后顺序扫描；`'%abc'` 中 'abc' 可能出现在任意位置，'xxxabc'、'abc'、'xabcy' 在树中分散，无法确定扫描起点——类似字典无法直接查"以 ing 结尾的单词"。

### 5. 解决方案

**全文索引：**

```sql
ALTER TABLE orders ADD FULLTEXT INDEX ft_remark (remark);
EXPLAIN SELECT * FROM orders WHERE MATCH(remark) AGAINST('退款' IN NATURAL LANGUAGE MODE);
-- type=fulltext, key=ft_remark
```

适用于分词搜索，短字符串和精确子串支持有限（InnoDB 默认最小词长 3）。

**搜索引擎（Elasticsearch）：** 复杂模糊搜索、相关性排序走 ES 倒排索引；MySQL 保留 B+ 树处理精确过滤。架构：`MySQL → Canal/Debezium → ES`。

**覆盖索引优化（无法消除全表扫描，仅减回表）：**

```sql
CREATE INDEX idx_remark_id ON orders (remark, id);  -- InnoDB 二级索引自动含主键，显式列出 id 使覆盖意图更明确
SELECT id FROM orders WHERE remark LIKE '%退款%';
-- 仍 type=ALL，但 Extra=Using index，宽表场景降低 IO
```

## 六、NOT、!=、<>、NOT IN、NOT EXISTS

否定条件是否走索引是长期误解——并非所有否定条件都导致全表扫描。

### 1. 否定条件是否能用索引

**常见误解：并非所有否定条件都不走索引。**

```sql
EXPLAIN FORMAT=TRADITIONAL SELECT * FROM orders WHERE status != 0;
-- type=range, key=idx_status_amount, rows≈666666
```

优化器将 `!= 0` 转化为 `status < 0 OR status > 0`（或 `IN (1,2,...)`），在索引中扫描多个不连续区间。

**优化器的成本判断**取决于：匹配行数占比（排除 1 行 vs 排除 99%）、索引选择性、是否有更优路径。`id != 100` 在主键上几乎必然全表扫描。

### 2. 哪些情况下确实不走索引

| 场景                          | 典型计划      | 原因          |
|-----------------------------|-----------|-------------|
| `NOT IN (大量值)`              | ALL       | 排除集过大，区间过多  |
| `col != value` 且 value 为常见值 | ALL       | 匹配行接近全表     |
| `NOT IN (子查询)` 子查询无索引       | ALL       | 关联子查询代价高    |
| `IS NOT NULL` 且 NULL 极少     | index/ALL | 取决于 NULL 比例 |

```sql
EXPLAIN SELECT * FROM orders o
WHERE o.user_id NOT IN (SELECT user_id FROM blacklist);
-- blacklist.user_id 无索引时，外层每行执行子查询，极慢
```

InnoDB 二级索引**会存储 NULL 值**，`WHERE col IS NULL` 可以使用索引（type=ref）。但 `IS NOT NULL` 在 NULL 值极少时可能退化为全索引/全表扫描，因为优化器认为返回行数过多、走索引不划算。

### 3. 替代方案

```sql
-- NOT IN → NOT EXISTS（确保 blacklist.user_id 有索引）
SELECT * FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM blacklist b WHERE b.user_id = o.user_id);

-- NOT IN → LEFT JOIN + IS NULL
SELECT o.* FROM orders o
LEFT JOIN blacklist b ON o.user_id = b.user_id WHERE b.user_id IS NULL;

-- != 排除少数值 → 正向 IN
-- status 仅有 0,1,2：WHERE status IN (1, 2) 优于 WHERE status != 0

-- IS NOT NULL → 设计层避免 NULL
-- remark 设 NOT NULL DEFAULT '' 后无 NULL 值，用 WHERE remark <> '' 筛非空行（注意：<> '' 与 IS NOT NULL 语义不同，仅在列定义为 NOT NULL 且意图排除空串时等效）
```

## 七、联合索引相关的失效场景

联合索引是 InnoDB 最强优化工具，失效几乎都与最左前缀和范围截断有关。

### 1. 不满足最左前缀

联合索引 `(a, b, c)` 仅当查询包含**从最左列开始的连续等值条件**时才能充分利用。

```sql
-- idx_user_created (user_id, created_at)

-- 有效
WHERE user_id = 10086 AND created_at >= '2026-01-01';

-- 失效：跳过最左列
EXPLAIN SELECT * FROM orders WHERE created_at >= '2026-01-01';
-- type=range, possible_keys=idx_created, key=idx_created
-- 注意：此处使用了单列索引 idx_created，联合索引 idx_user_created 因不满足最左前缀未被选用
```

索引物理顺序 `(10086,2026-01-01), (10086,2026-01-02), (10087,2026-01-01)...` — `created_at` 在不同 `user_id`间交错，不存在全局连续范围，因此联合索引无法使用——如上方 EXPLAIN 所示，优化器仅能使用单列索引 `idx_created`。

`(user_id, status, created_at)` 索引下：`WHERE status=1` 无法用；`WHERE user_id=10086 AND status=1` 可用前两列。

### 2. 范围条件中断

等值条件从左连续匹配后，**第一个范围条件所在列右侧**无法继续用于索引过滤或排序。

```sql
-- 假设索引 (user_id, created_at, amount)
EXPLAIN SELECT * FROM orders
WHERE user_id = 10086 AND created_at >= '2026-01-01' AND amount > 100;
-- key_len=13（user_id 等值 8 + created_at 范围 5），amount 在 Server 层过滤
-- filtered≈33.33 表示 amount 在 Server 层过滤
```

范围条件包括 `<`、`<=`、`>`、`>=`、`BETWEEN`、`LIKE 'prefix%'`；MySQL 8.0 中 `IN` 有时视为 range。设计：等值列在前、范围列在后。

### 3. EXPLAIN 中 key_len 的判断方法

`key_len` 是判断联合索引实际用到几列的关键（字节数）。

| 类型                 | key_len（非 NULL）       |
|--------------------|-----------------------|
| TINYINT            | 1                     |
| INT                | 4                     |
| BIGINT             | 8                     |
| DATETIME           | 5                     |
| VARCHAR(N) utf8mb4 | N×4+2（实际以 EXPLAIN 为准） |

允许 NULL 的列 +1 字节。对 `idx_user_created (user_id BIGINT, created_at DATETIME)`：

```sql
WHERE user_id = 10086;                              -- key_len=8
WHERE user_id = 10086 AND created_at = '2026-01-01'; -- key_len=13 (8+5)
WHERE user_id = 10086 AND created_at >= '2026-01-01'; -- key_len=13 (8+5)，范围列也计入 key_len
```

判断技巧：`key_len=8` 于 `(user_id, created_at)` 表示仅 user_id 被使用（无 created_at 条件）；`key_len=13` 表示两列均参与（等值+等值 或 等值+范围）。不确定时直接 EXPLAIN 对比，比手工计算更可靠。

## 八、索引合并（Index Merge）

Index Merge 是优化器合并多条索引扫描结果集的策略，不是"失效"，但常是索引设计与访问模式未对齐的信号。

### 1. Index Merge Intersection

适用于 AND 连接、每列各有独立索引：

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 10086 AND status = 1;
```

```
+-------------+------+------+-----------------------------------------------------------+
| type        | key  | rows | Extra                                                     |
+-------------+------+------+-----------------------------------------------------------+
| index_merge | ...  |   40 | Using intersect(idx_user_created,idx_status_amount); Using where |
+-------------+------+------+-----------------------------------------------------------+
```

流程：分别扫描两索引得主键集合 A、B → 求 A∩B → 回表。两路过滤后交集很小时有效；A 10 万行 ∩ B 50 万行时求交成本可能很高。

### 2. Index Merge Union

适用于 OR 连接（见第四节），`Extra=Using union(idx_a,idx_b)`。回表行数 ≈ |A|+|B|，两集合均大时可能劣于全表扫描。

### 3. 什么时候 Index Merge 是优化器的妥协

出现 Index Merge 通常意味着：

- 缺乏覆盖全部谓词的联合索引，多个单列索引各管一列；
- 统计信息过时：已有 `(user_id, status)` 仍走 Merge → 执行 `ANALYZE TABLE`；
- 联合索引存在但最左前缀不满足；
- 测试环境与生产数据量差异导致成本估算偏差。

合理选择场景：临时报表、不值得建联合索引；多低选择性 AND 条件交集极小；宽表回表行数经合并后仍很少。

排查：`SHOW INDEX FROM orders` → `ANALYZE TABLE orders` → `SELECT @@optimizer_switch` 查看 `index_merge` 开关。

### 4. 更好的替代：建联合索引

```sql
CREATE INDEX idx_user_status ON orders (user_id, status);

EXPLAIN SELECT * FROM orders WHERE user_id = 10086 AND status = 1;
-- type=ref, key=idx_user_status, key_len=9 (8+1), 单次查找直达，无两路扫描
```

原则：AND 等值列按选择性从高到低排列；一个联合索引替代多个重叠单列索引；避免"每列各建一个索引"把合并成本推给优化器。

## 九、其他失效场景

### 1. 对索引列做运算

与函数同理，列侧算术破坏有序访问：

```sql
EXPLAIN SELECT * FROM orders WHERE id + 1 = 100;
-- type=ALL, possible_keys=PRIMARY, key=NULL  （即使主键）
```

改写：`WHERE id = 99`。`amount * 2 > 1000` → `amount > 500`。`user_id % 10 = 3` 无法简单改写，考虑冗余列或函数索引。

### 2. IS NULL 与 IS NOT NULL

如前文所述，InnoDB 二级索引会存储 NULL 值，`WHERE col IS NULL` 可以使用索引（type=ref）。但 `IS NOT NULL` 在绝大多数行非 NULL 时，优化器可能因回表成本过高而选择全表扫描（type=ALL）或全索引扫描（type=index），尤其在被评估的行占比超过阈值时。设计建议：若业务允许，列定义加 `NOT NULL DEFAULT ''` 以消除 NULL 语义带来的优化器不确定性；此时 `WHERE col = ''` 可替代 `WHERE col IS NULL`。

### 3. 字符集不一致导致索引失效

JOIN/WHERE 两侧 collation 不一致时，MySQL 对索引列做 `CONVERT()`：

```sql
-- t1.name: utf8mb4_general_ci, t2.name: utf8mb4_unicode_ci
EXPLAIN SELECT * FROM t1 JOIN t2 ON t1.name = t2.name;
SHOW WARNINGS;  -- 可能看到 convert(t1.name using utf8mb4)
```

排查：`information_schema.columns` 查 `character_set_name`/`collation_name`，统一为相同 collation。

### 4. 数据分布极端：全表只有两种值

语法上索引可用，优化器也可能放弃：

```sql
CREATE INDEX idx_gender ON users (gender);  -- 仅 'M'/'F'
EXPLAIN SELECT * FROM users WHERE gender = 'M';
-- 5000 行: type=ref, rows=2500
-- 1000 万行且 M 占 500 万: type=ALL（随机读 500 万索引项+回表 > 顺序扫全表）
```

这类"失效"是优化器正确决策。对策：低选择性列不单独建索引；与其他高选择性列组联合索引；分区/归档缩小表规模。

## 总结

索引失效是 SQL 性能排查的核心议题。本文从设计原则与失效场景两个维度梳理了 InnoDB 索引无法按预期工作的根因与对策。

**设计原则是索引有效的前提**

- 选择性决定索引价值：高选择性列作联合索引前缀，低选择性列避免单独建索引。
- 覆盖索引消除回表，但需权衡索引宽度与写成本。
- 有序性要求联合索引列顺序服务 WHERE 等值与范围；范围条件截断其右侧列的索引利用。

**函数与类型转换是最常见的可避免失效**

- 对索引列施加函数（`YEAR()`、`UPPER()`、算术运算）破坏 B+ 树有序访问，优先改写为范围查询，或使用 MySQL 8.0 函数索引。
- 隐式类型转换等效于列侧 CAST；VARCHAR 列用数字、INT 列用字符串是高频陷阱。用 `EXPLAIN` + `SHOW WARNINGS` 排查。

**谓词形态决定索引访问模式**

- OR 可能触发 Index Merge Union 或全表扫描，可评估 UNION ALL 改写。
- LIKE 仅支持 `'abc%'` 前缀固定；`'%abc'`/`'%abc%'` 需 FULLTEXT 或外部检索引擎。
- 否定条件并非绝对不走索引，取决于匹配行占比；优先改写为正向 IN 或 NOT EXISTS。

**联合索引遵循最左前缀与范围截断**

- 跳过最左列、范围条件后的后续列，无法在索引层有效过滤。
- 通过 `key_len` 判断联合索引实际用到的列数。

**Index Merge 是妥协而非目标**

- Intersection/Union 是多单列索引场景下的备选；高频 AND/OR 查询应建联合索引从根本上消除 Merge。

**其他注意事项**

- 列侧运算、IS NULL/IS NOT NULL、字符集不一致、极端数据分布，都可能导致索引不被选用。
- 索引不被使用有时是优化器基于成本的正确决策，而非 SQL 写法错误。

索引问题没有一劳永逸的规则表。建立"查看表结构 → EXPLAIN → SHOW WARNINGS → 对照 B+ 树访问模式"的排查习惯，比死记失效场景清单更能应对复杂生产环境。当语法正确的索引仍不被使用时，请回到选择性、数据分布与统计信息是否准确这三个根本问题上来。
