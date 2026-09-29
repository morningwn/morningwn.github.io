---
title: Undo Log 深入：版本链构建、事务回滚与 MVCC 的完整实现
summary: 深入 Undo Log 的内部结构，拆解版本链的物理构建过程、Read View 的可见性判断算法以及 Purge 线程的回收机制，补全 MVCC 的实现全貌。
created: 2026-07-02
updated: 2026-07-08
tags: MySQL, InnoDB, Undo Log, MVCC, 事务
---

# Undo Log 深入：版本链构建、事务回滚与 MVCC 的完整实现

版本链、Read View 与 MVCC 协同工作的核心问题是"一致性读为什么能读到某个历史版本"，以及不同隔离级别如何调整快照的创建与复用策略。本文补全 Undo Log 在磁盘上的物理形态、版本链指针的编码方式，以及 Purge 线程何时真正回收历史版本这一层实现细节。

阅读本文后，你应当能够回答以下问题：

- 一条 UPDATE 语句在物理层面如何同时完成"覆盖当前记录"与"保留旧版本"。
- Insert Undo 与 Update Undo 为何生命周期不同，回滚与 MVCC 分别依赖哪一种。
- `DB_TRX_ID`、`DB_ROLL_PTR`、`DB_ROW_ID` 三个隐藏列如何在聚簇索引记录上串联版本链。
- Read View 内部字段 `m_ids`、`m_low_limit_id`、`m_up_limit_id` 各自代表什么边界，如何用具体数值理解。
- InnoDB 判定某个历史版本可见或不可见时的完整分支逻辑。
- Purge 线程如何按 History List 顺序回收 Undo，以及长事务为何会导致 Undo 表空间持续膨胀。

本文仍聚焦 InnoDB 存储引擎，不涉及 Server 层优化器或 Binlog 复制细节。

## 一、Undo Log 在 InnoDB 中的定位

### 1. 双重职责：事务回滚与 MVCC

InnoDB 的行更新遵循"原地修改 + 保留旧版本"策略。聚簇索引页上的记录被 UPDATE 或 DELETE 修改时，当前内存中的最新值会直接写入数据页；与此同时，被覆盖的旧版本必须被保存下来，否则系统无法完成两类关键操作。

**职责一：事务回滚**

当事务执行 ROLLBACK，或崩溃恢复阶段发现某事务未提交时，InnoDB 必须能够撤销该事务对数据页的全部修改。Undo Log 中保存的正是"修改前的旧值"或"插入行的主键定位信息"。回滚时，InnoDB 沿事务私有的 Undo 链逆向重放，将数据页恢复到事务开始前的状态。

**职责二：MVCC 一致性读**

其他事务的快照读（普通 SELECT）可能看不到最新版本——Read View 会将某些未提交或"未来"事务的修改判定为不可见。此时，InnoDB 不能返回数据页上的最新值，而必须沿版本链回溯到对该快照可见的历史版本。Undo Log 正是这些历史版本的物理载体。

**两套职责如何共用一套数据结构**

Insert Undo 与 Update Undo 的划分，本质上是在"回滚专用"与"回滚 + MVCC 共用"之间做分流：

| 操作类型   | Undo 类型     | 回滚用途               | MVCC 用途     |
|--------|-------------|--------------------|-------------|
| INSERT | Insert Undo | 根据主键删除新插入的行        | 不参与版本链      |
| UPDATE | Update Undo | 用旧列值覆盖当前记录         | 提供历史版本供回溯   |
| DELETE | Update Undo | 清除 delete mark，恢复行 | 提供删除前的完整行副本 |

两类 Undo 在物理上存储在同一套 Undo 表空间与 Rollback Segment 体系中，由事务子系统统一分配页槽、统一受 Redo Log 保护。差别在于生命周期：Insert Undo 提交后即可丢弃；Update Undo 必须进入 History List，等待 Purge 线程在确认无快照依赖后回收。

从实现角度看，一次 UPDATE 的完整路径同时服务两个职责：

```
UPDATE 执行
  ├─ 写 Update Undo（保存旧值）        ← 供回滚撤销 + 供 MVCC 回溯
  ├─ 修改聚簇索引记录（写入新值）       ← 当前最新版本
  └─ 更新 DB_TRX_ID 与 DB_ROLL_PTR    ← 串联版本链
```

若事务随后 ROLLBACK，InnoDB 沿该事务的 Undo 链逆向恢复，MVCC 从未"发布"这些中间版本给其他事务的快照读。若事务 COMMIT，数据页上的新版本成为"当前最新"，Undo 中的旧版本则进入全局 History List，供其他事务的快照读按需回溯。

### 2. Undo Log 与 Redo Log 的关系

Redo Log 与 Undo Log 在 InnoDB 事务体系中分工明确，但相互依赖，共同保证 ACID 中的原子性与持久性。

**Redo Log 记录"做了什么"**

Redo Log 以物理日志的形式记录数据页（以及 Undo 页）的修改内容，例如"在表空间 5、页号 1000、偏移 200 处写入 4 字节"。其目的是崩溃恢复时的前滚（Redo）：将尚未落盘的脏页恢复到崩溃前的最新一致状态。Redo Log 不关心"如何撤销"，只关心"页面上最终应该是什么"。

**Undo Log 记录"如何撤销"**

Undo Log 以逻辑行的形式记录被修改列的旧值、主键信息、以及指向更早 Undo 的 roll_pointer。其目的是：

- 事务回滚时，根据 Undo 逆向修改数据页。
- MVCC 一致性读时，根据 Undo 构造历史版本。

