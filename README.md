# Trading Matcher

## 项目简介

本项目实现了一个面向高频交易场景的 **实时行情处理与订单撮合引擎**。核心代码使用现代 C++（C++20）编写，重点关注低延迟、缓存友好、固定点价格表示、无锁队列、内存池和 Linux 平台下的系统级优化。

项目主要目标：

- **低延迟撮合**：订单处理路径尽量减少动态分配和系统调用。
- **固定点价格**：使用整数表示价格，避免浮点误差。
- **缓存友好结构**：`Order`、队列和内存池使用对齐优化。
- **热路径零分配**：通过预分配内存池减少运行时堆分配。
- **异步日志**：日志写入放到后台线程，避免阻塞撮合路径。
- **Linux 优化**：包含 CPU 亲和性、NUMA、内存锁定、实时调度、epoll 等优化接口。

> 注意：项目包含 Linux 专用优化代码。核心撮合逻辑可以在 macOS 上阅读和部分构建，但 `epoll`、NUMA、CPU affinity 等能力需要 Linux 环境。

---

## 核心功能

### 订单与撮合

- `Order` 使用 32 字节对齐结构，字段紧凑，适合缓存访问。
- 价格使用 `int64_t` 固定点表示，默认精度为 `1e6`。
- `OrderBook` 使用价格档位结构维护买卖盘。
- `Matcher` 基于价格优先、时间优先原则执行撮合。
- 成交后更新订单数量，并移除完全成交的订单。

### 性能组件

- `MemoryPool`：预分配 `Order` 对象，减少热路径动态分配。
- `SPSCQueue`：单生产者单消费者无锁环形队列。
- `AsyncLogger`：异步日志器，使用 SPSC 队列将日志交给后台线程写入。
- `Timestamp`：提供纳秒级时间戳和 RDTSC 计时工具。
- `Benchmark`：提供基础延迟统计，包括 P50、P90、P95、P99、P99.9。

### 网络模块

- `EpollServer`：Linux 下基于 epoll 的高性能 TCP server。
- `UDPServer`：面向低延迟 UDP 场景的网络接口。
- `Client` / `Server`：基础网络客户端和服务端示例。

---

## 目录结构

```bash
.
├── CMakeLists.txt
├── include/
│   ├── order_matching/
│   ├── market_data/
│   ├── networking/
│   └── utils/
├── src/
│   ├── order_matching/
│   ├── market_data/
│   └── networking/
├── HFT_OPTIMIZATIONS.md
├── SYSTEM_ARCHITECTURE.md
└── README.md
```

主要目录说明：

- `include/order_matching/`：订单、订单簿、撮合器等核心数据结构。
- `include/utils/`：内存池、无锁队列、异步日志、时间戳、benchmark 工具。
- `include/networking/`：TCP/UDP 网络相关封装。
- `src/`：示例程序和部分模块实现。

---

## 构建方式

### Linux

推荐使用 Linux 构建和运行完整功能：

```bash
git clone git@github.com:qujianbo/trading-matcher.git
cd trading-matcher
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/bin/MarketDataEngine
```

如果需要启用部分系统级 HFT 优化，可能需要 root 权限或额外系统配置，例如内存锁定、实时调度、huge pages、CPU 亲和性等。

### macOS

macOS 没有 Linux 的 `epoll` 和部分系统 API，因此 Linux 专用网络优化不会启用。当前环境下如果 C++ 标准库路径没有被自动识别，可以显式指定：

```bash
cmake -S . -B build -G "Unix Makefiles" \
  -DCMAKE_CXX_FLAGS="-isystem /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1"
cmake --build build
./build/bin/MarketDataEngine
```

运行后日志默认写入：

```bash
market_engine.log
```

可以查看：

```bash
tail -f market_engine.log
```

---

## 示例流程

`src/main.cpp` 演示了一个简单撮合流程：

1. 初始化 RDTSC 校准器。
2. 启动异步日志器。
3. 创建 `OrderBook` 和 `Matcher`。
4. 使用内存池分配订单。
5. 加入买单和卖单。
6. 执行撮合。
7. 输出成交结果和订单簿状态。
8. 演示 SPSC 队列在线程间传递订单。

---

## 设计重点

### 固定点价格

价格类型定义为：

```cpp
using Price = int64_t;
constexpr Price PRICE_SCALE = 1'000'000LL;
```

例如 `100.50` 会被转换成 `100500000`，避免浮点数在金融计算中的精度问题。

### 订单结构对齐

`Order` 使用：

```cpp
struct alignas(32) Order
```

目标是让订单对象保持 32 字节大小，两个订单可以较好地落在一个常见的 64 字节 cache line 中，提高缓存利用率。

### 异步日志

日志调用不会直接写文件，而是将 `LogMessage` 推入无锁 SPSC 队列，由后台线程负责格式化和输出，减少热路径阻塞。

### 头文件内联

`Matcher` 和 `OrderBook` 的主要逻辑放在头文件中，目的是让编译器在撮合热路径上更容易进行内联和跨函数优化。

---

## 相关文档

- [HFT_OPTIMIZATIONS.md](HFT_OPTIMIZATIONS.md)：HFT 优化说明。
- [SYSTEM_ARCHITECTURE.md](SYSTEM_ARCHITECTURE.md)：系统架构说明。

---

## 后续改进方向

- 完善订单生命周期管理。
- 增加账户、持仓、成交回报和 P&L 模块。
- 为核心撮合逻辑补充单元测试。
- 改进 `OrderBook` 数据结构以减少 `std::map` 带来的分配成本。
- 将异步日志格式化从 `ostringstream` 改成固定缓冲区。
- 增加 Linux 环境下的 benchmark 和延迟统计报告。
- 补充 CI 构建流程。
