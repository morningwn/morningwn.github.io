---
title: MySQL 整体架构：连接管理、SQL 解析、优化器与存储引擎的分层协作
summary: 从连接管理到存储引擎，系统梳理 MySQL 的分层架构，理解一条 SQL 从接收到返回结果的完整生命周期。
created: 2026-07-02
updated: 2026-07-02
tags: MySQL, InnoDB, 架构
---

# MySQL 整体架构：连接管理、SQL 解析、优化器与存储引擎的分层协作

讨论 MySQL 的执行流程，不能只盯着 SQL 语法本身。真正决定一次查询或修改如何落地的，是连接层、Server 层、优化器、事务系统、日志系统以及 InnoDB 锁管理器之间的配合。

从客户端发出一条 SQL，到结果集返回客户端，中间至少经过连接管理、协议解析、语义检查、执行计划生成、存储引擎访问、日志持久化等多个阶段。理解这种分层，是为了建立稳定的分析框架：当遇到慢查询、锁等待、死锁、主从延迟或崩溃恢复问题时，能够迅速判断问题落在哪一层、该从哪个方向排查。

## 一、为什么要先理解架构分层

### 1. 分层让职责边界清晰

MySQL 采用经典的三层结构：

```
┌─────────────────────────────────────────────────────────────┐
│                        客户端应用层                           │
│          (JDBC / Python mysql-connector / Go driver)         │
└────────────────────────────┬────────────────────────────────┘
                             │ MySQL 协议 (TCP / SSL)
┌────────────────────────────▼────────────────────────────────┐
│                      连接层 (Connectors)                     │
│   连接接入 · 认证 · 线程调度 · 协议编解码 · 会话上下文维护      │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                    Server 层 (MySQL Server)                  │
│  解析器 → 预处理器 → 优化器 → 执行器                          │
│  权限 · MDL · Binlog · 慢查询日志 · 复制协调 · 元数据管理       │
└────────────────────────────┬────────────────────────────────┘
                             │ Handler API (存储引擎接口)
┌────────────────────────────▼────────────────────────────────┐
│                   存储引擎层 (Storage Engines)                │
│  InnoDB · MyISAM · Memory · Archive · ...                   │
│  数据页 · 索引 · 事务 · 行锁 · Redo/Undo · 崩溃恢复            │
└────────────────────────────┬────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   文件系统 / 磁盘  │
                    └─────────────────┘
```

连接层负责"谁可以连进来、以什么身份连进来"。Server 层负责"这条 SQL 是什么意思、怎么执行最划算、如何协调多个表和多个引擎"。存储引擎层负责"数据在磁盘上怎么存、怎么读、怎么保证事务和并发安全"。

每一层只关心自己该做的事。Server 层不需要知道 B+ 树页的具体布局；InnoDB 也不需要理解 `JOIN` 的重排规则。Server 层与存储引擎之间通过`handler` 接口交互，接口稳定意味着引擎可以独立演进——InnoDB 可以在内部引入 Buffer Pool、Change Buffer、Adaptive Hash Index 等复杂机制，而不必让 Server 层感知这些细节。

### 2. 分层是排查问题的地图

有了分层视角，可以把症状映射到层次，从而缩小排查范围：

| 症状           | 优先排查层次            | 典型工具 / 视图                                  |
|--------------|-------------------|--------------------------------------------|
| 连接数打满、握手超时   | 连接层               | `SHOW PROCESSLIST`、`max_connections`       |
| 语法报错、权限拒绝    | Server 层（解析 / 权限） | 错误码、`SHOW GRANTS`                          |
| 执行计划不合理、全表扫描 | Server 层（优化器）     | `EXPLAIN`、`optimizer_trace`                |
| 行锁等待、死锁      | 存储引擎层             | `performance_schema.data_locks`、`SHOW ENGINE INNODB STATUS` |
| DDL 阻塞 DML   | Server 层（MDL）     | `performance_schema.metadata_locks`        |
| 主从数据不一致      | Server 层 + 引擎层    | Binlog、Redo Log、复制线程状态                     |
| 崩溃后数据丢失      | 存储引擎层             | Redo Log、`innodb_force_recovery`           |

例如，`Lock wait timeout exceeded` 是 InnoDB 行锁问题；`Too many connections` 则应检查连接层配置和客户端连接池，而非去调 Buffer Pool。

### 3. 分层解释了 MySQL 的设计取舍

MySQL 的许多设计选择，只有放在分层语境下才容易理解：

- **查询缓存 vs Buffer Pool。** 查询缓存在 Server 层，以 SQL 文本为键；Buffer Pool 在 InnoDB 层，以数据页为单位。前者因全局锁争用已在 MySQL 8.0 移除；后者仍是核心组件。
- **优化器在 Server 层，行锁在 InnoDB。** 执行计划需要跨表统一视图；锁语义与存储结构紧密相关，由引擎自行管理。
- **DDL 会阻塞 DML。** MDL 由 Server 层维护，与 InnoDB 内部的页 latch 是不同层次、不同粒度的锁。
- **同一条 SQL 在不同引擎上行为可能不同。** `SELECT ... FOR UPDATE` 在 InnoDB 上加行锁，在 MyISAM 上加表锁。

## 二、连接层

连接层是客户端与 MySQL Server 之间的第一道关口。它不负责 SQL 语义，但决定了"谁能连进来、连接如何被调度、会话上下文如何维护"。

### 1. 客户端连接协议

MySQL 客户端与 Server 基于 MySQL 协议通信，运行在 TCP 之上（默认端口 3306，也可通过 Unix Socket 本地连接）。完整连接建立经历：TCP 三次握手 → MySQL 握手协议 → 认证。

**MySQL 握手协议（TCP 建立之后）：**

