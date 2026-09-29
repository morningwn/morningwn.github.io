---
title: 联合索引、覆盖索引与索引下推：高效利用索引的进阶策略
summary: 深入讲解联合索引的存储结构与最左前缀原则、覆盖索引避免回表的机制、索引下推的优化原理，以及前缀索引的设计方法。
created: 2026-07-02
updated: 2026-07-06
tags: MySQL, InnoDB, 索引, 覆盖索引, 联合索引, 索引下推
---

# 联合索引、覆盖索引与索引下推：高效利用索引的进阶策略

在 InnoDB 中，单列索引只能解决"某一列上的过滤或排序"问题；真实业务查询几乎总是多列组合：按用户查订单、按城市与姓名定位联系人、按状态与时间范围筛选任务。联合索引（Composite Index）将多个列组织在同一棵 B+ 树上，是 OLTP 系统中最常用的索引形态。

然而，"建了联合索引"与"查询真正高效地使用了联合索引"之间存在显著差距。覆盖索引可以消除回表，索引下推（Index Condition Pushdown, ICP）可以在回表前过滤无效行，前缀索引可以在长字符串列上控制索引体积——这些机制都建立在联合索引的存储结构之上。

本文从联合索引的 B+ 树组织方式出发，系统讲解最左前缀原则、范围查询对后续列的截断效应、覆盖索引的设计与量化、ICP 的执行路径、前缀索引的选择性计算，以及 MySQL 8.0 引入的索引跳跃扫描（Index Skip Scan）。文中示例基于如下测试表，读者可在本地复现 EXPLAIN 输出：

```sql
CREATE TABLE contacts (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    city        VARCHAR(50)     NOT NULL,
    name        VARCHAR(100)    NOT NULL,
    age         TINYINT         NOT NULL,
    zipcode     VARCHAR(10)     NOT NULL,
    lastname    VARCHAR(50)     NOT NULL,
    firstname   VARCHAR(50)     NOT NULL,
    email       VARCHAR(255)    NOT NULL,
    phone       VARCHAR(20)     DEFAULT NULL,
    status      TINYINT         NOT NULL DEFAULT 1,
    created_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_city_name_age (city, name, age),
    KEY idx_zip_last_first (zipcode, lastname, firstname),
    KEY idx_status_created (status, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

分析索引行为时，建议始终结合 `EXPLAIN` 或 `EXPLAIN FORMAT=JSON` 观察 `type`、`key`、`key_len`、`rows`、`filtered` 与 `Extra` 字段；MySQL 8.0.18+ 可用 `EXPLAIN ANALYZE` 获取实际执行时间与循环次数。

## 一、联合索引的存储结构

### 1. 联合索引的 B+ 树如何组织

联合索引是在多个列上建立的单一 B+ 树，而不是"每列各建一棵树"。InnoDB 按索引定义中列的顺序，依次比较各列的值来决定键值的排序位置：

```sql
CREATE INDEX idx_city_name_age ON contacts (city, name, age);
```

这棵 B+ 树的叶节点中，每条索引项逻辑上是一个三元组 `(city, name, age)`，后面还附带主键 `id` 用于回表。排序规则等价于 SQL 中的多列 ORDER BY：

```sql
ORDER BY city ASC, name ASC, age ASC
```

具体而言：

- 先按第一列 `city` 的字典序排序。
- 当 `city` 相同时，再按第二列 `name` 排序。
- 当 `city` 和 `name` 都相同时，再按第三列 `age` 排序。

可以用电话簿来类比：电话簿先按姓氏排序，同一姓氏内再按名字排序，同一姓名内再按中间名或后缀排序。联合索引 `(city, name, age)`的组织方式与此完全一致——第一列是"大类"，后续列是"细分类"。

假设表中有如下数据，索引 `idx_city_name_age` 叶节点中的逻辑排列顺序为：

| city | name | age | id（主键） |
|------|------|-----|--------|
| 北京   | 李四   | 25  | 1003   |
| 北京   | 李四   | 30  | 1007   |
| 北京   | 张三   | 28  | 1001   |
| 北京   | 王五   | 22  | 1012   |
| 上海   | 赵六   | 35  | 1005   |
| 上海   | 钱七   | 40  | 1009   |
| 深圳   | 孙八   | 29  | 1002   |

注意排序细节：

- "北京"的所有记录排在一起，其内部按 `name` 排序："李四" < "张三" < "王五"（字典序）。
- 同为"北京 + 李四"的两条记录，再按 `age` 排序：25 < 30。
- 不同 `city` 之间，仅比较 `city`，不会跨城市按 `name` 全局排序——"上海/赵六"不会插入到"北京"各条目之间。

这一有序性决定了优化器能否利用索引做范围扫描、排序剪枝，以及最左前缀原则是否成立。B+ 树非叶节点同样只存储索引键的前缀与子节点指针；叶节点之间通过双向链表连接，便于在同一 `city` 范围内顺序扫描所有`(name, age)` 组合。

从物理存储角度，InnoDB 将 `(city, name, age, id)` 作为一个复合键写入 B+ 树节点。变长列（VARCHAR）的键长不固定，会影响每个节点能容纳的键数量，进而影响树高。这也是联合索引列不宜过多、不宜纳入大字段的原因之一。

### 2. 最左前缀原则

B+ 树索引只能从最左列开始，连续利用有序性。这称为最左前缀原则（Leftmost Prefix Rule）：索引 `(a, b, c)` 的有序性对查询 `(a)`、`(a, b)`、`(a, b, c)` 有效，但对 `(b)`、`(c)`、`(b, c)` 无效，因为 B+ 树节点键值并非按 `b` 或 `c` 全局排序。

#### 可以使用索引的查询模式

以下查询可以有效利用 `idx_city_name_age (city, name, age)`：

```sql
-- 模式 1：仅第一列等值
SELECT * FROM contacts WHERE city = '北京';

