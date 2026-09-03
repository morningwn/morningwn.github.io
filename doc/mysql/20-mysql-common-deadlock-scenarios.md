---
title: MySQL 常见死锁场景及成因
summary: 通过最小事务时序，说明 InnoDB 中交叉更新、索引路径、范围锁、唯一键冲突和批量处理导致死锁的原因。
created: 2026-09-01
updated: 2026-09-01
tags: MySQL, InnoDB, 死锁, 锁, 事务
cover: /img/mysql/mysql-common-deadlock-scenarios-cover.webp
source: https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html
---

# MySQL 常见死锁场景及成因

本文基于 MySQL 8.0、InnoDB 和默认的 `REPEATABLE READ` 隔离级别，列举常见死锁场景及其直接原因。示例中的语句需要在独立会话中按编号交错执行。单个事务等待锁不等于死锁；只有多个事务各自持有锁，同时等待其他事务持有的锁，等待关系形成闭环时，死锁才成立。

InnoDB 的行锁实际作用于索引记录。使用二级索引查询时，语句还可能锁定对应的聚簇索引记录；范围扫描可能涉及间隙锁或临键锁。这些行为决定了下面各场景中的实际锁定对象。

## 一、多行记录的加锁顺序相反

假设 `account` 表存在主键为 `1` 和 `2` 的两行记录，两个事务以相反顺序更新它们：

| 次序 | 事务 A | 事务 B |
| --- | --- | --- |
| 1 | `BEGIN` | `BEGIN` |
| 2 | `UPDATE account SET balance = balance - 100 WHERE id = 1` |  |
| 3 |  | `UPDATE account SET balance = balance - 100 WHERE id = 2` |
| 4 | `UPDATE account SET balance = balance + 100 WHERE id = 2`，等待事务 B |  |
| 5 |  | `UPDATE account SET balance = balance + 100 WHERE id = 1`，等待事务 A |

事务 A 持有主键 `1` 的排他记录锁，等待主键 `2`；事务 B 持有主键 `2` 的排他记录锁，等待主键 `1`。

**形成原因**：两个事务更新相同的记录集合，但加锁顺序相反，形成 `A → B → A` 的等待环。相同结构也可能出现在两个事务以相反顺序访问多张表时。

## 二、不同索引访问路径导致加锁顺序不同

假设 `account` 表的主键为 `id`，`email` 上存在唯一二级索引 `uk_email`，并已有记录 `(id=1, email='a@example.com')`：

| 次序 | 事务 A | 事务 B |
| --- | --- | --- |
| 1 | `BEGIN` | `BEGIN` |
| 2 | `UPDATE account SET balance = balance + 1 WHERE id = 1`，持有聚簇索引记录锁 |  |
| 3 |  | `SELECT * FROM account FORCE INDEX (uk_email) WHERE email = 'a@example.com' FOR UPDATE`，持有二级索引记录锁，等待聚簇索引记录锁 |
| 4 | `UPDATE account SET email = 'b@example.com' WHERE id = 1`，等待事务 B 持有的旧二级索引记录锁 |  |

事务 B 通过 `uk_email` 定位记录时，需要继续锁定对应的聚簇索引记录，但该记录已被事务 A 锁定。事务 A 随后修改 `email`，需要更新原二级索引记录，又被事务 B 已取得的锁阻塞。

**形成原因**：事务 A 的加锁路径是“聚簇索引 → 二级索引”，事务 B 的路径是“二级索引 → 聚簇索引”。两个路径锁定同一组索引记录，但顺序相反。

## 三、范围锁定与区间插入相互等待

假设 `slot` 表以 `id` 为主键，已有 `10`、`20`、`30` 三条记录。在 `REPEATABLE READ` 下，两个事务先锁定不同的空区间，再向对方锁定的区间插入：

| 次序 | 事务 A | 事务 B |
| --- | --- | --- |
| 1 | `BEGIN` | `BEGIN` |
| 2 | `SELECT * FROM slot WHERE id > 10 AND id < 20 FOR UPDATE` |  |
| 3 |  | `SELECT * FROM slot WHERE id > 20 AND id < 30 FOR UPDATE` |
| 4 | `INSERT INTO slot(id) VALUES (25)`，等待事务 B |  |
| 5 |  | `INSERT INTO slot(id) VALUES (15)`，等待事务 A |

两个范围查询没有返回记录，但锁定扫描涉及的索引间隙。事务 A 插入 `25` 时申请的插入意向锁受事务 B 的间隙锁阻塞；事务 B 插入 `15` 时又受事务 A 的间隙锁阻塞。