1. **Server → Client：初始握手包。** 包含协议版本、Server 版本字符串、连接 ID、认证 scramble 数据、能力标志位（`CLIENT_SSL`、`CLIENT_COMPRESS`、`CLIENT_PLUGIN_AUTH` 等）、字符集、状态标志、认证插件名称（MySQL 8.0 默认 `caching_sha2_password`）。
2. **Client → Server：握手响应包。** 包含 Client 能力标志、字符集、用户名、认证响应（密码哈希）、可选默认数据库、可选认证插件名。
3. **Server → Client：认证结果。** 成功返回 OK 包；失败返回 ERR 包并断开。`caching_sha2_password` 在缓存未命中时可能触发额外的 AuthMoreData 交换（RSA 公钥加密或 fast auth 路径）。

**认证插件：**

`mysql_native_password`（旧版默认，8.0 已 deprecated）：

```
response = SHA1(password) XOR SHA1(scramble + SHA1(SHA1(password)))
```

Server 端存储 `SHA1(SHA1(password))`；认证时 Server 发送 20 字节随机 scramble，Client 按上述公式计算 response 返回。

`caching_sha2_password`（MySQL 8.0 默认）：

```
response = SHA256(password) XOR SHA256(scramble + SHA256(SHA256(password)))
```

Server 端存储 `SHA256(SHA256(password))`；认证时 Server 发送随机 scramble，Client 按上述公式计算 response。其中 `+` 表示字节拼接。

Server 维护认证缓存：缓存命中走 fast auth（仅一轮 scramble/response 交换）；缓存未命中时，若连接已启用 SSL 则在加密通道中传输密码完成认证，若未启用 SSL 则通过 RSA 公钥加密传输密码（full auth）。这也是 8.0 首次连接偶发 "Authentication requires secure connection" 提示的原因。

**SSL/TLS 加密连接：**

```ini
[mysqld]
ssl-ca=ca.pem
ssl-cert=server-cert.pem
ssl-key=server-key.pem
require_secure_transport=ON
```

Client 在 Handshake Response 中设置 `CLIENT_SSL` 标志时，TLS 握手在 MySQL 协议之前或之中完成。SSL 保护传输通道（SQL 文本、结果集、密码交换），并不替代 Server 端权限验证。对 `caching_sha2_password`，SSL 连接可走 fast auth，避免 RSA 额外往返，生产环境推荐启用。

MySQL 协议在连接建立后的数据交互以 **Command Packet** 为单位。常见命令类型包括：`COM_QUERY`（文本 SQL）、`COM_STMT_PREPARE`/`COM_STMT_EXECUTE`（Prepared Statement）、`COM_PING`（心跳）、`COM_QUIT`（断开）。每个 Command Packet 由 1 字节命令码 + 可选 payload 组成，Server 处理完毕后返回 OK/ERR/Result Set 等响应包。理解这一协议分层，有助于分析连接假死（Client 等响应、Server 等锁）和网络层面的超时问题。

### 2. 线程模型：一连接一线程

MySQL 社区版（非 Thread Pool 模式）采用**一连接一线程**模型：每个客户端连接对应一个服务线程，负责接收请求、执行 SQL、返回结果，直到连接断开。

**为什么选线程而非进程：** 线程创建/切换开销更小；Buffer Pool、数据字典等全局资源位于进程地址空间内，线程可直接访问；每个连接独立线程、顺序执行 SQL，编程模型简单。连接数在数百以内时表现良好。

**线程创建与复用：** 新连接到达时，MySQL 优先从**线程缓存**（Thread Cache）取空闲线程，而非每次 `pthread_create`。缓存大小由`thread_cache_size` 控制（默认 `-1` 自动，约 8 + `max_connections`/100）。流程：accept → 握手 → 从缓存取线程或新建 → 进入`thread_sql` 循环处理 Command Packet（如 `COM_QUERY` → `dispatch_command` → `mysql_parse`）→ 连接断开时线程回缓存或销毁。

```sql
SHOW STATUS LIKE 'Threads_created';  -- 累计创建线程数
SHOW STATUS LIKE 'Threads_cached'; -- 当前缓存线程数
```

若 `Threads_created` 持续接近 `Connections`，说明缓存命中率低。但增大 `thread_cache_size` 无法突破根本限制——每个活跃连接仍占一个线程及栈空间（`thread_stack`，默认 256 KB ~ 1 MB）。连接数达数千时，上下文切换、内存占用、调度器性能均成为瓶颈。

### 3. 连接池与线程池

**客户端连接池：** 生产环境几乎总是用连接池复用连接（HikariCP、Druid 等），而非每次请求新建连接。100 个应用线程共享 20 个 MySQL 连接，Server 端只需 20 个服务线程。关键参数：`maximumPoolSize`（不超过 Server `max_connections` 合理比例）、`minimumIdle`（预热）、`connectionTimeout`、`idleTimeout`、`maxLifetime`（定期轮换）。

**MySQL Enterprise Thread Pool：** 用固定数量工作线程服务大量连接，SQL 任务入队调度。连接上的 SQL 因 I/O 阻塞时，工作线程切换到其他任务，提高 CPU 利用率。注意 Thread Pool 为 MySQL Enterprise Edition 商业功能；社区版默认仍是一连接一线程，如需类似能力可考虑 ProxySQL 连接池复用或 MariaDB/Percona 分支的 Thread Pool。

**max_connections 调优：**

```sql
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
```

原则：不是越大越好；`Max_used_connections / max_connections > 0.8` 时应先查连接泄漏或连接池配置；内存估算约`max_connections × 1 MB`（含线程栈与连接缓冲区）；多实例部署时，`max_connections` 应大于各实例连接池上限之和并留余量。

### 4. 会话上下文

每个连接对应一个**会话**（Session），持有独立上下文，贯穿连接生命周期。

**会话变量 vs 全局变量：**

```sql
SET GLOBAL max_connections = 500;           -- 影响新连接
SET SESSION sql_mode = 'STRICT_TRANS_TABLES'; -- 仅当前连接
```

