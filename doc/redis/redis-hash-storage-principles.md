---
title: Redis Hash 类型的数据存储原理：listpack 与 hashtable 的内部编码
summary: 以 Redis 7.4 为边界，解释 Hash 值在内存中的 robj 对象、listpack 与 hashtable 两种内部编码、编码切换的阈值与触发条件，以及不同编码下常见命令的实际代价。
created: 2026-07-20
updated: 2026-07-20
tags: Redis, Hash, listpack, 对象编码, 内存结构
---

# Redis Hash 类型的数据存储原理：listpack 与 hashtable 的内部编码

Redis 的 Hash 在外表现是一个字段-值的映射，`HSET user:1001 name "张三" age "28"` 看起来再自然不过。但落到内存里，同一个 Hash key 存 5 个字段和存 600 个字段，`HGET` 的耗时可以差一个量级——差别不在网络，而在 Redis 为这个 Hash 选择了不同的内部数据结构。

本文只讨论 Redis 7.4 下 Hash 值的内存表示、listpack 和 hashtable 两种编码的存储结构与查找机制、编码切换的阈值和触发时机，以及相关命令在不同编码下的时间复杂度。Hash 的 field 过期（`HEXPIRE`）、Redis Cluster 的 hash tag 路由、Lua 脚本内的 Hash 原子操作、Stream 和 Pub/Sub 不展开。

一句话概括：**Hash 的逻辑类型固定是键值对集合，但 Redis 会根据字段数量和单值大小，在紧凑连续内存（listpack）和标准哈希表（hashtable）之间自动切换；切换是单向且阻塞的，大 Hash 升级时需要留意。**

## 从 robj 看 Hash 的第一层包装

和 String 一样，Redis 键空间中保存的 Hash 值也是一个 `robj`。对 Hash 来说，`robj` 回答三个问题：

- `type`：这个对象的逻辑类型是 `OBJ_HASH`；
- `encoding`：当前用 `OBJ_ENCODING_LISTPACK` 还是 `OBJ_ENCODING_HT`；
- `ptr`：指向实际数据——要么是一段 listpack 内存，要么是一个 `dict` 结构。

执行 `OBJECT ENCODING key` 时，Redis 返回的是内部编码名。对一个少量字段的 Hash，看到的是 `listpack`；字段数或单值大小超过阈值后，编码变为 `hashtable`。和 String 一样，这个编码是实现细节，不应写进业务兼容判断。

两种编码的设计意图不同：

- **listpack**：把 field 和 value 交替平铺在一块连续内存中，追求紧凑和缓存友好，但查找只能靠线性扫描。
- **hashtable**：用标准哈希表组织 field-value 对，O(1) 查找，代价是每个 entry 都有 dict 结构的额外内存开销。

Redis 7.0 之前，紧凑编码用的是 ziplist；7.0 起换成了 listpack。listpack 在结构上解决了 ziplist 的连锁更新问题，但两种编码在 Hash 上的使用逻辑是一致的——都是"字段少且值短时用紧凑结构，超过阈值时切换为标准哈希表"。

## listpack 编码：紧凑但 O(N) 扫描

listpack 是 Redis 7.0 引入的序列化结构，用来替代 ziplist 作为 String、Hash、ZSet 等类型的小量数据编码。在 Hash 场景下，listpack 的内存布局大致为：

```text
listpack 整体布局:
┌──────────────┬───────────────┬─────────┬─────────┬───┬─────────┬──────┐
│ total_bytes  │ num_elements  │ field 1 │ value 1 │ … │ value N │ 0xFF │
│ (4 bytes)    │ (2 bytes)     │         │         │   │         │ end  │
└──────────────┴───────────────┴─────────┴─────────┴───┴─────────┴──────┘
```

- `total_bytes`：整个 listpack 的字节数，4 字节。
- `num_elements`：元素个数，2 字节。对于 Hash，这里的值是 `字段数 × 2`（每个 entry 代表一个 field 或一个 value）。
- entry 序列：field 和 value 交替排列，entry 格式根据数据类型而变化：小整数直接编码进 encoding byte，字符串则为 `<encoding-byte><data><backlen>`，其中 backlen 在 entry 尾部编码总长度，支持从后向前遍历。
- 结尾 `0xFF`：一个字节的结束标记。