Undo 本身不直接保证持久性——Undo 页的修改同样必须先写 Redo Log，再写 Undo 页，遵循 WAL 原则。

**Undo Log 本身也需要被 Redo 保护**

这是初学者容易忽略的一点：Undo Log 页也是 Buffer Pool 中的普通数据页，其修改同样会生成 Redo Log 记录。一条 DML 语句的典型写入顺序如下：

```
1. 分配或定位 Undo 页，将 Undo 记录写入 Undo 页（生成 Undo 页的 Redo）
2. 修改聚簇索引数据页（生成数据页的 Redo）
3. 上述 Redo 记录写入 Log Buffer
4. 事务提交时，Log Buffer 中的 Redo fsync 落盘
5. （异步）脏页（含 Undo 页与数据页）刷回表空间
```

崩溃恢复时，恢复流程分两个阶段：

```
崩溃恢复
  ├─ Redo 阶段：重放所有 Redo Log，恢复 Undo 页与数据页到崩溃前状态
  └─ Undo 阶段：扫描未提交事务，沿 Undo 链回滚其对数据页的修改
```

若没有 Redo 对 Undo 页的保护，崩溃后 Undo 记录本身可能丢失或损坏，未提交事务将无法正确回滚，已提交事务的 MVCC 历史版本也可能不完整。

**Redo Log 与 Undo Log 的分工边界**

| 维度      | Redo Log   | Undo Log                  |
|---------|------------|---------------------------|
| 记录内容    | 物理页修改      | 逻辑行旧值                     |
| 主要用途    | 崩溃前滚、持久性   | 事务回滚、MVCC                 |
| 空间管理    | 循环覆盖（固定大小） | History List + Purge 异步回收 |
| 是否参与版本链 | 否          | 是（Update Undo）            |

两者在崩溃恢复中缺一不可：Redo 保证页级一致，Undo 保证未提交事务被正确撤销。

### 3. Undo Log 的存储位置

Undo Log 并不存放在用户表的 `.ibd` 文件中，而是由 InnoDB 事务子系统（`trx_sys`）统一管理，存储在专门的 Undo 表空间中。

**Undo 表空间（MySQL 8.0 独立化）**

MySQL 5.7 及更早版本中，Undo Log 默认存放在共享表空间 `ibdata1` 内，与数据字典、系统表等共享同一文件。这带来两个问题：

- Undo 膨胀会撑大 `ibdata1`，且空间无法单独回收。
- 高并发写入时，Undo 与数据页竞争同一文件的 I/O 带宽。

MySQL 8.0 起，Undo Log 默认存放在独立的 Undo 表空间中，文件命名形如 `undo_001`、`undo_002`，位于数据目录下。独立化的好处包括：

- Undo 膨胀不影响用户表空间文件。
- 支持对单个 Undo 表空间执行 Truncate，归还磁盘空间。
- 可通过 `innodb_undo_directory` 将 Undo 表空间放到独立磁盘，隔离 I/O。

相关配置参数：

| 参数                         | 默认值（8.0）         | 含义                         |
|----------------------------|------------------|----------------------------|
| `innodb_undo_tablespaces`  | 2                | Undo 表空间数量（静态参数，可通过 `CREATE UNDO TABLESPACE` 动态添加） |
| `innodb_undo_directory`    | 数据目录             | Undo 表空间文件存放路径             |
| `innodb_max_undo_log_size` | 1073741824（1 GB） | 单个 Undo 表空间触发 Truncate 的阈值 |
| `innodb_undo_log_truncate` | ON               | 是否启用 Undo 表空间自动 Truncate   |

**回滚段（Rollback Segment）的组织**

Rollback Segment（简称 rseg）是 Undo Log 的逻辑容器。每个 rseg 维护：

- 一组 Undo 页链表，用于追加 Undo 记录。
- Insert Undo 与 Update Undo 的分离管理。
- 指向 History List 的链接，供 Purge 线程遍历。

InnoDB 在初始化时创建多个 rseg，分散在不同 Undo 表空间中，以减少并发事务分配 Undo 页时的锁竞争。事务首次执行 DML 时，从可用 rseg 中分配一个 Undo 页槽位，后续该事务的所有 Undo 记录追加到同一 rseg 的链上。

逻辑结构示意：

```
Undo 表空间 undo_001
  ├─ Rollback Segment 0
  │    ├─ Undo 页链（活跃事务的 Undo）
  │    └─ History List 入口
  ├─ Rollback Segment 1
  │    └─ ...
  └─ ...

Undo 表空间 undo_002
  ├─ Rollback Segment 2
  └─ ...
```

**undo tablespace 的数量与大小**

`innodb_undo_tablespaces` 控制 Undo 表空间文件的数量。增加 Undo 表空间可以：

- 分散 rseg，降低单文件内的页分配竞争。
- 在 Truncate 时，允许一个表空间正在被清理而另一个继续服务新事务。

单个 Undo 表空间的大小没有硬编码上限，会随事务写入持续增长，直到 Purge 回收空间或 Truncate 触发。生产环境中，Undo 表空间大小是监控重点——若 Purge 滞后，文件可能达到数十 GB 甚至更大。

可通过以下语句查看 Undo 表空间状态（MySQL 8.0）：

```sql
SELECT NAME, SPACE_TYPE, STATE, FILE_SIZE, ALLOCATED_SIZE
FROM information_schema.INNODB_TABLESPACES
WHERE SPACE_TYPE = 'Undo';
```

## 二、Insert Undo 与 Update Undo

### 1. Insert Undo Log