会话建立时从全局默认值初始化。常见会话级变量：`transaction_isolation`、`autocommit`、`sql_mode`、`character_set_client/results`、`optimizer_switch`、`max_execution_time`。

**字符集与排序规则协商：** Client 在 Handshake Response 中声明字符集，Server 设置`character_set_client/connection/results`。五层（Client、Connection、Server、Database、Table）统一使用 `utf8mb4` +`utf8mb4_0900_ai_ci`，避免隐式转换导致索引失效或乱码。

**事务状态维护：** THD（Thread Handler）结构维护会话状态——是否在事务中、隔离级别、持有的 MDL 锁、临时表、Prepared Statement 缓存、用户变量（`@var`）。连接断开时，未提交事务自动回滚（InnoDB），并释放所有会话资源。

THD 还承载当前语句的诊断上下文：`current_db`（当前数据库）、`query_string`（正在执行的 SQL）、`start_time`（语句开始时间，慢查询日志依赖此字段）、`lex`（当前语句的 AST）。`SHOW PROCESSLIST` 和 Performance Schema 的 `threads` 表展示的 Id、User、Host、db、Command、Time、State 等信息，均来自 THD 及其关联结构。排查"哪个连接持锁不放"或"哪条 SQL 执行了多久"，本质上是读取 THD 状态。

## 三、Server 层

Server 层是 MySQL 的"大脑"：理解 SQL 含义，决定如何执行，协调存储引擎，处理复制、权限、元数据等横切逻辑。

### 1. 解析器（Parser）

解析器将 SQL 文本转换为内部数据结构，分词法分析和语法分析两阶段。

**词法分析：SQL 文本 → Token**

词法分析器（Lexer）扫描 SQL，识别 Token（最小有意义单元）：

```sql
SELECT name, age FROM users WHERE id = 42;
```

```
SELECT → KEYWORD    name → IDENTIFIER    , → COMMA
age → IDENTIFIER    FROM → KEYWORD       users → IDENTIFIER
WHERE → KEYWORD     id → IDENTIFIER      = → OPERATOR
42 → LITERAL(INT)   ; → SEMICOLON       EOF → END
```

词法分析器需处理：关键字 vs 标识符（反引号 `` `select` ``）、字符串转义、多种数字字面量、注释跳过、用户变量 `@var`/`@@global.var`。输出 Token 流供语法分析器消费。

**语法分析：Token → 语法树（AST）**

语法分析器（Yacc/Bison）按 SQL 语法规则构建 AST：

```
                    ┌──────────────┐
                    │  SELECT_STMT │
                    └──┬───┬───┬───┘
           ┌───────────┘   │   └──────────┐
           ▼               ▼              ▼
     ┌───────────┐   ┌────────┐   ┌────────────┐
     │SELECT_LIST│   │  FROM  │   │   WHERE    │
     └─────┬─────┘   └───┬────┘   └─────┬──────┘
       ┌───┴───┐         ▼              ▼
       ▼       ▼     ┌───────┐   ┌──────────────┐
     name     age    │ users │   │  EQ(id, 42)  │
                     └───────┘   └──────────────┘
```

AST 节点涵盖 DML、DDL、DCL、TCL 全部语句类型。语法分析器还处理运算符优先级、语句类型识别（如 `SELECT ... INTO OUTFILE` vs`SELECT ... FROM`）、优化器 Hint 解析（`/*+ INDEX(users idx_name) */`）。

**语法错误的检测时机：**

```sql
SELECT * FORM users;              -- ERROR 1064：解析阶段（语法错误）
SELECT * FROM nonexistent_table;  -- ERROR 1146：预处理器阶段（语义错误）
```

语法错误在解析阶段检测，尚未访问数据字典。MySQL 8.0 Parser 改进了错误提示，但复杂嵌套 SQL 的定位有时仍不够精确。

### 2. 预处理器

预处理器（Preprocessor / Resolver）在 AST 上进行语义检查与变换。

**语义检查：** 查询数据字典（MySQL 8.0 起为 InnoDB 表空间中的 DD 表），验证表、列、索引、视图是否存在，类型是否兼容。对`SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id`，依次检查两表存在性、列存在性、JOIN 条件类型可比性、WHERE 字面量与列类型兼容性。失败返回 `1146`、`1054` 等错误码。

**权限验证：** 检查当前用户是否具备所需权限（SELECT/INSERT/UPDATE/DELETE 等，粒度到表/列）。视图有 DEFINER 和 SQL SECURITY 属性；`SQL SECURITY INVOKER` 模式下以调用者身份检查底层表权限。

**视图展开与子查询重写：**

```sql
CREATE VIEW active_users AS SELECT id, name FROM users WHERE status = 'active';
SELECT name FROM active_users WHERE id > 100;
-- 展开后等价于：
SELECT name FROM users WHERE status = 'active' AND id > 100;
```

含 `GROUP BY`、`DISTINCT`、聚合、`LIMIT` 的视图可能需**物化**（materialize）为临时表。子查询重写（如 `IN (SELECT ...)` → Semi Join）也在此阶段或优化器阶段进行。

### 3. 优化器（Optimizer）

优化器接收预处理后的 AST，生成**执行计划**——决定索引选择、JOIN 顺序与算法、子查询处理方式，以最低估算成本完成查询。

**基于成本的优化（CBO）原理：**

```
Total Cost = CPU Cost + I/O Cost
```

优化器对每种可行方案估算成本，选最低者。I/O Cost 是主因（顺序读、随机读次数 × 权重）；CPU Cost 含行评估、排序、哈希构建等。

对 `SELECT * FROM orders WHERE user_id = 42`：

| 方案 | 描述                      | 估算成本                                          |
|----|-------------------------|-----------------------------------------------|
| A  | 全表扫描（读取所有数据页）          | P × seq_page_cost（P 为数据页数）                      |
| B  | idx_user_id 索引 + 回表 M 行 | log(N) × idx_read + M × rnd_page_cost（M 为匹配行数） |

