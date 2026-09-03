---
title: Redo Log：WAL 机制、Checkpoint 与崩溃恢复
summary: 深入 InnoDB 的 Redo Log 机制，从 WAL 原则、日志物理结构、刷盘策略到 LSN 与 Checkpoint，系统拆解崩溃恢复的完整过程。
created: 2026-07-02
updated: 2026-07-12
tags: MySQL, InnoDB, Redo Log, WAL, 崩溃恢复
---

# Redo Log：WAL 机制、Checkpoint 与崩溃恢复

InnoDB 的持久性（Durability）承诺，建立在 Redo Log 与 WAL 机制之上。事务一旦提交，即使进程被 `kill -9`、服务器突然断电或操作系统崩溃，已提交的数据也不应丢失。与此同时，为了获得可接受的 OLTP 写入吞吐，InnoDB 并不会在每次修改数据页时立刻将其刷回磁盘，而是优先在内存中的 Buffer Pool 里完成修改，再由后台线程异步地将脏页写回表空间文件。

这种"内存先改、磁盘后写"的策略，与"提交即持久"的承诺之间存在天然张力。Redo Log 正是弥合这一矛盾的核心机制：它以顺序追加的方式，记录数据页的物理修改，在崩溃后可以将尚未落盘的脏页恢复到一致状态。Checkpoint 机制则界定恢复边界，使 Redo Log 可以在有限空间内循环复用。

本文聚焦 Redo Log 及其周边的 WAL 原则、物理结构、刷盘策略、LSN 与 Checkpoint，以及崩溃恢复的完整流程。阅读本文后，你应当能够回答：

- 为什么 InnoDB 不直接在每次提交时刷数据页，而需要 Redo Log。
- WAL 的三个保证分别约束哪一阶段的写入顺序。
- Redo Log 文件组如何循环写入，`write pos` 与 `checkpoint` 如何协同。
- `innodb_flush_log_at_trx_commit` 三种模式在崩溃时各可能丢失多少数据。
- 崩溃恢复时 Redo 前滚与 Undo 回滚各自解决什么问题。
- Double Write Buffer 为何是 Redo Log 不可或缺的补充。

不涉及 Undo Log 版本链与 MVCC 的完整实现，也不展开 Binlog 复制拓扑；但会在必要处说明 Redo Log 与 Binlog 在两阶段提交中的分工。

## 一、为什么需要 Redo Log

### 1. 数据页随机写的代价

InnoDB 将表数据和索引组织为 **16 KB 页（Page）**，存放在表空间文件（`.ibd` 或共享表空间）中。B+ 树的一次索引查找，可能触发从根到叶的多级页访问；一次 `UPDATE` 可能只修改行中的几个字节，但在存储引擎层面，仍然需要：

1. 根据主键或二级索引定位到目标页；
2. 若页不在内存中，从磁盘随机读取该页到 Buffer Pool；
3. 在页内修改记录，更新页头元信息；
4. 最终将修改后的整页写回磁盘上的原位置。

第 4 步是性能瓶颈所在。OLTP 负载中，不同事务往往修改不同表、不同页，写请求分散在表空间文件的各个偏移位置，形成 **随机写（Random Write）**。与顺序写相比，随机写面临：

| 维度          | 随机写                          | 顺序写          |
|-------------|------------------------------|--------------|
| 磁盘寻道        | 频繁切换物理位置，HDD 上延迟显著           | 连续地址，可流水线化   |
| I/O 合并      | 难以合并，每次 commit 可能触发独立 fsync  | 追加写天然可批量     |
| 写入放大        | 16 KB 页写回，实际变更可能仅数十字节        | 日志记录通常远小于整页  |
| 与 commit 耦合 | 若每次 commit 刷页，延迟被磁盘 fsync 主导 | 日志追加与数据页刷盘解耦 |

假设一个简单场景：每秒 1000 个短事务，每个事务修改不同行、落在不同数据页。若每次 commit 都将对应脏页 fsync 到表空间，每秒可能产生 1000 次随机写 fsync——这在传统 HDD 上几乎不可接受，在 SSD 上也会严重限制吞吐。

InnoDB 的设计选择是：**不在事务提交时刷数据页**，而是将页的落盘延迟到后台。这一策略的前提是：必须存在另一种机制，在崩溃时仍能恢复已提交事务对页的修改。Redo Log 承担的正是这一职责。

### 2. Buffer Pool 的"延迟写"策略

Buffer Pool 是 InnoDB 在内存中缓存数据页的区域，默认大小由 `innodb_buffer_pool_size` 控制。写操作的标准路径如下：

```
读/写请求
    -> 在 Buffer Pool 页哈希表中查找 (space_id, page_no)
    -> 未命中则从磁盘读入
    -> 在内存页中修改记录
    -> 页标记为脏页（dirty page），加入 Flush List
    -> 生成 Redo Log 记录
    -> 事务提交时刷 Redo Log（而非刷脏页）
    -> 返回客户端
    -> （异步）Page Cleaner 线程将脏页写回表空间
```

脏页在内存中停留的时间取决于多种因素：LRU 淘汰压力、Flush List 长度、`innodb_max_dirty_pages_pct` 阈值、Checkpoint 推进速度等。在典型 OLTP 负载下，脏页可能在内存中停留数百毫秒到数秒不等；极端情况下，若刷脏跟不上写入，脏页可能停留更久。

延迟写带来的好处是显而易见的：

- **读路径加速**：热点页长期驻留内存，避免重复磁盘 I/O。
- **写合并**：多个事务先后修改同一页时，只需刷一次脏页，Redo Log 中保留每次修改的记录。
- **刷盘批量化**：Page Cleaner 可以按表空间、按 LSN 顺序批量刷脏，提高 I/O 效率。
- **commit 延迟降低**：事务提交只需等待 Redo Log fsync，而非等待数据页随机写。

Buffer Pool 与 Redo Log 的分工可以概括为：

| 组件          | 存储内容     | 正常运行职责     | 崩溃后状态         |
|-------------|----------|------------|---------------|
| Buffer Pool | 数据页完整副本  | 加速读写，承载脏页  | 内存丢失，脏页未落盘    |
| Redo Log    | 页的物理修改记录 | 保证持久性，支持恢复 | 已 fsync 部分可重放 |
| 表空间文件       | 数据页持久化基准 | 提供"旧版本"快照  | 可能是过期的旧页      |

