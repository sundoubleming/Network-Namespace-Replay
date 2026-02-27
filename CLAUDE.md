# Network Namespace Behavior Tracker - 开发文档

## 项目概述

这是一个基于C语言实现的**插件化网络监控框架**，用于在指定时间内（或持续运行）监控Linux network namespace中的网络状态变化，并提供完整的时间序列记录以支持网络状态复现。

### 核心特性

1. **统一时间轴**：所有事件使用纳秒级时间戳，保证时间顺序（同一纳秒内的事件视为同时发生）
2. **插件化架构**：支持动态加载/卸载插件（.so文件），易于扩展
3. **事件驱动**：基于回调机制，不使用轮询，保证时间戳准确性
4. **可复现性**：输出完整的时间序列日志，支持网络状态回放和分析
5. **Per-Plugin并发**：每个插件独立的事件队列和处理线程，插件间完全隔离

### 设计目标

- **时间戳灵活性**：插件自主管理时间戳（如libpcap使用内核时间戳），框架提供工具函数
- **零轮询设计**：使用Linux异步通知机制（libpcap回调、netlink事件）
- **热插拔支持**：运行时动态加载/卸载插件
- **配置灵活性**：每个插件有独立的配置文件

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        Main Thread                          │
│  - 初始化框架、加载插件、启动各模块线程                      │
│  - 等待信号(SIGINT/SIGTERM)                                 │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────┐      ┌──────────────┐
│ Capture插件  │    │ Network插件  │      │ Plugin N     │
│   线程       │    │   线程       │      │   线程       │
└──────────────┘    └──────────────┘      └──────────────┘
        │                     │                     │
        │ emit_event()        │ emit_event()        │
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────┐      ┌──────────────┐
│   Queue 1    │    │   Queue 2    │      │   Queue N    │
│ (无锁队列)   │    │ (无锁队列)   │      │ (无锁队列)   │
│ Concurrency  │    │ Concurrency  │      │ Concurrency  │
│    Kit       │    │    Kit       │      │    Kit       │
└──────────────┘    └──────────────┘      └──────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────┐      ┌──────────────┐
│  Worker 1    │    │  Worker 2    │      │  Worker N    │
│   线程       │    │   线程       │      │   线程       │
└──────────────┘    └──────────────┘      └──────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────┐      ┌──────────────┐
│capture.jsonl │    │network.jsonl │      │ plugin_n.jsonl│
│  (文件)      │    │  (文件)      │      │  (文件)      │
└──────────────┘    └──────────────┘      └──────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     Framework Core                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │Plugin Manager│  │  Config Mgr  │  │    Logger    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### 核心模块

#### 1. Plugin Manager（插件管理器）
- 负责插件的加载、卸载、启动、停止
- 使用 `dlopen/dlsym/dlclose` 实现动态加载
- 管理插件生命周期和状态

#### 2. Per-Plugin Worker（每插件独立处理）
- **每个插件独立的事件队列**：使用Concurrency Kit的无锁SPSC队列
- **每个插件专门的Worker线程**：从队列取事件并持久化
- **插件间完全隔离**：某个插件阻塞不影响其他插件
- **无需全局序列号**：同一纳秒内的事件视为同时发生，按到达顺序处理

#### 3. Config Manager（配置管理器）

**设计原则**：
- **统一解析**：框架负责解析所有配置文件（YAML格式）
- **分离配置**：框架配置和插件配置分离，每个插件有独立配置文件
- **类型安全**：插件定义配置结构体，框架自动映射YAML到结构体

**配置文件组织**：
```
/etc/ns-tracker/
├── framework.yaml          # 框架配置
└── plugins/
    ├── capture.yaml        # Capture插件配置
    ├── network.yaml        # Network插件配置
    └── plugin_n.yaml       # 其他插件配置
```

**框架配置内容**：
- 输出目录配置
- 轮转策略配置（时长、大小、事件数）
- 日志配置
- 启用的插件列表

**插件配置内容**：
- 插件基本信息（名称、版本、是否启用）
- 插件特定配置（由插件自己定义结构）

**配置加载流程**：
1. 框架启动时加载framework.yaml
2. 根据启用的插件列表，加载对应的插件配置文件
3. 框架解析YAML并映射到插件定义的配置结构体
4. 验证配置有效性（必需字段、类型检查、自定义验证）
5. 将配置对象传递给插件初始化函数

**配置优先级**（从高到低）：
1. 命令行参数
2. 环境变量（NS_TRACKER_*前缀）
3. 配置文件
4. 代码默认值

**错误处理**：
- 配置文件不存在
- YAML解析失败
- 类型不匹配
- 缺少必需字段
- 自定义验证失败

