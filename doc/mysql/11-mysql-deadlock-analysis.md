---
title: 死锁的产生条件、检测机制与实战排查
summary: 结合 InnoDB 锁类型，拆解典型死锁场景的形成过程，深入等待图检测算法，并给出基于死锁日志的实战排查方法与预防策略。
created: 2026-07-02
updated: 2026-07-12
tags: MySQL, InnoDB, 死锁, 锁, 事务
---

# 死锁的产生条件、检测机制与实战排查

InnoDB 的行锁体系基于四种基本锁类型：Record Lock、Gap Lock、Next-Key Lock 与 Insert Intention Lock。死锁不是"锁很多"的副产品，而是多个事务以不同顺序争夺同一组锁资源时，等待链闭合的必然结果。

线上排查时，勿将 `Error 1213 (40001)` 当作数据库故障——它是 InnoDB 主动打破环路的正常机制。关注环路成因、访问顺序优化与应用重试策略即可。

## 一、死锁的四个必要条件在 InnoDB 中的体现

经典并发理论将死锁归结为四个必要条件同时成立。InnoDB 行锁机制为这四个条件提供了完整实现土壤。

### 1. 互斥：排它锁在同一时刻只能被一个事务持有

InnoDB 的 Record Lock 和 Gap Lock 遵循排它语义。对同一条索引记录，两个事务不能同时持有 X Lock；对同一间隙，Gap Lock 与 Insert Intention Lock 不兼容。

| 已持有 \ 请求         | X（Record） | Gap | Insert Intention |
|------------------|-----------|-----|------------------|
| X（Record）        | 不兼容       | 兼容  | 不兼容              |
| Gap              | 兼容        | 兼容  | 不兼容              |
| Insert Intention | 不兼容       | 不兼容 | 兼容               |

> **说明**：Record Lock 锁定索引记录，Gap Lock 锁定记录之间的间隙，两者作用在不同资源上，互不阻塞。两个 Gap Lock 也互相兼容（同间隙可被多事务共享）。死锁的根源在于 Gap Lock 与 Insert Intention Lock 互斥（见第三行/第三列）。

`UPDATE`、`DELETE`、`SELECT ... FOR UPDATE` 等当前读在命中记录时，会在聚簇索引（及涉及的二级索引）上申请 X 型 Record Lock：

```sql
-- 会话 A
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;
-- 持有 id=1 聚簇索引 X Record Lock

-- 会话 B（并发）
BEGIN;
UPDATE account SET balance = balance + 50 WHERE id = 1;
-- 请求同一 X 锁，进入等待（单向等待，尚未死锁）
```

REPEATABLE READ 下，当前读扫描到的间隙会被加 Gap Lock，阻止其他事务在该间隙插入。

### 2. 持有并等待：事务持有部分锁的同时请求更多锁

InnoDB 遵循两阶段锁协议（2PL）：事务逐步获取锁，提交或回滚前不释放已持有的锁。

```sql
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 持有 id=1 的 X 锁
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 等待 id=2 的 X 锁
```

单个事务可同时处于占有者与请求者角色。锁持有时间与事务生命周期绑定，事务中间调用外部接口会放大死锁窗口。

### 3. 不可剥夺：事务锁不能被其他事务抢占

InnoDB 没有锁抢占机制。已持有的锁只能由该事务 `COMMIT` 或 `ROLLBACK` 释放。`innodb_lock_wait_timeout` 到期后终止的是**等待方**，被等方持有的锁不受影响。

检测到死锁时，InnoDB **回滚整个参与环路的事务之一**，而非剥夺单个锁：

| 机制                | 触发条件   | 被终止对象   | 环路是否打破 |
|-------------------|--------|---------|--------|
| 死锁检测              | 等待图出现环 | 权重较小的事务 | 是      |
| lock_wait_timeout | 单向等待超时 | 等待方事务   | 否      |

### 4. 循环等待：等待关系形成环

当事务 A 等待 B 持有的锁，而 B 又等待 A 持有的锁时，等待图出现环，死锁成立：

```
事务 A ──等待──> B 持有的锁(id=2)
   ^                    |
   └──等待── 事务 B ──等待──> A 持有的锁(id=1)
```

除交叉更新外，二级索引路径差异、Gap/Insert Intention 冲突、批量隐含顺序、外键级联均可形成隐蔽环路。

### 5. 结合 InnoDB 锁类型的综合说明

