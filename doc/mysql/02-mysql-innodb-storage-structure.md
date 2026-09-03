---
title: InnoDB 存储结构：表空间、数据页、行格式与磁盘组织
summary: 从表空间到数据页，从行格式到页内记录组织，系统拆解 InnoDB 在磁盘上如何存储数据，为理解索引、日志和 Buffer Pool 建立物理基础。
created: 2026-07-02
updated: 2026-07-03
tags: MySQL, InnoDB, 存储结构, 数据页
---

# InnoDB 存储结构：表空间、数据页、行格式与磁盘组织

讨论 InnoDB 的索引、事务、日志和 Buffer Pool 之前，需要先回答一个更底层的问题：数据在磁盘上究竟以什么形态存在。逻辑上我们操作的是表和行，物理上 InnoDB 管理的是表空间、段、区、页以及页内的记录组织。只有建立这一层物理视图，后续关于 B+ 树分裂、页分裂、Redo Log 重放、Buffer Pool 刷脏等机制才有明确的锚点。

本文聚焦 InnoDB 的存储结构，从 `CREATE TABLE` 执行后磁盘文件的变化出发，逐层展开表空间分类、段区页层次、16 KB 数据页内部布局、行格式编码、大字段溢出策略，以及页内记录链表与页目录查找算法。每个知识点都会给出原理说明、实现细节与具体示例，目标是为理解索引、日志和 Buffer Pool 建立完整的物理基础。

## 一、从逻辑表到物理文件

### 1. CREATE TABLE 后磁盘上发生了什么

当我们在 MySQL 中执行一条建表语句时，Server 层与 InnoDB 引擎层会协同完成从逻辑定义到物理存储的映射。以 MySQL 8.0 默认配置为例：

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(64) NOT NULL,
    email VARCHAR(128),
    created_at DATETIME NOT NULL
) ENGINE=InnoDB;
```

这条语句触发的操作远不止"在内存里注册一张表"。Server 层解析 SQL、在数据字典（MySQL 8.0 存储于 InnoDB 表空间内）注册表元信息，并调用 InnoDB 创建存储结构。InnoDB 层在 `innodb_file_per_table=ON`（8.0 默认）下创建 `users.ibd`，初始化 FSP Header 页（第 0 页），为聚簇索引分配根节点页，并将表空间 ID 回写数据字典。未显式创建二级索引时，表空间内仅有一棵聚簇索引 B+ 树。

执行完成后，在数据目录（如 `/var/lib/mysql/test/`）下可以看到：

```
test/
├── users.ibd          # InnoDB 独立表空间，存放 users 表数据
└── ...                # 其他系统文件
```

注意：`users.frm` 文件在 MySQL 8.0 中已不存在，表结构完全由 InnoDB 数据字典管理。若使用 MySQL 5.7，仍可能看到 `.frm`文件，但数据本身仍在 `.ibd` 中。

**首次 INSERT 时**

空表刚创建时，`.ibd` 文件很小（通常只有若干页，约 112 KB 量级），聚簇索引 B+ 树只有一个空的叶子根节点。当执行第一条 INSERT：

```sql
INSERT INTO users (name, email, created_at) VALUES ('Alice', 'alice@example.com', '2026-07-02 10:00:00');
```

InnoDB 会：

1. 在 Buffer Pool 中定位或加载聚簇索引叶子页。
2. 按行格式编码记录，插入页内 User Records 区。
3. 更新 Page Header、Page Directory 等元信息。
4. 写 Redo Log，标记页为脏页，后续异步刷盘。

此时 `.ibd` 文件大小基本不变，变化的是页内二进制内容。随着数据增长，InnoDB 会从段中申请新页，`.ibd` 文件逐步扩大。

### 2. .ibd 文件与表空间的对应关系

`.ibd` 是 InnoDB 表空间文件在文件系统上的载体，二者是一一对应关系：一个独立表空间对应一个 `.ibd` 文件，文件内容即该表空间的全部页集合。

**表空间的寻址方式**

InnoDB 用 `(space_id, page_no)` 二元组在全局范围内定位一个页：

- `space_id`：表空间 ID，由 InnoDB 在创建表空间时分配，写入数据字典。
- `page_no`：页在表空间内的逻辑序号，从 0 开始，每个页固定大小（默认 16 KB）。

在 `.ibd` 文件中，第 `page_no` 页的物理偏移为：

```
offset = page_no × innodb_page_size
```

例如，默认 16 KB 页大小下，第 3 页的偏移为 `3 × 16384 = 49152` 字节。

**`.ibd` 文件内部结构概览**

打开 `.ibd` 文件，看到的不是文本或逐行排列的记录，而是按页切分的二进制序列：

```
+--------+--------+--------+-----+
| Page 0 | Page 1 | Page 2 | ... |   每个 Page 16 KB
+--------+--------+--------+-----+
```

各页职责不同：

| 页号 | 典型类型           | 作用               |
|----|----------------|------------------|
| 0  | FSP_HDR / XDES | 表空间头，管理区、段、空闲页   |
| 1  | IBUF_BITMAP    | Insert Buffer 位图 |
| 2  | INODE          | 段 inode 入口       |
| 3+ | INDEX / BLOB 等 | 索引数据页、溢出页等       |

**逻辑表与物理文件的映射关系**

需要强调：逻辑上的"一张表"并不对应"一个文件里连续排列的行"。一张 InnoDB 表在物理上对应：

- 一个表空间（通常一个 `.ibd` 文件）。
- 表空间内一棵聚簇索引 B+ 树（叶子节点即数据页，行数据按主键顺序组织）。
- 每个二级索引各有一棵独立 B+ 树，同样存储在同一表空间内。

因此，查询 `SELECT * FROM users WHERE id = 100` 的物理路径是：通过 B+ 树从根节点逐层下降，最终在某一叶子页内定位 `id=100`的记录——而非在文件中"顺序扫描到第 100 行"。

### 3. innodb_file_per_table 的历史与现状

`innodb_file_per_table` 控制用户表是否使用独立表空间，是理解 InnoDB 文件布局的关键参数。

**历史演进**

| 时期                 | 默认配置                  | 用户表数据存放位置          |
|--------------------|-----------------------|--------------------|
| MySQL 4.x / 5.0 早期 | OFF                   | 全部在共享表空间 `ibdata1` |
| MySQL 5.5 / 5.6    | OFF（可改）               | 仍以 `ibdata1` 为主    |
| MySQL 5.6.6+       | ON（可改）                | 每表独立 `.ibd`        |
| MySQL 8.0          | ON（不可改回 OFF 的某些场景需注意） | 每表独立 `.ibd`        |

**共享表空间时代的问题**

`innodb_file_per_table=OFF` 时，所有用户表数据写入 `ibdata1`，导致：`DROP TABLE` 后文件系统空间无法回收；无法单表备份迁移；单文件过大且损坏风险集中。

**独立表空间的优势**

`innodb_file_per_table=ON` 下每表独立 `.ibd`：`DROP TABLE` 即释放空间；支持 Transportable Tablespace 单表迁移；可按表启用压缩与加密；故障影响范围限定于单表。

**现状与注意事项**

MySQL 8.0 默认且推荐 `innodb_file_per_table=ON`。即使启用独立表空间，以下信息仍存放在系统表空间 `ibdata1` 中：

- Change Buffer。
- Doublewrite Buffer（可选独立配置）。
- 部分系统内部结构。

数据字典在 MySQL 8.0 中已独立为 InnoDB 内部表，存放在 `mysql.ibd` 表空间中，不再驻留 `ibdata1`（注意与 5.7 及更早版本的区别）。

Undo Log 在 MySQL 8.0 默认独立到 `undo_001`、`undo_002` 等文件，不再写入 `ibdata1`。因此，现代 InnoDB 实例的磁盘布局是"系统表空间 + 每表 `.ibd` + Undo 表空间 + 临时表空间"的组合，而非早期"一个 `ibdata1` 包打天下"的模式。

## 二、表空间的种类与作用

InnoDB 将磁盘空间划分为多种表空间类型，各自承担不同职责。理解这些类型，是分析 I/O 分布、空间膨胀和备份策略的前提。

### 1. 系统表空间（ibdata1）

系统表空间（System Tablespace）是 InnoDB 实例启动时创建的核心表空间，默认文件为 `ibdata1`，可通过 `innodb_data_file_path`配置多个文件或自动扩展。

**主要存储内容**

| 内容                      | 说明                           |
|-------------------------|------------------------------|
| Change Buffer           | 对二级索引页的 deferred 写操作的缓冲      |
| Doublewrite Buffer      | 防止 partial page write 的页副本   |
| 系统事务信息                  | 部分内部系统表                      |
| Undo Log（MySQL 5.7 及以前） | 旧版本将 Undo 放在 ibdata1，8.0 已独立 |

> **MySQL 8.0 的重要变化**：数据字典已从 `ibdata1` 中迁出，改为 InnoDB 内部系统表，存储在 `mysql.ibd` 表空间中。`ibdata1` 在 8.0 中的职责大幅简化，主要保留 Change Buffer、Doublewrite Buffer 及少量内部结构。

MySQL 8.0 将数据字典重构为 InnoDB 内部系统表，存放于 `mysql.ibd` 表空间，不再依赖 `ibdata1`。可通过`innodb_data_file_path = ibdata1:12M:autoextend` 配置自动扩展。

### 2. 独立表空间（file-per-table）

独立表空间（File-Per-Table Tablespace）是每张用户表一个 `.ibd` 文件的模式，由 `innodb_file_per_table=ON` 启用。

独立表空间文件位于数据库目录，命名为 `<表名>.ibd`，与表一一对应。每个 `.ibd` 包含聚簇索引与全部二级索引的 B+ 树、可能的 BLOB 溢出页，以及 FSP Header 等管理页；各索引共享表空间但拥有独立段。

### 3. 通用表空间

通用表空间（General Tablespace）是 MySQL 5.7 引入的特性，允许 DBA 显式创建表空间，并将多张表放入其中。

**创建与使用**

```sql
CREATE TABLESPACE ts1 ADD DATAFILE 'ts1.ibd' ENGINE=InnoDB;