-- 模式 2：前两列等值
SELECT * FROM contacts WHERE city = '北京' AND name = '张三';

-- 模式 3：三列等值
SELECT * FROM contacts
WHERE city = '北京' AND name = '张三' AND age = 28;

-- 模式 4：第一列等值 + 第二列范围
SELECT * FROM contacts
WHERE city = '北京' AND name LIKE '张%';

-- 模式 5：第一列范围
SELECT * FROM contacts WHERE city BETWEEN '北京' AND '上海';
```

对上述查询执行 EXPLAIN，典型输出如下（数值因数据量而异）：

```sql
EXPLAIN SELECT * FROM contacts WHERE city = '北京';
-- type=ref, key=idx_city_name_age, key_len=202, ref=const, rows≈120, Extra=NULL

EXPLAIN SELECT * FROM contacts
WHERE city = '北京' AND name = '张三' AND age = 28;
-- type=ref, key=idx_city_name_age, key_len=605, ref=const,const,const, rows=1
```

解读要点：

- `type = ref`：通过索引等值查找，命中索引前缀。
- `key_len` 随使用的列数增加：仅 `city` 时约 202 字节；加上 `name`、`age` 后增至 605，说明三列均参与匹配。
- `ref` 出现多个 `const` 表示对应列均为等值条件。
- `Extra` 为空表示 `SELECT *` 需回表读取不在索引中的列（如 `email`）。

#### 不能使用索引的查询模式

以下查询无法有效利用 `idx_city_name_age` 的有序性：

```sql
-- 跳过第一列，直接从第二列过滤
SELECT * FROM contacts WHERE name = '张三';

-- 跳过第一列
SELECT * FROM contacts WHERE age = 28;