| 锁类型                   | 典型语句                    | 死锁关联                  |
|-----------------------|-------------------------|-----------------------|
| Record Lock           | `UPDATE ... WHERE pk=?` | 交叉更新                  |
| Gap / Next-Key Lock   | RR 下 `FOR UPDATE` 范围读   | 与 Insert Intention 冲突 |
| Insert Intention Lock | `INSERT`                | 等待 Gap Lock           |
| 隐式锁                   | 普通 UPDATE               | 冲突时 materialize       |

RR 下 Gap/Next-Key Lock 扩大互斥范围；RC 可显著减少插入类死锁。分析日志：锁模式 → 加锁顺序 → 等待边是否成环。

## 二、典型死锁场景一：主键交叉更新

两个事务以相反顺序更新同一组主键行，各自持有第一个目标行的锁，再请求第二个目标行的锁，等待链闭合。这是线上最常见的死锁形式。

### 1. 建表和数据准备

```sql
DROP TABLE IF EXISTS account;
CREATE TABLE account (
    id      INT PRIMARY KEY,
    balance DECIMAL(12, 2) NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

INSERT INTO account (id, balance) VALUES
    (1, 1000.00), (2, 2000.00), (3, 3000.00);

SELECT @@transaction_isolation;  -- 默认 REPEATABLE-READ，本场景 RR/RC 行为一致
```

### 2. 事务 A 和事务 B 的时序

两个独立会话逐步执行（每步暂停等待对端）：

```sql
-- ========== 会话 A ==========          -- ========== 会话 B ==========
BEGIN;                                   BEGIN;

UPDATE account                           UPDATE account
SET balance = balance - 100              SET balance = balance - 50
WHERE id = 1;                            WHERE id = 2;
-- 步骤 1：持有 id=1 的 X Record Lock   -- 步骤 1：持有 id=2 的 X Record Lock

UPDATE account                           UPDATE account
SET balance = balance + 100              SET balance = balance + 50
WHERE id = 2;                            WHERE id = 1;
-- 步骤 2：等待 id=2 的 X 锁             -- 步骤 2：等待 id=1 的 X 锁
-- 死锁！其中一个会话收到 Error 1213
```

时序：`A: UPDATE id=1(获锁) → UPDATE id=2(等B)` 与 `B: UPDATE id=2(获锁) → UPDATE id=1(等A)` 并发到步骤 2 时环路闭合。

### 3. 锁的获取过程分析

A 第一步经聚簇索引定位 `id=1` 获 X Record Lock；第二步请求 `id=2` 时 B 已持有，A 进入等待且**不释放 id=1 锁**。B 对称持有
`id=2`、等待 `id=1`：

```
会话 A ──wait──> 锁(id=2) <──hold── 会话 B
   ^                                  |
   └──────── wait ── 锁(id=1) <──hold─┘
```

### 4. 死锁形成的原因

根因：**加锁顺序不一致**（A: 1→2，B: 2→1）。统一顺序则仅为单向等待。死锁需两事务各完成第一步后同时进入第二步。

### 5. 完整的复现步骤

```sql
-- 终端 1                          -- 终端 2
SET autocommit = 0;                SET autocommit = 0;
BEGIN;                             BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;
                                   UPDATE account SET balance = balance - 50 WHERE id = 2;
UPDATE account SET balance = balance + 100 WHERE id = 2;
                                   UPDATE account SET balance = balance + 50 WHERE id = 1;
-- 预期：其中一个终端 ERROR 1213，另一个 COMMIT 成功

SHOW ENGINE INNODB STATUS\G
-- 查看 LATEST DETECTED DEADLOCK 段
```

**预防**：多行更新统一按主键升序加锁，或使用单条 SQL 批量更新。

## 三、典型死锁场景二：Gap Lock 与 Insert Intention Lock 的冲突

REPEATABLE READ 的"特产"。两个事务对不存在记录做等值当前读，各获 Gap Lock；随后都 INSERT 同一间隙，Insert Intention Lock 与对方 Gap Lock 互斥，形成环路。

### 1. 在 REPEATABLE READ 下的场景

```sql
DROP TABLE IF EXISTS user_slot;
CREATE TABLE user_slot (
    id   INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(64) NOT NULL,
    UNIQUE KEY uk_name (name)
) ENGINE=InnoDB;

INSERT INTO user_slot (name) VALUES ('alice'), ('carol');
SELECT @@transaction_isolation;  -- 必须为 REPEATABLE-READ
```

### 2. 两个事务同时对不存在的记录做等值查询（FOR UPDATE）