CREATE TABLE t1 (id INT PRIMARY KEY) TABLESPACE ts1;
CREATE TABLE t2 (id INT PRIMARY KEY) TABLESPACE ts1;
```

通用表空间适合将多表集中存放于指定磁盘路径、跨库共享表空间的 I/O 规划场景；DROP TABLE 仅移除表数据，表空间文件保留。日常 OLTP 仍以 file-per-table 为主。

### 4. 临时表空间

临时表空间（Temporary Tablespace）存放临时表数据，包括用户显式创建的 `CREATE TEMPORARY TABLE` 和优化器内部临时表。

默认文件为 `ibtmp1`，由 `innodb_temp_data_file_path` 配置；实例重启时重建，内容不持久化，也不纳入常规备份。MySQL 8.0 改进了临时表空间回收并引入独立 Temp Undo。复杂查询产生大临时表时需监控 `ibtmp1` 增长。

### 5. Undo 表空间（MySQL 8.0 独立化）

Undo Log 用于 MVCC 读视图和事务回滚。MySQL 8.0 将 Undo 从系统表空间独立出来，默认使用 Undo 表空间（Undo Tablespace）。

MySQL 8.0 默认创建 `undo_001`、`undo_002` 两个 Undo 表空间（`innodb_undo_tablespaces` 可增），采用循环写满后切换、旧空间 truncate 回收的策略，避免 Undo 撑大 `ibdata1`。Undo 表空间存放 Undo Segment 与 Undo Page，记录 insert/update 的前镜像；长事务会延迟 purge 并撑大 Undo 空间，需运维关注。

## 三、表空间的层次结构：段、区、页

InnoDB 在表空间内部采用四级层次组织磁盘空间：表空间 → 段 → 区 → 页。这一设计面向 B+ 树索引的访问模式，在分配效率、空间局部性和碎片控制之间取得平衡。

### 1. 页（Page）：最小 IO 单位

页（Page）是 InnoDB 管理磁盘和内存的最小逻辑单元，也是与操作系统、Buffer Pool、Redo Log 交互的基本粒度。

**默认 16 KB，为什么选择这个大小**

InnoDB 默认页大小为 16 KB，可在实例初始化时通过 `innodb_page_size` 设为 4 KB、8 KB、16 KB 或 32 KB，一旦初始化完成不可更改。16 KB 成为默认值的原因包括：

1. **与操作系统页对齐**：多数 Linux 系统内存页为 4 KB，16 KB 是其整数倍，便于内存映射与 DMA。
2. **B+ 树扇出**：16 KB 可容纳数百个索引项（取决于键长），使树高控制在 2–4 层，单次查找只需少量随机 IO。
3. **顺序 IO 效率**：16 KB 在随机读延迟与顺序读吞吐之间折中；过小则树高增加、IO 次数增多，过大则单页缓存占用高、更新粒度粗。
4. **历史兼容**：早期 InnoDB 与 MySQL 生态均围绕 16 KB 优化，工具链和运维经验成熟。

**页的全局标识**

每个页由 `(space_id, page_no)` 唯一标识。Buffer Pool 中的 Page Frame、Redo Log 中的页修改记录、Doublewrite 中的页副本，都使用这一标识。

**不同页类型**

InnoDB 定义多种页类型，通过 File Header 中的 `FIL_PAGE_TYPE` 区分：

| 类型                      | 值（示意） | 用途                 |
|-------------------------|-------|--------------------|
| FIL_PAGE_INDEX          | 17855 | B+ 树索引节点（数据页/索引页）  |
| FIL_PAGE_UNDO_LOG       | 2     | Undo Log 页         |
| FIL_PAGE_INODE          | 3     | 段 inode 页          |
| FIL_PAGE_IBUF_FREE_LIST | 4     | Insert Buffer 空闲列表 |
| FIL_PAGE_TYPE_ALLOCATED | 0     | 新分配未使用             |
| FIL_PAGE_BLOB           | 10    | 溢出页（存储 BLOB/TEXT）  |
| FIL_PAGE_TYPE_FSP_HDR   | 8     | 表空间头               |
| FIL_PAGE_TYPE_XDES      | 9     | 区描述页               |

日常开发中接触最多的是 `FIL_PAGE_INDEX`（索引/数据页）和 `FIL_PAGE_BLOB`（大字段溢出页）。Undo 页、系统页在分析事务和崩溃恢复时会涉及。

**页与 IO 的关系**

无论读取 1 行还是 1 字节，InnoDB 都必须读取整页到 Buffer Pool。这是磁盘索引与内存指针结构的根本差异，也是为什么索引设计强调"减少页访问次数"而非"减少字节比较次数"。

### 2. 区（Extent）

区（Extent）是连续的 64 个页组成的分配单元。在 16 KB 页大小下，一个区占 1 MB 磁盘空间。

**连续 64 页的意义**

B+ 树同一层的相邻叶子节点在范围扫描时往往被顺序访问。将同一索引段的页尽量分配在物理连续的区中，可以提高顺序 IO 效率，降低磁盘寻道开销。区也是 InnoDB 向磁盘"批量要空间"的单位，比逐页分配更高效。

**区的分配策略**

InnoDB 在**表空间级别**维护区链表，管理所有不属于任何段的空闲或碎片区：

| 链表                  | 含义                       |
|---------------------|--------------------------|
| FSP_FREE            | 完全空闲的区，64 页均未使用          |
| FSP_FREE_FRAG       | 部分使用的碎片区，单页可被段独立分配       |
| FSP_FULL_FRAG       | 碎片区中已用完的区               |

每个**段（Segment）**在其 inode 中维护自己的区链表：

| 链表               | 含义            |
|------------------|---------------|
| FSEG_FREE        | 已分配给段、尚未使用的区  |
| FSEG_NOT_FULL    | 段内部分使用的区      |
| FSEG_FULL        | 段内已用完的区       |

**分配流程（简化）**

1. 段需要新页时，优先从 FSEG_NOT_FULL 中取单个空闲页。
2. 若 FSEG_NOT_FULL 无可用页，从 FSEG_FREE 取一个区，移入 FSEG_NOT_FULL，再从中取页。
3. 若 FSEG_FREE 也无可用区，则从表空间的 FSP_FREE 链表分配一个完整区给该段。
4. 当某区 64 页全部用完，该区移入 FSEG_FULL。
5. 段内某区全部页被释放时，整区归还表空间 FSP_FREE 链表。

这种"优先用碎片区、必要时分配新区"的策略，在减少碎片和控制分配开销之间折中。对于自增主键顺序插入，InnoDB 还会尝试在同一 Extent 内连续分配，利用 `PAGE_LAST_INSERT` 等优化顺序写。

**XDES 与 FSP Header**

表空间的第 0 页（FSP_HDR）内含 XDES（Extent Descriptor）数组，记录前 256 个区的元数据：属于哪个段、已用页数、空闲页 bitmap 等。当区数量超过 256 时，InnoDB 会分配独立的 XDES 页（`FIL_PAGE_TYPE_XDES`）来扩展管理能力。

### 3. 段（Segment）

段（Segment）是 InnoDB 为每棵 B+ 树索引分配的空间管理单元，是连接"逻辑索引"与"物理页集合"的桥梁。

**段的类型**

| 段类型                      | 对应 B+ 树部分 | 说明           |
|--------------------------|-----------|--------------|
| 叶子节点段（Leaf Segment）      | 叶子层全部页    | 聚簇索引叶子段即数据页  |
| 非叶子节点段（Non-Leaf Segment） | 内部节点页     | 根到叶路径上的索引项   |
| 回滚段（Rollback Segment）    | Undo 相关   | 存放在 Undo 表空间 |

InnoDB 将叶子与非叶子分开管理，因为二者的增长模式不同：叶子段随数据插入持续扩展，非叶子段仅在树增高或分裂上层时增长。分离管理可避免叶子大量分配时与非叶子争抢同一区链表。

**段与 B+ 树的对应关系**

一张有主键和两个二级索引的 InnoDB 表，在表空间内至少有：

```
聚簇索引
  ├── 非叶子段
  └── 叶子段（存行数据）

