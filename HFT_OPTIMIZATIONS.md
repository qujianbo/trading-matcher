# HFT 优化说明

本文档说明本项目中使用的高频交易（HFT）相关优化思路。项目目标不是实现完整交易所系统，而是通过一个可运行的订单簿与撮合示例，展示低延迟 C++ 系统中常见的数据结构、内存管理、计时、日志和 Linux 系统优化方法。

## 总览

高频交易系统对延迟和抖动都很敏感。一次订单处理可能只允许微秒级甚至更低的预算，因此代码设计需要尽量减少：

- 动态内存分配
- 锁竞争
- 不必要的系统调用
- cache miss
- 分支预测失败
- 线程迁移
- 日志和 I/O 对热路径的阻塞

本项目围绕这些目标做了一组基础优化。

---

## 1. 固定点价格表示

### 问题

金融系统中不适合直接使用浮点数表示价格。浮点数可能产生舍入误差，而且在部分场景中会带来不确定性。

### 做法

项目中价格使用 `int64_t` 表示，并按 `1,000,000` 放大：

```cpp
using Price = int64_t;
constexpr Price PRICE_SCALE = 1'000'000LL;
```

例如：

```text
100.50 -> 100500000
```

### 好处

- 避免浮点精度问题。
- 整数比较和计算更确定。
- 适合价格优先、时间优先的撮合逻辑。

---

## 2. 缓存友好的数据结构

### Order 结构

`Order` 被设计为 32 字节：

```cpp
struct alignas(32) Order
```

并通过静态断言固定大小和对齐：

```cpp
static_assert(sizeof(Order) == 32, "Order must be exactly 32 bytes");
static_assert(alignof(Order) == 32, "Order must be 32-byte aligned");
```

常见 CPU cache line 是 64 字节，因此 32 字节的订单结构可以让两个订单较好地落在一条 cache line 中，提高缓存利用率。这里的目标不是让每个订单独占一条 cache line，而是在对象紧凑和访问效率之间做折中。

### Price Ladder

订单簿中使用价格档位管理订单：

- 买盘按价格从低到高存储，最高买价通过 `rbegin()` 获取。
- 卖盘按价格从低到高存储，最低卖价通过 `begin()` 获取。
- 每个价格档位内部用 FIFO 顺序维护订单。

当前实现使用 `std::map<Price, PriceLevel>`。它不是极致低延迟的数据结构，但结构清晰，适合展示价格档位和最优价查询逻辑。

---

## 3. 内存管理

### Memory Pool

项目提供了 `OrderMemoryPool`：

```cpp
template<size_t PoolSize = 1024 * 1024>
class OrderMemoryPool
```

它会预先分配一批 `Order` 对象，避免热路径中频繁调用 `new` / `delete`。

主要特点：

- 默认预分配约 100 万个订单对象。
- 使用原子递增索引分配对象。
- 池内对象按数组连续存储，缓存局部性较好。
- 池耗尽时才退回到堆分配。

### 内存锁定

Linux 下提供：

```cpp
mlockall(MCL_CURRENT | MCL_FUTURE)
```

作用是尽量防止进程内存被换出到磁盘。对低延迟系统来说，swap 会造成不可接受的延迟抖动。

---

## 4. Linux 系统级优化

这些能力主要在 `LinuxOptimizations.hpp` 中实现，只在 Linux 下启用。

### CPU 亲和性与线程绑定

```cpp
hft::setCpuAffinity(cpu_id);
hft::pinThreadToCpu(cpu_id);
hft::pinThreadToCpu(thread, cpu_id);
```

作用：

- 减少线程在不同 CPU 核之间迁移。
- 提高 cache locality。
- 降低调度带来的延迟抖动。

### NUMA 感知

```cpp
int numa_node = hft::getNumaNode(cpu_id);
hft::setMemoryPolicy(numa_node);
hft::bindThreadToNumaNode(numa_node);
void* ptr = hft::allocateOnNumaNode(size, numa_node);
```

在多路 CPU 服务器上，不同 CPU 访问不同 NUMA 节点的内存延迟不同。NUMA 优化的目标是让线程尽量访问本地节点内存。