#### 4. Worker线程（持久化层）
- 每个插件有专门的Worker线程
- 从无锁队列取事件并写入文件（JSON Lines格式）
- **每个插件的数据分开保存**到独立文件
- **统一轮转控制**：所有插件文件同步轮转
- **轮转条件由框架配置**：文件大小、时间间隔、事件数量
- **同步文件切换**：当任一插件触发轮转条件时，所有插件同时切换到新文件
- 生成元数据文件（每个session一个）
- 支持数据压缩

#### 5. Logger（日志系统）
- 统一的日志接口
- 支持多级别日志（DEBUG/INFO/WARN/ERROR）
- 插件日志自动标记来源

#### 6. Common Library（通用库）
- **时间戳工具函数**：格式转换、获取当前时间等
- **无锁队列**：基于Concurrency Kit的SPSC队列封装
- 哈希表
- 其他工具函数

## 核心设计原则

### 1. 时间戳管理

**设计原则**：
- 时间戳尽可能由各插件自己负责获取并上报给框架，这样才能保证准确性
- 如果插件获取不到再由框架补充
- 框架提供 Common Library 工具函数辅助
- **不使用全局序列号**：同一纳秒内的事件视为同时发生，接受按到达顺序处理

### 2. 零轮询设计
- 使用主动通知的形式而不是轮询获取事件，减少时间误差

### 3. Per-Plugin并发模型

**设计思想**：
- 每个插件有独立的事件队列（Concurrency Kit无锁SPSC队列）
- 每个插件有专门的Worker线程处理事件
- 插件之间完全隔离，互不影响

**优势**：
- 某个插件阻塞不会影响其他插件
- 每个插件内的事件严格按时间戳顺序处理
- 无需跨插件同步，设计简单
- 易于扩展（新增插件只需添加新的队列+Worker）
- 高性能：无锁队列减少竞争开销

**队列选择**：
- 使用Concurrency Kit (libck)的无锁SPSC队列
- 单生产者（插件线程）单消费者（Worker线程）模式
- 轻量级依赖：`apt-get install libck-dev`
- 无需DPDK的复杂配置

### 4. 插件热插拔

- 使用 `dlopen/dlsym/dlclose` 动态加载插件
- 插件实现标准接口 `plugin_api_t`
- 运行时可加载/卸载插件

### 4. 持久化策略

**文件组织**：
```
output/
├── session_001/
│   ├── capture.jsonl     # capture插件的事件
│   ├── network.jsonl     # network插件的事件
│   └── metadata.yaml     # 这个session的元数据
├── session_002/
│   ├── capture.jsonl
│   ├── network.jsonl
│   └── metadata.yaml
└── global_metadata.yaml  # 全局元数据
```

**轮转策略**：
- 框架统一控制轮转条件（文件大小、时间、事件数）
- 当任一插件满足轮转条件时，所有插件同步轮转
- 保证同一session内所有文件的时间范围一致
- 每个Worker线程独立写入自己的文件，无锁竞争

## 输出格式

### Session元数据 (metadata.yaml)

```yaml
session: 1
start_time: "2026-01-30T15:04:05.123456789Z"
end_time: "2026-01-30T15:09:05.987654321Z"
duration_seconds: 300.864197532

files:
  - plugin: "capture"
    file: "capture.jsonl"
    size_bytes: 95234567
    event_count: 15234

  - plugin: "network"
    file: "network.jsonl"
    size_bytes: 123456
    event_count: 42

rotation_reason: "max_duration_reached"
```

## 依赖库

- **libck (Concurrency Kit)**：无锁数据结构（SPSC队列）
- **libyaml**：YAML配置解析
- **pthread**：多线程支持
- **libdl**：动态库加载

## 开发计划

### Phase 1: 框架核心（优先级：P0）
- [ ] 基础框架结构
- [ ] 插件管理器
- [ ] Per-Plugin Worker模型（无锁队列 + Worker线程）
- [ ] 配置管理器
- [ ] 日志系统
- [ ] Common Library（时间戳工具函数、无锁队列封装、哈希表）

### Phase 2: 持久化和输出（优先级：P0）
- [ ] 事件持久化（JSON Lines，每插件独立文件）
- [ ] 统一轮转控制
- [ ] 元数据生成

### Phase 3: 高级特性（优先级：P1）
- [ ] 插件热插拔
- [ ] 配置热加载
- [ ] 运行时控制接口
- [ ] 性能优化

### Phase 4: 工具和文档（优先级：P2）
- [ ] 状态查询工具
- [ ] 完整文档
- [ ] 示例插件

## 扩展性

### 如何开发新插件

1. 实现 `plugin_api_t` 接口
2. 导出 `get_plugin_api()` 函数
3. 编译为 `.so` 文件
4. 添加配置文件
5. 在框架配置中注册插件

### 插件示例场景

- **DNS监控插件**：记录DNS查询和响应
- **连接跟踪插件**：记录TCP连接状态变化
- **性能指标插件**：记录CPU、内存、网络带宽
- **eBPF插件**：记录内核级别的网络事件
- **防火墙规则插件**：监控iptables/nftables变化