-- 仅中间列 + 最后一列
SELECT * FROM contacts WHERE name = '张三' AND age = 28;
```

```sql
EXPLAIN SELECT * FROM contacts WHERE name = '张三';
-- type=ALL, key=NULL, rows≈500000, Extra=Using where
```

`type = ALL` 表示全表扫描。B+ 树按 `(city, name, age)` 排序，`name = '张三'` 的记录分散在各个 `city`分组中，无法通过单一有序扫描定位，优化器只能遍历聚簇索引全部行并在 Server 层过滤。

#### 跳过中间列的情况

当查询包含第一列和第三列，但跳过第二列时，只能部分利用索引：

```sql
SELECT * FROM contacts WHERE city = '北京' AND age = 28;
```

```sql
EXPLAIN SELECT * FROM contacts WHERE city = '北京' AND age = 28;
-- type=ref, key_len=202（仅 city）, Extra=Using where
```

索引仅用于 `city = '北京'` 的定位（`key_len` 只覆盖第一列），`age = 28` 无法通过索引有序性进一步剪枝——同一 `city` 下不同`name` 的 `age` 值交错排列，不满足"先按 name 再按 age"的连续区间扫描。`Extra = Using where` 表示 Server 层（或 ICP 层）需对索引扫描结果额外过滤 `age`。

若业务中频繁出现 `WHERE city = ? AND age = ?` 而很少按 `name` 过滤，应考虑调整索引为 `(city, age, name)` 或额外建立`(city, age)` 索引，但需评估与现有查询的冲突。

#### key_len 与索引利用深度的验证

`key_len` 是判断联合索引实际使用了多少列的可靠依据。MySQL 文档中对各类型的长度计算规则如下（简化）：

| 类型                 | key_len 计算           |
|--------------------|----------------------|
| TINYINT            | 1                    |
| INT                | 4                    |
| BIGINT             | 8                    |
| VARCHAR(N) utf8mb4 | N × 4 + 2（变长额外 2 字节） |
| 允许 NULL            | 额外 +1                |

对 `idx_city_name_age (city, name, age)`，结合 utf8mb4 字符集计算：

- 仅 `city`：约 202 字节（50 × 4 + 2）。
- `city + name`：约 604 字节（202 + 100 × 4 + 2）。
- 三列全用：约 605 字节（202 + 402 + 1）。

通过对比不同 SQL 的 `key_len`，可以验证优化器是否如预期使用了索引前缀，而不被 `possible_keys` 的"可能使用"所误导。

### 3. 范围查询对后续列的影响

范围条件（`>`、`<`、`>=`、`<=`、`BETWEEN`、`<>`、`LIKE 'prefix%'`）会破坏联合索引中"范围列右侧列"的有序性利用。原因是：B+ 树在范围列上仍能顺序扫描，但范围列取值的"跳跃"使得后续列不再形成单一连续有序区间。

#### 范围条件后面的列无法使用索引

```sql
SELECT * FROM contacts
WHERE city = '北京' AND age > 25;
```

索引 `(city, name, age)` 中，列顺序是 city → name → age。查询跳过了 `name`，直接对 `age` 做范围条件。即使 `city` 等值匹配，由于中间缺少`name` 条件，`age` 的有序性无法在该 `city` 分组内直接用于范围剪枝（见上一节）。若写成：

```sql
SELECT * FROM contacts
WHERE city = '北京' AND name > '李' AND age > 25;
```

此时 `city` 等值、`name` 范围匹配后，`age` 列同样无法继续用于精确过滤或排序——`name > '李'` 已是范围条件，其右侧的 `age` 被截断。

```sql
EXPLAIN SELECT * FROM contacts
WHERE city = '北京' AND name > '李' AND age > 25;
-- type=range, key=idx_city_name_age, key_len=604, Extra=Using index condition
```

`type = range`：`city` 等值 + `name` 范围。`age > 25` 若出现在索引中，可能通过 ICP 在存储引擎层过滤（`Using index condition`），但不再是"按索引有序性直接定位"的精确匹配。

#### 等值 + 范围 + 等值的情况

考虑索引 `(a, b, c)` 上的查询：

```sql
SELECT * FROM t WHERE a = 1 AND b > 10 AND c = 5;
```

索引利用情况：

- `a = 1`：等值定位，有效。
- `b > 10`：在 `a = 1` 分组内做范围扫描，有效。
- `c = 5`：无法通过 B+ 树有序性加速——在 `a = 1` 且 `b > 10` 的扫描结果中，`c` 的值不是单调递增的，不能二分定位。

若 `c` 的过滤性极强（例如唯一标识某个子集），可以考虑把列顺序调整为 `(a, c, b)`：

```sql
-- 原索引
KEY idx_acb (a, b, c)

-- 针对 WHERE a = ? AND c = ? AND b > ? 的查询，调整为：
KEY idx_acb_v2 (a, c, b)
```

```sql
SELECT * FROM t WHERE a = 1 AND c = 5 AND b > 10;
```

此时 `a` 等值、`c` 等值定位后，`b > 10` 可在 `(a, c)` 分组内做范围扫描。列顺序必须服务于**实际的高频查询模式**，不能孤立地讨论"高区分度列放前面"。

#### 范围截断与 ORDER BY、LIKE

```sql
-- 可借助索引排序：city 等值后 name 在索引中有序
SELECT * FROM contacts WHERE city = '北京' ORDER BY name;

-- 无法借助索引排序：city 为范围，name 全局无序，Extra 出现 Using filesort
SELECT * FROM contacts WHERE city > '北京' ORDER BY name;

-- LIKE '张%' 视为 name 上的范围，后续 age 列不能继续精确匹配
SELECT * FROM contacts WHERE city = '北京' AND name LIKE '张%';

-- LIKE '%三' 前缀通配，name 列无法走索引
SELECT * FROM contacts WHERE city = '北京' AND name LIKE '%三';
```

### 4. 联合索引的列顺序设计

列顺序没有 universally optimal 的公式，必须结合查询模式、数据分布与排序需求综合决策。

**区分度 vs 等值列**：所有列均为等值条件时，高区分度列放前往往更优（如 `(user_id, gender)` 服务
`WHERE user_id = ? AND gender = ?`）；低区分度等值列 + 高区分度范围列时，等值列必须放前（如 `(status, created_at)` 服务
`WHERE status = 1 AND created_at >= ?`）。

**查询模式驱动设计**：收集 TOP N 慢查询，列出 WHERE / ORDER BY / GROUP BY 列后决定顺序。示例：

```sql
-- 高频查询 1
SELECT id, amount FROM orders
WHERE user_id = ? AND status = 1
ORDER BY created_at DESC LIMIT 20;