其中 `seq_page_cost` 和 `rnd_page_cost` 分别代表顺序和随机读取一个页的 I/O 成本（MySQL 默认值分别为 1.0 和 4.0）。

M 远小于 N 时选 B；M 接近 N（低选择性）时可能选 A。

**统计信息的来源与更新：**

```sql
SHOW TABLE STATUS LIKE 'orders';
SELECT * FROM mysql.innodb_table_stats WHERE table_name = 'orders';
SELECT * FROM mysql.innodb_index_stats WHERE table_name = 'orders';
ANALYZE TABLE orders;  -- 手动触发采样统计
```

关键统计量：`table_rows`（行数估算）、`avg_row_length`、`index cardinality`（索引选择性）、`histogram`（8.0+ 列值分布）。8.0 默认自动收集；`innodb_stats_persistent` 控制持久化。统计不准是"优化器选错计划"最常见原因——1000 万行的表若统计为 1000 行，优化器可能选全表扫描。

**优化器的典型决策：**

1. **索引选择：** 评估联合索引最左前缀、`index_merge` 多索引组合。
2. **JOIN 顺序：** 少表用动态规划，多表用贪心启发式；8.0 还需决定 Hash Join vs Nested Loop Join。
3. **子查询优化：** `IN` → Semi Join、`NOT IN` → Anti Join、相关子查询 → Derived Table + JOIN。
4. **排序与分组：** 能否利用索引有序性（`Using index`）避免 filesort。
5. **访问方法：** `const`、`ref`、`range`、`index`、`ALL` 对应 EXPLAIN 的 `type` 列。

**optimizer_trace 的使用：**

```sql
SET optimizer_trace = 'enabled=on';
SELECT * FROM orders WHERE user_id = 42 AND status = 'paid';
SELECT * FROM information_schema.optimizer_trace\G
SET optimizer_trace = 'off';
```

trace 输出包含各访问方法的成本计算、索引排除原因、JOIN 搜索过程、子查询重写步骤。是诊断"为什么不走索引"的利器，但不应在生产长期开启。

**成本模型示例（简化）：** MySQL 8.0 内置的 `server_cost` 和 `engine_cost` 表定义了各操作的单位成本。例如`memory_block_read_cost`（内存读）、`io_block_read_cost`（磁盘读）等。优化器将"预计读取多少个索引页/数据页"乘以对应单位成本，累加得到总成本。DBA 可通过修改这些表微调优化器行为，但生产环境极少这样做——更常见的做法是修正统计信息（`ANALYZE TABLE`）或调整索引/schema。

**JOIN 优化深入：** 对 `SELECT * FROM a JOIN b JOIN c ON ...`，优化器需决定三表连接顺序（如 a→b→c 或 b→a→c）及每步使用的算法。Nested Loop Join 适合小驱动表 + 索引良好的被驱动表；Hash Join（8.0.18+）适合等值连接且一侧无索引或数据量较大（8.0.20 起全面取代 Block Nested Loop）。MySQL 不支持 Sort-Merge Join。优化器对 N 表 JOIN 的搜索空间是 N! 量级，表数 ≤ `optimizer_search_depth`（默认 62，实际受`eq_range_index_dive_limit` 等约束）时用动态规划精确搜索，超出则用贪心算法。错误连接顺序可能导致中间结果集爆炸——这是多表 JOIN 性能问题的重要排查方向。

### 4. 执行器

执行器按执行计划调用存储引擎接口，读取或修改数据，返回结果。

**调用存储引擎接口：** 每个表对应一个 `handler` 实例。执行器通过 handler 方法与引擎交互——`ha_open` 打开表 → `index_init` +`index_read_map` 初始化索引扫描 → 循环 `ha_index_next` 逐行读取 → 评估 WHERE、投影 SELECT 列、发送结果 → `ha_close`。UPDATE/DELETE 先定位行再调 `ha_update_row`/`ha_delete_row`；INSERT 直接调 `ha_write_row`。

**迭代器模型（Volcano Model）：**

```
┌──────────────┐      ┌───────────────────┐      ┌────────────────┐      ┌────────────┐
│   Project    │      │      Filter       │      │   IndexScan    │      │   Table    │
│ (name, age)  │ ───▶ │ (status='active') │ ───▶ │  (idx_status)  │ ───▶ │  (users)   │
└──────────────┘      └───────────────────┘      └────────────────┘      └────────────┘
   Init()/Next()         Init()/Next()              Init()/Next()          Init()/Next()
```

每个节点实现 `Init()`/`Next()`/`Close()`，逐行向上传递。优点是算子独立可组合（8.0 Hash Join 即新增算子）；缺点是 tuple-at-a-time 在大结果集下不如向量化执行高效。

**结果集组装与返回：** 发送列定义包 → 逐行发送 Row Data Packet（MySQL 二进制协议编码）→ EOF/OK 包。`LIMIT 100` 时取够 100 行即停止。聚合查询在内部维护状态，返回一行汇总。

### 5. Server 层的横切关注点

**MDL（元数据锁）：** Server 层表级锁，保护表结构一致性。

| MDL 类型            | 触发场景       | 阻塞关系                        |
|-------------------|------------|-----------------------------|
| SHARED_READ       | SELECT     | 与 SHARED_WRITE、EXCLUSIVE 互斥 |
| SHARED_WRITE      | DML        | 与 EXCLUSIVE 互斥              |
| SHARED_UPGRADABLE | ALTER 准备阶段 | 与 EXCLUSIVE 互斥              |
| EXCLUSIVE         | DDL 执行     | 阻塞所有其他 MDL                  |

MDL 在打开表时获取，语句结束或事务提交时释放。长事务持有 SHARED_WRITE 会阻塞 DDL。Online DDL 分阶段升级 MDL 以减少阻塞。排查：`performance_schema.metadata_locks WHERE LOCK_STATUS='PENDING'`。