### 3. 延迟写与持久性的矛盾

延迟写策略引入了一个直接风险：**若在脏页刷回磁盘之前发生崩溃，内存中的最新修改将永久丢失**。考虑以下时间线：

```
T1: 事务 A 修改页 P，生成脏页，写 Redo Log
T2: 事务 A COMMIT，Redo Log 已 fsync
T3: 客户端收到"提交成功"
T4: （尚未发生）Page Cleaner 将页 P 刷回表空间
T5: 服务器断电
```

在 T5 时刻，表空间文件中的页 P 仍是 T1 之前的旧内容，但事务 A 已经提交。若没有 Redo Log，恢复后用户会看到"提交成功但数据丢失"——这直接违反 ACID 中的 Durability。

Redo Log 解决这一矛盾的方式是 **Write-Ahead Logging（预写式日志）**：

- 事务提交前，描述页 P 修改的 Redo 记录必须已持久化到磁盘。
- 崩溃恢复时，以表空间中的旧页 P 为起点，重放 Redo Log 中针对页 P 的记录，将页恢复到崩溃前的最新状态。
- 数据页何时刷盘不再影响已提交事务的安全性，只影响恢复时需要重放的 Redo 范围。

换言之，持久性的锚点从"数据页落盘"转移到了"Redo Log 落盘"。脏页是性能优化手段，Redo Log 是持久性保证手段，二者通过 WAL 原则绑定在一起。

### 4. 随机写变顺序写

Redo Log 存放在独立于表空间的专用日志区域（传统上为 `ib_logfile0`、`ib_logfile1`，MySQL 8.0.30+ 为 `#innodb_redo` 目录）。写入方式是**顺序追加（Append-Only）**：

- 新的 Redo Record 总是写到当前 `write pos` 指向的位置；
- 写满日志文件末尾后，回到第一个文件继续写（循环）；
- 不修改已有日志内容，不随机定位历史位置。

顺序追加写与表空间随机页写相比，Redo Record 只记录变更部分（表空间 ID、页号、偏移、字节），体积远小于 16 KB 整页；多个事务的 Redo 还可通过组提交一次 fsync，使 commit 延迟不与事务数线性相关。

用一句话概括：**让数据页的随机写变成后台异步操作，让持久性保证变成日志的顺序写。**

## 二、WAL 核心原则

### 1. Write-Ahead Logging 的含义

WAL（Write-Ahead Logging，预写式日志）是数据库与文件系统中广泛采用的持久性范式。其核心原则可以表述为：

**在修改持久化存储上的数据之前，必须先将描述该修改的日志记录持久化。**

对于 InnoDB，"持久化存储上的数据"主要指表空间和索引空间中的数据页；"描述该修改的日志记录"指 Redo Log 中的 Redo Record。WAL 并不要求"日志永久先于数据页存在"，而是要求在 **关键时刻** 日志已经落盘：

| 关键时刻  | WAL 要求                                   |
|-------|------------------------------------------|
| 事务提交时 | 该事务产生的 Redo Log 必须已 fsync 到磁盘            |
| 脏页刷盘前 | 使该页产生最后一次修改的 Redo Log 必须已在磁盘上            |
| 崩溃恢复时 | 从 Checkpoint LSN 起的 Redo 足以将未落盘脏页恢复到一致状态 |

"先写日志，再写数据页"中的 **"先"**，需要精确理解：

- **不是**指每次修改内存页之前必须先写磁盘日志（实际上 Redo 先写入 Log Buffer，与内存页修改几乎同步）。
- **而是**指：在事务对外宣告提交成功之前，Redo Log 必须完成持久化；在脏页写回表空间之前，对应的 Redo 必须已在磁盘上。

第二条约束（刷脏前日志已在磁盘）通常自然满足：脏页由 Redo 修改产生，Redo 写入 Log Buffer 的时间不晚于页变脏；而事务提交时的 fsync 或后台刷盘，都晚于 Redo 进入 Log Buffer 的时间。InnoDB 的刷脏逻辑还会检查页的 LSN，确保不会跳过必要的 Redo。

### 2. WAL 的三个保证

InnoDB 的 WAL 实现可以拆解为三个相互衔接的保证，分别对应写路径的不同阶段：

**保证一：数据页修改前，对应的 Redo Log 已在 Log Buffer 中**

当事务修改 Buffer Pool 中的数据页时，InnoDB 在 **Mini-Transaction（mtr）** 框架内完成这一操作。mtr 是 InnoDB 内部的原子操作单元，例如 B+ 树页分裂、单行更新、分配新页等。每个 mtr 结束时，会将本次修改产生的 Redo Record 写入 Log Buffer，然后才释放页锁。

这保证了：任何导致页变脏的修改，都有对应的 Redo 记录存在于内存日志缓冲区中。若在此阶段崩溃（Redo 尚未刷盘），该修改将被视为未提交或使用 Undo 回滚，不会破坏一致性。

**保证二：事务提交时，Redo Log 已刷到磁盘**

当事务执行 `COMMIT`（或在 autocommit 模式下语句结束）时，InnoDB 将本事务相关的 Redo 从 Log Buffer 刷出。在`innodb_flush_log_at_trx_commit=1`（默认）下，刷出包含 **fsync**，即 Redo 已持久化到磁盘存储。

只有保证二成立，事务才满足 Durability。客户端收到"提交成功"时，可以认为：即使下一秒发生整机断电，该事务的修改也能在重启后通过 Redo 恢复。

**保证三：数据页刷盘前，对应的 Redo Log 已在磁盘上**

Page Cleaner 将脏页写回表空间时，InnoDB 要求：使该页达到当前状态所需的全部 Redo，必须已经持久化。这通过 LSN 比较实现——每个数据页页头有`FIL_PAGE_LSN` 字段，记录最后一次修改该页的 Redo 的 LSN；刷脏前，系统确保 `flushed_lsn >= page_lsn`（概念上）。

保证三的意义在于：表空间中的页要么是"旧但完整"的版本，要么是"已通过 Redo 可重建"的版本。不会出现"页已落盘但对应 Redo 被覆盖"的情况——后者将导致崩溃后无法恢复。

三个保证串联起来，构成完整的 WAL 闭环：