-- 高频查询 2
SELECT COUNT(*) FROM orders
WHERE user_id = ? AND created_at >= ?;

-- 推荐索引
KEY idx_user_status_created (user_id, status, created_at)
```

Q1 中 `user_id`、`status` 等值，`created_at` 排序——索引顺序完全匹配。Q2 中 `user_id` 等值、`created_at` 范围——仍可使用前缀`(user_id, created_at)`，但 `status` 夹在中间导致 Q2 无法利用 `created_at` 范围（若 Q2 频率更高，可能需要`(user_id, created_at, status)` 或两条索引）。

单一联合索引无法完美服务所有 SQL，需在 Index Merge 与"一条主索引 + 少量妥协"之间权衡。排序需求方面，`WHERE dept_id = 5 ORDER BY salary DESC` 适合 `(dept_id, salary)`；MySQL 8.0+ 可声明降序索引 `(dept_id, salary DESC)`。`(a, b, c)` 已存在时，`(a)` 通常冗余（最左前缀已覆盖），但 `(b, c)` 不冗余。

## 二、覆盖索引

### 1. 什么是覆盖索引

覆盖索引（Covering Index）不是一种独立的索引类型，而是指：**查询所需的所有列都包含在某条索引中，优化器可以直接从索引叶节点读取结果，无需回表到聚簇索引**。

InnoDB 二级索引叶节点存储的是 `(索引列..., 主键列)`。当 SELECT 的列、WHERE 中需要读取的列（在 Index Only Scan 场景下）以及 ORDER BY 涉及的列均落在同一索引内时，引擎执行 Index Only Scan，`EXPLAIN` 的 `Extra` 列显示 `Using index`。

```sql
EXPLAIN SELECT city, name, age FROM contacts WHERE city = '北京';
-- type=ref, key=idx_city_name_age, Extra=Using index

EXPLAIN SELECT city, name, age, email FROM contacts WHERE city = '北京';
-- type=ref, key=idx_city_name_age, Extra=NULL（email 不在索引中，需回表）
```

`Using index` 是覆盖扫描的标志：所有需要的列均从 `idx_city_name_age` 获得，不回表。对比可见，仅增加一个非索引列就会使`Extra` 失去 `Using index`，触发回表。

聚簇索引本身也可视为"覆盖"主键查询，但通常所说的覆盖索引特指二级索引避免回表的场景。

### 2. 覆盖索引的设计方法

从慢查询日志提取高频 SQL，列出 WHERE 过滤列、ORDER BY/GROUP BY 列与 SELECT 投影列。示例：订单列表页仅展示`order_no, amount, created_at`：

```sql
SELECT order_no, amount, created_at
FROM orders
WHERE user_id = 12345 AND status = 1
ORDER BY created_at DESC LIMIT 20;

-- 覆盖索引：等值列在前，排序列居中，投影列追加在末尾
CREATE INDEX idx_user_status_created_cover
ON orders (user_id, status, created_at, order_no, amount);