**Binlog 写入位置：** Binlog 由 Server 层维护，记录在引擎操作之后、事务提交之前：

```
ha_write_row()
      │
      ▼
┌───────────────────────────┐
│ InnoDB 写 Undo/Redo、加行锁 │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│   写 Binlog Cache (内存)    │
└─────────────┬─────────────┘
              │
              ▼
          COMMIT
              │
              ▼
┌───────────────────────────┐
│  InnoDB Prepare (Redo)    │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│   Flush Binlog (fsync)    │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  InnoDB Commit (Redo)     │
└───────────────────────────┘
```

两阶段提交（XA）保证 Binlog 与 Redo Log 一致，是复制正确性的基础。

**慢查询日志：** 语句执行完成后判定——耗时超过 `long_query_time`（默认 10s）则写入。记录的是 Server 层总耗时（含解析、优化、执行、等锁），不能区分优化器 vs InnoDB I/O，需结合 Performance Schema 或 `EXPLAIN ANALYZE`（8.0.18+）进一步分析。

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
```

## 四、存储引擎层

存储引擎层负责数据物理组织与访问。Server 层通过 Handler API 调用引擎，引擎将逻辑操作转化为磁盘读写，并提供事务、锁、崩溃恢复能力。

### 1. 插件式架构设计

引擎以动态库（`.so`/`.dll`）形式加载，通过 Handler API 与 Server 交互。

```sql
SHOW ENGINES;
```

| Engine    | Support | 定位                  |
|-----------|---------|---------------------|
| InnoDB    | DEFAULT | 事务、行锁、外键、崩溃恢复       |
| MyISAM    | YES     | 表锁、无事务，只读/读多写少      |
| MEMORY    | YES     | 内存 Hash/B+ 树，临时表/缓存 |
| ARCHIVE   | YES     | 高压缩归档，仅 INSERT      |
| BLACKHOLE | YES     | 丢弃写入，复制拓扑测试         |

注册流程：Server 启动扫描 `plugin_dir` → 插件通过 `mysql_declare_plugin` 注册 → `CREATE TABLE ... ENGINE=InnoDB` 时查找 Handler 工厂创建实例。

**引擎对比：**

| 特性   | InnoDB   | MyISAM       | Memory | Archive    |
|------|----------|--------------|--------|------------|
| 事务   | 支持       | 不支持          | 不支持    | 不支持        |
| 锁粒度  | 行锁       | 表锁           | 表锁     | 行锁(INSERT) |
| 崩溃恢复 | Redo Log | REPAIR TABLE | 数据丢失   | 无          |
| 适用场景 | OLTP 通用  | 只读报表         | 临时中间结果 | 日志归档       |

MySQL 5.5 起 InnoDB 为默认引擎；8.0 系统表全部迁入 InnoDB。生产环境几乎全部使用 InnoDB。

### 2. InnoDB 为什么是默认引擎

**事务支持（ACID）：**

- **Atomicity：** Undo Log 记录反向操作，ROLLBACK 时恢复。
- **Consistency：** 主键、外键、CHECK 约束（8.0.16+）及业务逻辑共同保证。
- **Isolation：** MVCC + 行锁，不同隔离级别下 Read View 不同。
- **Durability：** Redo Log + 刷脏，COMMIT 后 Redo 落盘即持久。

MyISAM 无事务，`UPDATE` 中途崩溃可能导致表部分更新。

**行级锁与 MVCC：**

```
Tx A: UPDATE users SET age=30 WHERE id=1;  -- 加 X 行锁
Tx B: SELECT * FROM users WHERE id=1;       -- 无锁，读 MVCC 快照
```

读写不互斥，是 OLTP 高并发基石。MyISAM 表锁使任何写阻塞所有读写。

**崩溃恢复：** 重启时扫描 Redo Log 重放已提交修改，通过 Undo Log 回滚未提交事务，自动完成。MyISAM 崩溃后 `REPAIR TABLE`可能耗时数小时且不保证完整。

**外键支持：**

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY, user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

引用完整性由引擎保证；MyISAM 只能由应用层维护。

### 3. InnoDB 的内部组件概览

**Buffer Pool：** 内存缓存数据页与索引页，读写优先在内存完成。含改良 LRU（Young/Old 分区，防止全表扫描污染热点）、Free List（空闲帧）、Flush List（脏页链表，按最早修改 LSN 排序）。Buffer Pool 与 Redo Log 分工明确：前者缓存完整数据页以加速 I/O，后者记录页的物理修改以保证持久性——二者协同实现 WAL 思想。

**Redo Log：** 物理日志，记录"In 页 X 的偏移 Y 处将 Z 改为 W"。采用 WAL 策略——日志先行，脏页延迟刷盘。Redo Log 循环写入（ib_logfile0/1 或 8.0.30+ 的 `#innodb_redo` 目录），大小由 `innodb_redo_log_capacity` 控制。Checkpoint 机制保证 Redo Log 空间可回收：当脏页刷盘进度推进到某个 LSN，该 LSN 之前的 Redo 条目可被覆盖。

**Undo Log：** 逻辑日志，记录"如何将行改回旧值"。每条 UPDATE/DELETE 在 Undo 中写入反向记录，ROLLBACK 时按 Undo 链恢复；普通 SELECT 通过 Read View + Undo 链读取历史版本（MVCC）。长事务会阻止 Purge 清理 Undo，导致 Undo Tablespace 膨胀——这是"长事务危害"的重要表现之一。

**Change Buffer：** 仅对**非唯一**二级索引生效。当目标索引页不在 Buffer Pool 时，INSERT 操作及 UPDATE/DELETE 引起的二级索引变更（如插入新索引项、标记删除）先写入 Change Buffer（原名 Insert Buffer），避免立即随机读盘。后续该页被任何读操作加载进 Buffer Pool 时，触发 merge 将 Change Buffer 中的变更应用到页上。唯一索引不适用 Change Buffer，因为必须立即读盘验证唯一性约束。