```
[mtr 修改页] -> 保证一：Redo 进 Log Buffer
      |
[COMMIT]     -> 保证二：Redo fsync
      |
[异步刷脏]   -> 保证三：刷页前 Redo 已在磁盘
```

### 3. WAL 与 Redo Log、Binlog 的关系

MySQL 存在两层日志，初学者容易混淆：

| 日志       | 层级             | 内容形式            | 主要用途         |
|----------|----------------|-----------------|--------------|
| Redo Log | InnoDB 引擎层     | 物理日志：页号、偏移、字节变更 | 崩溃恢复、WAL     |
| Binlog   | MySQL Server 层 | 逻辑日志：SQL 或行变更事件 | 主从复制、PITR、审计 |

Redo Log 与 InnoDB 页格式紧密耦合，恢复时在引擎启动阶段以物理方式重放，速度快，无需解析 SQL。Binlog 是 Server 层独立于存储引擎的逻辑变更流，复制 slave 重放 Binlog，而非 Redo Log。

当 `binlog_format=ROW` 且开启 Binlog 时，一次事务提交涉及 **两份日志的协调**。InnoDB 使用 **两阶段提交（Two-Phase Commit, 2PC）** 保证 Redo Log 与 Binlog 的一致性：

```
阶段一（Prepare）:
  1. 事务修改数据页，写 Redo Log（可能仍在 Log Buffer）
  2. InnoDB 进入 Prepare 状态，Redo Log 刷盘（Prepare 标记）

阶段二（Commit）:
  3. Binlog 刷盘（fsync）
  4. InnoDB 提交，Redo Log 写入 Commit 标记并刷盘
  5. 返回客户端成功
```

2PC 解决的核心问题是：Redo 与 Binlog 必须 **要么都持久化，要么都不持久化**。若只写了 Redo 没写 Binlog，主从复制会丢失变更；若只写了 Binlog 没写 Redo，崩溃恢复后 InnoDB 层数据不一致。

崩溃恢复时，InnoDB 检查 Redo Log 中处于 Prepare 状态的事务，并扫描 Binlog 判断是否已有对应记录：若 Binlog 中有完整事件，则提交该事务；若无，则回滚。这一机制保证 Redo 与 Binlog 的最终一致。

需要强调的是：**InnoDB 的崩溃恢复不依赖 Binlog**。Binlog 服务于复制与逻辑恢复；Redo Log 服务于引擎层物理恢复。WAL 原则仅直接约束 Redo Log 与数据页的关系；Binlog 通过 2PC 间接纳入提交协议。

## 三、Redo Log 的物理结构

### 1. ib_logfile 文件组

InnoDB 的 Redo Log 存储在一组 **固定大小** 的文件中，构成 Redo Log 文件组（Redo Log Group）。

**MySQL 5.7 及更早版本：**

- 默认两个文件：`ib_logfile0`、`ib_logfile1`，位于 `datadir` 目录。
- 单文件大小由 `innodb_log_file_size` 控制，默认 48 MB（两文件共 96 MB，具体因版本而异）。
- 文件数量由 `innodb_log_files_in_group` 控制，默认 2。
- 总容量 = `innodb_log_file_size` × `innodb_log_files_in_group`。
- 修改文件大小通常需要停机、删除旧文件、重启重建。

**MySQL 8.0.30 及之后：**

- Redo Log 迁移至 `#innodb_redo` 目录，文件命名形如 `#ib_redo<N>`。
- 使用 `innodb_redo_log_capacity` 控制总容量，默认 100 MB。
- 支持 **在线动态调整** 容量，无需停机删除文件。
- 旧参数 `innodb_log_file_size`、`innodb_log_files_in_group` 被弃用或忽略。

文件组的关键特征：

- **大小固定**：运行时不会动态增长，采用循环覆盖策略。
- **顺序写入**：逻辑上形成一个环形缓冲区。
- **预分配空间**：创建实例或调整容量时一次性分配，避免运行时扩容开销。

查看 Redo Log 配置的常用方式：

```sql
-- MySQL 8.0
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';

-- 查看 Redo Log 相关状态
SHOW ENGINE INNODB STATUS\G
-- 关注 LOG 段落中的 Log sequence number、Log flushed up to、Last checkpoint at
```

### 2. Log Block（512 字节）

Redo Log 文件在物理上被划分为 **512 字节** 的 Log Block，与磁盘扇区（Sector）大小对齐。这一设计源于传统磁盘以 512 字节为原子读写单位的假设，便于：

- 单次 I/O 读写完整 Block，避免扇区边界问题；
- 在 Block 级别计算校验和，检测日志 corruption；
- 与操作系统和存储驱动的块 I/O 对齐，提高吞吐。

每个 Log Block 的结构如下：

```
+--------------------------------------------------+
| Block Header                          12 bytes   |
+--------------------------------------------------+
| Block Body                           496 bytes   |
|   （一条或多条 Redo Log Record）                  |
+--------------------------------------------------+
| Block Trailer                          4 bytes   |
+--------------------------------------------------+
| Total                                  512 bytes |
+--------------------------------------------------+
```

**Block Header（12 字节）** 主要字段：

| 字段                          | 含义                      |
|-----------------------------|-------------------------|
| LOG_BLOCK_HDR_NO            | Block 序号，用于检测日志连续性      |
| LOG_BLOCK_HDR_DATA_LEN      | 本 Block 中有效数据长度         |
| LOG_BLOCK_HDR_FLUSH_BIT     | 标记是否为 flush block 的边界   |
| LOG_BLOCK_HDR_CHECKPOINT_NO | 关联的 Checkpoint 编号（部分场景） |

**Block Body（496 字节）** 存放实际的 Redo Log Record。一条 Redo Record 可能跨越多个 Block；一个 Block 也可能包含多条小 Record。

**Block Trailer（4 字节）** 存放 Block 级别的校验和（checksum），用于检测磁盘 corruption 或不完整写入。

InnoDB 写入 Redo Log 时，以 Block 为单位组织数据。Log Buffer 中的内容刷盘时，按 Block 边界写入 ib_logfile，并在 Trailer 中更新校验和。

### 3. 循环写入机制

Redo Log 文件组在逻辑上构成 **环形缓冲区**。InnoDB 维护两个关键位置：