Insert Undo 由 INSERT 操作产生，是 Undo 体系中结构最简单、生命周期最短的一类。当事务向聚簇索引（或二级索引）插入新行时，InnoDB 在写入数据页的同时生成 Insert Undo 记录，核心内容是**新插入行的主键值**及必要的索引定位信息——因为 INSERT 之前该行不存在，没有"旧值"可言，Undo 只需保存足够在回滚时删除该行的定位信息。

事务 ROLLBACK 时，InnoDB 沿 Insert Undo 链逆向处理：根据主键在聚簇索引 B+ 树中定位并物理删除记录，同时删除各二级索引项。提交后可以**立即清理**——其他事务的快照读不可能回溯到"尚未插入"的行，已提交 INSERT 的 Undo 不再被任何快照需要。Insert Undo 只服务回滚，不参与版本链，不进入 History List。

### 2. Update Undo Log

Update Undo 由 UPDATE 和 DELETE 操作产生，是 MVCC 版本链的物理载体。UPDATE 在修改聚簇索引记录前将被覆盖列的旧值写入 Undo；DELETE 在设置 delete mark 前将被删除行的完整副本写入 Undo。Update Undo 记录典型包括：被修改列的旧值（或 DELETE 的完整行副本）、该行当前的 `DB_TRX_ID` 与 `DB_ROLL_PTR`、主键列值，以及涉及二级索引列变更时的索引项旧值。

回滚时，UPDATE 用 Undo 中的旧列值原地恢复记录并还原 `DB_TRX_ID`/`DB_ROLL_PTR`；DELETE 清除 delete mark 恢复行可见性。提交后**不能立即清理**——其他事务的 Read View 可能仍需回溯该 Undo 代表的历史版本，必须等 Purge 线程异步确认无引用后回收。Update Undo 提交后进入 History List，按提交顺序排队。

### 3. 两种 Undo 的生命周期差异

两种 Undo 的生命周期差异，根源于 MVCC 的需求不同。

Insert Undo 可早清理，因为 INSERT 之前该行不存在，不存在"回溯到 INSERT 之前"的 MVCC 需求；未提交 INSERT 由 Read View 判定不可见，已提交 INSERT 则数据页最新版本即为正确结果。Update Undo 必须保留至"不存在任何 Read View 仍可能需要该历史版本"，由 Purge 线程根据全局最旧 Read View 的 `m_low_limit_no` 异步判断——从事务提交到真正回收，可能间隔数秒甚至数小时（若存在长事务）。

生命周期对比：

```
Insert Undo:
  事务 BEGIN → INSERT → 写 Insert Undo → COMMIT/ROLLBACK
                                              ↓
                                    立即释放（不进 History List）

Update Undo:
  事务 BEGIN → UPDATE/DELETE → 写 Update Undo → COMMIT
                                                    ↓
                                          进入 History List
                                                    ↓
                                    Purge 线程异步扫描
                                                    ↓
                              确认无 Read View 依赖 → 物理删除
```

| 维度      | Insert Undo | Update Undo     |
|---------|-------------|-----------------|
| 触发操作    | INSERT      | UPDATE、DELETE   |
| 记录内容    | 新行主键        | 旧列值 / 完整行副本     |
| 参与版本链   | 否           | 是               |
| 提交后     | 立即释放        | 进入 History List |
| 清理者     | 事务提交路径      | Purge 线程        |
| MVCC 依赖 | 无           | 有               |

## 三、版本链的物理构建过程

### 1. 行记录的隐藏列

InnoDB 聚簇索引记录除了用户定义的列之外，还包含若干隐藏列，其中与 MVCC 直接相关的有三个。这些列对用户不可见，但在每条记录的物理存储中始终存在（在`COMPACT`/`DYNAMIC` 行格式下）。

**DB_TRX_ID（6 字节）**

最后一次插入或更新该行的事务 ID。InnoDB 为每个读写事务分配单调递增的 `trx_id`（只读事务通常不分配）。`DB_TRX_ID` 标识"这个版本由谁最后修改"，是 Read View 可见性判断的核心输入。

**DB_ROLL_PTR（7 字节）**

回滚指针，指向 Undo Log 中该行的上一个版本。它是版本链的物理链接——通过 `DB_ROLL_PTR`，InnoDB 可以从聚簇索引上的最新记录跳转到 Undo 页中保存的旧版本副本，再继续沿 Undo 中的 roll_pointer 向前回溯。

`DB_ROLL_PTR` 并非简单的文件偏移量，而是一个复合地址，逻辑上可拆为：

```
| 1 bit: 类型标志 | 7 bit: rseg id | 32 bit: 页号 | 16 bit: 页内偏移 |
```

- **类型标志**：区分 Insert Undo 指针（值为 1）与 Update Undo 指针（值为 0）。MVCC 回溯时只沿 Update Undo 指针前进，遇到 Insert Undo 指针表示链尾。
- **rseg id**：回滚段标识符，7 bit 可标识 0~127 个回滚段。MySQL 8.0 最多支持 128 个回滚段（`INNODB_ROLLBACK_SEGMENTS` = 128）。
- **页号与页内偏移**：定位 Undo 记录所在的 Undo 页（32 bit 页号）及页内记录位置（16 bit 偏移，适配 16 KB 默认页大小）。

**DB_ROW_ID（6 字节）**

隐藏主键。仅当表没有显式主键且没有 NOT NULL 的唯一索引时，InnoDB 会自动生成 `DB_ROW_ID` 作为聚簇索引键。若表已有主键，`DB_ROW_ID` 不存在于记录中。

三个隐藏列在 MVCC 中的角色：