目标：插入 `name = 'bob'`，当前不存在。

```sql
-- ========== 会话 A ==========          -- ========== 会话 B ==========
BEGIN;                                   BEGIN;

SELECT * FROM user_slot                  SELECT * FROM user_slot
WHERE name = 'bob'                       WHERE name = 'bob'
FOR UPDATE;                              FOR UPDATE;
-- 未命中，对 uk_name 上 'bob' 插入位置加 Gap Lock
-- RR 下两个 Gap Lock 可共存，步骤 1 互不阻塞
```

### 3. 都获得 Gap Lock 后的状态

`uk_name` 索引上 `('alice','carol')` 之间用于插入 `'bob'` 的间隙，被两事务各持一把 Gap Lock。无 Record Lock，因 `'bob'`尚不存在。

MySQL 8.0+ 验证：

```sql
SELECT engine_transaction_id AS trx_id, lock_mode, lock_status, index_name
FROM performance_schema.data_locks
WHERE object_name = 'user_slot';
-- 预期：lock_mode 含 GAP，lock_status = GRANTED
```

### 4. 然后都尝试 INSERT

```sql
-- 会话 A                                -- 会话 B
INSERT INTO user_slot (name)           INSERT INTO user_slot (name)
VALUES ('bob');                         VALUES ('bob');
-- 申请 Insert Intention Lock            -- 申请 Insert Intention Lock
-- 被 B 的 Gap Lock 阻塞                   -- 被 A 的 Gap Lock 阻塞
```

Insert Intention Lock 与 Gap Lock **不兼容**。INSERT 流程：定位插入位置 → 设置 Insert Intention Lock → 无 Gap 冲突则继续。

### 5. Insert Intention Lock 与 Gap Lock 的冲突机理

```
会话 A：持有 Gap(bob) ──请求──> Insert Intention(bob) ──被 B 的 Gap 阻塞
会话 B：持有 Gap(bob) ──请求──> Insert Intention(bob) ──被 A 的 Gap 阻塞
```

成环后检测回滚其一。仅单事务 INSERT 时单向等待，不死锁；关键在**双事务先 FOR UPDATE 再同时 INSERT**。

### 6. 完整的复现步骤

```sql
-- 终端 1
BEGIN;
SELECT * FROM user_slot WHERE name = 'bob' FOR UPDATE;  -- Empty set

-- 终端 2（终端 1 未 COMMIT）
BEGIN;
SELECT * FROM user_slot WHERE name = 'bob' FOR UPDATE;  -- Empty set，不阻塞

-- 终端 1
INSERT INTO user_slot (name) VALUES ('bob');  -- 阻塞

-- 终端 2
INSERT INTO user_slot (name) VALUES ('bob');  -- ERROR 1213 或终端 1 报 1213
```

**预防**：避免"先 SELECT FOR UPDATE 再 INSERT"；改用 `INSERT ... ON DUPLICATE KEY UPDATE` 或捕获唯一键冲突；评估 RC 隔离级别；缩短 FOR UPDATE 事务。

## 四、典型死锁场景三：二级索引与聚簇索引的加锁顺序不一致

两个事务通过不同二级索引定位同一行并更新时，需在二级索引与聚簇索引分别加锁。加锁路径顺序相反，可能在聚簇索引上形成死锁。

### 1. 表结构与数据

```sql
DROP TABLE IF EXISTS employee;
CREATE TABLE employee (
    id         INT PRIMARY KEY,
    emp_no     VARCHAR(32) NOT NULL,
    dept_code  VARCHAR(32) NOT NULL,
    salary     DECIMAL(12, 2) NOT NULL,
    UNIQUE KEY uk_emp_no (emp_no),
    KEY idx_dept (dept_code)
) ENGINE=InnoDB;

INSERT INTO employee (id, emp_no, dept_code, salary) VALUES
    (100, 'E001', 'D-A', 8000.00),
    (101, 'E002', 'D-A', 9000.00);
```

### 2. 通过不同索引更新同一行

```sql
-- ========== 会话 A ==========                    -- ========== 会话 B ==========
BEGIN;                                             BEGIN;

UPDATE employee                                    UPDATE employee
SET salary = salary + 100                          SET salary = salary + 200
WHERE emp_no = 'E001';                             WHERE dept_code = 'D-A';
-- 路径：uk_emp_no → 聚簇 id=100                    -- 路径：idx_dept → 聚簇 id=100
```

### 3. 一个走二级索引 A → 聚簇索引