- **write pos（写入指针）**：当前 Redo Log 的写入位置，随新 Record 追加而单调递增（模文件组总大小）。
- **checkpoint（检查点位置）**：对应 `last_checkpoint_lsn`，表示该 LSN 之前的 Redo 所关联的脏页 **均已刷回表空间**，因此该 LSN 之前的日志空间可以被覆盖。

可用空间的计算（概念上）：

```
可用空间 ≈ checkpoint 到 write pos 之间的"环距"
```

图示（两个文件的简化环形视图）：

```
                    ib_logfile0              ib_logfile1
              +---------------------+---------------------+
              |                     |                     |
 checkpoint ->*==== 可覆盖 ========|==== 可用写入 ======>*<- write pos
              |                     |                     |
              +---------------------+---------------------+
                              循环方向 ->
```

当 **write pos 追上 checkpoint**（可用空间耗尽）时，InnoDB 无法继续写入新 Redo，事务可能被阻塞，状态变量 `Innodb_log_waits` 增加。此时系统会 **急切刷脏（Aggressive Flush）**：

1. 加速 Page Cleaner 刷脏，推进 checkpoint；
2. 可能同步刷脏，阻塞写入直到腾出空间；
3. 若长时间无法推进，出现明显的写入停顿。

这一现象俗称 **"Redo Log 满了"**，是写密集型实例常见的性能瓶颈。根本原因是：刷脏速度跟不上 Redo 产生速度，导致 checkpoint 推进缓慢。

监控 Checkpoint Age 有助于预警：

```
Checkpoint Age ≈ current_lsn - last_checkpoint_lsn
```

Age 持续接近 Redo Log 总容量，说明需要增大日志容量或提升刷脏能力。

### 4. Redo Log 记录的内容

Redo Log 是 **物理日志（Physical Log）**：它描述的是"在哪个表空间的哪一页、哪个偏移、写入了什么字节"，而非"执行了什么 SQL"。

**物理日志 vs 逻辑日志：**

| 类型   | 示例                               | 优点       | 缺点              |
|------|----------------------------------|----------|-----------------|
| 物理日志 | 表空间 5，页 1000，偏移 200，写入 4 字节      | 恢复快，直接改页 | 与页格式强耦合，跨版本兼容性差 |
| 逻辑日志 | `UPDATE t SET age=21 WHERE id=1` | 人类可读，跨引擎 | 恢复慢，需解析执行       |

InnoDB 选择物理 Redo，是因为崩溃恢复发生在引擎启动的关键路径上，必须尽可能快地将所有脏页恢复到一致状态。

**Mini-Transaction（mtr）** 是 Redo 产生的原子单元。一个用户级 SQL 语句可能触发多个 mtr，例如：

- 单行 `UPDATE`：至少一个 mtr 修改聚簇索引页；
- 若更新了索引列，还需额外 mtr 修改二级索引页；
- 若页空间不足，可能触发 mtr 做页分裂、分配新页等。

每个 mtr 在修改页之前持有页锁，结束时：

1. 将本次修改的 Redo Record 写入 Log Buffer；
2. 释放锁；
3. 递增 LSN。

**一条 UPDATE 产生多少 Redo Log？** 典型单行 UPDATE（仅改非索引列）约数十到数百字节；若更新二级索引列，额外产生索引项删除/插入的 Redo；若触发页分裂，单次可能产生数 KB 甚至更多。Redo Record 由 `MLOG_*` 类型标识（如 `MLOG_1BYTE`、`MLOG_WRITE_STRING`），恢复时按类型解析应用。

Redo Log 还记录对 Undo 页、索引页、系统页的修改——凡是 InnoDB 页被修改都会产生 Redo，保证 Undo Log 本身也受 WAL 保护。

## 四、Log Buffer 与刷盘策略

### 1. Log Buffer 的作用

Redo Log 并非每次 mtr 结束都直接写磁盘。InnoDB 在内存中维护 **Log Buffer**，作为 Redo 写入磁盘前的缓冲区。参数 `innodb_log_buffer_size` 控制其大小，默认通常为 **16 MB**（可配置，过大对多数场景收益有限）。

Log Buffer 的职责：

- **合并写入**：多个 mtr、多个事务的 Redo Record 在 Buffer 中连续排列，刷盘时一次 I/O 写入多条记录。
- **减少 fsync 频率**：没有 Log Buffer，每次 mtr 都可能触发磁盘 I/O；有了 Buffer，可以按事务、按组、按时间批量刷出。
- **解耦前台与 I/O**：事务线程只负责写 Log Buffer（内存操作，极快），刷盘由 commit 路径或后台线程异步完成。

Log Buffer 的内部结构可简化为：一个连续字节数组 + 写指针。新 Redo 追加到尾部，刷盘时从头部或 Checkpoint 位置开始将内容写入 ib_logfile。

### 2. innodb_flush_log_at_trx_commit 三种模式

参数 `innodb_flush_log_at_trx_commit` 控制 **事务提交时** Redo Log 的刷盘行为，是 OLTP 性能与持久性之间最重要的权衡旋钮。

三种模式的行为差异如下：

| 模式        | commit 时行为     | 后台刷盘     | 持久性说明           | 适用场景           |
|-----------|----------------|----------|-----------------|----------------|
| **1（默认）** | write + fsync  | 辅助       | 进程崩溃、断电均不丢已提交事务 | 生产 OLTP，金融/订单等 |
| **0**     | 仅写 Log Buffer  | 每秒 fsync | 崩溃可能丢失最近约 1 秒事务 | 分析库、可丢数据的缓存写入  |
| **2**     | write（无 fsync） | OS 调度    | 进程崩溃通常不丢；断电可能丢失 | 持久性要求略低的场景，需实测 |

**三种模式对比表（安全性维度）：**

| 场景            | 模式 1 | 模式 2      | 模式 0     |
|---------------|------|-----------|----------|
| mysqld 被 kill | 无    | 无         | 最多 1 秒事务 |
| 服务器断电         | 无    | 取决于 OS 缓存 | 最多 1 秒事务 |
| 相对性能          | 最低   | 中等        | 最高       |
| 生产推荐          | 是    | 谨慎        | 否        |

注意：即使模式 1，若磁盘或 RAID 缓存谎报 fsync 成功（write cache 未掉电保护），仍可能丢数据。生产环境应确认存储的 **Write Through** 或 **BBU/电容保护** 配置。