| 隐藏列         | 大小   | 作用                       |
|-------------|------|--------------------------|
| DB_TRX_ID   | 6 字节 | 标识版本归属事务，供可见性判断          |
| DB_ROLL_PTR | 7 字节 | 指向 Undo 中的上一版本，串联版本链     |
| DB_ROW_ID   | 6 字节 | 无主键时的聚簇索引键（与 MVCC 无直接关系） |

**二级索引记录的差异**

二级索引记录不包含 `DB_TRX_ID` 与 `DB_ROLL_PTR`，只保存索引列值与对应聚簇索引主键的引用。MVCC 判可见性时，必须先通过二级索引项回表到聚簇索引记录，读取`DB_TRX_ID` 与 `DB_ROLL_PTR`，再执行版本链回溯。这是"回表"操作除了读取完整行数据之外的另一层 MVCC 含义。

### 2. 一次 UPDATE 如何构建版本链

用一个具体示例演示一次 UPDATE 如何同时完成"覆盖当前记录"与"保留旧版本"。

**初始状态**

表 `accounts` 有一行，`id = 1`：

```
聚簇索引记录：
  id = 1, balance = 1000
  DB_TRX_ID = 100    （由事务 T100 插入并提交）
  DB_ROLL_PTR = NULL （无更早版本）
```

**事务 T200 执行 UPDATE**

```sql
-- 事务 T200 (trx_id = 200)
BEGIN;
UPDATE accounts SET balance = 1500 WHERE id = 1;
```

InnoDB 内部执行以下步骤：

**步骤 1 — 写 Update Undo**

在 T200 的 Undo 页中追加一条 Update Undo 记录，保存修改前的旧版本：

```
Update Undo 记录：
  主键 id = 1
  balance = 1000          （旧值）
  DB_TRX_ID = 100         （旧 trx_id）
  DB_ROLL_PTR = NULL      （旧 roll_ptr，指向链尾）
```

**步骤 2 — 原地修改聚簇索引记录**

在 Buffer Pool 的数据页上，直接覆盖该行的列值与隐藏列：

```
聚簇索引记录（修改后）：
  id = 1, balance = 1500   （新值）
  DB_TRX_ID = 200          （更新为 T200）
  DB_ROLL_PTR = 0x...      （指向步骤 1 写入的 Undo 记录）
```

**步骤 3 — 写 Redo Log**

上述 Undo 页修改与数据页修改均生成 Redo Log 记录，写入 Log Buffer。

**步骤 4 — T200 提交**

Redo Log fsync 落盘，T200 的 Update Undo 从活跃链移入 History List。数据页上的最新版本变为 `(balance=1500, trx_id=200)`，Undo 中保留 `(balance=1000, trx_id=100)` 的副本。

版本链逻辑结构：

```
聚簇索引 (balance=1500, trx=200)
    │
    │ DB_ROLL_PTR
    ▼
Undo 页 (balance=1000, trx=100)
    │
    │ roll_ptr = NULL
    ▼
  （链尾）
```

**关键点**：数据页上始终只有一个物理记录（最新版本），历史版本存放在 Undo 页中；版本链是逻辑单向链表，每次 UPDATE 延长链条而非替换。

### 3. 多次 UPDATE 形成链表

若同一行被多个事务连续 UPDATE，版本链会不断延长。

**接续上例**

事务 T300 再次 UPDATE：

```sql
-- 事务 T300 (trx_id = 300)
BEGIN;
UPDATE accounts SET balance = 2000 WHERE id = 1;
COMMIT;
```

物理过程与单次 UPDATE 相同：

1. 写 Update Undo：保存 `(balance=1500, trx_id=200, roll_ptr→Undo_T200)`。
2. 修改聚簇索引：`(balance=2000, trx_id=300, roll_ptr→Undo_T300)`。
3. 提交后，T300 的 Undo 进入 History List。

版本链变为：

```
聚簇索引 (balance=2000, trx=300)
    │ roll_ptr
    ▼
Undo (balance=1500, trx=200)
    │ roll_ptr
    ▼
Undo (balance=1000, trx=100)
    │ roll_ptr = NULL
    ▼
  （链尾）
```

**版本链的方向：从最新到最旧**

版本链是单向链表，方向固定为：**从聚簇索引上的最新版本，沿 `DB_ROLL_PTR` 指向 Undo 页中的旧版本，再沿 Undo 内的 roll_pointer 继续向前，直到链尾**。

一致性读总是从最新版本出发，逐节向前回溯，直到找到对当前 Read View 可见的版本。

**链表的终止条件**

版本链在以下两种情况下终止：

1. **roll_pointer 为 NULL**：表示该行由 INSERT 创建，不存在更早的 Undo 版本。这是最常见的链尾。
2. **roll_pointer 指向 Insert Undo**：Insert Undo 不参与 MVCC 回溯，InnoDB 将其视为链尾。语义上表示"再往前就是该行不存在"。

若一致性读沿链回溯到链尾仍未找到可见版本，则该行对当前事务"不存在"（等效于 DELETE 或未 INSERT）。

**版本链长度与性能**

版本链的长度等于该行自上次 Purge 以来被 UPDATE/DELETE 的次数（减去已被 Purge 回收的节点）。热点行被频繁 UPDATE 且 Purge 滞后时，链条可能积累数十甚至数百个节点，每次一致性读的回溯成本与链长成正比。

### 4. DELETE 操作的特殊处理

DELETE 在 InnoDB 中通常并非立即物理抹除记录，而是采用"标记删除 + 延迟 Purge"的两阶段策略。