A 路径：`uk_emp_no(E001)` 加 X 锁 → 回表 CL(id=100) 加 X 锁 → 修改。

### 4. 另一个走二级索引 B → 聚簇索引

B 路径：`idx_dept(D-A)` 加 X 锁 → 回表 CL(id=100) 加 X 锁 → 修改。

### 5. 加锁顺序不同导致死锁

单行竞争时看似普通 Record Lock 等待。更隐蔽的是**多行交叉**——A 按 `emp_no` 逐行更新 E001、E002（CL 加锁顺序 100→101），B 通过 `idx_dept` 以相反顺序更新同两行（CL 加锁顺序 101→100），回表聚簇索引时形成交叉：

```sql
-- 会话 A（按 uk_emp_no 索引顺序加锁）
UPDATE employee SET salary = salary + 1 WHERE emp_no = 'E001';  -- 锁 CL(100)
UPDATE employee SET salary = salary + 1 WHERE emp_no = 'E002';  -- 锁 CL(101)

-- 会话 B（通过 idx_dept 定位，先命中 E002 再命中 E001）
UPDATE employee SET salary = salary + 1
    FORCE INDEX (idx_dept) WHERE dept_code = 'D-A' AND emp_no = 'E002';  -- 锁 CL(101)
UPDATE employee SET salary = salary + 1
    FORCE INDEX (idx_dept) WHERE dept_code = 'D-A' AND emp_no = 'E001';  -- 锁 CL(100)
-- 注：不加 FORCE INDEX 时优化器可能选择 uk_emp_no，两会话走相同路径则不死锁
```

两会话通过不同二级索引定位相同两行，但加锁顺序相反（A: CL(100)→CL(101)，B: CL(101)→CL(100)）。交叉执行时聚簇索引上形成：

```
A 持有 CL(id=100)，等待 CL(id=101)
B 持有 CL(id=101)，等待 CL(id=100)
```

### 6. 完整的复现步骤

```sql
-- 终端 1（走 uk_emp_no 索引）
BEGIN;
UPDATE employee SET salary = salary + 1 WHERE emp_no = 'E001';
-- 持有 uk_emp_no(E001) + CL(id=100)

-- 终端 2（走 idx_dept 索引，先定位 E002）
BEGIN;
UPDATE employee SET salary = salary + 1
    FORCE INDEX (idx_dept) WHERE dept_code = 'D-A' AND emp_no = 'E002';
-- 持有 idx_dept(D-A→E002) + CL(id=101)

-- 终端 1
UPDATE employee SET salary = salary + 1 WHERE emp_no = 'E002';
-- 需要 CL(id=101)，被终端 2 持有 → 等待

-- 终端 2
UPDATE employee SET salary = salary + 1
    FORCE INDEX (idx_dept) WHERE dept_code = 'D-A' AND emp_no = 'E001';
-- 需要 CL(id=100)，被终端 1 持有 → 死锁！
-- 环路：终端 1 持有 CL(100) 等待 CL(101)，终端 2 持有 CL(101) 等待 CL(100)
-- 预期 ERROR 1213
```

**预防**：更新条件尽量用主键/唯一键；多行更新按主键排序；避免混用多种索引路径更新重叠行集。

## 五、典型死锁场景四：批量操作的隐含死锁

批量 UPDATE/DELETE 中，加锁顺序在不同事务间不一致，即使访问相同 ID 集合，也可能因物理加锁顺序不同而死锁。

### 1. 两个事务批量更新，记录范围有交集

```sql
DROP TABLE IF EXISTS order_item;
CREATE TABLE order_item (
    id       INT PRIMARY KEY,
    order_id INT NOT NULL,
    status   TINYINT NOT NULL,
    amount   DECIMAL(10, 2) NOT NULL,
    KEY idx_order (order_id)
) ENGINE=InnoDB;

INSERT INTO order_item (id, order_id, status, amount) VALUES
    (1, 100, 0, 10.00), (2, 100, 0, 20.00), (3, 100, 0, 30.00),
    (4, 200, 0, 40.00), (5, 200, 0, 50.00);

-- 会话 A
BEGIN;
UPDATE order_item SET status = 1 WHERE id IN (3, 1, 2);

-- 会话 B
BEGIN;
UPDATE order_item SET status = 2 WHERE id IN (2, 4, 1);
```

### 2. IN 子句的加锁顺序问题

InnoDB 对 `WHERE id IN (...)` 通常按**主键 B+ 树顺序**加锁，而非 IN 列表书写顺序。但以下因素导致顺序差异：