**后台线程：**

| 线程                | 职责                     |
|-------------------|------------------------|
| IO Thread         | 异步读/写、Change Buffer 合并  |
| Page Cleaner      | 从 Flush List 选脏页异步刷盘   |
| Purge Thread      | 清理不再需要的 Undo 版本        |
| Lock Wait Timeout | 检测并回滚锁等待超时事务           |

## 五、一条 SQL 从接收到返回结果的完整生命周期

### 1. SELECT 示例

```sql
SELECT name, age FROM users WHERE id = 42;
```

**阶段 1 — 连接层接收：**

1. Client 从连接池获取已认证连接（TCP 长连接，无需重复握手）。
2. 发送 COM_QUERY 包，payload 为 UTF-8 编码的 SQL 文本。
3. 服务线程从 socket 读包，识别命令码 `0x03`，调用 `dispatch_command()` → `mysql_parse()`。

**阶段 2 — 解析：**

4. 词法分析器切分 Token 流。
5. 语法分析器构建 AST：`SelectStmt(select_list=[name,age], from=users, where=Eq(id,42))`。
6. 语法检查通过，进入预处理器。

**阶段 3 — 预处理：**

7. 查询 Data Dictionary，确认 `users` 表存在。
8. 验证 `name`、`age`、`id` 列存在且类型合法。
9. Access Control 模块检查 SELECT 权限。
10. 对 `users` 获取 MDL SHARED_READ 锁。
11. `open_table()` 创建 InnoDB handler，加载表元信息（列定义、索引列表）。

**阶段 4 — 优化：**

12. 读取 `users` 统计信息：`table_rows ≈ 1000000`，主键 cardinality ≈ 1000000。
13. 估算全表扫描成本：约 `10000 × io_block_read_cost`（按数据页数估算，每页约 100 行）。
14. 估算主键等值查找：`≈ 4 × io_block_read_cost`（B+ 树 3~4 层，每层一次随机读）。
15. 选择 `type=const` 的主键查找，生成计划树。

**阶段 5 — 执行：**

16. 执行器 `handler->index_read_map(pk, 42, HA_READ_KEY_EXACT)`。
17. InnoDB：Buffer Pool page hash 查找 `(space_id, page_no)` → 命中则直接内存读；miss 则发起异步 I/O 请求并等待其完成，将页加载进 Buffer Pool。
18. 在聚簇索引 B+ 树叶子节点定位 `id=42` 的完整行。
19. 执行器从 record 缓冲区提取 `name`、`age`，组装结果。
20. 发送 Column Definition × 2 → Row Data Packet × 1 → EOF/OK 包。

**阶段 6 — 收尾：**

21. 释放 MDL SHARED_READ。
22. `close_table()`，handler 实例销毁或回表缓存。
23. 线程回到 `thread_sql` 循环，等待下一 Command。

Buffer Pool 命中时总耗时通常在 1 ms 以内；优化器耗时微秒级；主要变量是 InnoDB 页是否在内存中。

### 2. UPDATE 示例

```sql
BEGIN;
UPDATE users SET age = 31 WHERE id = 42;
COMMIT;
```

与 SELECT 的差异：

**执行阶段：**

1. 预处理器获取 MDL SHARED_WRITE（而非 SHARED_READ），允许读但标记为写意图。
2. 优化器同样选择主键等值查找（`type=const`）。
3. 执行器 `index_read_map` 定位行 → `ha_update_row(old_row, new_row)`。
4. InnoDB 写路径：
    - 对 `id=42` 加排他行锁（X Lock），阻止其他事务修改或 `SELECT ... FOR UPDATE`。
    - 写 Undo Log：记录 `age` 旧值 30，trx_id，roll_ptr 链接到 Undo 链。
    - 在 Buffer Pool 中修改行（`age=31`），页加入 Flush List 标记为脏页。
    - 写 Redo Log Buffer：记录该页的物理变更（仅在内存中；真正持久化需 COMMIT 时 fsync Redo Log）。
    - 若 `age` 有二级索引且值变化：直接更新索引页，或通过 Change Buffer 延迟（非唯一索引且页不在内存时）。
5. Server 层将变更写入 Binlog Cache（`binlog_format=ROW` 时记录变更前后镜像；`STATEMENT` 时记录 SQL 文本）。

**提交（两阶段提交）：**

1. InnoDB Prepare：Redo Log 标记 prepare。
2. flush Binlog 到磁盘。
3. InnoDB Commit：Redo Log commit 标记，释放行锁。

若 Binlog 写入成功但 InnoDB commit 前崩溃，重启后 XA recover 完成 commit；若无对应 Binlog 则回滚。COMMIT 返回后脏页由 Page Cleaner 异步刷盘，Redo 已持久化即可保证崩溃恢复。

### 3. 完整路径总览

```
Client ──COM_QUERY──→ 连接层 ──dispatch──→ Server层
                                              ├─ Parse
                                              ├─ Preprocess
                                              ├─ Optimize
                                              └─ Execute ──→ InnoDB
                                                              ├─ Buffer Pool
                                                              ├─ 加锁/MVCC
                                                              └─ Undo/Redo
              ←──结果/OK──←──────────────←── 返回行数据
                                              ├─ Binlog Cache
                                              └─ COMMIT(2PC) ──→ Prepare/Commit
                                                                  └─ (异步)刷脏
```

## 六、Server 层与存储引擎的接口边界

### 1. handler API 的设计

Handler API 定义在 `sql/handler.h`，每个表一个 `handler` 实例，Server 通过虚函数与引擎交互。设计原则：

- **面向表、面向行：** 每表一个 handler，CRUD 以行为粒度。
- **引擎无关：** Server 不感知底层 B+ 树或 Hash 表。
- **能力声明：** `table_flags()` 声明是否支持事务、行锁、Online DDL 等，Server 据此调整行为。

### 2. 关键方法