**阶段一：标记删除（delete_flag = 1）**

事务执行 DELETE 时，InnoDB：

1. 写 Update Undo，保存删除前的完整行副本（含 `DB_TRX_ID` 与 `DB_ROLL_PTR`）。
2. 在聚簇索引记录上设置 **delete mark**（记录头中的 `deleted_flag` 位）。
3. 更新 `DB_TRX_ID` 为当前事务 ID，`DB_ROLL_PTR` 指向刚写入的 Undo。

此时，记录在 B+ 树中仍然存在，但被标记为"已删除"。二级索引项同样打 delete mark。

**阶段二：Purge 物理删除**

Purge 线程在确认没有任何 Read View 仍需要该行的历史版本后，才真正从 B+ 树中移除索引项，回收页空间。

**在版本链中的位置与一致性读处理**

DELETE 的 Update Undo 与 UPDATE 在链上位置相同，差别在语义：UPDATE 节点回溯时返回旧列值；DELETE 节点若可见则行仍存在，若最新可见版本带 delete mark 则行"不存在"。高并发 DELETE 后表空间不立即缩小，因 delete mark 记录仍占 B+ 树节点，直到 Purge 物理移除。

## 四、Read View 的创建时机与内部结构

### 1. Read View 是什么

Read View 是一致性读的**可见性快照边界**——不是数据的物理副本，而是一组事务 ID 边界值与活跃事务列表。普通 SELECT 依赖它判定每个版本的可见性，从最新版本沿 Undo 链回溯到第一个可见版本。创建开销极低（拷贝事务 ID 列表），多个事务共享 Undo 中的历史版本，写入路径与 Read View 无锁竞争。

### 2. Read View 的核心字段

InnoDB 源码中（MySQL 8.0），Read View 结构体 `ReadView` 包含以下核心字段。理解这些字段是掌握 MVCC 可见性算法的前提。

| 字段                 | 类型        | 含义                                                  |
|--------------------|-----------|-----------------------------------------------------|
| `m_creator_trx_id` | trx_id_t  | 创建该 Read View 的事务 ID                                |
| `m_ids`            | trx_ids_t | 创建快照时，系统中所有活跃（未提交）事务 ID 的有序列表                       |
| `m_up_limit_id`    | trx_id_t  | 活跃事务列表中的最小 trx_id；若 `m_ids` 为空，则等于 `m_low_limit_id` |
| `m_low_limit_id`   | trx_id_t  | 创建快照时，InnoDB 已分配的下一个 trx_id（即"未来"事务的起始 ID）          |
| `m_low_limit_no`   | trx_id_t  | 用于 Purge：小于该值的 Undo 可以被清理（与 trx_no 相关）              |

**用具体数值示例解释各字段**

假设某时刻系统状态如下：

- 已提交事务的最大 trx_id = 95。
- 当前活跃（未提交）事务：T96、T98、T99（trx_id 分别为 96、98、99）。
- 下一个将分配的 trx_id = 100。

事务 T97（trx_id = 97）在 REPEATABLE READ 下执行第一次一致性读，InnoDB 创建 Read View：

```
m_creator_trx_id = 97

m_ids = [96, 98, 99]        （活跃事务 ID 列表，有序）

m_up_limit_id = 96          （m_ids 中的最小值）

m_low_limit_id = 100        （下一个将分配的 trx_id）
```

各字段划分的可见性区间：

```
trx_id 轴：
  ... 94  95  |  96  97  98  99  |  100  101  ...
              ↑                   ↑
         m_up_limit_id      m_low_limit_id
              |←── m_ids 区间 ──→|
              |                   |
         小于 96：可能可见    大于等于 100：一定不可见
         96~99 在 m_ids 中：一定不可见
         97 是 creator：对自己可见
```

**m_creator_trx_id 与边界字段**

当前事务自己的修改始终可见（`trx_id == m_creator_trx_id`）。`m_up_limit_id` 以下的事务在快照创建前已提交，其版本可能可见；`m_low_limit_id` 及以上的事务在快照创建后才启动，一定不可见；落在 `[m_up_limit_id, m_low_limit_id)` 且在 `m_ids` 中的事务快照创建时仍活跃，不可见；同区间但不在 `m_ids` 中的，说明启动早、提交快，可见。

`m_low_limit_no` 是 trx_no（提交顺序号），非 trx_id。Purge 线程用它判断 History List 中哪些 Undo 可安全清理——只能清理 trx_no 小于全局最旧 Read View 的 `m_low_limit_no` 的 Undo。

### 3. 不同隔离级别下的创建时机

Read View 的创建时机由隔离级别决定，其创建时点直接影响与 Undo 实现相关的版本链回溯行为。

READ COMMITTED 下每条一致性读语句创建新 Read View，快照边界"更晚"，Undo 回溯链通常更短；REPEATABLE READ 下事务内首次一致性读创建后复用，边界固定在事务早期，后续 UPDATE 更可能不可见；READ UNCOMMITTED 不构造标准 Read View，直接读最新版本。无论哪种级别，创建时需短暂持有 `trx_sys` 互斥锁，原子拷贝活跃事务列表，创建后内容不再变化。

### 4. Read View 的回收

**事务结束时释放**

Read View 的生命周期与创建它的事务绑定。当事务 COMMIT 或 ROLLBACK 时，InnoDB 释放该事务的 Read View 结构体，将其从全局 Read View 列表中移除。

**与 Purge 的关系**

Read View 的释放直接影响 Purge 线程的回收能力。Purge 维护全局最旧 Read View 的 `m_low_limit_no` 水位：