| 因素             | 影响                   |
|----------------|----------------------|
| 索引路径           | 主键 vs 二级索引           |
| Optimizer 重写   | IN 可能转为 range、多次 ref |
| 应用层循环单条 UPDATE | HashSet 迭代顺序不确定      |

主键 IN 通常按升序加锁，`IN (1,2)` 与 `IN (2,1)` 可能不死锁。但若 B 走二级索引：

```sql
UPDATE order_item SET status = 2
WHERE order_id = 100 AND id IN (2, 1);
-- 先锁 idx_order 再锁聚簇，顺序与 A 纯主键路径不同
```

更常见的陷阱是循环单条 UPDATE：

```java
for (Long id : idsFromHashSet) {  // 迭代顺序不确定
    jdbc.update("UPDATE order_item SET status=? WHERE id=?", status, id);
}
```

两线程以不同顺序逐行加锁，等价于场景一的交叉更新。

### 3. 典型死锁时序

```sql
-- 会话 A                          -- 会话 B
BEGIN;                             BEGIN;
UPDATE order_item SET status = 1 WHERE id = 1;
                                   UPDATE order_item SET status = 2 WHERE id = 2;
UPDATE order_item SET status = 1 WHERE id = 2;
                                   UPDATE order_item SET status = 2 WHERE id = 1;
-- 与场景一相同，隐藏在 ORM/Job 批量逻辑中
```

### 4. 解决方案：排序后批量操作

**应用层排序**：

```java
List<Long> ids = new ArrayList<>(idSet);
Collections.sort(ids);
for (Long id : ids) {
    jdbc.update("UPDATE order_item SET status = ? WHERE id = ?", status, id);
}
```

**单条 SQL + ORDER BY**：

```sql
UPDATE order_item SET status = 1
WHERE id IN (3, 1, 2)
ORDER BY id;
```

**根本原则**：并发事务对重叠资源集必须遵循相同全序（通常主键升序）。

## 六、InnoDB 的死锁检测机制

InnoDB 默认启用主动死锁检测，通过等待图（Wait-for Graph）判断是否存在环。

### 1. 等待图（Wait-for Graph）算法

#### 构建等待关系图

等待图 `G = (V, E)`：顶点 V 为活跃事务，边 `T1 → T2` 表示 T1 等待 T2 持有的锁。

```
TRX_A → TRX_B  （A 等待 B 持有的 id=2 锁）
TRX_B → TRX_A  （B 等待 A 持有的 id=1 锁）
```

#### 检测环的算法（深度优先搜索）

环检测采用 DFS：沿 wait-for 边遍历，若回到路径上节点则判定死锁，回滚牺牲者释放其全部锁。

#### 检测的触发时机

主要在**事务请求锁进入等待状态时**触发，从当前等待事务出发做有限深度搜索，而非每次重建全图。极端高并发下检测本身可能成为瓶颈。

### 2. innodb_deadlock_detect 参数

```sql
SHOW VARIABLES LIKE 'innodb_deadlock_detect';  -- 默认 ON
```

| 值   | 行为                       |
|-----|--------------------------|
| ON  | 检测死锁，回滚牺牲者，报错 1213       |
| OFF | 不检测，依赖 lock_wait_timeout |

高并发热点场景（秒杀等）可能临时关闭以降低 CPU 开销：

```sql
SET GLOBAL innodb_deadlock_detect = OFF;
```

代价：真正死锁不再快速识别，会话长时间阻塞直到 1205：