项目中如果检测到 `libnuma`，会使用 NUMA API；否则退化为普通分配。

### 实时调度

```cpp
hft::setRealtimePriority(50);
```

底层使用 `SCHED_FIFO`。它可以减少普通进程对交易线程的抢占，但通常需要 root 权限。生产环境中必须谨慎使用，避免高优先级线程长期占用 CPU。

### Huge Pages

普通页通常是 4KB，huge page 常见大小是 2MB。使用 huge pages 可以减少 TLB miss，适合大块内存池或高频访问的大数组。

---

## 5. 高精度时间戳

### RDTSC

RDTSC 是 x86 CPU 的时间戳计数器读取指令，速度很快，适合测量短路径延迟。

项目提供：

```cpp
hft::rdtsc();
hft::rdtscp();
hft::LatencyTimerRDTSC;
```

`RDTSCCalibrator` 会在启动时估算 cycles 和 nanoseconds 之间的换算关系：

```cpp
hft::RDTSCCalibrator::calibrate();
```

### clock_gettime

项目也提供基于 `clock_gettime` 的时间函数：

```cpp
hft::getTimestampNs();
hft::getMonotonicNs();
hft::LatencyTimer;
```

一般区分：

- 订单时间戳：使用 wall-clock 时间。
- 延迟测量：使用 monotonic 时间或 RDTSC。

### 示例

```cpp
hft::LatencyTimerRDTSC timer;

// 执行待测逻辑

int64_t latency_us = timer.elapsedUs();
uint64_t cycles = timer.elapsedCycles();
```

---

## 6. 无锁 SPSC 队列

SPSC 是 Single Producer Single Consumer，即单生产者、单消费者队列。

项目中的 `SPSCQueue<T, Size>` 使用固定大小环形缓冲区：

```cpp
template<typename T, size_t Size>
class SPSCQueue
```

特点：

- 单生产者单消费者场景下无需互斥锁。
- 使用 `std::atomic` 管理读写位置。
- 队列容量在编译期确定。
- 使用 cache line 对齐减少 false sharing。
- push/pop 都是 O(1)。

适用场景：

- 行情线程到撮合线程的数据传递。
- 异步日志消息传递。
- 延迟统计采样传递。

限制：

- 只适合单生产者、单消费者。
- 多生产者或多消费者场景需要 MPSC/MPMC 队列。

---

## 7. 异步日志

日志如果直接写文件，会引入 I/O 阻塞，影响热路径延迟。

项目中的 `AsyncLogger` 做法是：

1. 业务线程构造 `LogMessage`。
2. 将日志消息推入 SPSC 队列。
3. 后台日志线程从队列取出消息。
4. 后台线程负责格式化和写文件。

示例：

```cpp
LOG_INFO("order accepted");
LOG_INFO_F("order id=%u price=%ld", order_id, price);
```

优点：

- 业务线程不直接做文件 I/O。
- 日志消息使用固定大小 buffer。
- 队列满时可以选择丢弃日志，避免阻塞热路径。

当前可改进点：

- `writeLogMessage()` 中仍使用 `std::ostringstream`。
- 时间格式化返回 `std::string`。
- 如果追求更低延迟，可以改为固定栈缓冲区和 `snprintf`。

---

## 8. 网络优化

### epoll TCP Server

Linux 下的 `EpollServer` 使用：

- `epoll_create1`
- `epoll_ctl`
- `epoll_wait`
- 非阻塞 socket
- edge-triggered 模式
- `TCP_NODELAY`
- `SO_REUSEADDR`
- `SO_REUSEPORT`

它适合高并发连接和事件驱动网络处理。`TCP_NODELAY` 用于关闭 Nagle 算法，减少小包等待。

### UDP Busy Polling

项目中的 UDP 模块包含 busy polling 思路。busy poll 可以让内核在短时间内轮询 socket 队列，减少睡眠/唤醒带来的延迟。

代价是 CPU 占用更高，通常需要绑定独立 CPU 核。

### 未来网络方向