```
全局 Purge 水位 = min(所有活跃 Read View 的 m_low_limit_no)
```

只有当某个 Read View 被释放后，Purge 水位才可能下降，History List 中更多的 Undo 才变得可以回收。因此：

- 长事务（长时间不提交）→ Read View 长时间存在 → Purge 水位被抬高 → Undo 无法回收。
- 即使长事务是只读事务（在 RR 下首次 SELECT 也会创建 Read View），同样会阻塞 Purge。

这是长事务导致 Undo 膨胀的核心机制，详见第七节。

## 五、MVCC 可见性判断的完整算法

### 1. 判断流程（伪代码级详解）

InnoDB 对版本 `V`（其事务 ID 为 `trx_id`）的可见性判定，在源码中由 `RowVers::row_vers_old_has_visible_version`等路径实现。以下伪代码按判断优先级排列，与源码逻辑一致。

```
函数 visible(trx_id, read_view):

  // 步骤一：自己修改的，始终可见
  若 trx_id == read_view.m_creator_trx_id
    → 返回 可见

  // 步骤二：快照创建前已提交（trx_id 小于活跃列表最小 ID）
  若 trx_id < read_view.m_up_limit_id
    → 返回 可见

  // 步骤三：快照创建后才启动的事务，一定不可见
  若 trx_id >= read_view.m_low_limit_id
    → 返回 不可见

  // 步骤四：快照创建时仍活跃的事务，一定不可见
  若 trx_id 存在于 read_view.m_ids 中
    → 返回 不可见

  // 步骤五：落在 [m_up_limit_id, m_low_limit_id) 之间且不在活跃列表
  //         说明快照创建前已提交（启动早、提交快，拷贝 m_ids 时已不在列表）
  否则
    → 返回 可见
```

**各步骤的语义解释**

| 步骤 | 条件                         | 结论  | 语义                                                     |
|----|----------------------------|-----|--------------------------------------------------------|
| 一  | trx_id == m_creator_trx_id | 可见  | 读己之所写：当前事务自己的修改始终可见                                    |
| 二  | trx_id < m_up_limit_id     | 可见  | 版本所属事务在快照创建前已提交                                        |
| 三  | trx_id >= m_low_limit_id   | 不可见 | 版本所属事务在快照创建之后才启动                                       |
| 四  | trx_id 在 m_ids 中           | 不可见 | 快照创建时该事务仍活跃，即使随后提交也不可见                                 |
| 五  | 其余情况                       | 可见  | 落在 `[m_up_limit_id, m_low_limit_id)` 但不在 m_ids：启动早、提交快 |

步骤五是最容易忽略的场景：事务 T50 在 T60 创建 Read View 前已提交，T50 的 trx_id=50 不在 `m_ids` 中，因此对 T60 的快照可见。

### 2. 沿版本链查找

单个版本的可见性判定只是 MVCC 的第一步。一致性读的完整流程是：从最新版本出发，沿版本链逐节回溯，直到找到第一个可见版本。

```
函数 mvcc_find_visible_version(聚簇索引记录 R, read_view):

  version ← R 的最新版本（数据页上的记录）
            含：列值, DB_TRX_ID, DB_ROLL_PTR, delete_flag

  loop:
    result ← visible(version.trx_id, read_view)

    若 result == 可见:
      若 version.delete_flag == 1
        → 返回 NULL（行不存在，已被删除）
      否则
        → 返回 version 的列值

    // 当前版本不可见，沿链回溯
    若 version.roll_ptr 为空 或 指向 Insert Undo
      → 返回 NULL（无更早历史，行不存在）

    version ← 从 Undo 页加载 version.roll_ptr 指向的旧版本
              含：旧列值, 旧 DB_TRX_ID, 旧 roll_ptr, 旧 delete_flag

    继续 loop
```

**回溯与二级索引**

每次沿 `DB_ROLL_PTR` 回溯可能触发 Undo 页 I/O（Buffer Pool 命中则免）。二级索引扫描时，通常须回表到聚簇索引执行上述流程；但当二级索引页头部的 `max_trx_id` 小于 Read View 的 `m_low_limit_id` 时，表示该页所有记录对当前快照均可见，可跳过逐行回表。覆盖索引在此优化命中时可避免聚簇索引访问，否则仍需回表判定可见性。

### 3. 完整示例

以下示例演示多个事务交错执行时，各事务看到的版本分析。假设隔离级别为 REPEATABLE READ。

**初始状态**

```sql
CREATE TABLE accounts (id INT PRIMARY KEY, balance INT);
INSERT INTO accounts VALUES (1, 1000);  -- 事务 T100 提交，trx_id=100
```

聚簇索引：`id=1, balance=1000, DB_TRX_ID=100, DB_ROLL_PTR=NULL`

**时间线**

| 时间 | 事件                                  | 聚簇索引最新版本          | 活跃事务         |
|----|-------------------------------------|-------------------|--------------|
| t1 | T200 BEGIN                          | trx=100, bal=1000 | {T200}       |
| t2 | T200 首次 SELECT，创建 RV200             | trx=100, bal=1000 | {T200}       |
| t3 | T300 BEGIN; UPDATE bal=1500; COMMIT | trx=300, bal=1500 | {T200}       |
| t4 | T400 BEGIN; UPDATE bal=2000; COMMIT | trx=400, bal=2000 | {T200}       |
| t5 | T200 再次 SELECT                      | trx=400, bal=2000 | {T200}       |
| t6 | T500 BEGIN; 首次 SELECT，创建 RV500      | trx=400, bal=2000 | {T200, T500} |
| t7 | T200 COMMIT                         | trx=400, bal=2000 | {T500}       |