| 方法                          | 作用                                     |
|-----------------------------|----------------------------------------|
| `ha_open` / `ha_close`      | 打开/关闭表，初始化 `.ibd`、索引元信息                |
| `ha_index_read_map`         | 按索引键精确查找或范围扫描起点                        |
| `ha_index_next`             | 沿索引读下一行                                |
| `ha_rnd_next`               | 全表扫描下一行                                |
| `ha_write_row`              | INSERT                                 |
| `ha_update_row`             | UPDATE（内部完成加锁、Undo/Redo、改 Buffer Pool） |
| `ha_delete_row`             | DELETE                                 |
| `ha_external_lock`          | 语句开始/结束时获取释放表级意向锁与资源                   |
| `ha_commit` / `ha_rollback` | 事务提交/回滚，管理 trx_id、Read View、锁释放        |

执行器的 IndexScan/TableScan 算子最终都调用上述读方法；写操作在 `ha_update_row` 等内部完成引擎全部写路径逻辑。

### 3. 为什么有些操作在 Server 层，有些在引擎层

**Server 层：** SQL 解析与优化（跨表统一视图）、JOIN 执行、聚合/排序（filesort）、权限检查、MDL、Binlog、临时表——通用逻辑，与存储格式无关。

**引擎层：** 物理读写、B+ 树索引查找、行锁/表锁、MVCC Read View、Redo/Undo Log、Buffer Pool、崩溃恢复——与存储结构紧密绑定。

**协作优化（灰色地带）：**

- **ICP（Index Condition Pushdown）：** Server 将 WHERE 条件下推到引擎，索引扫描时过滤，减少回表。
- **MRR（Multi-Range Read）：** 批量索引范围请求，引擎按物理页顺序读，减少随机 I/O。
- **Covering Index：** 优化器（Server）判断列全在索引中，引擎直接返回索引记录。

### 4. 这种分层对性能的影响

**正面：** 引擎可替换（MyISAM 只读、Memory 临时）；引擎独立优化（Buffer Pool、Change Buffer、AHI）；Server 层优化（CBO、Hash Join、子查询重写）对所有引擎生效。

**负面：** 逐行接口——每次 `ha_index_next` 是虚函数调用 + B+ 树遍历，大结果集开销显著。例如返回 10 万行的全表扫描，意味着 10 万次虚函数调用和 B+ 树 leaf 遍历，CPU 开销可观。MySQL 8.0 的 Hash Join 在 Server 层批量构建哈希表，部分缓解但仍未完全转向向量化执行。

引擎到 Server 的 record 缓冲区存在多次拷贝：InnoDB 将行写入 handler 的 `record[0]` 缓冲区 → 执行器从中提取列值 → 编码为 MySQL 协议格式发送 Client。大结果集场景下，拷贝和协议编码可能占 CPU 的显著比例。

两阶段提交每次 COMMIT 至少涉及 Redo Log fsync + Binlog fsync（取决于 `sync_binlog` 和 `innodb_flush_log_at_trx_commit`配置），这是分层架构为保证 Binlog 与 InnoDB 一致性必须付出的代价。高写入吞吐场景下，fsync 延迟往往是瓶颈——这与 Handler API 设计无关，而是复制架构的固有成本。

优化器无法精确感知 Buffer Pool 当前是否缓存了目标页、Change Buffer 中有多少待合并条目。它只能基于统计信息和成本模型**估算** I/O 次数，无法像引擎内部那样做运行时自适应。这可能导致：数据已在内存中但优化器仍按磁盘 I/O 估算成本；或 Change Buffer 积压严重但优化器不知情。ICP、MRR、AHI（Adaptive Hash Index）等机制是引擎层在 Handler API 之下做的运行时优化，弥补优化器静态估算的不足。

## 七、查询缓存的废弃

### 1. 查询缓存的工作原理

Query Cache 在 Server 层实现（5.7 及以前），位于解析之后、优化之前：

```
                      ┌──────────┐
                      │ SQL 文本  │
                      └────┬─────┘
                           ▼
                      ┌──────────┐
                      │   解析   │
                      └────┬─────┘
                           ▼
                   ┌───────────────┐
                   │ 查 Query Cache │
                   └───────┬───────┘
                     ┌─────┴─────┐
                     ▼           ▼
                   命中         未命中
                     │           │
                     ▼           ▼
               ┌──────────┐ ┌──────────┐
               │ 直接返回  │ │ 优化+执行 │
               │ 结果集    │ └────┬─────┘
               └──────────┘      ▼
                          ┌──────────────┐
                          │ 写入 Cache   │
                          └──────┬───────┘
                                 ▼
                          ┌──────────────┐
                          │   返回结果    │
                          └──────────────┘
```

- **缓存键：** 规范化 SQL 文本 + 当前数据库名。
- **缓存值：** 完整 Result Set。
- **失效：** 表上任何写操作（INSERT/UPDATE/DELETE/ALTER）使该表所有缓存条目失效。

```sql
SET GLOBAL query_cache_type = ON;
SET GLOBAL query_cache_size = 67108864;
SHOW STATUS LIKE 'Qcache%';  -- Qcache_hits, Qcache_inserts, Qcache_lowmem_prunes
```

### 2. 为什么在高并发下成为瓶颈

根本缺陷是**全局互斥锁**：

- 读路径：每次 SELECT 加全局读锁查缓存，99% 未命中也要加锁。
- 写路径：任何 DML 加全局写锁失效缓存，阻塞所有读锁。
- 写缓存：结果写入 Cache 也需写锁。

```
         Query Cache 全局互斥锁下的并发冲突

Thread1 │ SELECT ── 读锁(命中检查) ── 未命中 ── 执行SQL
        │                                         │
        │                 ┌── 写锁·写入Cache ◀─────┘
        │                 │        │
Thread2 │ UPDATE ── 写锁·失效缓存 ◀──┘    ← 写锁阻塞所有其他操作
        │         │
        │         └─────────────────────────▶ Thread3/4 读锁等待
        │
Thread3 │ SELECT ── 等待读锁...
Thread4 │ SELECT ── 等待读锁...
```