二级索引 idx_a
  ├── 非叶子段
  └── 叶子段

二级索引 idx_b
  ├── 非叶子段
  └── 叶子段
```

共 6 个段。每个段拥有独立的 inode 结构，记录该段拥有的区列表、已用页数、空闲页数。B+ 树节点分裂需要新页时，向所属段申请；段内空闲页不足时，段向表空间申请新区。

当段内空闲页积累超过阈值（约 32 页），InnoDB 可能将整区归还表空间。自增主键顺序插入时，聚簇索引叶子段从 FSEG_NOT_FULL（或表空间碎片区 FSP_FREE_FRAG 单页分配）取页写行，Extent 填满后申请新区，页满则分裂——逻辑上的"插入一行"即转化为页内写入与必要的段、区级空间扩展。

## 四、数据页的内部结构（重点展开）

InnoDB 数据页（Index Page，`FIL_PAGE_INDEX`）是 B+ 树节点的载体，也是行数据最终落盘的位置。一个 16 KB 的页并非简单堆叠记录，而是有严格的区域划分和元数据管理。理解页内布局，是分析页内查找、插入、分裂的基础。

### 1. File Header（38 字节）

File Header 位于页的起始 38 字节，存放与页定位、校验、链表相关的元信息，对所有页类型通用。

**关键字段**

| 字段                            | 含义                 |
|-------------------------------|--------------------|
| FIL_PAGE_OFFSET               | 页号（page_no）        |
| FIL_PAGE_PREV / FIL_PAGE_NEXT | 同层 B+ 树叶子页双向链表     |
| FIL_PAGE_LSN                  | 页最后一次修改对应的 LSN     |
| FIL_PAGE_TYPE                 | 页类型（INDEX=17855 等） |
| FIL_PAGE_SPACE_ID             | 表空间 ID             |

**页号与 LSN 的作用**

- **页号**：InnoDB 通过 `(space_id, page_no)` 定位页。File Header 中的 `FIL_PAGE_OFFSET` 必须与页在文件中的实际位置一致，否则说明页损坏或指针错误。
- **LSN（Log Sequence Number）**：记录页最后一次被 Redo Log 覆盖的日志序号。崩溃恢复时，InnoDB 比较页的 LSN 与 Redo Log 的 LSN，决定是否需要重放。Buffer Pool 刷脏时也依赖 LSN 协调顺序。
- **FIL_PAGE_PREV / FIL_PAGE_NEXT**：构成 B+ 树同层叶子节点的双向链表。范围扫描 `WHERE id BETWEEN 100 AND 200` 在定位起始叶后，可沿 NEXT 指针顺序读后续页，无需反复从根查找。

**示例：叶子页链表**

```
Leaf Page 5  <--->  Leaf Page 8  <--->  Leaf Page 12
   PREV=0          PREV=5             PREV=8
   NEXT=8          NEXT=12            NEXT=0