**T200 的 Read View（t2 创建）**

```
m_creator_trx_id = 200
m_ids = [200]
m_up_limit_id = 200
m_low_limit_id = 300
```

**t5：T200 再次 SELECT 的版本分析**

从聚簇索引最新版本出发：

1. 最新版本 trx=400：`400 >= 300 (m_low_limit_id)` → 不可见，沿 Undo 回溯。
2. Undo 中 trx=300 的版本：`300 >= 300` → 不可见，继续回溯。
3. Undo 中 trx=100 的版本：`100 < 200 (m_up_limit_id)` → **可见**，返回 `balance=1000`。

T200 两次 SELECT 均读到 `1000`，符合 REPEATABLE READ 语义。

**T500 的 Read View（t6 创建）**

```
m_creator_trx_id = 500
m_ids = [200, 500]
m_up_limit_id = 200
m_low_limit_id = 501
```

**T500 首次 SELECT 的版本分析**

1. 最新版本 trx=400：`200 <= 400 < 501`，400 不在 m_ids [200,500] 中 → **可见**，返回 `balance=2000`。

T500 读到最新值 `2000`，因为 T300、T400 的修改在其 Read View 创建前已提交。

补充：若 T200 在 t3 前执行 `UPDATE balance=1200` 但未提交，t5 时步骤一优先判定 `200 == m_creator_trx_id`，T200 读到 `1200`而非 `1000`——"读己之所写"优先于链上其他版本。Undo 链在 t4 后保留 T300、T400 的版本，供 T500 等更晚快照使用，直到 Purge 回收。

## 六、Undo Log 的回收：Purge 线程

### 1. 为什么需要 Purge

Update Undo 不能随事务提交立即删除，这是 MVCC 正确性的硬性要求。若提交后立即删除 Undo，其他事务的 Read View 在回溯版本链时会遇到"断链"——指向已回收的 Undo 页，导致读到错误数据或崩溃。

Purge 机制的存在，是为了在**保证 MVCC 正确性**的前提下，**尽可能早地**回收不再被任何 Read View 引用的 Undo，归还磁盘空间与 Buffer Pool 缓存。

核心矛盾：

```
MVCC 需要旧版本尽可能长时间保留  ←→  磁盘空间与性能需要旧版本尽快回收
                              Purge 线程调和这一矛盾
```

### 2. History List

**Undo Log 按提交顺序进入 History List**

事务 COMMIT 时，其 Update Undo 段从活跃 Undo 链摘下，按 **trx_no**（事务提交顺序号，单调递增）链接到全局 History List 尾部。History List 是一个单向链表，头部是最早提交的、尚未 Purge 的 Undo，尾部是最新提交的 Undo。

Insert Undo 不进入 History List——提交后直接释放槽位。

**History List Length 的含义**

`SHOW ENGINE INNODB STATUS` 输出中的 `History list length` 表示当前 History List 中尚未 Purge 的 Undo 记录数量（近似值，按 Undo 日志段计）。

```
History list length = 0     → Purge 完全跟上，无积压
History list length = 100   → 少量积压，通常正常
History list length = 10000+ → 显著积压，需排查长事务或 Purge 性能
History list length 持续增长 → 危险信号，Undo 表空间将持续膨胀
```

History List Length 是生产环境监控 Undo 健康的核心指标之一。

### 3. Purge 线程的工作方式

InnoDB 后台运行 Purge 线程（数量由 `innodb_purge_threads` 控制，默认 4）。Purge Coordinator 扫描 History List 头部，读取全局最旧 Read View 的 `m_low_limit_no` 水位，将 trx_no 小于该水位的 Undo 清理任务分发给 Purge Worker。Worker 删除 Undo 记录、物理移除 delete mark 行、标记 Undo 页为空闲。Purge 不能跳过 History List 中更旧的条目——必须按 trx_no 顺序处理。DELETE 的物理移除也在此阶段完成：确认无 MVCC 依赖后，从聚簇索引与二级索引 B+ 树中移除索引项。

### 4. Undo 表空间的 Truncate

MySQL 8.0 支持对 Undo 表空间执行自动 Truncate，在 Purge 回收大量空间后，将缩小的物理文件归还操作系统。

**innodb_undo_log_truncate 参数**

默认值为 `ON`。启用后，InnoDB 会在 Undo 表空间大小超过 `innodb_max_undo_log_size`（默认 1 GB）且其中至少有一半空间可回收时，异步 Truncate 该表空间。

**Truncate 的触发条件**

Truncate 并非随时触发，需同时满足：

1. `innodb_undo_log_truncate = ON`。
2. 某个 Undo 表空间的大小超过 `innodb_max_undo_log_size`。
3. 该表空间中至少有一半的 Undo 页已被 Purge 标记为空闲。
4. 该 Undo 表空间不在 History List 的活跃引用中（即所有引用它的 Undo 均已 Purge）。

**Truncate 的过程**

```
1. 选择一个满足条件的 Undo 表空间
2. 将该表空间标记为 inactive（新事务不再分配）
3. 等待所有引用该表空间的 Undo 均被 Purge
4. 将 active 的 Undo 页迁移到其他 Undo 表空间
5. Truncate 物理文件至初始大小（通常 10 MB）
6. 将该表空间重新标记为 active
```

Truncate 期间，其他 Undo 表空间继续服务新事务，不影响正常业务。但若 Purge 长期滞后，Truncate 条件长期不满足，Undo 表空间文件会持续占用磁盘。