### 3. 组提交（Group Commit）

组提交（Group Commit）的核心理念是：**将多个并发事务的 fsync 合并为一次**，从而打破"事务数 = fsync 次数"的线性瓶颈。MySQL 5.6 之前的实现受限于内部的 prepare_commit_mutex，高并发下提交串行化；5.6 起引入 **Binary Log Group Commit（BLGC）**，将提交过程拆分为 FLUSH → SYNC → COMMIT 三个阶段，每个阶段独立批量处理：

- **FLUSH 阶段**：将多个事务的 Binlog 从线程 Cache 刷入 Binlog 文件缓冲区，Leader 线程一次 write 完成整批；
- **SYNC 阶段**：Leader 线程对 Binlog 文件执行一次 fsync，覆盖本批次所有事务；
- **COMMIT 阶段**：Leader 线程依次调用各事务的引擎层 commit，InnoDB 侧同样将多个事务的 Redo 合并为一次 fsync。

BLGC 中，Redo 的组提交与 Binlog 的组提交在 2PC 框架下自然协调：Prepare 后进入 FLUSH 队列，SYNC 完成后全部进入 COMMIT 队列，使 commit 延迟在高并发下几乎不随事务数线性增长——实测数千并发时 TPS 可提升数倍。

### 4. Log Buffer 的其他刷盘时机

除事务 commit 外，以下情况也会触发 Log Buffer 刷盘：

| 触发条件              | 说明                                          |
|-------------------|---------------------------------------------|
| Log Buffer 使用超过一半 | 半满刷盘，避免大事务或高 Redo 速率导致 Buffer 溢出            |
| 后台线程定期刷盘          | Master Thread / log writer 约每秒刷盘；模式 0 依赖此路径 |
| Checkpoint / 空间不足 | Redo Log 环距紧张时强制刷 Log Buffer，准确推进 LSN       |
| 手动 `FLUSH LOGS`   | 备份前、切换 Binlog 前常用；只刷 Redo，不刷脏页              |
| Redo Log 空间不足     | 急切刷脏同时刷 Log Buffer，以便计算可用环距                 |

完整的刷盘路径视图：

```
Redo 产生 -> Log Buffer
                |
    +-----------+-----------+-----------+
    |           |           |           |
 commit      半满触发    后台每秒    空间不足
 (模式1,2)              刷盘        急切刷盘
    |           |           |           |
    +-----------+-----------+-----------+
                |
           write + fsync -> ib_logfile
```

## 五、LSN 与 Checkpoint 机制

### 1. LSN（Log Sequence Number）

LSN（Log Sequence Number，日志序列号）是 InnoDB 为 Redo Log 分配的 **全局单调递增** 字节偏移量。从实例创建到销毁，LSN 持续递增（模 2^64 溢出可忽略）。每向 Redo Log 写入 1 字节有效数据，LSN 增加 1。

LSN 贯穿日志子系统、Buffer Pool 和崩溃恢复，是 InnoDB 内部最重要的坐标之一。用途包括：

- 标识 Redo Log 中的逻辑位置；
- 标记数据页的"最后修改版本"（页头 `FIL_PAGE_LSN`）；
- 协调 Log Buffer 刷盘、脏页刷盘与 Checkpoint；
- 界定崩溃恢复的重放起点与终点。

**几个关键 LSN 的含义：**

| 名称                      | 常见别名                | 含义                             |
|-------------------------|---------------------|--------------------------------|
| current_lsn             | Log sequence number | 当前 Redo 写入位置，最新 LSN            |
| log_flushed_up_to_lsn   | Log flushed up to   | 已刷入 ib_logfile 并 fsync 的 LSN   |
| pages_flushed_up_to_lsn | （概念性）               | 脏页刷盘进度对应的 LSN，脏页 LSN 低于此值的页已落盘 |
| last_checkpoint_lsn     | Last checkpoint at  | 最后一次 Checkpoint 记录的 LSN，恢复起点   |

在 `SHOW ENGINE INNODB STATUS` 的 LOG 段落中，可以看到类似输出：

```
LOG
---
Log sequence number          3847291056
Log buffer assigned up to    3847291056
Log buffer completed up to   3847291056
Log written up to            3847291056
Log flushed up to            3847291056
Added dirty pages up to      3847291056
Pages flushed up to          3847289000
Last checkpoint at           3847285000
```

各 LSN 之间的关系（正常运行时）：

```
last_checkpoint_lsn  <=  pages_flushed_up_to_lsn  <=  log_flushed_up_to_lsn  <=  current_lsn
```

**Checkpoint Age** = `current_lsn - last_checkpoint_lsn`，表示尚未被 Checkpoint "消化" 的 Redo 量，也是崩溃恢复时至少需要重放的范围的上界估计。

每个数据页页头存储 **FIL_PAGE_LSN**，记录使该页达到当前状态的最后一次 Redo 的 LSN。崩溃恢复和刷脏逻辑都依赖 LSN 比较：

- 恢复时：仅当 Redo Record 的 LSN **大于** 页 LSN 时才应用（保证幂等）；
- 刷脏时：确保 log_flushed_up_to_lsn **不小于** 页的 FIL_PAGE_LSN，才允许将该页写回表空间。

### 2. Checkpoint 的作用

Checkpoint（检查点）是 InnoDB 定期执行的 **推进恢复边界** 操作。其核心目的：

**缩短崩溃恢复时间**

若无 Checkpoint，崩溃恢复需要从 Redo Log 的物理起点重放所有历史 Redo——随着运行时间增长，恢复时间不可接受。Checkpoint 记录一个 LSN 位置，声明：**该 LSN 之前产生的 Redo，其对应脏页均已刷回表空间**。恢复时只需从 `last_checkpoint_lsn` 扫描到`current_lsn`，大幅缩小重放范围。

**释放 Redo Log 循环空间**

Redo Log 文件组大小固定，采用循环覆盖。只有 checkpoint 推进后，旧日志区域才能被新 Redo 覆盖。Checkpoint 是 Redo Log 空间回收的前提。

**控制脏页比例**

Checkpoint 过程伴随刷脏，将 Buffer Pool 中的脏页写回表空间，防止脏页无限堆积占满 Buffer Pool，保障读路径有足够空闲页。