每个 entry 的 encoding byte 决定了 data 的类型和长度。小整数可以编码进 encoding byte 本身（0-12），短字符串用变长整数存长度，长字符串则包含 backlen（从后向前读取 entry 长度的能力，支持反向遍历）。

从查找角度看，listpack 结构本身不提供索引。要找到一个 field，Redis 必须从第一个 entry 开始逐对扫描：读 field，比较；不匹配则跳过对应 value，继续读下一个 field。因此 listpack 编码下 `HGET`、`HSET`（更新已有 field）、`HDEL` 的复杂度都是 O(N)，N 为字段数。

不过 `HLEN` 在 listpack 下是 O(1)：只需要读 `num_elements` 并除以 2。`HGETALL` 遍历也是 O(N)，无需理会 field 是否匹配，直接顺序读取所有 entry 即可，反而比随机 field 查找更经济。

listpack 还有一个与写操作相关的特性：当 `HSET` 修改一个已有 field 且新 value 长度与旧 value 不同时，如果 listpack 尾部没有足够的空闲空间，Redis 需要在 entry 位置原地腾出或收缩空间，这涉及 `memmove` 移动后续所有 entry。这一操作的复杂度也是 O(N)。而删除一个 entry 时，listpack 会真正移除该 entry 并紧缩后续数据。

## hashtable 编码：标准哈希表

当 Hash 的字段数或单值大小超过阈值后，Redis 调用 `hashTypeConvert` 将 listpack 整体转换为 `dict`。转换过程遍历 listpack 中的每一对 field-value，逐对插入新 dict。

转换后的 Hash 内部是一个 Redis `dict` 结构（搭配 `dictType hashDictType`）。dict 使用 SipHash 对 field 做哈希，支持渐进式 rehash，字段查找为 O(1)。转换为 hashtable 后，`HGET`、`HSET`、`HDEL`、`HEXISTS` 都可以直接在 dict 中定位目标 entry，不再依赖线性扫描。

与 listpack 不同，hashtable 编码下每个 field-value 对都是一个独立的 dict entry，包含 `dictEntry` 结构（key 指针、v 联合体、next 指针），再加上 field 和 value 各自的 SDS/sds 分配。这意味着内存开销显著高于 listpack：一个只有两三个字段的 Hash，在 hashtable 编码下不仅要为 dict 本身分配哈希桶数组，还要为每个 entry 单独分配内存。

Redis 不会在字段数回落到阈值以下时把 hashtable 降级回 listpack。编码切换是单向的：listpack → hashtable，且不可逆。因此，即使后续 `HDEL` 删到只剩 3 个字段，Hash 仍然保持 hashtable 编码。

## 编码切换的阈值与触发条件

编码升级由两个配置参数控制：

| 配置项 | 默认值 | 含义 |
|---|---|---|
| `hash-max-listpack-entries` | 512 | 字段数超过该值时，升级为 hashtable |
| `hash-max-listpack-value` | 64 | 任一 value 的字节长度超过该值时，升级为 hashtable |

注意第二个参数检查的是 **value 的长度**，不是 field 的长度。field 长度不受此阈值限制——尽管在 listpack 中 field 和 value 都是 entry，但配置只判断 value。

检查逻辑位于 `hashTypeTryConversion` 函数中，大致等价于：

```text
如果当前编码 == OBJ_ENCODING_LISTPACK:
    如果 (字段数 > hash-max-listpack-entries) 或 (待写入 value 的字节长度 > hash-max-listpack-value):
        调用 hashTypeConvert 将 listpack 转为 dict
```

这个检查发生在每次写操作（`HSET`、`HMSET`、`HINCRBY`、`HINCRBYFLOAT` 等）之前。转换本身是 O(N) 的阻塞操作——需要遍历当前所有 field-value 并逐一插入新 dict。在默认 512 字段的阈值下，这个转换通常很快完成；但如果将 `hash-max-listpack-entries` 调到比如 100000，转换发生时就需要遍历 10 万个 entry，消耗明显 CPU 时间。