## 七、长事务为什么会导致 Undo Log 膨胀

### 1. 因果链分析

长事务本身不一定写入大量 Undo，但会**长时间持有 Read View**，从而阻塞 Purge 回收。完整的因果链如下：

```
长事务开启
  │
  ├─ REPEATABLE READ 下首次一致性读创建 Read View
  │     （只读长查询同样会创建 Read View）
  │
  ▼
Read View 长时间不释放
  │
  ├─ 全局 Purge 水位 = min(所有活跃 Read View 的 m_low_limit_no)
  │     被该长事务的 Read View 抬高
  │
  ▼
Purge 线程无法清理 History List 中更旧的 Undo
  │
  ├─ History List Length 持续增长
  │
  ▼
Update Undo 记录持续累积
  │
  ├─ Undo 表空间文件持续增大
  │
  ▼
热点行版本链变长（Purge 无法回收链上节点）
  │
  └─ 其他事务一致性读回溯成本上升
```

**关键洞察**

Undo 膨胀不是因为长事务"写了很多 Undo"，而是因为长事务"阻止别人清理 Undo"。即使长事务是纯只读（没有任何 DML），只要它在 REPEATABLE READ 下执行过 SELECT，就会创建 Read View 并持有至事务结束，等效于阻塞 Purge。

**数值示例**

假设：

- 业务每秒产生 100 个 UPDATE 事务，每个产生 1 条 Update Undo。
- 正常情况：Purge 每秒回收 100 条，History List Length 稳定在低位。
- 某时刻：一个只读事务开启，RR 下 SELECT 创建 Read View，然后运行 1 小时不提交。
- 1 小时内：新增 100 × 3600 = 360,000 条 Update Undo 进入 History List，Purge 无法回收任何一条。
- History List Length 从 ~100 飙升至 ~360,000，Undo 表空间可能增长数百 MB 甚至 GB。

### 2. 连锁影响

Undo 膨胀引发连锁影响：Undo 页长期驻留 Buffer Pool 挤占数据页缓存；热点行版本链延长导致一致性读回溯变慢；Undo 表空间文件只增不减直至 Truncate 触发，极端情况下可能耗尽磁盘。与 Redo Log 不同——Redo 循环覆盖、大小固定，膨胀主因是写入压力；Undo 膨胀主因是长事务阻塞 Purge，极端后果是磁盘耗尽、新事务无法分配 Undo 页。

### 3. 监控与预防

核心监控指标：`SHOW ENGINE INNODB STATUS` 中的 `History list length`（通常超过 10000 应告警）和 `Purge done for trx's n:o`（与 trx 计数器的差距反映 Purge 滞后程度）。长事务排查：

```sql
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS running_seconds,
       trx_mysql_thread_id, trx_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started;
```

关注 `running_seconds` 超阈值（如 60 秒）的会话，尤其是 `trx_query` 为空（应用开启事务后长时间未操作）的情况。确保`innodb_undo_log_truncate = ON`，但 Truncate 前提是 Purge 正常——长事务阻塞 Purge 时 Truncate 无法触发。应用层应设置事务超时，避免 RR 下"BEGIN → SELECT → 长业务逻辑 → COMMIT"模式；只读长查询考虑 READ COMMITTED 或拆分为无事务 SELECT。

## 总结

Undo Log 是 InnoDB 同时实现**原子回滚**与**多版本一致性读**的物理基础。Insert Undo 仅服务 INSERT 的回滚，记录新行主键，提交后即可丢弃，不参与版本链；Update Undo 承载 UPDATE/DELETE 的旧版本，串联成以 `DB_ROLL_PTR` 为链、以`DB_TRX_ID` 标识版本归属的单向版本链，提交后进入 History List，直到 Purge 确认无 Read View 依赖后才回收。

版本链的构建发生在每次 UPDATE/DELETE 的写入路径上：旧值写入 Update Undo，数据页原地更新为新值，`DB_TRX_ID` 与 `DB_ROLL_PTR` 将最新版本与 Undo 中的历史版本链接。DELETE 采用标记删除策略，Purge 时才物理移除 B+ 树索引项。聚簇索引记录上的三个隐藏列——`DB_TRX_ID`、`DB_ROLL_PTR`、`DB_ROW_ID`——是 MVCC 的物理锚点；二级索引记录不含完整 MVCC 信息，一致性读必须回表到聚簇索引。

MVCC 的一致性读并不"复制一份数据"，而是在最新版本上执行一套基于 Read View 的五步可见性算法：优先判定是否为自己修改；再比较 `trx_id` 与 `m_up_limit_id`、`m_low_limit_id` 及活跃事务列表 `m_ids`；不可见则沿 Undo 链回溯，直至找到可见版本或确认行不存在。Read View 的创建时机由隔离级别决定（RC 每条语句创建，RR 事务内首次创建后复用），但其内部结构与判定算法对所有级别一致。

Purge 线程按 trx_no 顺序扫描 History List，以全局最旧 Read View 的 `m_low_limit_no` 为水位，异步清理不再被任何快照引用的 Update Undo，并物理删除已标记删除的行。MySQL 8.0 支持 Undo 表空间自动 Truncate，在 Purge 回收空间后归还磁盘。长事务通过长时间持有 Read View 抬高 Purge 水位，导致 History List 堆积、Undo 表空间膨胀、版本链延长与一致性读变慢——这是生产环境中 Undo 相关故障的重要实现层根因。

理解上述机制后，可以将隔离级别语义规则与 InnoDB 源码级的字段、算法一一对应，形成从 SQL 现象到存储引擎实现的完整闭环。