EXPLAIN SELECT order_no, amount, created_at
FROM orders
WHERE user_id = 12345 AND status = 1
ORDER BY created_at DESC LIMIT 20;
-- 期望：Extra=Using index，无 Using filesort
```

覆盖索引以空间换时间，需权衡：

| 维度          | 窄索引（仅过滤列）    | 宽覆盖索引                   |
|-------------|--------------|-------------------------|
| 索引体积        | 小            | 大，随 VARCHAR/DECIMAL 列膨胀 |
| 读性能         | 需回表，随机 I/O 多 | Index Only Scan，顺序读索引页  |
| 写性能         | 维护成本低        | 每次 UPDATE 涉及列都要改索引      |
| Buffer Pool | 同样内存缓存更多索引项  | 缓存条目减少，可能降低命中率          |

原则：对 QPS 极高、延迟敏感的只读或读多写少查询，覆盖索引收益最大；对宽列（TEXT、长 VARCHAR）、频繁 UPDATE 的列，谨慎纳入索引。

### 3. 覆盖索引的收益量化

假设 `idx_city_name_age` 匹配 1000 行：无覆盖时需 1000 次回表（缓冲池未命中时为随机 I/O）；有覆盖时仅扫描二级索引叶节点，0 次回表。二级索引叶节点比聚簇索引更紧凑（仅存索引列 + 主键），同样行数占用更少页，对 Buffer Pool 更友好。

```sql
EXPLAIN ANALYZE SELECT city, name, age FROM contacts WHERE city = '北京';
EXPLAIN ANALYZE SELECT city, name, age, email, phone FROM contacts WHERE city = '北京';
-- 对比实际耗时与 loops 差异
```

典型高收益场景：

- **分页列表**：`KEY idx_cat_pub (category_id, publish_at, id, title)` 使 `LIMIT 20` 可在索引层早停。
- **COUNT**：`SELECT COUNT(*) FROM orders WHERE user_id = ? AND status = 1` 可在 `(user_id, status)` 索引层计数。
- **延迟关联**：深分页时内层 `SELECT id ... LIMIT offset, n` 走覆盖索引，外层仅回表 n 次。

### 4. 覆盖索引的限制

- **SELECT \***：几乎无法覆盖；ORM 默认 `SELECT *` 是覆盖失效的常见原因。
- **索引列数与大小**：InnoDB 单索引最多 16 列，键长约 3072 字节；纳入过多宽列会增加树高与写成本。
- **频繁更新的列**：如 `last_login_at` 每次登录都 UPDATE，不宜纳入宽覆盖索引。
- **最左前缀**：`SELECT city, name FROM contacts WHERE name = '张三'` 无法走 `(city, name, age)`，覆盖无从谈起。

## 三、索引下推（Index Condition Pushdown, ICP）

### 1. ICP 之前的执行方式

MySQL 5.6 之前，存储引擎与 Server 层的职责划分较为僵硬：

1. Server 层解析 SQL，将**能用索引前缀处理的条件下推**给 InnoDB。
2. InnoDB 按索引前缀扫描，对每条匹配的索引项，将其对应主键交给 Server 层。
3. Server 层根据主键回表（或引擎内部回表）获取完整行。
4. Server 层对完整行应用**剩余 WHERE 条件**（包括索引中但不能按前缀利用的列）。

问题出在步骤 2–4：联合索引 `(zipcode, lastname, firstname)` 上查询：

```sql
SELECT * FROM contacts
WHERE zipcode = '95054' AND lastname LIKE '%son';
```

- 索引前缀：`zipcode = '95054'` 可定位。
- `lastname LIKE '%son'`：前缀通配，不能按 B+ 树有序性剪枝，属于"剩余条件"。

无 ICP 时，InnoDB 扫描所有 `zipcode = '95054'` 的索引项，**每一条都回表**，再由 Server 层对完整行做 `lastname LIKE '%son'`过滤。若 `95054` 下有 5000 人，则 5000 次无效回表可能占绝大多数。

### 2. ICP 的优化原理

Index Condition Pushdown（MySQL 5.6+，默认开启 `optimizer_switch='index_condition_pushdown=on'`）将部分"剩余条件"下推到存储引擎，在**回表之前**用索引项中的列值过滤。

有 ICP 时：

1. InnoDB 扫描 `zipcode = '95054'` 的索引项。
2. 对每条索引项，直接读取索引中的 `lastname` 值，执行 `lastname LIKE '%son'` 判断。
3. **仅当条件满足时**，才拿主键回表。

若 5000 条中只有 50 条 lastname 含 `son`，回表次数从 5000 降至 50。

```sql
EXPLAIN SELECT * FROM contacts
WHERE zipcode = '95054' AND lastname LIKE '%son';
-- type=ref, key=idx_zip_last_first, rows≈5000, Extra=Using index condition
```

`Using index condition` 表示 ICP 生效：`lastname` 条件在引擎层用索引项过滤，而非回表后过滤。ICP 过滤使用的是**索引项中已存储的列值**，不要求该列满足最左前缀匹配规则——`lastname` 是索引第二列，虽不能用于 B+ 树定位，但仍可在索引项中读取并判断。

### 3. ICP 的适用条件

ICP 的适用边界如下：

| 条件    | 说明                                                             |
|-------|----------------------------------------------------------------|
| 索引类型  | 仅适用于二级索引（聚簇索引无回表概念）                                            |
| 访问类型  | `range`、`ref`、`eq_ref`、`ref_or_null`；`ALL`/`index`/`const` 不适用 |
| 条件下推  | 条件列必须在索引定义中，但不要求满足最左前缀                                         |
| 与覆盖索引 | 已 `Using index` 时 ICP 收益有限；默认开启 `index_condition_pushdown=on`  |

测试环境可对比开关效果：

```sql
SET optimizer_switch = 'index_condition_pushdown=off';
EXPLAIN ANALYZE SELECT * FROM contacts
WHERE zipcode = '95054' AND lastname LIKE '%son';