**形成原因**：两个事务分别持有一个索引间隙的锁，又交叉向对方的间隙申请插入意向锁。间隙锁可以彼此共存，但会阻止其他事务向相应间隙插入。该时序依赖范围扫描产生间隙锁；在 `READ COMMITTED` 下，普通查询和索引扫描通常不会产生同样的间隙锁。

## 四、同一唯一键的多个并发插入

假设 `unique_item(id)` 的 `id` 是主键，表中尚无 `id=1`。这一场景需要三个并发事务：

| 次序 | 事务 A | 事务 B | 事务 C |
| --- | --- | --- | --- |
| 1 | `BEGIN` | `BEGIN` | `BEGIN` |
| 2 | `INSERT INTO unique_item(id) VALUES (1)`，取得排他记录锁 |  |  |
| 3 |  | `INSERT INTO unique_item(id) VALUES (1)`，等待事务 A |  |
| 4 |  |  | `INSERT INTO unique_item(id) VALUES (1)`，等待事务 A |
| 5 | `ROLLBACK` | 继续插入 | 继续插入 |

事务 B 和事务 C 检测到未提交的重复键后，都会请求该索引记录的共享锁。事务 A 回滚并释放排他锁后，两个共享锁请求都可能被授予；随后 B 和 C 都需要取得排他锁才能完成插入，但各自持有的共享锁会阻塞对方。

**形成原因**：同一唯一键上存在多个等待插入者，原排他锁持有者回滚后，等待者从重复键检查所需的共享锁转向插入所需的排他锁，形成相互等待。只有两个事务时，第二个事务通常只是等待第一个事务结束，并不会仅因唯一键相同就形成死锁。

## 五、删除记录后并发插入同一唯一键

假设 `unique_item` 表已经存在 `id=1`。一个事务删除该记录，另外两个事务同时插入相同主键：

| 次序 | 事务 A | 事务 B | 事务 C |
| --- | --- | --- | --- |
| 1 | `BEGIN` | `BEGIN` | `BEGIN` |
| 2 | `DELETE FROM unique_item WHERE id = 1`，持有排他记录锁 |  |  |
| 3 |  | `INSERT INTO unique_item(id) VALUES (1)`，等待事务 A |  |
| 4 |  |  | `INSERT INTO unique_item(id) VALUES (1)`，等待事务 A |
| 5 | `COMMIT` | 继续插入 | 继续插入 |

事务 A 提交后，B 和 C 对重复索引记录申请的共享锁被授予。删除结果生效后，两者继续尝试插入并申请排他锁，但各自持有的共享锁会阻塞另一个事务的排他锁请求。

**形成原因**：删除事务释放排他锁后，多个等待插入者同时从共享锁阶段转入排他锁阶段，对同一唯一索引记录形成锁升级式的循环等待。该场景与上一场景的区别是：原锁持有者删除已有记录并提交，而不是插入新记录后回滚。

## 六、批量更新的记录集合重叠且顺序不同

批处理、ORM 循环更新或多个任务分片可能处理重叠的记录集合。假设事务 A 按 `1 → 2 → 3` 更新，事务 B 按 `3 → 2 → 1` 更新：

| 次序 | 事务 A | 事务 B |
| --- | --- | --- |
| 1 | `UPDATE task SET status = 1 WHERE id = 1` |  |
| 2 |  | `UPDATE task SET status = 2 WHERE id = 3` |
| 3 | `UPDATE task SET status = 1 WHERE id = 2` |  |
| 4 |  | `UPDATE task SET status = 2 WHERE id = 2`，等待事务 A |
| 5 | `UPDATE task SET status = 1 WHERE id = 3`，等待事务 B |  |

事务 B 等待事务 A 持有的主键 `2` 记录锁，事务 A 又等待事务 B 持有的主键 `3` 记录锁。

**形成原因**：两个批处理的目标集合有交集，但应用层迭代顺序、分片顺序或 SQL 执行计划产生的实际加锁顺序不同。该场景的锁结构与多行交叉更新相同，区别在于相反顺序通常隐藏在批处理实现中，而不是直接出现在两条手写 SQL 中。`IN` 列表的书写顺序也不能视为 InnoDB 的加锁顺序保证。

以上场景虽然由不同 SQL 形态触发，但共同点都是：事务已经持有部分索引记录或间隙上的锁，又以不同顺序申请其余锁，最终形成闭合的等待关系。
