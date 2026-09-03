---
title: Redis ZSet 的实现原理：为什么用哈希表与跳表维护有序集合
summary: 以 Redis 7.4 为边界，解释 ZSet 如何在紧凑 listpack 与哈希表加跳表之间选择，并说明成员查找、排序、排名和范围查询背后的设计取舍。
created: 2026-07-22
updated: 2026-07-22
tags: Redis, ZSet, 跳表, listpack, 数据结构
---

# Redis ZSet 的实现原理：为什么用哈希表与跳表维护有序集合

排行榜里既要快速得到“用户 A 的分数”，又要取“前 100 名”，还可能查询“分数在 80 到 90 之间的人”。如果只把数据放进哈希表，按用户查分快，却没有顺序；如果只维护一个有序链表，顺序有了，查某个用户又会变慢。Redis ZSet（有序集合）正是为这类双重访问模式设计的。

本文以 Redis 7.4 为边界，只讨论 ZSet 的内存表示、查询与更新逻辑，以及这些设计背后的取舍；不逐行讲解源码，也不展开持久化、集群、阻塞弹出命令或多集合聚合。

一句话结论：**小 ZSet 使用紧凑的 listpack 节省内存；规模或成员长度超过阈值后，Redis 用“哈希表 + 跳表”保存同一批成员——哈希表按成员定位，跳表按 `(score, member)` 排序并维护排名。**

## ZSet 要同时满足什么需求

ZSet 是一组唯一字符串成员，每个成员关联一个浮点分值。成员按分值从低到高排列；分值相同时，按成员的字典序排列。这条同分规则很重要：它让顺序完全确定，而不依赖插入先后。

以游戏排行榜为例：

```redis
ZADD leaderboard 100 alice 120 bob 120 carol 90 dave
ZRANGE leaderboard 0 -1 WITHSCORES
# 1) "dave"  2) "90"
# 3) "alice" 4) "100"
# 5) "bob"   6) "120"
# 7) "carol" 8) "120"
```

这里需要三类不同的访问：

| 问题 | 对应命令 | 需要的能力 |
| --- | --- | --- |
| `alice` 当前多少分？ | `ZSCORE` | 按成员快速定位 |
| `alice` 排第几？ | `ZRANK` | 从成员找到其有序位置 |
| 前 100 名是谁？ | `ZRANGE ... REV` | 按全局顺序遍历 |
| 100 到 120 分有哪些人？ | `ZRANGE ... BYSCORE` | 按分值定位一个连续区间 |

单一结构很难同时把这些操作都做好。因此 Redis 根据数据规模选择两种内部编码。

## 小 ZSet：listpack 用连续内存换空间

当 ZSet 较小且成员较短时，Redis 使用 `listpack`。它把成员与分值交替放在一段连续内存中，可粗略理解为：

```text
member-1 | score-1 | member-2 | score-2 | ... | member-N | score-N
```

连续存储避免了哈希桶、节点指针和独立内存分配的额外开销，对几十个短成员尤其省内存，也更容易命中 CPU 缓存。代价是没有成员索引：要找一个成员，通常需要顺序比较；插入或删除也可能移动后续数据。

Redis 7.x 默认在以下条件都满足时使用这种紧凑编码：成员数不超过 `zset-max-listpack-entries`（默认 `128`），且成员字节长度不超过 `zset-max-listpack-value`（默认 `64`）。这些阈值不是性能“推荐值”，而是内存与 CPU 的折中开关；修改前应按真实成员长度、读写比例做压测。

可以通过下面的命令观察编码，但业务代码不应据此分支，因为它属于实现细节：

```redis
ZADD leaderboard 100 alice 120 bob
OBJECT ENCODING leaderboard
# 预期：listpack

CONFIG GET zset-max-listpack-entries zset-max-listpack-value
```

## 大 ZSet：两种视图服务两类查询

超过紧凑编码适用范围后，Redis 使用常规编码：一个哈希表和一个跳表共同表示同一组成员。它们不是两份互不相干的数据，而是针对不同访问路径建立的两种视图。

