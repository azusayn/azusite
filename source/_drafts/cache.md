## 缓存一致性

blog 里面有图，关于谁管理缓存、如何更新缓存

## 缓存雪崩、缓存穿透、缓存击穿

> 缓存设置过期是为了解决一些极端情况下系统存在旧数据，但缓存写不进去，（比如很倒霉我们的 service 没法访问 cache，但是用户查询先过 cache）。导致
> 请求一直被响应过期的数据。如果设置了过期时间至少可以等数据过期，请求直达 db。

<https://philosophyotaku.medium.com/a-complete-beginner-guide-for-cache-penetration-stampede-avalanche-ecadd7f16009>
avalanch、penatration、stampede

- penatration 查询不存在的 key 导致请求持续击打 db （尽管我们建立了缓存、但是仍存在取值是不在缓存 key 集合内）
  - bloom filter
  - 查不到的 key 也存一个 "" 进去

- stampede/breakdown 某一个 key 过期的时候正好有大量请求一起攻击 db 去重建 cache，导致 db 读压力大（就算有对应缓存，但是缓存有过期时间）
  - 设置一把锁去保证只有一个 worker 做缓存重建。可以直接用 golang 的 singleflight
  - 比较激进： 热点数据不设置 TTL，但是有别的 worker 定时读 db 更新 cache（微博热搜/秒杀商品）

- avalanche （不是热点 key 过期，但是是大量 key 集中过期，可能是时间到了，也可能是 redis 崩了）
  <!-- 尽量避免 -->
  - 给 redis 集群做高可用
  - 设置随机抖动分散缓存过期的时间
  <!-- 发生后的措施 -->
  - 做多级缓存（利用 service 内存），同时 db 前做限流保护
  - service 层做三态 circuit break （然后可以 fail fast 也可以 fail silent）
  
- Failure Model 里，degradation 指的是系统丧失一些性能但是业务逻辑仍然正常处理。而 Fail Silent 指的是错误不直接返回给下
- 我这里上游的说法主要来自于 envoy 的 terminology，大家的习惯不一样 <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/terminology>
- 就算数据设置了 Redis 的 TTL，也有可能因为外部因素（后端 worker 并发交错执行导致旧数据反刷），系统没有真的在 TTL 内失效。所以解决这个问题需要状态同步。（单 service 实例可以 singleflight， 多 service 分布式锁）

## master slave 模式下如何保证一致性

## 更多问题

你对缓存核心知识点的梳理非常清晰！从**模式**（Cache-Aside / Read-Through / Write-Through 等）到**稳定性三件套**（雪崩、击穿、穿透），再到**一致性兜底**（TTL 与并发反刷），你已经抓住了缓存架构最核心的主线。

如果在工业界大厂或高并发系统架构中继续深挖，缓存领域还有以下 **5 个非常大头、且经常被重点关注和考察的问题**：

### 1. 缓存与数据库的“一致性终极战”

虽然你已经掌握了 Cache-Aside 模式和 TTL 兜底，但在复杂分布式系统里，大家依然会疯狂卷一致性细节：

- **先删缓存再改 DB，还是先改 DB 再删缓存？**
- 为什么业界公认 **“先改 DB，再删缓存（Write to DB then Evict Cache）”** 是最佳实践？
- 如果“删缓存”这一步失败了怎么办？（引入 **Canal / Binlog 订阅异步重试机制**，或者 MQ 异步解耦，保证必定擦除）。

- **延迟双删（Delayed Double Delete）**：
- 在先删缓存的方案里，为了防止并发读写把旧数据刷回，在改完 DB 后 sleep 几百毫秒再删一次缓存。这种机制的“Sleep 时间”怎么评估？

### 2. 热点 Key / 突发热点（Hot Key）

不同于缓存击穿（单个 Key 刚好过期），热点 Key 指的是**某个 Key 访问量极高（比如爆款商品、微博大V发文），QPS 达到几万甚至数十万，直接把单台 Redis 节点的网卡或 CPU 跑满**。