```
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

**与 `innodb_lock_wait_timeout` 配合**（默认 50 秒）：

| 配置                     | 死锁行为            | 适用         |
|------------------------|-----------------|------------|
| detect=ON, timeout=50  | 快速 1213         | 默认生产环境     |
| detect=OFF, timeout=50 | 长时间等待后 1205     | 极少数热点优化    |
| detect=ON, timeout=5   | 1213 + 更快失败单向等待 | 重试成熟的低延迟应用 |

生产环境不建议长期关闭 detect，除非有完善监控与重试。

### 3. 检测的性能开销

极端场景检测复杂度近似 **O(n²)**，高并发下 CPU 开销显著。MySQL 8.0 锁系统重构有所优化。应对：固定加锁顺序、缩小事务、拆分热点、RC 减少 Gap Lock。

## 七、被回滚事务的选择策略

### 1. InnoDB 如何选择牺牲者

基本原则：**回滚代价最小的事务**。InnoDB 比较环内候选事务的 undo 代价——修改行数越多、持有锁越多、运行时间越长，权重越高，越不应被牺牲（工程理解模型，具体实现因版本略有差异）。

### 2. 权重计算：修改的行数、持有的锁数

| 因素               | 说明                  |
|------------------|---------------------|
| 已修改行数            | undo 回滚工作量          |
| 持有锁数量            | 释放后对其他事务影响          |
| undo log entries | STATUS 中可见，与回滚成本正相关 |
| 事务启动顺序           | 权重接近时，可能回滚较新事务      |

STATUS 示例：

```
TRANSACTION 421234, ACTIVE 0.012 sec
4 lock struct(s), 2 row lock(s), undo log entries 1
UPDATE account SET balance = balance + 100 WHERE id = 2
------- TRX HAS BEEN CHOSEN AS THE DEADLOCK VICTIM AND ROLLED BACK
```

较短、较新、修改较少的事务更可能成为牺牲者。

### 3. innodb_force_recovery 与死锁

`innodb_force_recovery`（1–6 级）仅用于崩溃恢复应急，与日常死锁无关，正常运行时不应设置。

### 4. Error 1213 的处理

```
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

SQLSTATE `40001` 表示整事务已被回滚。处理要点：

1. 不要假设部分语句生效——必须整事务重试。
2. 重新 `BEGIN` 后再执行业务逻辑。
3. 指数退避 + 随机抖动，避免同步重试风暴。
4. 记录 1213 频率与 SQL 模板，关联死锁日志做根因分析。
5. 区分 1213 与 1205（1205 仅终止等待方）。

```java
for (int i = 0; i < maxRetries; i++) {
    try { executeBusinessTransaction(); break; }
    catch (DeadlockException e) {
        if (i == maxRetries - 1) throw e;
        Thread.sleep(50 * (1 << i) + random.nextInt(50));
    }
}
```

## 八、SHOW ENGINE INNODB STATUS 死锁日志解读

`SHOW ENGINE INNODB STATUS\G` 的 `LATEST DETECTED DEADLOCK` 段是排查最重要的一手资料。InnoDB 只保留**最近一次**
死锁，发生后应尽快抓取。MySQL 8.0.1+ 可设 `innodb_print_all_deadlocks = ON` 将所有死锁写入错误日志。

### 1. 完整的死锁日志示例

以下对应场景一（主键交叉更新）：

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2026-07-02 14:32:18 0x7f8b1c0d9700
*** (1) TRANSACTION:
TRANSACTION 421234, ACTIVE 0.045 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1128, 2 row lock(s), undo log entries 1
MySQL thread id 88, OS thread handle 0x7f8b1c0d9700, query id 123456 root localhost
UPDATE account SET balance = balance + 100 WHERE id = 2

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 5 page no 3 n bits 80 index PRIMARY of table `test`.`account`
 trx id 421234 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 5 page no 3 n bits 80 index PRIMARY of table `test`.`account`
 trx id 421234 lock_mode X locks rec but not gap waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0

*** (2) TRANSACTION:
TRANSACTION 421235, ACTIVE 0.038 sec starting index read
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1128, 2 row lock(s), undo log entries 1
MySQL thread id 89, OS thread handle 0x7f8b1c0e0800, query id 123457 root localhost
UPDATE account SET balance = balance + 50 WHERE id = 1

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 5 page no 3 n bits 80 index PRIMARY of table `test`.`account`
 trx id 421235 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 5 page no 3 n bits 80 index PRIMARY of table `test`.`account`
 trx id 421235 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0

