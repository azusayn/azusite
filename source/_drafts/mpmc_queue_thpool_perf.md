## 开始一些硬核的性能测试

## 3. 性能测试（mutex vs atomic 线程池）

环境：macOS Darwin arm64（Apple Silicon），Release 构建。

Workload（突出队列锁竞争，而非算力）：

- 固定 16 workers
- 外部生产者并发 `Submit`（见下表）
- 极轻任务：`atomic` 计数 +1
- 计时范围：全部 `Submit` 完成 + `Wait` 结束
- atomic 池队列容量 `1<<20`，减少「队列满重试」干扰

| producers | tasks | mutex wall | mutex throughput | atomic wall | atomic throughput | 比值 (mutex/atomic) |
| ----------- | ------: | -----------: | -----------------: | ------------: | ------------------: | --------------------: |
| 8 | 2,000,000 | 984ms | ~2.03M tasks/s | 684ms | ~2.92M tasks/s | ~1.44x |
| 16 | 5,000,000 | 2365ms | ~2.11M tasks/s | 1688ms | ~2.96M tasks/s | ~1.40x |
| 32 | 5,000,000 | 2391ms | ~2.09M tasks/s | 1677ms | ~2.98M tasks/s | ~1.43x |

约 **1.4x**：mutex 更慢，但不是数量级差距。短临界区 + 两边都有调度/唤醒开销时，这很常见；不要指望端到端线程池测试打出 10x。

生产者从 8 提到 32，吞吐和比值几乎不变，说明入队侧竞争已饱和，再堆生产者不会线性拉开差距。

## 4. 性能测试（仅队列：TaskQueue vs LockFreeQueue，同构 function）

两边 payload 都是 `std::function<void()>`。环境：macOS Darwin arm64，Release。

各阶段独立程序：

| 阶段 | TaskQueue | LockFreeQueue |
| ------ | ----------- | --------------- |
| 单线程基线 | `perf/perf_mutex_queue_st.cc` | `perf/perf_lock_free_queue_st.cc` |
| 多线程 + 生产者扩展曲线 | `perf/perf_mutex_queue_mt.cc` | `perf/perf_lock_free_queue_mt.cc` |

### 4.1 单线程基线

每轮：`Push` 一个 task → 立刻 `Pop` → 执行（避免有界无锁队列装不下）。`kTotal = 5,000,000`。

| 队列 | wall | throughput |
| ------ | ------: | -----------: |
| `TaskQueue`（mutex 链表） | 159ms | ~31.4M ops/s |
| `LockFreeQueue<Task>` | 31ms | ~161M ops/s |

单线程下无锁约 **5x** 更快：没有多线程 CAS 互殴，且避开了 mutex 链表每次 `new`/`delete` 节点的固定成本（相对无锁环槽位复用）。

### 4.2 多线程 + 生产者扩展曲线

固定 16 consumers，`kTotal = 5,000,000`，消费者忙等 `Pop`，payload 同为 `std::function`。  
同一二进制内扫 `producers ∈ {1,2,4,8,16,32,48,64}`。

| producers | TaskQueue wall | TaskQueue thrput | LockFree wall | LockFree thrput |
| ----------: | ---------------: | -----------------: | --------------: | ----------------: |
| 1 | 7115ms | ~0.70M ops/s | 711ms | ~7.03M ops/s |
| 2 | 4165ms | ~1.20M ops/s | 518ms | ~9.65M ops/s |
| 4 | 1571ms | ~3.18M ops/s | 960ms | ~5.21M ops/s |
| 8 | 965ms | ~5.18M ops/s | 1042ms | ~4.80M ops/s |
| 16 | 662ms | ~7.55M ops/s | 1020ms | ~4.90M ops/s |
| 32 | 642ms | ~7.79M ops/s | 851ms | ~5.88M ops/s |
| 48 | 683ms | ~7.32M ops/s | 1049ms | ~4.77M ops/s |
| 64 | 773ms | ~6.47M ops/s | 1184ms | ~4.22M ops/s |

读法：

- **低生产者（1–4）**：LockFree 明显更快。
- **中高生产者（8–32）**：TaskQueue 更快或接近；约在 16–32 达到自身最好吞吐。
- **48–64**：两边都开始变差（过度订阅）；TaskQueue 仍略好于 LockFree。

原理（不完全是「无竞争就走几个原子」这么简单，但有关）：

1. **无竞争 mutex 的 fast path** 确实多半是用户态原子（CAS）改锁字，不进内核；这点和「轻量原子操作」同类。但 TaskQueue 每次还有链表 `new`/`delete`，单线程就已经比环缓冲槽复用贵（见 §4.1）。
2. **本测试消费者固定 16 且空队列时忙等**：TaskQueue 的空 `Pop` 仍要 `lock/unlock`，1 个生产者时队列经常偏空，16 个消费者轮流抢同一把锁 → 巨慢（7115ms）。LockFree 空 `Pop` 主要是读 `sequence`，空转便宜得多。
3. **生产者变多**后队列更不容易被抽干，空转抢锁变少；这时 LockFree 要付的是多线程 CAS / 缓存行来回打的成本，TaskQueue 则把入出队串行化，空转更少，所以中段反超。
4. **48–64** 线程过多，调度与伪共享变差，两边吞吐都掉。