- **问题**：Redis 是单线程模型（网络 IO 虽有多线程，但执行依然是单线程），主从架构或 Cluster 架构下，热点 Key 无法通过 Sharding 散列到多台机器，压力全部集中在一台 Redis 上。
- **主流解决方案**：
- **二级缓存 / 本地缓存（Local Cache）**：在 Go 进程内部（`freecache` / `bigcache` / `ristretto`）放一份短 TTL 的本地缓存，流量在 Service 内存直接拦截，打不到 Redis。
- **Hot Key 动态探测与广播**：引入 J2Cache 或 ByteDance 开源的 Hermes / HotKey 机制，实时统计高频访问的 Key，动态拉取并推送至应用本地内存。
- **Key 加随机前缀散列**：把 `hot_key` 拆成 `hot_key_1`, `hot_key_2` ... `hot_key_N`，分散到 Redis Cluster 的不同 Slot/节点上，读请求随机路由。

### 3. 大 Key 问题（Big Key）

指的是 Redis 中某个 Key 对应的 **Value 极其庞大**（比如一个包含了几十万元素的 `Set` / `Hash` / `ZSet`，或者单个 `String` Value 高达数 MB）。

- **危害**：
- **阻塞单线程**：读取或删除 Big Key（如 `DEL` 一个百万元素的 Hash）会导致 Redis 单线程长时间阻塞，造成整个 Redis 集群耗时拉爆甚至超时。
- **网卡堵塞**：每次拉取几 MB 的 Big Key，瞬间吃光 Redis 节点的网卡带宽。

- **解决办法**：
- **大 Key 拆分**：将大 Hash 按照 `hash(field) % N` 拆分成多个小 Hash 存储。
- **异步删除**：使用 Redis 4.0+ 的 `UNLINK` 替代 `DEL`，将 Big Key 的内存释放操作丢给 Background 线程异步处理。

---

### 4. 内存淘汰机制与内存碎片（Memory Eviction & OOM）

Redis 内存是有限且昂贵的，当 Redis 内存满了（达到 `maxmemory`）会发生什么？

- **内存淘汰策略（Eviction Policies）**：
- 应该配置 `volatile-lru`、`allkeys-lru`，还是 `volatile-lfu` / `allkeys-lfu`？
- **LRU（最近最少使用） vs LFU（最不经常使用）**：为什么 Redis 4.0 之后更推荐 LFU？（因为 LRU 无法识别“偶发性批量扫描”导致的缓存污染）。

- **内存碎片（Memory Fragmentation）**：
- 频繁的 `SET` / `DEL` 会导致内存碎片率（`mem_fragmentation_ratio`）飙升，导致物理内存占用高但有效数据少。如何配置 Redis 的 `activedefrag` 进行动态内存碎片整理？

### 5. 缓存穿透的工业级防线：布隆过滤器（Bloom Filter）与 Cuckoo Filter

之前讨论过，面对故意攻击的不存在 Key，除了缓存空值（`nil`），最硬核的防御是**布隆过滤器**。

- **核心机理**：用极小的内存空间（Bit Array + 多个 Hash 函数），判断“某个 Key **一定不存在** 或 **可能存在**”。
- **工业落地方式**：
- **RedisBloom 模块**：直接在 Redis 端加载 `BF.ADD` / `BF.EXISTS` 插件。
- **Client 侧本地布隆过滤器**：在 Go 进程内部维护 Bloom Filter，写 DB 时同步更新 Filter。

- **局限性与演进**：标准布隆过滤器**无法删除元素**。如果 DB 里删除了数据，Bloom Filter 无法同步移除。解决这个问题需要使用**计数布隆过滤器（Counting Bloom Filter）** 或 **布谷鸟过滤器（Cuckoo Filter）**。

### 总结建议

你目前已经把 **“缓存穿透/击穿/雪崩”、“缓存读写模式”、“一致性与并发反刷/SingleFlight”** 这三大板块咬得很透了。

如果还想拓展视野，下一步最值得深入研究的就是 **“热点 Key（Hot Key）的本地二级缓存方案”** 以及 **“Big Key 对 Redis 单线程的影响与拆分实战”**。这两个也是工业界日常排查和架构改造中最常遇到的“硬骨头”。