*** WE ROLL BACK TRANSACTION (2)
```

### 2. LATEST DETECTED DEADLOCK 段的结构

| 部分                                        | 含义      |
|-------------------------------------------|---------|
| 时间戳 + OS 线程                               | 死锁发生时刻  |
| `(n) TRANSACTION`                         | 参与死锁的事务 |
| `(n) HOLDS THE LOCK(S)`                   | 已持有的锁   |
| `(n) WAITING FOR THIS LOCK TO BE GRANTED` | 正在等待的锁  |
| `WE ROLL BACK TRANSACTION (n)`            | 牺牲者编号   |

### 3. TRANSACTION 信息的阅读

```
TRANSACTION 421234, ACTIVE 0.045 sec starting index read
LOCK WAIT 4 lock struct(s), 2 row lock(s), undo log entries 1
MySQL thread id 88
UPDATE account SET balance = balance + 100 WHERE id = 2
```

| 字段                                 | 解读                |
|------------------------------------|-------------------|
| `LOCK WAIT`                        | 处于锁等待             |
| `row lock(s)` / `undo log entries` | 持锁规模 / 修改量        |
| `MySQL thread id`                  | 对应 PROCESSLIST Id |
| 最后一行 SQL                           | 触发等待的语句           |

### 4. WAITING FOR THIS LOCK TO BE GRANTED

```
lock_mode X locks rec but not gap waiting
index PRIMARY of table `test`.`account`
Record lock, heap no 3
```

- `index PRIMARY`：聚簇索引。
- `rec but not gap`：纯 Record Lock。
- `waiting`：尚未授予。
- `heap no`：页内记录槽位，结合 SQL 推断 id 值。

Gap 锁：`lock_mode X locks gap before rec`。Insert Intention：`lock_mode X, INSERT INTENTION waiting`。

### 5. HOLDS THE LOCK(S)

`(2) HOLDS` 中 heap no 3 与 `(1) WAITING` 中 heap no 3 对应，则 `(1) → (2)` 存在 wait-for 边。对照两事务的 HOLDS/WAITING
即可还原完整环路。

### 6. 锁类型标识的含义

| 标识                        | 含义            |
|---------------------------|---------------|
| `X locks rec but not gap` | 记录排它锁，不含间隙    |
| `X locks gap before rec`  | Gap Lock      |
| `X`                       | Next-Key Lock |
| `X, INSERT INTENTION`     | 插入意向锁         |
| `S locks rec but not gap` | 共享记录锁         |

二级索引 Gap Lock 示例：

```
index uk_name of table `test`.`user_slot`
 lock_mode X locks gap before rec
```

### 7. 被回滚事务的标记

`*** WE ROLL BACK TRANSACTION (2)` 表示事务 (2) 被牺牲，对应会话收到 1213，事务 (1) 继续执行。牺牲者 TRANSACTION 段末尾可能出现：

```
------- TRX HAS BEEN CHOSEN AS THE DEADLOCK VICTIM AND ROLLED BACK
```

## 九、performance_schema 锁诊断

MySQL 8.0 中 `performance_schema.data_locks` 与 `data_lock_waits` 替代了 5.7 的 `INFORMATION_SCHEMA.INNODB_LOCKS` /`INNODB_LOCK_WAITS`。

### 1. data_locks 表（MySQL 8.0+）

```sql
SELECT
    engine_transaction_id AS trx_id,
    object_schema, object_name, index_name,
    lock_type, lock_mode, lock_status, lock_data
FROM performance_schema.data_locks
WHERE object_schema = 'test' AND object_name = 'account'
ORDER BY engine_transaction_id, lock_data;
```

场景一中间态示例：

```
+--------+ object_name | lock_mode     | lock_status | lock_data |
| 421234 | account     | X,REC_NOT_GAP | GRANTED     | 1         |
| 421234 | account     | X,REC_NOT_GAP | WAITING     | 2         |
| 421235 | account     | X,REC_NOT_GAP | GRANTED     | 2         |
| 421235 | account     | X,REC_NOT_GAP | WAITING     | 1         |
```

### 2. data_lock_waits 表

```sql
SELECT
    requesting_engine_transaction_id AS waiting_trx,
    blocking_engine_transaction_id AS blocking_trx,
    object_name, index_name, lock_mode
FROM performance_schema.data_lock_waits;
```

双向等待即死锁前兆：

```
+-------------+--------------+-------------+
| waiting_trx | blocking_trx | object_name |
+-------------+--------------+-------------+
| 421234      | 421235       | account     |
| 421235      | 421234       | account     |
+-------------+--------------+-------------+
```

关联 SQL 可通过 `data_lock_waits` 的 `requesting_thread_id` / `blocking_thread_id` 关联 `threads` 与 `processlist` 获取。

### 3. 替代旧版 INFORMATION_SCHEMA 视图

| MySQL 5.7           | MySQL 8.0+                           |
|---------------------|--------------------------------------|
| `INNODB_LOCKS`      | `performance_schema.data_locks`      |
| `INNODB_LOCK_WAITS` | `performance_schema.data_lock_waits` |
| `INNODB_TRX`        | `information_schema.innodb_trx`（仍可用） |

`data_locks` 同时展示 GRANTED 与 WAITING 锁，`lock_data` 可读性更好。

### 4. 实时查看当前锁等待的 SQL

综合诊断：

```sql
SELECT
    dl.engine_transaction_id AS trx_id,
    dl.object_name, dl.index_name,
    dl.lock_mode, dl.lock_status, dl.lock_data,
    trx.trx_rows_locked, trx.trx_rows_modified,
    p.id AS conn_id, p.time AS exec_sec,
    LEFT(p.info, 200) AS sql_text