另一个值得注意的细节：`HMSET` 一次性写入多个 field 时，如果这些 field 使 Hash 跨过阈值，转换会在命令执行中途发生。这意味着同一个 `HMSET` 调用内的前几个 field 通过 listpack 写入，后几个 field 通过新 dict 写入——客户端感知不到，但命令的耗时分布会因这次转换而改变。

## 常见操作的时间复杂度对比

| 命令 | listpack 编码 | hashtable 编码 | 说明 |
|---|---|---|---|
| `HSET` | O(N) | O(1) | listpack 需扫描定位已有 field；新 field 直接追加为 O(1)，但检查重复仍需 O(N) |
| `HGET` | O(N) | O(1) | listpack 从头扫描 field |
| `HDEL` | O(N) | O(1) | listpack 扫描定位 + memmove 紧缩 |
| `HLEN` | O(1) | O(1) | listpack 直接读 `num_elements / 2`；dict 直接读 `dictSize` |
| `HEXISTS` | O(N) | O(1) | 同 HGET |
| `HGETALL` | O(N) | O(N) | 两者都要遍历所有 field-value |
| `HKEYS` / `HVALS` | O(N) | O(N) | 同上 |
| `HINCRBY` | O(N) | O(1) | listpack 下定位 field + 可能因 value 变长触发 memmove |
| `HSTRLEN` | O(N) | O(1) | listpack 需先定位 field |
| `HSCAN` | O(1) per call | O(1) per call | dict 用哈希桶游标保证无重复无遗漏；listpack 用位置偏移模拟游标，迭代期间删除 entry 可能导致重复或遗漏 |

这张表里最容易被忽略的是 listpack 下 `HSET` 对已有 field 的更新：扫描找到 field 后，如果新 value 长度与旧 value 不同，还需要 `memmove` 移动后续 entry，这在字段多时累积成本不小。

另一个反直觉的事实：`HGETALL` 在小 Hash（listpack 编码）下通常很快，因为它只是顺序拷贝内存块中的 entry 内容，没有随机指针跳转，对 CPU 缓存友好。切换到 hashtable 后，`HGETALL` 需要遍历 dict 的哈希桶和链表，虽然复杂度也是 O(N)，但在 N 不大时可能反而比 listpack 版本稍慢。

## 可以这样观察编码变化

下面的示例适合在 Redis CLI 中执行。不同平台和 Redis 版本可能影响内部编码结果，重点在于观察机制。

```redis
FLUSHDB

# 初始为空 Hash，写入第一个字段后自动创建
HSET user:1 name "Alice"
OBJECT ENCODING user:1
# 预期: listpack（字段少，值短）

# 写入多个字段，保持在阈值内
HSET user:1 age "30" city "Beijing" email "alice@example.com"
HLEN user:1
OBJECT ENCODING user:1
# 预期: listpack（字段数 4，未超 512）

# 观察阈值切换：写入一个超长 value
HSET user:1 bio "这是一个很长的个人简介...(此处省略超过 64 字节的内容)"
OBJECT ENCODING user:1
# 如果 bio 的字节长度超过 hash-max-listpack-value (默认 64)，编码变为 hashtable

# 确认切换不可逆
HDEL user:1 bio
HLEN user:1
OBJECT ENCODING user:1
# 仍然为 hashtable，不会降级回 listpack
```

也可以用一个脚本来批量填充并观察转换点：

```redis
FLUSHDB
HSET big:hash f1 v1
OBJECT ENCODING big:hash
# listpack

# 填充 512 个字段（默认阈值）
# 可以写一个循环：for i in $(seq 2 513); do redis-cli HSET big:hash "f$i" "v$i"; done
# 当字段数超过 512 时，OBJECT ENCODING 变为 hashtable
```

如果在循环过程中用 `OBJECT ENCODING` 查询，可以看到编码在恰好超出阈值那次 `HSET` 之后发生变化。

## 实践建议与边界