Sysbench OLTP 测试显示开启 Query Cache 性能反而低于关闭。此外：失效粒度太粗（一行 UPDATE 失效整表缓存）；内存碎片化（`Qcache_lowmem_prunes` 频繁增长）；Prepared Statement 参数不同导致无法命中。

### 3. MySQL 8.0 移除的原因

1. OLTP 场景下锁争用开销超过缓存收益（仅只读/极少写负载才有正收益）。
2. 缓存层次不合理——应在 Buffer Pool（页级）或应用层（业务对象级），而非 Server 层 SQL 文本级。
3. 与解析/执行路径深度耦合，维护成本高。
4. 误导用户分配大量内存（如 512 MB）却适得其反。

8.0 移除后 `query_cache_type`、`query_cache_size` 及 `Qcache%` 状态变量均不存在。

### 4. 替代方案

**应用层缓存（首选）：** Redis/Memcached 缓存热点数据；Caffeine/Guava 做进程内本地缓存；ORM 二级缓存做实体级缓存。优势：业务粒度精确失效、无全局锁、可缓存聚合/计算结果。挑战：缓存一致性——更新时需 Cache-Aside 失效或 Write-Through。

典型 Cache-Aside 模式：

```
    读路径                              写路径
  ──────────                          ──────────
      │                                   │
      ▼                                   ▼
  ┌────────┐                          ┌──────────┐
  │ 查缓存  │                          │ 更新 DB  │
  └───┬────┘                          └────┬─────┘
      │                                    │
    ┌─┴──┐                                 ▼
    ▼    ▼                           ┌──────────┐
  命中  未命中                        │ 删除缓存  │
    │    │                           └──────────┘
    │    ▼
    │  ┌────────┐
    │  │ 查 DB  │
    │  └───┬────┘
    │      ▼
    │  ┌──────────┐
    │  │ 写入缓存  │
    │  └────┬─────┘
    │       │
    └───┬───┘
        ▼
    ┌────────┐
    │ 返回结果 │
    └────────┘
```

写路径务必"先更新 DB 再删缓存"，而非"先删缓存再更新 DB"——否则并发读可能在 DB 更新前将旧值重新写入缓存。对一致性要求极高的场景，可考虑 Read-Through（缓存层代理读）或订阅 Binlog 异步失效缓存。

**ProxySQL 等中间件：** 在 Proxy 进程内缓存，不阻塞 MySQL Server，可配置精细规则（如只缓存特定 schema 的 SELECT、设置 TTL）。ProxySQL 还提供连接复用、读写分离、查询路由——与缓存协同时可实现"读走从库 + 缓存"的架构。但增加单点/运维复杂度，缓存一致性仍是挑战。大多数生产环境选 Redis + 应用层控制，而非 Proxy 层缓存。

**InnoDB Buffer Pool（引擎层）：** 严格说不是 Query Cache 替代——缓存的是数据页而非查询结果。但页命中意味着 repeated SELECT 无需磁盘 I/O，且不受 SQL 文本变化影响、无 DML 全局失效、无全局锁争用：

```
┌─────────────────────────────────────────────────────────────────────┐
│  Query Cache (SQL 级，已废弃)                                        │
│  相同 SQL 文本 ──▶ 直接返回 Result Set ──▶ 任一 DML 使全表缓存失效    │
│  瓶颈：全局互斥锁                                                    │
├─────────────────────────────────────────────────────────────────────┤
│  Buffer Pool (页级，核心机制)                                        │
│  数据页在内存 ──▶ 直接读取 ──▶ 不受 SQL 文本变化影响 ──▶ LRU 淘汰    │
│  无全局锁争用，按页粒度并发                                          │
└─────────────────────────────────────────────────────────────────────┘
```

## 总结

MySQL 的分层架构——连接层、Server 层、存储引擎层——是理解其运行机制、排查问题、做出架构决策的基础框架。

连接层负责客户端接入与线程调度。一连接一线程在连接数适中时简单有效，高并发下需客户端连接池和可选 Thread Pool 缓解压力。认证、SSL、会话上下文在此层建立。

Server 层是 SQL 的"大脑"：解析器将文本变为 AST，预处理器做语义检查与权限验证，优化器基于 CBO 选择执行计划，执行器通过 Handler API 调用引擎并返回结果。MDL、Binlog、慢查询日志贯穿所有 SQL 处理。

存储引擎层负责物理存取。InnoDB 通过 Buffer Pool、Redo/Undo Log、行锁和 MVCC 提供完整事务能力与崩溃恢复。插件式架构允许多引擎共存，生产环境几乎全部使用 InnoDB。

一条 SQL 的完整生命周期串联所有层次：COM_QUERY → 解析/优化/执行 → InnoDB 读写 Buffer Pool 和日志 → 结果返回或两阶段提交。UPDATE 相比 SELECT 多了写路径和 Binlog，COMMIT 时协调 InnoDB 与 Binlog 一致性。

Handler API 边界决定"什么在 Server 做、什么在引擎做"——带来引擎可替换性和独立优化，也引入逐行接口开销和跨层协调成本。ICP、MRR 等是两层协作弥合差距的尝试。

Query Cache 的废弃是分层架构的典型案例：缓存应放在正确层次——页缓存由 Buffer Pool 在引擎层完成，结果缓存由应用在 Server 之外完成。Server 层以 SQL 文本为键的全局缓存，在高并发下必然因锁争用而失败。

掌握这套分层视角后，面对慢查询可区分优化器选错计划还是 Buffer Pool 命中率低；面对锁等待可区分 MDL 阻塞还是 InnoDB 行锁冲突；面对主从延迟可从 Binlog 写入和复制线程角度分析。架构分层是 MySQL 运维和性能调优的地图，值得每一位使用者深入理解。