SET optimizer_switch = 'index_condition_pushdown=on';
EXPLAIN ANALYZE SELECT * FROM contacts
WHERE zipcode = '95054' AND lastname LIKE '%son';
```

### 4. 具体示例

沿用 `idx_zip_last_first (zipcode, lastname, firstname)`，核心查询与执行路径对比如下：

```sql
SELECT * FROM contacts
WHERE zipcode = '95054' AND lastname LIKE '%son';
```

| 阶段          | 无 ICP                   | 有 ICP                 |
|-------------|-------------------------|-----------------------|
| 索引定位        | zipcode = '95054'       | 同左                    |
| lastname 过滤 | 回表后 Server 层            | 索引项层 Engine           |
| 回表次数        | 等于 zipcode 匹配行数（如 5000） | 等于两条件均满足行数（如 50）      |
| Extra       | Using where             | Using index condition |

#### 再例：索引 `(dept_id, age, salary)` 上 `WHERE dept_id=5 AND age>30 AND salary>8000`

`dept_id` 等值、`age` 范围后，`salary > 8000` 不能用于 B+ 树定位，但可 ICP 下推；期望`type=range, Extra=Using index condition`。

ICP 不能下推不在索引中的列，不能替代最左前缀定位，也不能替代覆盖索引。高 QPS 场景仍应优先设计覆盖索引。

## 四、前缀索引

### 1. 为什么需要前缀索引

对 `VARCHAR(255)`、`TEXT` 等长字符串列建立完整索引时，每个索引项可能占用数百至数千字节，导致：

- B+ 树节点能容纳的键数量锐减，树高增加，单次查找 IO 增多。
- 索引总体积膨胀，Buffer Pool 中缓存的有效条目减少。
- INSERT/UPDATE 维护索引的开销上升。

前缀索引（Prefix Index）只对列值的前 N 个字符（utf8mb4 下为字节/字符，取决于版本与定义）建索引：

```sql
CREATE INDEX idx_email_prefix ON contacts (email(20));
```

对比（示意，100 万行，email 平均长度 30 字符）：

| 索引类型      | 单键长约    | 索引总大小（量级） | 单页键数 |
|-----------|---------|-----------|------|
| 完整 email  | ~120 字节 | ~200 MB   | 少    |
| email(10) | ~42 字节  | ~70 MB    | 多    |
| email(20) | ~82 字节  | ~130 MB   | 中    |

前缀越短，索引越小，但区分度（选择性）下降，碰撞增多，查询需扫描更多行并在 Server 层或 ICP 层验证。

### 2. 如何选择前缀长度

#### 区分度（选择性）的计算方法

选择性定义为：

```
选择性 = COUNT(DISTINCT 索引键值) / COUNT(*)
```

完整列选择性：

```sql
SELECT
    COUNT(DISTINCT email) / COUNT(*) AS full_selectivity
FROM contacts;
```

前缀选择性：

```sql
SELECT
    COUNT(DISTINCT LEFT(email, 5))  / COUNT(*) AS sel_5,
    COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) AS sel_10,
    COUNT(DISTINCT LEFT(email, 15)) / COUNT(*) AS sel_15,
    COUNT(DISTINCT LEFT(email, 20)) / COUNT(*) AS sel_20,
    COUNT(DISTINCT LEFT(email, 30)) / COUNT(*) AS sel_30
FROM contacts;
```

示例输出：

```
+------------------+--------+--------+--------+--------+
| sel_5            | sel_10 | sel_15 | sel_20 | sel_30 |
+------------------+--------+--------+--------+--------+
| 0.012000         | 0.450000 | 0.920000 | 0.985000 | 0.999800 |
+------------------+--------+--------+--------+--------+
```

解读：

- 长度 5：选择性 1.2%，大量碰撞，不可用。
- 长度 10：45%，仍偏低。
- 长度 15：92%，接近完整列。
- 长度 20：98.5%，通常可接受（一般目标 > 95% 或达到完整列的 90% 以上）。
- 长度 30：几乎等于完整列，但索引更大。

#### 找到区分度足够的最短前缀

实践步骤：

1. 计算 `full_selectivity` 作为上限。
2. 从短到长递增 N，直到 `COUNT(DISTINCT LEFT(col, N)) / COUNT(*) >= 0.95 * full_selectivity`（或业务可接受阈值）。
3. 在该 N 上建前缀索引并 EXPLAIN 验证 `rows` 估算。

```sql
-- 选定 email(20)
CREATE INDEX idx_email_20 ON contacts (email(20));

EXPLAIN SELECT * FROM contacts WHERE email = 'alice@example.com';
```

前缀索引等值查询时，InnoDB 先匹配 `LEFT(email, 20)`，可能对多条碰撞记录回表，再用完整 `email` 验证。

#### 具体的计算示例

```sql
-- 辅助：查看碰撞最严重的 prefix
SELECT
    LEFT(email, 20) AS email_prefix,
    COUNT(*) AS cnt