Checkpoint 信息存储在 Redo Log 的 **Checkpoint Block** 中（ib_logfile 的固定位置，通常两个交替写入以实现冗余）。包含：

- `checkpoint_lsn`：检查点 LSN；
- `checkpoint_no`：单调递增的检查点编号；
- `buf_block_t` 相关元数据等。

启动时，InnoDB 读取两个 Checkpoint Block，选择 `checkpoint_no` 较大且校验和有效的一个作为恢复起点。

### 3. Sharp Checkpoint vs Fuzzy Checkpoint

InnoDB 存在两种 Checkpoint 语义：

**Sharp Checkpoint（完全检查点）**

- **触发时机**：数据库 **正常关闭**（`SHUTDOWN`）、部分 ALTER TABLE 操作、将 Buffer Pool 完全刷空的场景。
- **行为**：强制将 Buffer Pool 中 **所有** 脏页刷回表空间，Checkpoint 推进到最新 LSN。
- **恢复影响**：重启后几乎无需 Redo 重放，启动极快。
- **代价**：关闭过程可能较慢，取决于脏页数量。

**Fuzzy Checkpoint（模糊检查点）**

- **触发时机**：实例 **运行期间** 周期性触发。
- **行为**： **选择性刷脏**，只刷部分脏页，Checkpoint **逐步** 推进，不要求所有脏页落盘。
- **恢复影响**：崩溃后需从 `last_checkpoint_lsn` 重放到 `current_lsn`，重放量取决于 Checkpoint 间隔内的 Redo 产生量。
- **优势**：刷脏分散在时间线上，避免 I/O 尖峰；关闭时不需额外长时间刷脏。

生产实例 99% 以上的 Checkpoint 都是 Fuzzy 的。Sharp Checkpoint 主要发生在计划内停机维护。

### 4. 触发 Checkpoint 的条件

Fuzzy Checkpoint 由多种条件触发，InnoDB 根据内部启发式选择刷脏批次：

| 触发条件                  | 机制说明                                                    |
|-----------------------|---------------------------------------------------------|
| Master Thread 定期触发    | 按时间或 LSN 间隔推进 Checkpoint，最常见的背景源                        |
| LRU List 空闲页不足        | 淘汰脏页前先刷脏，间接推进 checkpoint                                |
| Redo Log 空间不足         | 同步刷脏，用户线程可能阻塞；`Innodb_log_waits` 增加的主因                  |
| 脏页比例超阈值               | `innodb_max_dirty_pages_pct`（默认 90）触发 Page Cleaner 加大刷脏 |
| Flush List 老化         | 最老脏页（LSN 最小）优先刷，防止单页阻塞 checkpoint                       |
| `FLUSH LOGS` / 特定 DDL | 手动刷 Log 或表重建后影响 Checkpoint 状态                           |

核心目标：**在 Redo Log 写满之前推进 checkpoint，同时避免刷脏 I/O 压垮存储**。

## 六、崩溃恢复的完整过程

### 1. 恢复的整体流程

当 MySQL 异常退出（断电、`kill -9`、操作系统 panic）后再次启动，InnoDB 在初始化存储引擎阶段自动进入 **崩溃恢复（Crash Recovery）**。恢复目标：

- 已提交事务的修改生效且持久；
- 未提交事务的修改被完全撤销；
- 所有数据页在物理上自洽（无 torn page，页内结构合法）。

恢复分为四个逻辑阶段：

```
MySQL 启动
  -> 读取 Checkpoint Block，确定 last_checkpoint_lsn
  -> Redo 前滚：扫描 [checkpoint_lsn, current_lsn]，读页时遇 torn page（校验和不匹配）先用 Double Write 修复，再幂等应用 Redo
  -> Undo 回滚：撤销活跃事务列表中未提交事务的修改
  -> 恢复完成，接受新连接
```

Redo 与 Undo 的分工至关重要：**Redo 恢复物理页内容，Undo 处理事务语义**。两者缺一不可。

### 2. Redo 前滚

Redo 阶段从 `last_checkpoint_lsn` 开始，顺序扫描 Redo Log 直到 `current_lsn`（崩溃前最后的 LSN）。对每条 Redo Record：

1. 解析 Record 类型、表空间 ID、页号、偏移、变更内容；
2. 读取目标页（从表空间文件或 Buffer Pool；恢复阶段通常直接读盘）；
3. 比较 Redo Record 的 LSN 与页的 `FIL_PAGE_LSN`；
4. 若 Record LSN **>** 页 LSN，应用变更，更新页 LSN；
5. 若 Record LSN **<=** 页 LSN， **跳过**（该修改已体现在页上）。

**幂等性** 是 Redo 的核心属性：同一条 Redo Record 应用一次与应用多次，结果相同。这通过 LSN 比较保证——只有日志 LSN 严格大于页 LSN 时才应用。因此：

- 已刷盘的脏页不会被 Redo "改旧"；
- 部分应用的页可以安全地继续应用后续 Redo；
- 恢复过程可以安全重试。

Redo 是 **物理级别** 的：不解析 SQL，不遍历 B+ 树语义，只按字节修改页内容。恢复速度主要取决于：

- Checkpoint Age（需扫描的 Redo 量）；
- 磁盘顺序读 Redo Log 的吞吐；
- 涉及的 distinct 页数量（随机读表空间）。

一个常见误解："Redo 只重做已提交事务"。实际上，**Redo 重做所有已写入 Redo Log 的物理修改，不区分事务是否提交**。未提交事务的修改若已写入 Redo，同样会被 Redo 应用到页上——随后由 Undo 阶段撤销。这一设计简化了 Redo 逻辑：Redo 只管"页的最新物理状态"，事务语义交给 Undo。

### 3. Undo 回滚

Redo 阶段完成后，所有数据页在物理上已恢复到崩溃瞬间的状态——包括未提交事务已经修改但尚未 commit 的页。此时需要 **Undo 阶段** 清理这些不应可见的修改。

InnoDB 在事务执行过程中为 DML 操作写 **Undo Log**，记录如何反向撤销修改。崩溃时，系统通过 Redo 恢复后的 Trx Sys（事务系统表）找到**活跃事务列表** 中仍处于未提交状态的事务，对每个这样的事务：

1. 沿 Undo Log 链从最新到最旧依次应用 Undo Record；
2. 将数据页恢复到事务开始前的状态（或标记事务已回滚）；
3. 释放事务持有的锁资源（内存结构重建）。