## 已知限制

1. **跨插件事件顺序**：同一纳秒内来自不同插件的事件顺序不保证，视为同时发生
2. **事件丢失风险**：高负载下事件队列可能溢出
3. **单机部署**：当前不支持分布式部署
4. **时间戳由插件管理**：框架不强制统一时间源，插件需自行保证时间戳的准确性

## 参考资料

- [libpcap文档](https://www.tcpdump.org/manpages/pcap.3pcap.html)
- [Netlink协议](https://man7.org/linux/man-pages/man7/netlink.7.html)
- [rtnetlink文档](https://man7.org/linux/man-pages/man7/rtnetlink.7.html)
- [dlopen手册](https://man7.org/linux/man-pages/man3/dlopen.3.html)
- [Concurrency Kit文档](http://concurrencykit.org/)

---

**文档版本**：v0.4
**最后更新**：2026-02-07
**状态**：设计阶段

## Capture插件架构设计（XDP + AF_XDP方案）

### 设计概述

Capture插件采用**XDP + AF_XDP组合方案**实现高性能旁路抓包，在不影响业务流量的前提下，提供极致的抓包性能和纳秒级时间戳精度。

**核心特性**：
- **技术栈**：XDP (eXpress Data Path) + AF_XDP (Address Family XDP)
- **部署环境**：混合环境（物理网卡 + 虚拟网卡自动适配）
- **XDP模式**：Native模式（物理网卡）/ Generic模式（虚拟网卡）自动检测
- **处理策略**：XDP早期过滤 + 旁路抓包（所有包继续进入协议栈）
- **时间戳**：内核时间戳（纳秒级精度）
- **队列模式**：每网卡多队列并行处理（充分利用多核）
- **性能目标**：Native模式单核20-40 Mpps，支持10Gbps+网络

### 架构图

```
═══════════════════════════════════════════════════════════════════════════════
                              内核空间 (Kernel Space)
═══════════════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────────────────┐
│                         Network Interface (eth0)                            │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Queue 0     │  │  Queue 1     │  │  Queue 2     │  │  Queue 3     │  │
│  │  (RSS分流)   │  │  (RSS分流)   │  │  (RSS分流)   │  │  (RSS分流)   │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼──────────────────┼──────────────────┼──────────────────┼──────────┘
          │                  │                  │                  │
          │ [数据包到达]     │ [数据包到达]     │ [数据包到达]     │ [数据包到达]
          ▼                  ▼                  ▼                  ▼
    ┌──────────┐       ┌──────────┐       ┌──────────┐       ┌──────────┐
    │   XDP    │       │   XDP    │       │   XDP    │       │   XDP    │
    │ Program  │       │ Program  │       │ Program  │       │ Program  │
    │ Queue 0  │       │ Queue 1  │       │ Queue 2  │       │ Queue 3  │
    └────┬─────┘       └────┬─────┘       └────┬─────┘       └────┬─────┘
         │                  │                  │                  │
         │ 过滤判断         │ 过滤判断         │ 过滤判断         │ 过滤判断
         │ (BPF过滤器)      │ (BPF过滤器)      │ (BPF过滤器)      │ (BPF过滤器)
         │                  │                  │                  │
    匹配? │             匹配? │             匹配? │             匹配? │
    ┌────┴─────┐        ┌────┴─────┐        ┌────┴─────┐        ┌────┴─────┐
    │YES: 重定向│        │YES: 重定向│        │YES: 重定向│        │YES: 重定向│
    │NO:  PASS │        │NO:  PASS │        │NO:  PASS │        │NO:  PASS │
    └────┬─────┘        └────┬─────┘        └────┬─────┘        └────┬─────┘
         │                  │                  │                  │
         │ XDP_REDIRECT     │ XDP_REDIRECT     │ XDP_REDIRECT     │ XDP_REDIRECT
         │ 到AF_XDP         │ 到AF_XDP         │ 到AF_XDP         │ 到AF_XDP
         ▼                  ▼                  ▼                  ▼
    ┌──────────┐       ┌──────────┐       ┌──────────┐       ┌──────────┐
    │ AF_XDP   │       │ AF_XDP   │       │ AF_XDP   │       │ AF_XDP   │
    │ Socket 0 │       │ Socket 1 │       │ Socket 2 │       │ Socket 3 │
    │          │       │          │       │          │       │          │
    │  UMEM    │       │  UMEM    │       │  UMEM    │       │  UMEM    │
    │ RX Ring  │       │ RX Ring  │       │ RX Ring  │       │ RX Ring  │
    │Fill Ring │       │Fill Ring │       │Fill Ring │       │Fill Ring │
    └────┬─────┘       └────┬─────┘       └────┬─────┘       └────┬─────┘
         │                  │                  │                  │
         │ [零拷贝传输]     │ [零拷贝传输]     │ [零拷贝传输]     │ [零拷贝传输]
         │ + 自动PASS       │ + 自动PASS       │ + 自动PASS       │ + 自动PASS
         │ 到协议栈         │ 到协议栈         │ 到协议栈         │ 到协议栈
         │                  │                  │                  │
         ▼                  ▼                  ▼                  ▼
    ┌──────────────────────────────────────────────────────────────────────┐
    │                    Linux Network Stack                               │
    │                    (TCP/IP协议栈正常处理)                            │
    └──────────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════
                        内核-用户空间边界 (Kernel-User Boundary)
═══════════════════════════════════════════════════════════════════════════════

         │                  │                  │                  │
         │ mmap共享内存     │ mmap共享内存     │ mmap共享内存     │ mmap共享内存
         ▼                  ▼                  ▼                  ▼

═══════════════════════════════════════════════════════════════════════════════
                              用户空间 (User Space)
═══════════════════════════════════════════════════════════════════════════════

    ┌──────────┐       ┌──────────┐       ┌──────────┐       ┌──────────┐
    │ Plugin   │       │ Plugin   │       │ Plugin   │       │ Plugin   │
    │ Thread 0 │       │ Thread 1 │       │ Thread 2 │       │ Thread 3 │
    │          │       │          │       │          │       │          │
    │ 从AF_XDP │       │ 从AF_XDP │       │ 从AF_XDP │       │ 从AF_XDP │
    │ 读取包   │       │ 读取包   │       │ 读取包   │       │ 读取包   │
    │          │       │          │       │          │       │          │
    │ 获取内核 │       │ 获取内核 │       │ 获取内核 │       │ 获取内核 │
    │ 时间戳   │       │ 时间戳   │       │ 时间戳   │       │ 时间戳   │
    │          │       │          │       │          │       │          │
    │ 构造事件 │       │ 构造事件 │       │ 构造事件 │       │ 构造事件 │
    └────┬─────┘       └────┬─────┘       └────┬─────┘       └────┬─────┘
         │                  │                  │                  │
         │ emit_event()     │ emit_event()     │ emit_event()     │ emit_event()
         │                  │                  │                  │
         └──────────────────┴──────────────────┴──────────────────┘
                                      │
                                      ▼
         ┌────────────────────────────────────────────────────────────────┐
         │          Concurrency Kit SPSC Queue (无锁队列)                 │
         │              (Per-Plugin Event Queue)                          │
         └────────────────────────────┬───────────────────────────────────┘
                                      │
                                      │ dequeue()
                                      ▼
                                ┌──────────┐
                                │  Worker  │
                                │  Thread  │
                                └────┬─────┘
                                     │
                                     │ write()
                                     ▼
                                ┌──────────┐
                                │ capture. │
                                │  pcap    │
                                │  (文件)  │
                                └──────────┘
```

### 核心设计要点

#### 1. XDP程序（eBPF）
- 在网卡驱动层对数据包进行早期过滤
- 根据BPF过滤规则决定是否重定向到AF_XDP
- 使用`BPF_MAP_TYPE_XSKMAP`存储AF_XDP socket映射
- 支持动态更新过滤规则（通过BPF Map）
- 匹配的包：`XDP_REDIRECT`到AF_XDP
- 不匹配的包：`XDP_PASS`直接到协议栈

#### 2. AF_XDP Socket管理
- 每个网卡队列创建独立的AF_XDP socket
- UMEM配置：Frame大小2048/4096字节，Frame数量4096-8192
- 支持Copy模式（兼容性）和Zero-copy模式（性能）
- 通过mmap实现内核-用户空间零拷贝

#### 3. 插件线程模型
- 每队列一个插件线程，绑定到专用CPU核心
- 从AF_XDP批量读取数据包（批量大小32-64）
- 获取内核时间戳（通过XDP metadata或降级到用户空间）
- 构造事件并通过`emit_event()`发送到框架队列
- Frame循环使用（返回Fill Queue）

#### 4. 旁路抓包机制
- AF_XDP的Copy模式下，包经过用户空间后自动PASS到协议栈
- 所有包（无论是否匹配过滤器）最终都进入内核协议栈
- 业务流量不受影响，插件仅作为旁路监听

#### 5. 环境自适应
- 自动检测网卡是否支持Native XDP（物理网卡）
- 不支持则降级到Generic XDP（虚拟网卡veth/bridge）
- 自动检测网卡队列数量，优化线程配置
- 虚拟网卡使用单队列模式，物理网卡使用多队列模式

### 性能特性

| 模式 | 网卡类型 | 单核性能 | 多核性能(8核) | CPU占用 | 丢包率 |
|------|---------|---------|--------------|---------|--------|
| Native + Zero-copy | 物理网卡 | 20-40 Mpps | 160-320 Mpps | 20-30% | <0.01% |
| Native + Copy | 物理网卡 | 10-20 Mpps | 80-160 Mpps | 30-40% | <0.1% |
| Generic | 虚拟网卡 | 3-8 Mpps | 24-64 Mpps | 50-70% | <0.5% |

**延迟特性**：
- 端到端延迟（Native模式）：4-7 μs
- 时间戳精度：纳秒级（内核时间戳）
- 队列延迟：<1 μs（无锁队列）

### 技术挑战

1. **旁路抓包实现**：使用AF_XDP的Copy模式，包会自动PASS到协议栈
2. **内核时间戳获取**：使用XDP metadata功能（内核5.18+），降级方案为用户空间时间戳
3. **虚拟网卡性能**：接受Generic模式的性能限制，仍优于传统libpcap
4. **eBPF复杂度**：使用libbpf简化开发，提供预编译的XDP程序

### 参考文档

- [AF_XDP性能指南](docs/reference/af_xdp_guide.md)
- [XDP编程指南](docs/reference/xdp_guide.md)
- [Linux Kernel XDP Documentation](https://www.kernel.org/doc/html/latest/networking/af_xdp.html)

---

## 持久层设计方案

### 设计原则

持久层采用**插件自主持久化 + 框架统一轮转**的设计：

1. **插件自主性**：每个插件完全控制自己的输出格式（PCAP、JSON、二进制等）
2. **框架协调性**：框架统一控制轮转时机，保证所有插件的session边界一致
3. **职责分离**：Worker线程只负责从队列取事件并调用插件接口，不关心具体格式
4. **统一时间轴**：通过同步轮转，保证所有插件的数据可以按时间关联分析

### 架构层次

```
┌─────────────────────────────────────────────────────────┐
│                    Framework Core                       │
│  ┌──────────────────────────────────────────┐          │
│  │     Rotation Controller (轮转控制器)     │          │
│  │  - 监控全局轮转条件                      │          │
│  │  - 通知所有插件同步轮转                  │          │
│  │  - 管理session ID                        │          │
│  └──────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
                         │
                         │ 轮转通知
                         ▼
┌─────────────────────────────────────────────────────────┐
│                  Worker Threads                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │Worker 1  │  │Worker 2  │  │Worker N  │            │
│  │          │  │          │  │          │            │
│  │从队列取  │  │从队列取  │  │从队列取  │            │
│  │事件      │  │事件      │  │事件      │            │
│  │↓         │  │↓         │  │↓         │            │
│  │调用插件  │  │调用插件  │  │调用插件  │            │
│  │接口      │  │接口      │  │接口      │            │
│  └──────────┘  └──────────┘  └──────────┘            │
└─────────────────────────────────────────────────────────┘
                         │
                         │ 调用持久化接口
                         ▼
┌─────────────────────────────────────────────────────────┐
│              Plugin Persistence Layer                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │Capture   │  │Network   │  │Plugin N  │            │
│  │Plugin    │  │Plugin    │  │          │            │
│  │          │  │          │  │          │            │
│  │写PCAP    │  │写JSON    │  │写自定义  │            │
│  │格式      │  │格式      │  │格式      │            │
│  └──────────┘  └──────────┘  └──────────┘            │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   File System                           │
│  session_001/                                           │
│    ├── capture.pcap                                     │
│    ├── network.jsonl                                    │
│    ├── plugin_n.dat                                     │
│    └── metadata.yaml                                    │
└─────────────────────────────────────────────────────────┘
```

### 核心概念

- **Session**：一个轮转周期，所有插件在同一session内的数据时间范围一致
- **Rotation Controller**：框架的轮转控制器，负责决定何时轮转
- **Plugin Persistence API**：插件实现的持久化接口，框架通过这些接口调用插件

### 插件持久化接口

插件需要实现以下最小接口集：

```c
typedef struct {
    // 初始化持久化（打开第一个文件）
    // output_dir: 输出目录路径
    // session_id: 当前session ID
    int (*init_persistence)(void *plugin_ctx, const char *output_dir, int session_id);

    // 处理事件（从队列取出后调用，插件负责写入）
    // event: 事件对象
    int (*process_event)(void *plugin_ctx, event_t *event);

    // 轮转通知（框架通知插件切换到新session）
    // new_session_id: 新的session ID
    int (*on_rotation)(void *plugin_ctx, int new_session_id);

    // 清理持久化（关闭文件，生成元数据）
    int (*cleanup_persistence)(void *plugin_ctx);
} plugin_persistence_api_t;
```

### 轮转控制流程

```
1. Rotation Controller监控全局条件
   - 时间间隔（如每5分钟）
   - 总文件大小（如所有插件文件总和超过1GB）
   - 总事件数（如所有插件事件总数超过100万）

2. 条件满足时，Rotation Controller：
   ├─> 生成新的session_id
   ├─> 设置全局轮转标志
   └─> 通知所有Worker线程

3. 每个Worker线程：
   ├─> 检测到轮转标志
   ├─> 等待队列清空（处理完所有待处理事件）
   ├─> 调用插件的 on_rotation(new_session_id)
   └─> 继续处理新session的事件

4. 插件在 on_rotation() 中：
   ├─> 关闭当前文件
   ├─> 生成当前session的元数据
   ├─> 打开新session的文件
   └─> 返回成功

5. 所有插件完成轮转后：
   └─> Rotation Controller生成全局元数据
```

### 文件组织结构

```
output/
├── session_001/
│   ├── capture.pcap          # Capture插件：PCAP格式
│   ├── network.jsonl         # Network插件：JSON Lines格式
│   ├── plugin_n.dat          # 其他插件：自定义格式
│   └── metadata.yaml         # Session元数据（框架生成）
├── session_002/
│   ├── capture.pcap
│   ├── network.jsonl
│   ├── plugin_n.dat
│   └── metadata.yaml
└── global_metadata.yaml      # 全局元数据（框架生成）
```

### Session元数据格式

```yaml
session: 1
start_time: "2026-02-07T15:04:05.123456789Z"
end_time: "2026-02-07T15:09:05.987654321Z"
duration_seconds: 300.864197532

files:
  - plugin: "capture"
    file: "capture.pcap"
    format: "pcap"
    size_bytes: 95234567
    event_count: 15234

  - plugin: "network"
    file: "network.jsonl"
    format: "jsonl"
    size_bytes: 123456
    event_count: 42

rotation_reason: "max_duration_reached"
```

### 设计优势

1. **格式灵活性**：每个插件可以选择最适合的输出格式
2. **时间一致性**：所有插件的session边界对齐，便于关联分析
3. **性能隔离**：插件间完全独立，某个插件的I/O阻塞不影响其他插件
4. **易于扩展**：新增插件只需实现持久化接口，无需修改框架

---

## 插件接口规范

### 设计原则：避免循环依赖

采用**函数指针传递**的方式实现框架与插件的双向调用：

1. **编译时依赖**：
   - 框架编译时不依赖插件（插件是运行时动态加载的）
   - 插件编译时只依赖框架的**头文件**（接口定义），不依赖框架二进制

2. **运行时依赖**：
   - 框架通过 `dlopen()` 动态加载插件
   - 框架初始化插件时，传递包含框架接口的结构体（函数指针）
   - 插件保存框架接口指针，通过函数指针调用框架功能

### 框架提供给插件的接口

框架在初始化插件时，传递以下接口结构体：

```c
typedef struct {
    // 事件发送接口
    int (*emit_event)(void *framework_ctx, event_t *event);

    // 时间戳工具
    uint64_t (*get_timestamp_ns)(void);

    // 日志接口
    void (*log_debug)(const char *plugin_name, const char *fmt, ...);
    void (*log_info)(const char *plugin_name, const char *fmt, ...);
    void (*log_warn)(const char *plugin_name, const char *fmt, ...);
    void (*log_error)(const char *plugin_name, const char *fmt, ...);

    // 框架上下文（插件调用emit_event时需要传递）
    void *framework_ctx;
} framework_api_t;
```

**接口说明**：
- `emit_event()` - 将事件发送到插件的事件队列
- `get_timestamp_ns()` - 获取当前纳秒级时间戳
- `log_*()` - 统一的日志接口，自动标记插件来源

### 插件需要实现的接口

#### 1. 插件信息接口

```c
typedef struct {
    const char *name;           // 插件名称
    const char *version;        // 插件版本
    const char *description;    // 插件描述
} plugin_info_t;

// 获取插件信息（必需）
const plugin_info_t* get_plugin_info(void);
```

#### 2. 生命周期接口

```c
// 初始化插件（必需）
// plugin_ctx: 插件上下文指针（由框架分配）
// config: 插件配置对象（框架解析YAML后传入）
// framework_api: 框架提供的接口
int plugin_init(void *plugin_ctx,
                void *config,
                framework_api_t *framework_api);

// 启动插件（必需）
// 插件在此创建线程、打开资源等
int plugin_start(void *plugin_ctx);

// 停止插件（必需）
// 插件在此停止线程、等待线程退出
int plugin_stop(void *plugin_ctx);

// 清理插件（必需）
// 插件在此释放所有资源
int plugin_cleanup(void *plugin_ctx);
```

#### 3. 配置接口

```c
// 配置验证（可选）
// 返回0表示验证通过，-1表示验证失败
int validate_config(void *config, char *error_msg, size_t error_msg_size);

// 配置热加载回调（可选）
// 返回0表示接受新配置，-1表示拒绝
int on_config_reload(void *plugin_ctx, void *new_config);
```

#### 4. 持久化接口

```c
// 初始化持久化（必需）
// output_dir: 输出目录路径
// session_id: 当前session ID
int init_persistence(void *plugin_ctx,
                     const char *output_dir,
                     int session_id);

// 处理事件（必需）
// Worker线程从队列取出事件后调用此接口
// 插件负责将事件写入文件
int process_event(void *plugin_ctx, event_t *event);

// 轮转通知（必需）
// 框架通知插件切换到新session
// new_session_id: 新的session ID
int on_rotation(void *plugin_ctx, int new_session_id);

// 清理持久化（必需）
// 关闭文件，生成元数据
int cleanup_persistence(void *plugin_ctx);
```

#### 5. 监控接口（可选）

```c
typedef struct {
    uint64_t events_processed;   // 已处理事件数
    uint64_t events_dropped;     // 丢弃事件数
    uint64_t bytes_written;      // 已写入字节数
    uint64_t errors;             // 错误数
} plugin_stats_t;

// 获取统计信息（可选）
int get_stats(void *plugin_ctx, plugin_stats_t *stats);

// 健康检查（可选）
// 返回0表示健康，-1表示异常
int health_check(void *plugin_ctx);
```

### 插件API结构体

插件需要导出一个包含所有接口的结构体：

```c
typedef struct {
    // 插件信息
    const plugin_info_t* (*get_plugin_info)(void);

    // 生命周期接口（必需）
    int (*init)(void *plugin_ctx, void *config, framework_api_t *framework_api);
    int (*start)(void *plugin_ctx);
    int (*stop)(void *plugin_ctx);
    int (*cleanup)(void *plugin_ctx);

    // 配置接口（可选，可为NULL）
    int (*validate_config)(void *config, char *error_msg, size_t error_msg_size);
    int (*on_config_reload)(void *plugin_ctx, void *new_config);

    // 持久化接口（必需）
    int (*init_persistence)(void *plugin_ctx, const char *output_dir, int session_id);
    int (*process_event)(void *plugin_ctx, event_t *event);
    int (*on_rotation)(void *plugin_ctx, int new_session_id);
    int (*cleanup_persistence)(void *plugin_ctx);

    // 监控接口（可选，可为NULL）
    int (*get_stats)(void *plugin_ctx, plugin_stats_t *stats);
    int (*health_check)(void *plugin_ctx);
} plugin_api_t;

// 插件必须导出此函数
plugin_api_t* get_plugin_api(void);
```

### 接口调用时序

```
框架启动流程：
1. 加载插件
   └─> dlopen("plugin.so")
   └─> dlsym("get_plugin_api")

2. 获取插件信息
   └─> api->get_plugin_info()

3. 加载插件配置
   └─> 框架解析 plugins/xxx.yaml
   └─> 映射到插件配置结构体
   └─> api->validate_config() [可选]

4. 初始化插件
   └─> api->init(plugin_ctx, config, &framework_api)
   └─> 插件保存 framework_api 指针

5. 创建事件队列和Worker线程
   └─> 框架创建无锁SPSC队列
   └─> 框架创建Worker线程

6. 初始化持久化
   └─> api->init_persistence(output_dir, session_id)

7. 启动插件
   └─> api->start()
   └─> 插件线程开始运行

运行时流程：
8. 插件线程生成事件
   └─> framework_api->emit_event(framework_ctx, event)
   └─> 事件进入队列

9. Worker线程处理事件
   └─> 从队列取事件
   └─> api->process_event(event)
   └─> 插件写入文件

10. 轮转发生
    └─> Rotation Controller检测条件
    └─> 通知所有Worker线程
    └─> api->on_rotation(new_session_id)
    └─> 插件切换文件

11. 配置热加载（可选）
    └─> 配置文件变化
    └─> 框架重新解析配置
    └─> api->on_config_reload(new_config)

关闭流程：
12. 停止插件
    └─> api->stop()
    └─> 插件线程退出

13. 清理持久化
    └─> api->cleanup_persistence()
    └─> 关闭文件，生成元数据

14. 清理插件
    └─> api->cleanup()
    └─> 释放资源

15. 卸载插件
    └─> dlclose()
```

### 公共数据结构

#### 事件结构

```c
typedef struct {
    uint64_t timestamp_ns;      // 纳秒级时间戳
    char *plugin_name;          // 插件名称
    void *data;                 // 事件数据（插件自定义）
    size_t data_size;           // 数据大小
    void (*free_data)(void*);   // 释放data的函数（可选）
} event_t;
```

**说明**：
- 插件创建事件时分配 `event_t` 和 `data`
- 插件调用 `emit_event()` 将事件发送到队列
- Worker线程调用 `process_event()` 处理事件
- Worker线程处理完后调用 `free_data()` 释放内存（如果提供）

### 编译和链接

**框架编译**：
```bash
gcc -o ns-tracker framework.c -lpthread -lck -lyaml -ldl
```

**插件编译**：
```bash
gcc -fPIC -shared -o capture.so capture.c \
    -I/path/to/framework/include \
    -lpcap
```

**目录结构**：
```
project/
├── include/
│   ├── framework_api.h      # 框架提供给插件的接口
│   ├── plugin_api.h         # 插件需要实现的接口
│   └── common.h             # 公共数据结构（event_t等）
├── src/
│   ├── framework/           # 框架实现
│   └── plugins/             # 插件实现
│       ├── capture/
│       └── network/
└── build/
    ├── ns-tracker           # 框架可执行文件
    └── plugins/
        ├── capture.so
        └── network.so
```

### 接口设计原则

1. **无循环依赖**：插件只依赖头文件，不依赖框架二进制
2. **类型安全**：通过结构体和函数指针保证类型安全
3. **最小接口集**：只定义必需的接口，可选接口可为NULL
4. **插件自主性**：插件完全控制自己的行为和资源
5. **框架协调性**：框架负责生命周期管理和跨插件协调

---

## 线程模型和并发控制详细设计

### 线程模型概览

```
Main Thread (主线程)
  │
  ├─> 初始化框架
  ├─> 加载配置
  ├─> 创建插件实例
  ├─> 为每个插件创建：
  │     - 无锁SPSC队列 (Concurrency Kit)
  │     - Worker线程
  ├─> 启动所有插件线程
  └─> 等待信号 (SIGINT/SIGTERM)

Plugin Thread 1 (Capture)
  │
  ├─> pcap_loop() 回调
  ├─> 获取时间戳 (内核时间戳)
  ├─> 构造事件
  └─> ck_ring_enqueue() 入队

Worker Thread 1
  │
  ├─> ck_ring_dequeue() 出队
  ├─> 序列化事件 (JSON)
  ├─> 写入 capture.jsonl
  └─> 检查轮转条件

Plugin Thread 2 (Network)
  │
  ├─> recv() netlink消息
  ├─> 获取时间戳 (用户空间)
  ├─> 构造事件
  └─> ck_ring_enqueue() 入队

Worker Thread 2
  │
  ├─> ck_ring_dequeue() 出队
  ├─> 序列化事件 (JSON)
  ├─> 写入 network.jsonl
  └─> 检查轮转条件
```

### 并发控制机制

#### 1. 无锁队列（Concurrency Kit SPSC）

**队列特性**：
- 单生产者单消费者（SPSC）模式
- 无锁实现，使用原子操作
- 固定大小的环形缓冲区
- 生产者：插件线程
- 消费者：Worker线程

**队列操作**：
```c
// 插件线程：入队（非阻塞）
bool success = ck_ring_enqueue_spsc(ring, buffer, event);
if (!success) {
    // 队列满，记录丢包
    atomic_fetch_add(&stats.dropped_events, 1);
}

// Worker线程：出队（非阻塞）
bool success = ck_ring_dequeue_spsc(ring, buffer, &event);
if (!success) {
    // 队列空，短暂休眠
    usleep(100);  // 或使用条件变量
}
```

#### 2. 插件间隔离

**完全隔离**：
- 每个插件有独立的队列和Worker线程
- 插件之间无共享状态
- 某个插件崩溃不影响其他插件

**好处**：
- 无需跨插件锁
- 调试简单
- 易于扩展

#### 3. 文件轮转同步

**问题**：多个Worker线程需要同步轮转文件

**解决方案**：
```c
// 全局轮转状态（原子变量）
_Atomic uint64_t global_session_id = 1;
_Atomic bool rotation_requested = false;

// Worker线程检查轮转条件
if (should_rotate_local()) {
    // 请求全局轮转
    atomic_store(&rotation_requested, true);
}

// 定期检查全局轮转请求
if (atomic_load(&rotation_requested)) {
    // 等待所有Worker完成当前session
    barrier_wait(&rotation_barrier);

    // 切换到新session
    uint64_t new_session = atomic_fetch_add(&global_session_id, 1);
    open_new_file(new_session);

    // 重置轮转标志
    if (barrier_is_last()) {
        atomic_store(&rotation_requested, false);
    }
}
```

#### 4. 优雅关闭

**关闭流程**：
```
1. Main Thread收到SIGINT/SIGTERM
2. 设置全局标志：shutdown_requested = true
3. 通知所有插件线程停止
4. 等待插件线程退出（pthread_join）
5. 等待队列清空
6. 通知所有Worker线程停止
7. 等待Worker线程退出
8. 关闭所有文件
9. 生成元数据
```

### 性能考虑

**队列大小**：
- 默认：10000个事件
- 可配置：根据网络负载调整
- 监控：记录队列满的次数

**Worker线程策略**：
- 优先级：实时优先级（可选）
- CPU亲和性：绑定到特定CPU核心（可选）
- 批处理：一次处理多个事件减少系统调用

**内存管理**：
- 事件对象池：预分配事件对象，减少malloc/free
- 零拷贝：尽量避免数据复制