```mermaid
flowchart LR
    M[成员：alice] --> D[哈希表
成员 → 分值]
    D --> S[分值：100]
    S --> Z[跳表
按 score + member 排序]
    Z --> R[排名与范围结果]
```

- **哈希表**保存“成员 → 分值”的映射。`ZSCORE leaderboard alice` 先走它，因此无需扫描所有成员。
- **跳表**按“先分值、后成员字典序”组织节点。它负责 `ZRANGE`、按分值范围查询和排名定位。
- **共享成员数据**：实现会让两种结构协同管理成员字符串，避免为同一成员保存两份独立字符串；重点是用额外索引换取访问速度，而不是机械地复制两套完整数据。

这解释了 ZSet 的内存特征：常规编码比 listpack 占用更多内存，但把“按成员找值”和“按顺序找范围”都变成了高效路径。

## 为什么排序结构选跳表

平衡树同样能维持有序性，但跳表的思路更直接：最底层是按顺序串起来的完整链表；上层是逐渐稀疏的“快速通道”。查找时先在高层大步前进，快要越过目标时再下到低层细找。

```text
高层：head ───────────────→ 120:bob ─────→
中层：head ───→ 100:alice ─→ 120:bob ───→
底层：head → 90:dave → 100:alice → 120:bob → 120:carol
```

每个节点随机获得层高；层数越高的节点越少。这样不需要像平衡树那样在插入和删除后旋转，也能在期望意义上以 `O(log N)` 定位插入点、删除点和分值边界。

ZSet 的跳表还保存两个关键辅助信息：

- **跨度（span）**：一条前向指针跨过了底层的多少个节点。沿路累加跨度，就能计算排名；按排名找成员时，也能跳过整段节点。
- **后向指针**：从尾部向前遍历，因此高分在前的 `ZRANGE ... REV` 不必先正向走到末尾再倒序缓存。

排名并不是“每个成员都存一个会不断失效的 rank 字段”。排名取决于当前排序，插入一个新成员可能影响其后的所有名次；跨度让 Redis 在查询时计算位置，避免为每次更新重写大量成员的排名。

## 写入和更新：改变分值就是改变有序位置

`ZADD` 既可以新增成员，也可以更新已有成员的分值。逻辑上可分成三步：

1. 先按成员检查是否存在；常规编码下由哈希表完成。
2. 若是新增成员，把它插入有序结构的正确位置，并建立成员到分值的映射。
3. 若分值变化，修复其在有序结构中的位置，再更新映射的分值。

```redis
ZADD leaderboard 100 alice
ZADD leaderboard 150 alice
ZSCORE leaderboard alice
# "150"
```

第二条命令不会产生两个 `alice`：成员唯一性由成员索引保证。但 `alice` 从 100 分升到 150 分，可能跨越许多成员，旧位置不能继续使用。Redis 因而需要把节点移动到新的排序位置并维护相关跨度；这就是单个 `ZADD`/`ZINCRBY` 通常为 `O(log N)` 的原因，而不是简单的 `O(1)` 覆盖写。

若新分值仍落在相邻节点之间，位置没有变化，实现可以少做一些结构调整；从使用者视角看，仍应按 `O(log N)` 估算。

## 查询逻辑与实际成本

常规编码下常见操作的核心成本如下。`M` 表示返回的成员数量。

| 操作 | 主要路径 | 时间复杂度 | 说明 |
| --- | --- | --- | --- |
| `ZSCORE key member` | 哈希表 | `O(1)` | 按成员直接查分值 |
| `ZADD` / `ZINCRBY` | 哈希表 + 跳表 | `O(log N)` | 定位成员并维护有序位置 |
| `ZRANK key member` | 哈希表 + 跳表 | `O(log N)` | 先取分值，再借跨度计算排名 |
| `ZRANGE key start stop` | 跳表 | `O(log N + M)` | 定位起点后顺序输出结果 |
| `ZRANGE key min max BYSCORE` | 跳表 | `O(log N + M)` | 定位分值下界后顺序输出结果 |