FROM performance_schema.data_locks dl
JOIN information_schema.innodb_trx trx ON dl.engine_transaction_id = trx.trx_id
JOIN information_schema.processlist p ON trx.trx_mysql_thread_id = p.id
WHERE dl.object_schema NOT IN ('mysql', 'sys', 'performance_schema')
ORDER BY dl.engine_transaction_id, dl.lock_status DESC;
```

快捷视图：`SELECT * FROM sys.innodb_lock_waits\G`。避免高频轮询，告警触发后快照即可。

## 十、预防死锁的工程实践

死锁无法从理论上彻底消除（除非全局串行化），但可通过工程手段将频率降到可接受范围。

### 1. 固定访问顺序

最核心的原则：并发事务访问多组相同资源时，按相同顺序获取锁。

- 批量更新前对 ID 升序排序。
- 多表更新顺序固定（如始终 `account → order → order_item`）。
- 避免 HashSet、ConcurrentHashMap 无序迭代驱动逐行 UPDATE。

### 2. 缩小事务粒度和持续时间

- 将 RPC、文件 IO 移出事务；避免 `FOR UPDATE` 后长时间再 UPDATE。
- 大批量拆小批，缩短交叉持锁窗口。

外部调用完成后再开短事务更新，避免在 `FOR UPDATE` 与 `UPDATE` 之间长时间持锁。

### 3. 合理使用索引减少锁范围

- 更新/删除条件走索引，避免全表扫描引发大量 Next-Key Lock。
- 精确主键/唯一键定位，减少双路径加锁。
- 避免宽范围 `FOR UPDATE` 不必要锁定大量间隙。

### 4. 使用 RC 隔离级别减少 Gap Lock

```sql
SET SESSION transaction_isolation = 'READ-COMMITTED';
```

RC 下大多数当前读不加 Gap Lock，场景二类死锁显著减少。权衡：RR 幻读防护更强；许多 OLTP 系统选用 RC + 应用层幂等。

### 5. NOWAIT 和 SKIP LOCKED（MySQL 8.0）

不消除死锁，但改变竞争语义：

```sql
SELECT * FROM account WHERE id = 1 FOR UPDATE NOWAIT;
-- 无法立即获锁则报错 3572

SELECT id FROM task_queue
WHERE status = 'PENDING'
ORDER BY id LIMIT 1
FOR UPDATE SKIP LOCKED;
-- 跳过已锁定行，适合多 Worker 任务队列
```

| 语法          | 行为    | 用途     |
|-------------|-------|--------|
| NOWAIT      | 立即失败  | 热点行抢锁  |
| SKIP LOCKED | 跳过锁定行 | 队列分片消费 |

### 6. 应用层重试机制设计

1213 是可预期的并发冲突：仅对 transient 错误整事务重试（3–5 次上限），指数退避 + 抖动，保证幂等，监控 1213 率突增。配合
`innodb_print_all_deadlocks = ON` 建立排查闭环。

## 总结

InnoDB 死锁是互斥、持有并等待、不可剥夺、循环等待四个必要条件在 Record Lock、Gap Lock、Next-Key Lock 与 Insert Intention Lock 上的具体体现。主键交叉更新、Gap Lock 与 Insert Intention 冲突、二级索引与聚簇索引加锁顺序不一致、批量操作隐含顺序差异，构成线上最常见的四类场景。

InnoDB 通过等待图环检测主动发现死锁，默认回滚权重较低的事务并返回 Error 1213。`SHOW ENGINE INNODB STATUS` 的`LATEST DETECTED DEADLOCK` 段是还原加锁顺序的核心依据；MySQL 8.0 的 `data_locks` 与 `data_lock_waits` 提供实时等待链诊断。

工程上，**固定访问顺序**是预防死锁的首要手段；辅以缩小事务、合理索引、按需 RC、NOWAIT/SKIP LOCKED 队列模式，以及带退避的 1213 重试，可将死锁从"难以解释的偶发故障"转化为"可观测、可优化、可容忍的并发代价"。死锁本身不是 InnoDB 缺陷，而是并发正确性的边界；理解形成机理，比单纯关闭 `innodb_deadlock_detect` 更有长期价值。