```

`NEXT=0` 表示链表末尾。内部节点页通常不使用 PREV/NEXT（或值为 0），链表主要服务于叶子层范围扫描。

### 2. Page Header（56 字节）

Page Header 紧跟 File Header，占用 56 字节，存放索引页特有的管理信息。

**关键字段详解**

| 字段                                   | 含义                                        |
|--------------------------------------|-------------------------------------------|
| PAGE_N_DIR_SLOTS                     | Page Directory 中的槽数量                      |
| PAGE_HEAP_TOP                        | 堆顶指针，指向 User Records 与 Free Space 的分界     |
| PAGE_N_HEAP                          | 页内堆记录数（含 Infimum/Supremum 及已删除记录）         |
| PAGE_FREE                            | 已删除记录组成的空闲链表头偏移                           |
| PAGE_GARBAGE                         | 已删除记录占用的总字节数                              |
| PAGE_LAST_INSERT                     | 最近插入记录的位置                                 |
| PAGE_DIRECTION                       | 最近插入方向（LEFT=1 / RIGHT=2 / NO_DIRECTION=0） |
| PAGE_N_DIRECTION                     | 同一方向连续插入的次数                               |
| PAGE_N_RECS                          | 有效用户记录数（不含已删除、不含 Inf/Sup）                 |
| PAGE_MAX_TRX_ID                      | 页内记录的最大事务 ID（二级索引页）                       |
| PAGE_LEVEL                           | B+ 树层级，0 表示叶子                             |
| PAGE_INDEX_ID                        | 所属索引 ID                                   |
| PAGE_BTR_SEG_LEAF / PAGE_BTR_SEG_TOP | 叶子段、非叶子段的 inode 位置                        |

**PAGE_HEAP_TOP 与空间增长**

User Records 从 Infimum/Supremum 之后向高地址"堆式"增长。`PAGE_HEAP_TOP` 指向当前已使用空间的顶部，其上方至 Page Directory 之间为 Free Space。新记录插入时，通常从 `PAGE_HEAP_TOP` 处分配空间并上移堆顶；若 PAGE_FREE 链表有空闲槽，则优先复用已删除记录的空间。

**顺序插入优化字段**

`PAGE_LAST_INSERT`、`PAGE_DIRECTION`、`PAGE_N_DIRECTION` 三者配合，用于检测自增主键等顺序插入模式。当连续多次向同一方向（通常是 RIGHT）插入时，InnoDB 可跳过 Page Directory 的二分查找，直接追加到链表尾部，降低 CPU 开销。这是 InnoDB 对最常见插入模式的重要优化。

### 3. Infimum 和 Supremum 记录

Infimum 和 Supremum 是 InnoDB 自动维护的两条虚拟记录，不是用户数据，各占用 13 字节（记录头 + 固定内容），合计 26 字节。

**语义**

- **Infimum**：逻辑上"比页内任何用户记录都小"的边界记录。内容固定，相当于一个极小哨兵。
- **Supremum**：逻辑上"比页内任何用户记录都大"的边界记录。内容固定，相当于一个极大哨兵。

**在链表中的位置**

页内用户记录逻辑上严格位于 Infimum 与 Supremum 之间。物理链表结构为：

```
Infimum  -->  User Rec 1  -->  User Rec 2  -->  ...  -->  User Rec N  -->  Supremum
```

Infimum 的 `next_record` 指向键值最小的用户记录（或 Supremum，若页为空）。Supremum 的 `next_record` 通常为 0 或特殊值，表示链表结束。

**为何需要虚拟边界**

Infimum/Supremum 使页内查找始终在 `[Infimum, Supremum]` 闭区间内进行，且 Page Directory 首尾两槽固定指向二者（Slot 0 → Infimum，末槽 → Supremum），简化边界插入与空页处理。

### 4. User Records 区

User Records 区存放实际的用户记录（聚簇索引页中为完整行，二级索引页中为键值 + 主键）。记录在页内通过 `next_record`指针形成单向链表，但物理地址不必按主键顺序连续排列。

**记录的插入过程**

向叶子页插入 `(id=50, name='Bob')` 时：Page Directory 二分定位逻辑序 → 从 PAGE_FREE 或堆顶分配空间 → 按 DYNAMIC 格式编码（变长列表、NULL 位图、记录头、列数据及隐藏列）→ 更新 `next_record` 链表与槽位 → 更新 `PAGE_N_RECS`等；空间不足则页分裂。物理链表顺序 ≠ 主键逻辑顺序，有序性由 Page Directory 保证。

### 5. Free Space

Free Space 是 Page Header 下方 User Records 堆顶（`PAGE_HEAP_TOP`）与 Page Directory 之间的未使用区域。随着记录插入，堆顶上移，Free Space 缩小；记录删除后，空间进入 PAGE_FREE 链表，Free Space 不一定立即增大，直到页重组。

**未使用空间的管理**

InnoDB 采用两种空闲空间来源：

1. **PAGE_FREE 链表**：删除记录释放的、大小不等的块，通过链表串联，插入时优先匹配合适大小的块复用。
2. **堆顶 Free Space**：`PAGE_HEAP_TOP` 到 Page Directory 之间的连续空白，新记录在无合适 FREE 块时使用。

当 `PAGE_GARBAGE`（已删除记录占用字节）超过页大小约一半时，InnoDB 可能触发 **page reorganization**：compact 剩余记录，清除 PAGE_FREE 链表，回收碎片空间。重组期间页会被 latched，可能影响并发，但可恢复页内空间利用率。

**页满判断**

插入前 InnoDB 估算新记录所需空间（含变长列表、NULL 位图、记录头等）。若当前 Free Space + 可复用 FREE 块不足以容纳，则触发 **页分裂**，而非在页内强行写入。

### 6. Page Directory

Page Directory 位于页尾 File Trailer 之前，是一个稀疏的槽（Slot）数组。每个槽占 2 字节，存储指向页内某条记录的偏移量。

**槽的组织方式**

- 槽按所指向记录在**主键逻辑序**中的顺序排列（从小到大），即槽号递增对应键值递增。
- 每个槽指向其所属组的**最后一条**（键值最大）记录。
- InnoDB 规定：除 Infimum/Supremum 外，每组包含 4–8 条记录；组大小随页内记录总数动态调整。
- **首尾两个槽位固定**：Slot 0 指向 Infimum（最小键边界），最后一个槽指向 Supremum（最大键边界）。

> **关于槽的物理存储**：槽数组从页尾 File Trailer 之前向低地址方向增长，因此"物理存储顺序"上 Supremum 的槽紧邻 File Trailer，Infimum 的槽在最远处。本文以**逻辑键序**对槽编号（Slot 0 = 最小键 = Infimum），与 InnoDB 源码中的数组索引一致。

**示例**

假设页内有 19 条用户记录，可能划分为 4 组（不含 Infimum/Supremum），Page Directory 结构示意：

```
Slot 0  -->  Infimum          (最小边界)
Slot 1  -->  Rec 5            (第 1 组最后一条)
Slot 2  -->  Rec 11           (第 2 组最后一条)
Slot 3  -->  Rec 16           (第 3 组最后一条)
Slot 4  -->  Rec 19           (第 4 组最后一条)
Slot 5  -->  Supremum         (最大边界)
```

查找 `id=12` 时，对 Slot 1–4（用户记录对应的槽）二分，确定落在 Slot 2（Rec 11）与 Slot 3（Rec 16）之间，即从 Rec 11 的下一条起顺序扫描至 Rec 16（含），最多 4–8 条记录即可定位。

**槽数量**

`PAGE_N_DIR_SLOTS` 记录槽总数。槽数量远小于记录数（约每 4–8 条记录 1 槽），这是"稀疏目录"的设计——在空间开销与查找效率之间折中。

### 7. File Trailer（8 字节）

File Trailer 占用页的最后 8 字节，用于校验页的完整性。

**结构与一致性校验**

| 字段                       | 大小 | 内容          |
|--------------------------|----|-------------|
| FIL_PAGE_SPACE_OR_CHKSUM | 4  | 校验和（低 4 字节） |
| FIL_PAGE_LSN             | 4  | LSN 的后 4 字节 |

InnoDB 将页刷盘前，在 File Trailer 写入与 File Header 对应的校验和与 LSN 片段。读页时对比 Header 与 Trailer：

- 若不一致，说明发生了 **partial page write**（如操作系统只写了半页、断电等），该页不可信。
- Doublewrite 机制正是为此设计：先写 Doublewrite Buffer，再写数据文件，崩溃时从 Doublewrite 恢复完整页。

File Trailer 与 File Header 的"首尾呼应"，是 InnoDB 在页级别保证数据完整性的最后一道防线。Redo Log 保证逻辑可恢复，Doublewrite + 校验和保证页物理完整。

## 五、行格式详解

行格式（Row Format）定义 InnoDB 如何在数据页内编码一行记录：变长字段长度如何存储、NULL 如何标记、溢出列如何处理、记录头包含哪些信息。行格式在`CREATE TABLE` 时通过 `ROW_FORMAT` 指定，MySQL 8.0 默认 `DYNAMIC`。

### 1. COMPACT 格式

COMPACT 是 MySQL 5.0 引入的行格式，相比 REDUNDANT 显著减少存储开销，是 MySQL 5.x 时代的事实标准。

**变长字段长度列表**

对于 VARCHAR、VARBINARY、BLOB、TEXT 等变长列，InnoDB 在记录开头存储"变长字段长度列表"，按**列定义顺序的逆序**排列每个变长列的实际字节数：

- 长度 ≤ 255：1 字节。
- 长度 > 255：2 字节（第一字节标记 0xFF 表示扩展）。

逆序存储的原因是与记录头相邻，解析时从已知边界向前推算，实现上更紧凑。若列值为 NULL，则不在变长列表中占长度项，改由 NULL 位图标记。

**NULL 值标志位**

可空列对应 NULL 位图中的一个 bit，按列定义顺序排列，占用 `ceil(可空列数 / 8)` 字节。bit 为 1 表示 NULL，该列不占用数据区空间；bit 为 0 表示非 NULL，数据在变长列表之后按序存放。

**记录头信息（5 字节）**

COMPACT 记录头固定 5 字节，位域布局如下（逻辑字段，非逐字节）：

| 字段           | 位数 | 含义                                     |
|--------------|----|----------------------------------------|
| 未使用          | 1  | 保留                                     |
| delete_flag  | 1  | 删除标记，1 表示已逻辑删除                         |
| min_rec_flag | 1  | 1 表示该记录是所在 B+ 树层的最小记录                  |
| n_owned      | 4  | 该记录拥有的记录数（Page Directory 分组用）          |
| heap_no      | 13 | 堆号，页内记录编号，Infimum=0，Supremum=1，用户从 2 起 |
| record_type  | 3  | 0=普通，1=非叶子节点，2=Infimum，3=Supremum      |
| next_record  | 16 | 下一条记录的相对偏移（有符号）                        |

**next_record 的含义**

`next_record` 存储相对于当前记录**起始位置**的字节偏移（有符号 16 位）。通过该指针，页内记录形成单向链表。注意：偏移是相对于记录头所在位置计算的，解析时需按 InnoDB 源码约定处理符号和基准点。

**n_owned 与 Page Directory**

当某条用户记录是其分组的"最后一条"时，其 `n_owned` 字段记录该组包含的记录数（含自身）。Page Directory 槽位指向的记录通常`n_owned` 为 4–8。Infimum 和 Supremum 的 `n_owned` 特殊，用于管理边界组。

**真实数据部分：隐藏列**

InnoDB 在每条用户记录中自动添加隐藏列（对应用透明）：

| 列名          | 大小   | 何时存在                               | 作用                  |
|-------------|------|------------------------------------|---------------------|
| DB_ROW_ID   | 6 字节 | 无 PRIMARY KEY 且无 NOT NULL UNIQUE 时 | 聚簇索引键               |
| DB_TRX_ID   | 6 字节 | 始终                                 | 最后修改该行的事务 ID        |
| DB_ROLL_PTR | 7 字节 | 始终                                 | 指向 Undo Log 中该行的旧版本 |

`DB_ROLL_PTR` 由 Undo 表空间编号、Undo Segment 编号、Undo 页号等组成，是 MVCC 读视图链表的入口。即使 `SELECT`不显式查询这些列，它们也占用每行的物理空间（聚簇索引叶中）。

**COMPACT 记录布局示意**

```
+---------------------------+
| 变长字段长度列表（逆序）    |
+---------------------------+
| NULL 位图                  |
+---------------------------+
| 记录头（5 字节）            |
+---------------------------+
| 列1 | 列2 | ... | 隐藏列    |
+---------------------------+
```

### 2. DYNAMIC 格式

DYNAMIC 行格式最早在 InnoDB Plugin for MySQL 5.1 中作为 Barracuda 文件格式的一部分引入，MySQL 5.5 合入主线（需 `innodb_file_format=Barracuda`），MySQL 5.7 起设为默认行格式，MySQL 8.0 移除文件格式概念后 DYNAMIC 成为唯一推荐的现代行格式。记录在页内的编码方式（变长列表、NULL 位图、记录头）与 COMPACT **完全相同**，核心差异在于**溢出列的处理策略**。

**与 COMPACT 的区别：溢出处理方式**

| 维度      | COMPACT            | DYNAMIC      |
|---------|--------------------|--------------|
| 页内保留    | 变长列前 768 字节        | 仅 20 字节溢出页指针 |
| 溢出页     | 存放完整列数据（含前缀副本）   | 存放完整列数据        |
| 前缀索引    | 768 字节前缀可在聚簇索引页内匹配 | 前缀索引需访问溢出页   |
| 页内空间利用率 | 大字段占用多，单行占页大       | 高，单页可容纳更多行   |

DYNAMIC 将大字段几乎完全"赶出"数据页，仅留 20 字节指针（指向溢出页链表）。这样 VARCHAR(5000) 的列在页内只占 20 字节，而非 768+ 字节，显著减少页内浪费。

**为什么 MySQL 8.0 默认使用 DYNAMIC**

大字段普遍化使 COMPACT 768 字节前缀严重降低页内行密度；多数查询不读 TEXT 全量，完全溢出更合理；SSD 降低了读溢出页的随机 IO 代价。新建表应直接使用 DYNAMIC。

### 3. COMPRESSED 格式

COMPRESSED 在 DYNAMIC/COMPACT 记录编码基础上，对**整个页**进行压缩存储，适用于读多写少、数据可压缩性高的场景。

**压缩算法与适用场景**

- 支持 zlib、lz4、zstd（版本依赖），通过 `ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8` 等指定块大小（1/2/4/8/16 KB）。
- 页在 Buffer Pool 中以压缩形式缓存，读取时解压，写入时压缩，CPU 开销明显增加。
- 适合：归档表、日志历史表、文本重复度高的列。不适合：高频 OLTP 更新表。

**实现要点**

- 压缩单位通常小于 16 KB（如 8 KB 压缩块），一个 16 KB 逻辑页可能对应多个压缩块。
- 溢出页也可被压缩。
- 需监控 `Innodb_compression_padding_pct_max` 等参数，避免压缩失败导致插入变慢。

### 4. REDUNDANT 格式

REDUNDANT 是 InnoDB 最早的行格式，MySQL 5.0 之前默认。特点：

- 变长列长度用固定 1 或 2 字节前缀存储在每列前，而非逆序变长列表。
- 记录头 6 字节（比 COMPACT 多 1 字节）。
- 溢出策略类似 COMPACT（768 字节前缀）。

REDUNDANT 现已极少使用，仅在维护极老系统或迁移遗留库时可能遇到。了解即可，新建表不应使用。

## 六、行溢出与大字段存储

当一行数据的总大小超过数据页可用空间时，InnoDB 将变长列的部分或全部数据存放到独立的溢出页（Overflow Page），这种现象称为行溢出（Row Overflow）。

### 1. 什么时候发生行溢出

InnoDB 判断溢出时，考察的是**整行插入页内后是否放得下**，而非单列是否超过某一阈值。

**可用空间估算**

16 KB 页中，扣除 File Header、Page Header、Infimum/Supremum、Page Directory、File Trailer 等固定开销，用户记录可用空间约 **8126 字节**（COMPACT/DYNAMIC，非压缩）。若一行编码后（含变长列表、NULL 位图、记录头、所有列、隐藏列）超过此值，必须溢出。

**溢出列选择**

InnoDB 选择**最长的变长列**作为溢出候选。若多个列均较大，可能多列溢出。插入时从最长列开始，直到行能放入页内或所有候选列均已溢出。

**示例**

```sql
CREATE TABLE docs (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    body TEXT
) ROW_FORMAT=DYNAMIC;
```

`body` 为 50 KB 时，整行显然无法放入单页，`body` 被完全存到溢出页链，页内仅留 20 字节指针。`title` 若仅 50 字节，仍完整留在数据页内。

### 2. COMPACT 的 768 字节前缀策略

COMPACT（及 REDUNDANT）对溢出列在页内保留 **768 字节前缀**，同时将该列的**完整数据**（含前 768 字节的副本）存入溢出页。即溢出页中存放的是列数据的完整拷贝，而非仅存放"前缀之后的部分"。

**设计意图**

768 字节与 MySQL 索引前缀默认上限相关。若建前缀索引 `INDEX idx(body(768))`，前 768 字节在聚簇索引页内，**索引查找可在不读溢出页的情况下完成前缀匹配**（仍需读溢出页获取完整 body 做回表，但索引扫描阶段可过滤）。

**代价**

即使从不读 `body` 列，每行仍在数据页占 768 字节，导致每页行数极少。一张 100 万行的表，若每行 TEXT 前缀占 768 字节，数据页数量远大于 DYNAMIC 方案。

### 3. DYNAMIC 的 20 字节指针策略

DYNAMIC 对溢出列在页内仅保留 **20 字节指针**，指向溢出页链的第一页。20 字节包含表空间 ID、页号、偏移等定位信息。

**优势**

- 数据页内几乎不浪费空间于大字段，单行占用极小，B+ 树更"瘦"，Buffer Pool 可缓存更多行。
- 不访问大字段的查询（如 `SELECT id, title`）不会触发溢出页 IO。

**代价**

- 任何需要读大字段的操作都必须跟随指针读溢出页，可能多次随机 IO。
- 前缀索引在 DYNAMIC 下无法仅依赖页内数据，必须读溢出页。

### 4. VARCHAR(65535) 的实际限制

MySQL 语法允许 `VARCHAR(65535)`，但 InnoDB 下有严格物理限制：

受页大小与字符编码约束，utf8mb4 下 VARCHAR 最大约 16383 字符；且所有列长度之和（含隐藏列）不能超过行容量上限，极易触发溢出。超大文本应使用 TEXT / LONGTEXT 配合 DYNAMIC，并避免 `SELECT *` 触发不必要的溢出页 IO。

### 5. BLOB/TEXT 的存储方式

BLOB 和 TEXT 类型（TINYTEXT、TEXT、MEDIUMTEXT、LONGTEXT 等）在 InnoDB 中均按变长列处理，溢出机制与长 VARCHAR 相同。

**存储流程**

1. 数据较短：直接编码在聚簇索引叶子页的记录内。
2. 数据较长：DYNAMIC 下页内 20 字节指针 → 溢出页链表；COMPACT 下 768 前缀 + 溢出页。
3. 溢出页类型为 `FIL_PAGE_BLOB`，可跨多页链表存储。每页存部分数据 + 下一页指针。
4. COMPRESSED 格式下溢出页内容也可能被压缩。

**IO 路径示例**

```sql
SELECT body FROM docs WHERE id = 100;
```

1. B+ 树定位 `id=100` 的叶子页（1 次随机 IO，或 Buffer Pool 命中）。
2. 读取行，解析出 `body` 的 20 字节溢出指针。
3. 读第一个溢出页（1 次随机 IO）；若 body 跨页，沿链表继续读。

一次点查大字段可能触发 2 次以上随机 IO，这是 ORM `SELECT *` 在大表上的典型性能陷阱。

## 七、数据页内的记录组织

### 1. 单向链表

**next_record 指针的含义**

页内每条记录（含 Infimum、Supremum）的记录头中包含 `next_record` 字段，存储到下一条记录的相对偏移。所有用户记录与虚拟记录通过该指针串联成单向链表，遍历起点为 Infimum，终点为 Supremum。

**按主键顺序排列**

需要区分两个概念：

- **物理链表顺序**：由 `next_record` 连接，反映插入、删除、复用空间后的遍历顺序，通常与主键序不一致。
- **逻辑主键顺序**：由 Page Directory 槽位保证，二分查找 + 组内扫描按主键有序定位。

InnoDB 保证的是**逻辑主键有序**，而非物理存储有序。这一设计避免每次插入在中间位置时移动大量后续记录，代价是查找时必须借助 Page Directory，不能简单沿物理顺序扫描全页。

**与 B+ 树叶子页链表的区别**

| 链表     | 连接对象       | 指针位置                            | 作用          |
|--------|------------|---------------------------------|-------------|
| 页内记录链表 | 同页内记录      | next_record（记录头）                | 页内遍历、组内顺序扫描 |
| 叶子页链表  | 同层 B+ 树叶子页 | FIL_PAGE_PREV/NEXT（File Header） | 范围扫描跨页顺序访问  |

二者在不同粒度协同：页内链表服务单页查找，叶子页链表服务 `BETWEEN`、全表顺序扫描等跨页范围操作。

### 2. 记录的插入与删除

**插入时如何找到正确位置**

与第四节 User Records 插入过程相同：Page Directory 二分 → 组内 `next_record` 顺序扫描 → 插入并更新槽位。自增主键可借`PAGE_LAST_INSERT` 跳过二分直接追加。

**删除时的标记删除与空间回收**

设置 `delete_flag=1`，记录加入 PAGE_FREE 链表，`PAGE_GARBAGE` 增加；Purge 确认 MVCC 不可见后才真正回收。新插入优先复用 FREE 块，垃圾超过约半页时可能触发 page reorganization。

### 3. 页分裂与页合并

**页满时的分裂过程**

当插入导致页内空间不足，InnoDB 执行页分裂：

1. 申请新页（通常从同一索引段）。
2. 将原页约一半记录（按主键序）移动到新页。
3. 更新两页的 Page Directory、User Records 链表、FIL_PAGE_PREV/NEXT。
4. 在 B+ 树父节点插入新键和新页指针；若父节点也满，递归分裂。
5. 若分裂的是根节点且根仅两页，树高可能增加。

分裂是写入热点、索引碎片和 Redo Log 放大的重要来源。自增主键通常向页尾插入，分裂频率低于随机主键。

**删除过多时的合并条件**

InnoDB 的页合并（Page Merge）相对保守。当页内记录过少（如低于填充因子约 50%）且**相邻页**也稀疏时，可能将两页合并为一页，释放另一页回段。合并触发条件比分裂严格得多，生产环境中合并远少于分裂，因此 DELETE 大量数据后`.ibd` 文件往往不会明显缩小——空间留在段内供复用，或需 `OPTIMIZE TABLE` 重建表。

## 八、页目录与页内查找

### 1. 槽的分组规则

**每个槽管理 4–8 条记录**

InnoDB 将页内用户记录（逻辑主键序）划分为若干组，每组 4–8 条记录，组大小随 `PAGE_N_RECS`动态调整，目标是在槽数量与组内扫描代价之间平衡。每组最后一条记录的偏移写入 Page Directory 作为一个槽。

**Infimum 和 Supremum 的特殊分组**

- Infimum 单独作为一组，`n_owned` 可能为 1 或更多（管理最小键一侧的记录），对应 Page Directory 的 Slot 0。
- Supremum 同样特殊，是逻辑最大边界，对应 Page Directory 的最后一个槽。
- 首尾槽固定指向 Infimum 和 Supremum，保证二分查找的边界始终明确。

**n_owned 的维护**

当组内插入或删除导致组大小变化时，原"组尾记录"的 `n_owned` 需更新；若组尾变化，对应槽位指向新组尾记录。这是页内元数据维护的重要部分，分裂和重组时全部槽位会重建。

### 2. 二分查找 + 顺序扫描

**先通过槽做二分，再在组内顺序扫描**

查找键值 `K` 的算法：

1. 设槽数组为 `S[0..n-1]`，对应记录键值为 `K(S[i])`（槽指向记录的键）。
2. 二分查找：找到最大的 `i` 使得 `K(S[i]) < K`，或确定 `K` 落在 `(S[i], S[i+1])` 区间。
3. 从 `S[i]` 指向记录的下一条起，沿 `next_record` 顺序扫描，最多 4–8 条。
4. 找到 `K` 精确匹配，或第一个键 > `K` 的位置（插入/唯一约束失败）。

**时间复杂度分析**

设页内用户记录数为 `n`，槽数约为 `n/6`（每组约 6 条）。

- 二分查找：`O(log(n/6)) = O(log n)`。
- 组内顺序扫描：最多 8 次比较，**O(1)** 常数级。

总复杂度 `O(log n)`，其中 `n` 单页内通常不超过几百（取决于行宽）。对比纯链表 `O(n)` 和全记录槽位 `O(log n)` 但槽空间 `O(n)`，稀疏 Page Directory 是更优折中。

**与 B+ 树查找的衔接**

完整索引点查路径：

1. 从根节点起，每层页内执行上述"槽二分 + 组内扫描"，定位下一层子页号。
2. 重复至叶子层。
3. 叶子页内同样算法定位目标行或确定不存在。
4. 范围查询时，在叶子页内找到起点后，可沿页内链表和 `FIL_PAGE_NEXT` 顺序读取，直至键超出范围。

Page Directory 是 B+ 树每一层、每一页内部的查找加速结构；B+ 树保证页间有序，Page Directory 保证页内有序，二者共同实现全索引的有序访问能力。

## 总结

InnoDB 存储结构是从文件到记录的多层映射：**逻辑表**对应表空间内多棵 B+ 树，而非顺序行；**表空间**分系统、独立、通用、临时、Undo 等类型；**段 → 区（64 页/1 MB）→ 页（16 KB）** 面向 B+ 树局部性组织空间。

**16 KB 数据页**由 File Header（定位/LSN/叶子链表）、Page Header（堆与插入优化）、Infimum/Supremum、User Records 链表、Free Space、Page Directory 稀疏槽、File Trailer 校验组成。**行格式**上 COMPACT 定义变长列表、NULL 位图、5 字节记录头与隐藏列；DYNAMIC（8.0 默认）溢出列仅留 20 字节指针；大字段溢出至 BLOB 页链。

**页内查找**靠 Page Directory 每组 4–8 条记录的槽二分 + 常数级顺序扫描，复杂度 O(log n)。物理链表 ≠ 逻辑主键序；页满分裂、删除延迟回收。理解这些物理细节，是阅读 B+ 树索引、Buffer Pool、Redo 与 MVCC 的共同地基。