Undo 回滚有两种策略：**Eager Rollback** 在恢复阶段立即回滚所有未提交事务，启动较慢但完成后数据立即可用；**Lazy Rollback**仅标记待回滚，首次访问相关行时才触发 Undo，启动快但首批访问可能延迟。不同版本策略有差异，但最终所有未提交事务都必须被回滚。

| 阶段   | 处理对象            | 目的        | 日志来源     |
|------|-----------------|-----------|----------|
| Redo | 所有已写 Redo 的物理修改 | 恢复未落盘脏页   | Redo Log |
| Undo | 未提交事务的逻辑修改      | 撤销不应保留的变更 | Undo Log |

### 4. 恢复时间的影响因素

崩溃恢复耗时主要受以下因素影响：

| 因素             | 影响机制                  | 可控性            |
|----------------|-----------------------|----------------|
| Checkpoint Age | Age 越大，需重放的 Redo 越多   | 高：调 Redo 容量与刷脏 |
| Redo Log 总大小   | 间接影响 Age 累积上限         | 中              |
| distinct 页数量   | 重放时表空间随机读次数           | 中：取决于 workload |
| 未提交事务 / Undo 链 | 大事务未提交则回滚耗时           | 高：避免长事务        |
| torn page 修复   | Double Write 拷贝额外 I/O | 低              |
| 存储性能           | SSD 数秒级，HDD 可能数分钟     | 中              |

优化建议：Redo 容量设为峰值 1–2 小时写入量；调优 `innodb_io_capacity`；避免长事务未提交；计划内 `SHUTDOWN` 触发 Sharp Checkpoint。

## 七、Double Write Buffer

### 1. 页断裂问题

InnoDB 数据页大小为 **16 KB**，而传统磁盘和多数文件系统的 **原子写入单位是 512 字节**（一个扇区）。当 Page Cleaner 将脏页写回表空间时，一次 `write()` 系统调用写入 16 KB，底层可能分解为 32 个扇区写入。若在写入过程中发生断电或内核 panic，可能出现 **部分扇区已更新、部分扇区仍为旧值** 的情况。

这种 **页断裂（Partial Page Write / Torn Page）** 与"页未更新"有本质区别：

| 情况         | 页状态        | Redo 能否修复               |
|------------|------------|-------------------------|
| 页未更新（脏页未刷） | 完整旧页       | 能，Redo 前滚即可             |
| 页断裂        | 半新半旧，校验和失败 | **不能**，Redo 基于损坏基线会加剧错乱 |

Redo Log 是物理日志，恢复时读取表空间中的页作为基线，再应用 Redo。若基线页本身已损坏（例如 B+ 树节点指针指向非法地址），应用 Redo 不仅无法修复，还可能破坏相邻数据。

页断裂在以下环境更易发生：

- HDD + 无 BBU 的 RAID 控制器（写缓存未掉电保护）；
- 虚拟机突然断电；
- 部分老旧文件系统。

### 2. Double Write 的工作原理

**Double Write Buffer（双写缓冲）** 是 InnoDB 为页断裂提供的解决方案。刷脏页到表空间 **之前**，InnoDB 先将整页写入 Double Write 区域，待该写入完整落盘后，再写入表空间目标位置。

Double Write 区域位于 **系统表空间（ibdata1）** 的固定区域，或 MySQL 8.0 的 **独立 Double Write 文件** 中。该区域大小固定（通常 2 MB，可存放 128 个 16 KB 页），写入方式为 **顺序追加**。

刷脏流程：

```
1. Page Cleaner 从 Flush List 选取脏页 P
2. 将 P 的完整 16 KB 拷贝到内存中的 Double Write Buffer 批次
3. 批次凑满或超时后，顺序写入 Double Write 磁盘区域
4. fsync Double Write 区域
5. 将 P 写入表空间中的目标位置（随机写）
6. （若步骤 5 中断）页 P 在表空间中可能 torn
7. 恢复时从 Double Write 区域拷贝 P 的完整副本覆盖表空间中的损坏页
8. 对修复后的页应用 Redo Log（若页 LSN 落后于 Redo）
```

Double Write 的关键洞察：**顺序写 16 KB 到连续区域，原子性风险远低于随机写 16 KB 到分散位置**。即使步骤 5 失败，步骤 3–4 已保证完整页副本存在于 Double Write 区域。

MySQL 8.0 引入 **Double Write 多页并行刷盘** 和 **独立 doublewrite 文件** 等优化，但核心原理不变。

### 3. 崩溃恢复时的页修复

崩溃恢复在 Redo 前滚 **之前或期间**，会检查数据页完整性：

1. 读取表空间中的页，计算 **校验和（checksum）** ，与页头存储的 `FIL_PAGE_SPACE_OR_CHKSUM` 比较；
2. 若校验和不匹配，判定为 torn page 或 corruption；
3. 根据页号在 Double Write Buffer 中查找对应的完整页副本；
4. 若找到，用 Double Write 中的完整页 **覆盖** 表空间中的损坏页；
5. 对修复后的页继续正常的 Redo 前滚流程。

若 Double Write 中也没有有效副本（极罕见，如 Double Write 写入本身失败），InnoDB 可能报错拒绝启动，需要从备份恢复。

Double Write 与 Redo 的协作关系：

```
正常刷脏:  Double Write（完整页） -> 表空间（可能 torn）
崩溃恢复:  表空间（torn?） -> Double Write 修复 -> Redo 前滚 -> 一致页
```

Redo 假设表空间中的页 **要么是完整的旧页，要么是完整的新页（经 Double Write 保证）**。Double Write 确保这一假设在 partial write 场景下成立。

### 4. 对性能的影响

Double Write 的代价是 **写放大**：每个脏页实际写入磁盘两次——一次 Double Write 区域（顺序），一次表空间（随机）。理论上写 I/O 量增加近一倍。

实际影响通常 **小于理论值**：

- Double Write 写入是 **顺序 I/O**，延迟低，可与随机写并行；
- 批次写入摊薄 fsync 开销；
- 实测多数 OLTP 场景下性能影响约 **2%–5%**，写极密集场景可能达 10%。

**关闭 Double Write 的条件：**

参数 `innodb_doublewrite` 控制开关（默认 ON）。以下场景可考虑关闭，但 **必须充分验证**：