- DPDK：绕过内核网络栈。
- XDP：在内核早期路径处理网络包。
- zero-copy：减少数据拷贝。
- 硬件时间戳：提高行情与交易事件时间精度。

---

## 9. 编译器优化

Release 构建中会启用一组优化参数：

```cmake
-O3
-march=native
-mtune=native
-flto
-ffast-math
-funroll-loops
-fno-exceptions
-fno-rtti
-finline-functions
-fomit-frame-pointer
```

说明：

- `-O3`：启用较激进优化。
- `-march=native`：针对当前 CPU 指令集优化。
- `-flto`：链接时优化，允许跨翻译单元优化。
- `-fno-exceptions`：关闭异常支持，减少运行时开销。
- `-fno-rtti`：关闭 RTTI。
- `-fomit-frame-pointer`：释放寄存器，但可能影响调试和 profiling。

这些选项适合性能实验，但生产环境需要结合调试、可观测性和安全要求权衡。

---

## 10. 热路径头文件内联

项目中一些关键逻辑放在头文件里，例如：

- `Order` 内联访问函数。
- `OrderBook` 主要操作。
- `Matcher` 撮合逻辑。

这样做的原因是：当调用点能看到函数实现时，编译器更容易内联并进一步优化。

缺点是：

- 编译时间可能增加。
- 实现细节暴露在头文件中。
- 头文件改动会触发更多文件重新编译。

---

## 延迟目标

| 操作 | 目标 | 指标 |
| --- | --- | --- |
| 添加订单 | < 1 μs | P99 |
| 撤销订单 | < 1 μs | P99 |
| 撮合订单 | < 2 μs | P99 |
| 最优价查询 | < 100 ns | P99 |

这些目标用于指导优化方向，不代表当前代码在所有环境下都能达到。真实结果取决于硬件、编译器、系统配置、数据规模和 benchmark 方法。

---

## 性能监控

可以使用 `LatencyTimer` 和 `Benchmark` 统计延迟：

```cpp
hft::LatencyTimer timer;

// 执行待测逻辑

int64_t latency_us = timer.elapsedUs();
```

`Benchmark` 支持输出：

- count
- min
- max
- mean
- median
- P50
- P90
- P95
- P99
- P99.9

---

## Linux 系统配置建议

### Huge Pages

```bash
echo 1024 > /proc/sys/vm/nr_hugepages
```

### CPU 隔离

可以通过内核参数隔离交易线程使用的 CPU：

```bash
isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3
```

### CPU 频率策略

将 CPU governor 设置为 performance：

```bash
echo performance > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

### 网络缓冲区

```bash
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
```

### root 权限

以下能力通常需要 root 或额外 capability：

- `mlockall`
- `SCHED_FIFO`
- 某些 socket busy poll 设置
- huge page 配置

---

## 实践建议

1. 先测量，再优化。
2. 优先关注 P99 / P99.9，而不是只看平均值。
3. 热路径避免动态分配。
4. 热路径避免锁和阻塞 I/O。
5. 数据结构要考虑 cache locality。
6. 用 `[[likely]]` / `[[unlikely]]` 标注高频分支，但不要滥用。
7. benchmark 要固定 CPU、固定频率，并减少后台干扰。
8. 日志、监控、统计都应避免阻塞撮合路径。

---

## 后续优化方向

- [ ] 实现更低分配成本的价格档位结构。
- [ ] 为 `OrderBook` 和 `Matcher` 增加单元测试。
- [ ] 为撮合路径增加系统化 benchmark。
- [ ] 将异步日志格式化改成固定缓冲区。
- [ ] 引入批量订单处理。
- [ ] 增加持仓、成交回报、P&L 模块。
- [ ] 增加 Linux CI 构建。
- [ ] 探索 DPDK/XDP 网络路径。
- [ ] 增加硬件时间戳或 PTP 支持。

---

## 参考方向

- 低延迟 C++ 编程
- Linux 性能调优
- CPU cache 与 false sharing
- lock-free 数据结构
- exchange matching engine 设计
- market data feed handler 设计