**不要假设编码。** 和 String 一样，Hash 的内部编码是实现细节。业务代码应依赖命令语义（字段唯一、读写一致性），不应依赖 `OBJECT ENCODING` 结果做分支逻辑。编码可能因版本、配置或实现优化而变化。

**大 Hash 拆分的时机。** `HGETALL` 在万级字段时无论哪种编码都会产生大量网络输出和内存分配。如果业务需要频繁全量读取 Hash，应考虑拆分：用一个 Set 或 ZSet 维护字段索引，或者按业务维度拆成多个小 Hash。一次性返回几万个字段的网络包不仅影响客户端解析性能，也可能触发 Redis 的输出缓冲区限制。

**阈值调整的取舍。** 若业务确定某些 Hash 的字段数稳定在 1000 左右且需要频繁单字段读写，可以将 `hash-max-listpack-entries` 调大到 2000，让 Hash 保持在 listpack 编码以节省内存，但需要接受 O(N) 的 `HGET` 开销。反之，若内存充裕而查询延迟敏感，保持默认 512 即可。阈值调整是内存与 CPU 之间的权衡，不应盲目放大。

**用 Hash 存对象 vs 多个 String key。** Hash 存对象的优势在于一个 key 承载多个属性，减少键空间 key 数量和内存碎片。但如果对象的某些字段需要独立设置 TTL（Redis 7.4 起 `HEXPIRE` 支持字段级过期），或是需要原子操作多个对象级别属性，Hash 比多个 String key 更内聚。反过来，若各字段独立访问且生命周期不同，分开存为 String key 可能更灵活。两者之间没有绝对答案，核心判断标准是：这些字段在业务逻辑上是否属于同一个聚合。

**listpack 下删除字段的真实成本。** 尽管 `HDEL` 在 listpack 下是 O(N)，但在 N 较小（百级以内）时，CPU 缓存和连续内存的友好性通常会使操作肉眼不可见。真正的性能拐点出现在 N 数百以上，此时单次 `HGET` 的线性扫描开始产生可测量的延迟。hashtable 编码虽然消除了扫描，但内存占用会成倍增长，所以没有"免费"的切换。

**考虑 `HSCAN` 替代 `HGETALL`。** 当字段数较大（数千以上）且需要遍历时，`HSCAN` 的游标式分批迭代比一次 `HGETALL` 更安全。它在服务端不阻塞其他请求，客户端也能控制每批大小。listpack 编码下的 `HSCAN` 用位置偏移模拟游标，删除操作可能导致重复或遗漏，因此在遍历期间不要依赖精确的去重语义。

总结一下 Redis Hash 存储的核心要点：listpack 用连续内存和线性扫描换取空间效率，hashtable 用独立分配和 O(1) 指针跳转换取访问速度。编码在写入时单向切换，不可逆。具体选择由 `hash-max-listpack-entries` 和 `hash-max-listpack-value` 两个配置决定，用 `OBJECT ENCODING` 查看。业务上理解这些边界，不是为了依赖编码，而是为了在字段规模、读写模式和内存预算之间做出合理取舍。

## 参考资料

- [Redis 官方文档：Hashes](https://redis.io/docs/latest/develop/data-types/hashes/)
- [Redis 官方文档：HSET](https://redis.io/docs/latest/commands/hset/)
- [Redis 官方文档：OBJECT ENCODING](https://redis.io/docs/latest/commands/object-encoding/)
- [Redis 官方文档：Memory optimization — Hashes](https://redis.io/docs/latest/develop/reference/optimization/memory-optimization/#using-hashes-to-store-objects)
- [Redis 7.4 配置参考：hash-max-listpack-entries](https://redis.io/docs/latest/operate/oss_and_stack/management/config/)
- [Redis 7.4 源码：src/object.c](https://github.com/redis/redis/blob/7.4/src/object.c)
- [Redis 7.4 源码：src/t_hash.c](https://github.com/redis/redis/blob/7.4/src/t_hash.c)
- [Redis 7.4 源码：src/listpack.h](https://github.com/redis/redis/blob/7.4/src/listpack.h)
- [Redis 7.4 源码：src/listpack.c](https://github.com/redis/redis/blob/7.4/src/listpack.c)