这里的 `O(log N + M)` 有一个朴素但重要的含义：定位很快不代表返回海量结果也快。一次 `ZRANGE 0 -1` 仍要构造、传输并解析所有成员。带 `BYSCORE` 的 `LIMIT offset count` 也不是数据库游标：offset 很大时，Redis 需要跳过大量匹配成员，成本会随 offset 增加。

## 用排行榜把数据结构映射回命令

下面的例子既展示用户能看到的行为，也对应前文的三类访问路径。

```redis
ZADD game:rank 98 alice 100 bob 100 carol 82 dave

# 成员 -> 分值：哈希表擅长的路径
ZSCORE game:rank alice
# "98"

# 成员 -> 排名：先定位成员，再在跳表中计算位置；排名从 0 开始
ZRANK game:rank alice
# (integer) 1

# 有序结构从尾部反向遍历，得到前三名
ZRANGE game:rank 0 2 REV WITHSCORES
# 1) "carol" 2) "100"
# 3) "bob"   4) "100"
# 5) "alice" 6) "98"

# 从分数下界定位后，顺序读取分值区间
ZRANGE game:rank 90 100 BYSCORE WITHSCORES
# 1) "alice" 2) "98"
# 3) "bob"   4) "100"
# 5) "carol" 6) "100"
```

注意 `bob` 与 `carol` 同为 100 分：正向顺序中 `bob` 在前，反向顺序中 `carol` 在前。这是分值相同后按字典序、反向查询再反转字典序的直接结果，不应把它误解为“后写入者优先”。

## 实践边界：理解取舍，而不是依赖编码

**不要把内部编码当业务契约。** `OBJECT ENCODING` 很适合排查内存和观察阈值，但版本、配置及未来实现都可能改变编码。业务只应依赖 ZSet 的成员唯一与排序语义。

**避免大 offset 分页。** 排行榜首页取前 50 名适合 `ZRANGE ... REV`；如果用户不停翻到很靠后的页，offset 会让跳过成本不断增加。可按“上一页最后一个分值 + 成员”设计游标，或重新审视是否真的需要深翻页。

**分数必须能表达你的排序规则。** Redis 的 score 是双精度浮点数。对于很大的整数，超出精确表示范围后可能发生精度损失；需要精确大整数排序时，应重新设计编码，不能仅把任意长整数直接作为 score。

**警惕大范围返回和单个超大 ZSet。** ZSet 的范围操作虽然有对数级定位，但返回、网络传输和客户端反序列化均与结果数量线性相关。对持续膨胀的排行榜、时间窗口索引，应设置淘汰边界或按时间/业务维度拆分 key。

## 总结

ZSet 的设计不是“跳表很快”这么简单，而是在两类访问需求之间明确分工：listpack 为小集合降低内存成本；哈希表回答“某成员的分值是什么”；跳表回答“哪些成员位于这个顺序或范围内”；跨度再让排名成为可快速计算的结果。

当业务同时需要成员去重、可更新分值、排名和范围查询时，ZSet 很合适。若只需要唯一性，用 Set 更轻；若只按 key 查值，用 Hash 更直接；若只是追加事件流，则应考虑 Stream。先根据访问模式选类型，再把 ZSet 的内部取舍当作容量与性能评估依据，通常比死记具体编码更有价值。

## 参考资料

- [Redis 官方文档：Sorted sets](https://redis.io/docs/latest/develop/data-types/sorted-sets/)
- [Redis 官方文档：ZRANGE](https://redis.io/docs/latest/commands/zrange/)
- [Redis 官方文档：ZADD](https://redis.io/docs/latest/commands/zadd/)
- [Redis 官方文档：Memory optimization](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/)
- [Redis 7.4 源码：src/t_zset.c](https://github.com/redis/redis/blob/7.4/src/t_zset.c)