FROM contacts
GROUP BY LEFT(email, 20)
HAVING cnt > 1
ORDER BY cnt DESC
LIMIT 10;
```

若最高碰撞仅 2–3 行，前缀长度足够；若某前缀对应数千行，需加长前缀或换方案。

### 3. 前缀索引的功能限制

前缀索引在等值过滤场景可用，但功能上有明显边界：

| 限制          | 原因                 | 示例                                  |
|-------------|--------------------|-------------------------------------|
| 不能 ORDER BY | 索引仅按前缀排序，与完整列排序不等价 | `ORDER BY email` 无法走 `email(20)` 索引 |
| 不能 GROUP BY | 无法按完整列值分组          | `GROUP BY email` 需临时表               |
| 不能做覆盖索引     | 索引项不含完整列值          | `SELECT email` 仍需回表验证               |
| 不能 UNIQUE   | 不同完整值可能共享相同前缀      | `UNIQUE(email(20))` 无法保证业务唯一        |

| 场景                      | 前缀索引是否合适    |
|-------------------------|-------------|
| 长 URL/email 等值查找        | 合适          |
| 长文本 ORDER BY / GROUP BY | 不合适         |
| 需要覆盖索引或 UNIQUE          | 不合适，用完整列或哈希 |

### 4. 替代方案：哈希辅助列

对极长字符串，前缀索引的碰撞与功能限制可能不可接受。常见替代：增加固定长度哈希列，对其建索引。

#### CRC32 辅助列

```sql
ALTER TABLE contacts
ADD COLUMN email_crc INT UNSIGNED NOT NULL DEFAULT 0,
ADD INDEX idx_email_crc (email_crc);

-- 应用层或触发器维护
UPDATE contacts SET email_crc = CRC32(email);
```

查询：

```sql
SELECT * FROM contacts
WHERE email_crc = CRC32('alice@example.com')
  AND email = 'alice@example.com';
```

第二条件用于消除 CRC32 碰撞（概率低但业务上必须验证）。

CRC32 仅 4 字节，索引极紧凑；但无加密语义，且碰撞虽少却存在。

#### MD5 / SHA2 辅助列

```sql
ALTER TABLE contacts
ADD COLUMN email_hash BINARY(16)
    AS (UNHEX(MD5(email))) STORED,
ADD INDEX idx_email_hash (email_hash);
```

MySQL 8.0 支持生成列（Generated Column）自动维护。MD5 碰撞概率极低，索引固定 16 字节。

```sql
SELECT * FROM contacts
WHERE email_hash = UNHEX(MD5('alice@example.com'))
  AND email = 'alice@example.com';
```

#### 优缺点分析

| 方案       | 优点         | 缺点                           |
|----------|------------|------------------------------|
| 前缀索引     | 实现简单，无冗余列  | 碰撞、无 ORDER BY/GROUP BY/覆盖/唯一 |
| CRC32    | 索引小、查找快    | 需回表验证、碰撞、需维护哈希               |
| MD5/SHA2 | 碰撞极少、索引固定长 | 存储冗余、写时计算、仍建议回表验证            |
| 完整列索引    | 功能完整       | 体积大，长列不推荐                    |

MySQL 8.0 对长字符串还可考虑 `FULLTEXT` 索引（全文检索语义，非等值 B+ 树）或外部搜索引擎，已超出前缀索引讨论范围。

## 五、索引跳跃扫描（Index Skip Scan）

### 1. MySQL 8.0 新特性

在 MySQL 8.0 之前，联合索引 `(gender, age)` 无法用于仅过滤 `age` 的查询——不满足最左前缀，只能全表扫描。

Index Skip Scan 允许优化器在**第一列不在 WHERE 中、但第二列有等值条件**时，假设第一列基数较低，自动"跳过"第一列的不同取值，分别对第二列做子扫描，再合并结果。

```sql
CREATE TABLE people (
    id     INT NOT NULL AUTO_INCREMENT,
    gender CHAR(1) NOT NULL,
    age    INT NOT NULL,
    name   VARCHAR(50),
    PRIMARY KEY (id),
    KEY idx_gender_age (gender, age)
) ENGINE=InnoDB;

EXPLAIN SELECT * FROM people WHERE age = 30;
```

```sql
EXPLAIN SELECT * FROM people WHERE age = 30;
-- type=range, key=idx_gender_age, Extra=Using index for skip scan
```

`Using index for skip scan` 表示优化器选择跳跃扫描，逻辑上等价于对每个 `gender` distinct 值分别执行`WHERE gender = ? AND age = 30` 并合并结果。

### 2. 适用条件

#### 第一列的基数要低

Skip Scan 的成本 ≈ `distinct(第一列)` × 单次 `(第一列等值 + 第二列条件)` 扫描。第一列唯一值越少，越划算。

```sql
-- gender 仅 M/F：2 次子扫描，Skip Scan 理想
WHERE age = 30  -- 可用 Skip Scan on (gender, age)