- 文件系统或存储设备保证 **大于 16 KB 的原子写**（如 ZFS 的 recordsize 配置、部分企业级 SAN）；
- 使用支持原子写的特殊硬件（如 Fusion-io，已较少见）；
- MySQL 8.0.20+ 支持 `innodb_doublewrite=DETECT_AND_RECOVER` 等模式，可按需调整。

生产环境 **默认应开启** Double Write。关闭后若发生 torn page，只能依赖备份恢复，Redo Log 无法挽救。

## 八、Redo Log 大小设置

### 1. 太小的影响

Redo Log 容量过小（如默认 100 MB 用于写密集型实例）会导致：write pos 快速逼近 checkpoint，频繁同步刷脏；`Innodb_log_waits` 增加、commit 延迟飙升；写入 TPS 提前 plateau；Checkpoint Age 在低位与满容量间震荡，刷脏 I/O 呈脉冲型。

典型症状汇总：

| 现象                      | 可能原因       |
|-------------------------|------------|
| `Innodb_log_waits` 持续增加 | Redo 空间不足  |
| 写入 TPS 上不去              | Redo 满导致阻塞 |
| Checkpoint Age 接近总容量    | 刷脏跟不上      |
| 错误日志 Redo 相关警告          | 空间耗尽       |

### 2. 太大的影响

增大 Redo Log 可缓解 Log waits，但过大也有代价：长期高 Age 导致崩溃恢复从数秒延长到数分钟（影响 RTO）；预分配占用磁盘（如 8 GB 容量即占 8 GB 空间）；可能掩盖刷脏配置不当——Age 长期很高却不触发 Log waits，直到崩溃才暴露。过大 Redo **不会** 增加运行时内存（Log Buffer 独立配置）。

### 3. 合理的设置方法

**估算一小时内 Redo 产生量**

```sql
-- 记录起始 LSN
SHOW ENGINE INNODB STATUS;  -- 记下 Log sequence number

-- 等待 1 小时（或业务高峰 1 小时）

-- 再次查看
SHOW ENGINE INNODB STATUS;  -- 计算 LSN 差值
```

LSN 差值（字节）≈ 一小时内 Redo 写入量。也可通过 `Innodb_os_log_written` 状态变量的增量估算。

**建议设置为 1–2 小时的写入量**

行业常见经验：

- 容量 = 峰值写入速率 × 1~2 小时；
- 写密集型 OLTP：1 GB – 4 GB 常见，极端场景 8 GB 或更高；
- 读多写少：默认 100 MB – 512 MB 可能足够。

目标是：在业务高峰下，Checkpoint Age 通常不超过总容量的 75%，且 `Innodb_log_waits` 接近 0。

**MySQL 8.0 动态调整**

```sql
-- 在线调整，无需重启
SET GLOBAL innodb_redo_log_capacity = 2147483648;  -- 2 GB
```

MySQL 8.0.30+ 支持运行时调整 `#innodb_redo` 容量。调整期间 InnoDB 会平滑迁移，但建议在低峰期操作并观察 Checkpoint 行为。

MySQL 5.7 修改 `innodb_log_file_size` 需停机、删除 `ib_logfile*`、重启重建，应在维护窗口操作。Redo 容量需与刷脏能力匹配：`innodb_io_capacity`（SSD 2000–10000）、`innodb_io_capacity_max`（通常为 2 倍）、`innodb_max_dirty_pages_pct`（写密集可略降）。监控 `Innodb_os_log_written`、Checkpoint Age、`Innodb_log_waits`、`Innodb_buffer_pool_pages_dirty`。

## 总结

Redo Log 是 InnoDB 实现 WAL 与事务持久性的基石。它通过顺序追加物理日志，将数据页的随机写延迟到后台批量刷脏，在不牺牲 Durability 的前提下大幅提升写入性能。理解 Redo Log，需要把握以下主线：

**动机与矛盾。** Buffer Pool 的延迟写策略与提交持久性之间存在张力。Redo Log 以"先记日志、后刷数据页"的 WAL 方式解决这一矛盾，使 commit 延迟取决于日志顺序写而非数据页随机写。

**WAL 三保证。** 修改进 Log Buffer、提交时 Redo fsync、刷脏前 Redo 已在磁盘——三者串联构成完整 WAL 闭环。Binlog 通过两阶段提交与 Redo 协调，服务复制而非引擎崩溃恢复。

**物理结构。** ib_logfile 文件组（或 `#innodb_redo` 目录）循环写入；512 字节 Log Block 与扇区对齐；write pos 与 checkpoint 界定可用空间与恢复起点；mtr 产生类型化物理 Redo Record。

**刷盘策略。** Log Buffer 合并 Redo 写入；`innodb_flush_log_at_trx_commit` 在性能与持久性间权衡，生产 OLTP 应使用模式 1；组提交合并 fsync 提升高并发吞吐。

**LSN 与 Checkpoint。** LSN 是全局日志坐标；Checkpoint 推进恢复边界并回收日志空间；Fuzzy Checkpoint 运行时逐步刷脏，Sharp Checkpoint 在正常关闭时刷全部脏页。

**崩溃恢复。** Redo 从 checkpoint_lsn 前滚，幂等应用物理修改；Undo 回滚未提交事务；恢复时间主要取决于 Checkpoint Age 与存储 I/O。

**Double Write。** 防止 16 KB 页的部分写入（torn page）；Redo 无法修复损坏基线页，Double Write 提供完整页副本；与 Redo 互补构成完整安全网。

**容量调优。** Redo Log 过小导致 Log waits 与刷脏风暴；过大延长恢复时间；建议设为峰值 1–2 小时 Redo 写入量，MySQL 8.0 支持在线调整，并配合 `innodb_io_capacity` 保证刷脏跟上。

Redo Log 与 Buffer Pool、Undo Log、Binlog、Double Write 紧密协作，共同支撑 InnoDB 的 ACID 与崩溃恢复能力。掌握 Redo Log 的机制，是深入理解 InnoDB 事务、复制、备份恢复与性能调优的必要基础。在此基础上可继续学习 Undo Log 与 MVCC、Buffer Pool 内部机制，以及 Binlog 两阶段提交与基于 Binlog 的时间点恢复，以建立完整的 MySQL 持久化与恢复知识图谱。