-- user_id 接近行数：Skip Scan 等价于数百万次子扫描，绝不可能选用
KEY idx_user_age (user_id, age)
WHERE age = 30  -- 不会 Skip Scan
```

#### 第二列条件选择性要高

若 `age = 30` 匹配表内 30% 行，每次子扫描仍很大，Skip Scan 总成本可能高于全表扫描。优化器基于统计信息（`histogram`、NDV）估算。

#### 具体示例

```sql
-- 表 contacts 上假设有 (status, created_at) 索引，status 仅 3 个枚举值
EXPLAIN SELECT id FROM contacts
WHERE created_at >= '2026-01-01' AND created_at < '2026-02-01';
```

若 `status` 基数低且 `created_at` 范围选择性较好，可能触发 Skip Scan；`Extra` 含 `Using index for skip scan`。

控制开关（测试用）：

```sql
SET optimizer_switch = 'skip_scan=on';  -- 默认 on（8.0.13+）
```

### 3. 什么时候 Skip Scan 不如全表扫描

以下情况优化器通常拒绝 Skip Scan：

| 条件        | 原因                        |
|-----------|---------------------------|
| 第一列基数高    | 子扫描次数爆炸                   |
| 第二列条件选择性差 | 每次子扫描接近全表                 |
| 表很小       | 全表扫描更便宜                   |
| 需要回表的大量宽行 | 多次索引扫描 + 回表总 IO 超过顺序扫聚簇索引 |
| 第一列统计信息不准 | 可能误判，需 `ANALYZE TABLE`    |

```sql
-- 100 行小表
EXPLAIN SELECT * FROM tiny_table WHERE age = 30;
-- type = ALL，不会 skip scan
```

对比验证：

```sql
EXPLAIN FORMAT=JSON SELECT * FROM people WHERE age = 30\G
```

查看 `access_type` 是否为 `index_skip_scan` 及 `rows_examined_per_scan`。

Skip Scan 是优化器在特定统计特征下的自动改写，**不应**故意把低基数列放在联合索引最左 solely 为了 Skip Scan——首要原则仍是服务最左前缀匹配的主流查询。Skip Scan 只是没有合适前缀时的补充手段。

## 总结

联合索引、覆盖索引、索引下推与前缀索引，构成了 InnoDB 二级索引进阶优化的核心工具链。它们的共同基础是 B+ 树按列顺序物理有序这一存储特性。

**联合索引**：单列 B+ 树，按定义顺序排序，类似电话簿。最左前缀原则决定哪些 WHERE 模式能利用索引定位；范围条件截断后续列的有序性利用。列顺序应服从真实查询模式（等值列在前、范围列在后、排序列纳入设计），而非机械地"高区分度优先"。

**覆盖索引**：当查询列全部落在索引中时，Index Only Scan 消除回表，`Extra` 显示 `Using index`。对高频列表、COUNT、延迟关联场景收益显著；需权衡索引宽度与写成本，且 `SELECT *` 几乎永远无法覆盖。

**索引下推**：MySQL 5.6+ 在二级索引扫描路径上，将索引中但不能按前缀定位的条件下推到存储引擎，在回表前过滤，`Extra` 显示`Using index condition`。典型场景是联合索引第一列等值 + 后续列 LIKE/范围/filter。ICP 减少回表次数，但不能替代覆盖索引，也不能绕过最左前缀定位问题。

**前缀索引**：控制长字符串索引体积，通过 `COUNT(DISTINCT LEFT(col, N)) / COUNT(*)` 选择最短可用前缀。限制包括：无 ORDER BY/GROUP BY、无覆盖、无唯一。长列等值查找可配合 CRC32/MD5 哈希辅助列。

**Index Skip Scan**：MySQL 8.0 在首列低基数、次列高选择性时，自动跳过首列 distinct 值做多次子扫描。首列基数过高或表过小时不如全表扫描。

分析一条 SQL 时，建议按以下顺序检查：

1. `key` 与 `key_len`：是否命中预期联合索引？用了几列？
2. `type`：是否为 `ref`/`range` 而非 `ALL`？
3. `Extra`：是否有 `Using index`（覆盖）？`Using index condition`（ICP）？`Using filesort`（排序未走索引）？`Using index for skip scan`（8.0 跳跃扫描）？
4. 匹配行数与回表次数：是否需要覆盖索引或改写 SQL？
5. 长字符串列：前缀索引或哈希列是否更合适？

索引设计没有银弹。理解存储结构背后的有序性、Server 层与 Engine 层的分工，才能在高 QPS 生产环境中做出可验证、可维护的索引决策。
