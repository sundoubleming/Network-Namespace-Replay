# Network Monitor Framework - 开发文档

## 项目概述

这是一个基于C语言实现的**插件化网络监控框架**，用于在指定时间内（或持续运行）监控Linux network namespace中的网络状态变化，并提供完整的时间序列记录以支持网络状态复现。

### 核心特性

1. **统一时间轴**：所有事件使用纳秒级时间戳和全局序列号，保证严格的时间顺序
2. **插件化架构**：支持动态加载/卸载插件（.so文件），易于扩展
3. **事件驱动**：基于回调机制，不使用轮询，保证时间戳准确性
4. **可复现性**：输出完整的时间序列日志，支持网络状态回放和分析

### 设计目标

- **时间戳灵活性**：插件自主管理时间戳（如libpcap使用内核时间戳），框架提供工具函数
- **零轮询设计**：使用Linux异步通知机制（libpcap回调、netlink事件）
- **热插拔支持**：运行时动态加载/卸载插件
- **配置灵活性**：每个插件有独立的配置文件

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────┐
│                Application Layer                    │
│                  (network-monitor)                  │
├─────────────────────────────────────────────────────┤
│                  Framework Core                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │Plugin Manager│  │  Event Bus   │  │Config Mgr│ │
│  └──────────────┘  └──────────────┘  └──────────┘ │
│  ┌──────────────┐  ┌──────────────┐               │
│  │    Logger    │  │  Persister   │               │
│  └──────────────┘  └──────────────┘               │
├─────────────────────────────────────────────────────┤
│              Plugin Interface (API)                 │
│              (plugin_api.h)                         │
├─────────────────────────────────────────────────────┤
│                  Plugins (.so)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │capture.so│  │network.so│  │custom.so │         │
│  └──────────┘  └──────────┘  └──────────┘         │
├─────────────────────────────────────────────────────┤
│                  Common Library                     │
│  (时间戳工具、队列、哈希表等)                        │
└─────────────────────────────────────────────────────┘
```

### 核心模块

#### 1. Plugin Manager（插件管理器）
- 负责插件的加载、卸载、启动、停止
- 使用 `dlopen/dlsym/dlclose` 实现动态加载
- 管理插件生命周期和状态

#### 2. Event Bus（事件总线）
- 接收所有插件发送的事件
- 使用线程安全队列缓冲事件
- 负责事件的排序和分发
- **维护全局序列号**（用于相同时间戳的事件排序）

#### 3. Config Manager（配置管理器）
- 解析框架和插件配置文件（YAML格式）
- 支持配置热加载
- 提供配置查询接口

#### 4. Persister（持久化层）
- 将事件写入文件（JSON Lines格式）
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
- 线程安全队列
- 哈希表
- 其他工具函数

## 核心设计原则

### 1. 时间戳管理

**设计原则**：
- 时间戳由各插件自己负责获取和管理
- 框架提供 Common Library 工具函数辅助
- 框架提供全局序列号保证事件顺序

**Capture插件**：
- 使用 libpcap 提供的**内核时间戳**（最精确）

**Network插件**：
- 在收到 netlink 事件时立即获取时间戳
- 尽量减少用户空间延迟

**序列号**：
- 由 Event Bus 维护全局序列号
- 用于处理同一纳秒内的多个事件排序

### 2. 零轮询设计

**Capture插件**：
- 使用 `pcap_loop()` 阻塞等待数据包
- 内核在收到包时立即回调

**Network插件**：
- 使用 `recv()` 阻塞等待 netlink 消息
- 内核在配置变化时主动发送通知

### 3. 插件热插拔

- 使用 `dlopen/dlsym/dlclose` 动态加载插件
- 插件实现标准接口 `plugin_api_t`
- 运行时可加载/卸载插件

### 4. 持久化策略

**文件组织**：
```
output/
├── session_001/
│   ├── capture.jsonl      # capture插件的事件
│   ├── network.jsonl      # network插件的事件
│   └── metadata.yaml      # 这个session的元数据
├── session_002/
│   ├── capture.jsonl
│   ├── network.jsonl
│   └── metadata.yaml
└── global_metadata.yaml   # 全局元数据
```

**轮转策略**：
- 框架统一控制轮转条件（文件大小、时间、事件数）
- 当任一插件满足轮转条件时，所有插件同步轮转
- 保证同一session内所有文件的时间范围一致

### 5. 线程模型

```
Main Thread
  ├─> Plugin Manager Thread (监听插件管理请求)
  ├─> Event Bus Thread (处理事件队列)
  ├─> Plugin 1 Thread (capture插件)
  ├─> Plugin 2 Thread (network插件)
  └─> Signal Handler Thread (SIGINT/SIGTERM)
```

## 内置插件

### 1. Capture插件

**功能**：网络流量抓取

**技术实现**：
- 使用 libpcap 进行数据包捕获
- 基于 `pcap_loop()` 回调机制
- 使用 libpcap 提供的内核时间戳

**事件类型**：
- `packet`：捕获到的数据包

### 2. Network插件

**功能**：网络配置监控

**技术实现**：
- 使用 Netlink Socket 监听内核事件
- 订阅 RTM_NEWROUTE、RTM_NEWLINK、RTM_NEWNEIGH 等消息
- 事件驱动，无轮询

**事件类型**：
- `route_add` / `route_del`：路由表变化
- `link_add` / `link_del`：网卡变化
- `neigh_add` / `neigh_del`：ARP/邻居表变化
- `addr_add` / `addr_del`：IP地址变化

## 输出格式

### 事件日志 (每个插件独立文件)

**capture.jsonl**:
```json
{"timestamp_ns":1706601845234567890,"sequence":2,"source":"capture","type":"packet","data":{"len":1500,"proto":"TCP"}}
{"timestamp_ns":1706601845345678901,"sequence":4,"source":"capture","type":"packet","data":{"len":60,"proto":"TCP"}}
```

**network.jsonl**:
```json
{"timestamp_ns":1706601845123456789,"sequence":1,"source":"network","type":"route_add","data":{"dst":"192.168.1.0/24"}}
{"timestamp_ns":1706601846345678901,"sequence":3,"source":"network","type":"neigh_add","data":{"ip":"192.168.1.1"}}
```

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

- **libpcap**：数据包捕获
- **libyaml**：YAML配置解析
- **pthread**：多线程支持
- **libdl**：动态库加载

## 开发计划

### Phase 1: 框架核心（优先级：P0）
- [ ] 基础框架结构
- [ ] 插件管理器
- [ ] 事件总线（含序列号管理）
- [ ] 配置管理器
- [ ] 日志系统
- [ ] Common Library（时间戳工具函数、队列、哈希表）

### Phase 2: 内置插件（优先级：P0）
- [ ] Capture插件（基于libpcap）
- [ ] Network插件（基于netlink）

### Phase 3: 持久化和输出（优先级：P0）
- [ ] 事件持久化（JSON Lines，每插件独立文件）
- [ ] 统一轮转控制
- [ ] 元数据生成

### Phase 4: 高级特性（优先级：P1）
- [ ] 插件热插拔
- [ ] 配置热加载
- [ ] 运行时控制接口
- [ ] 性能优化

### Phase 5: 工具和文档（优先级：P2）
- [ ] 事件回放工具
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

1. **时间戳精度差异**：libpcap时间戳（内核）vs netlink时间戳（用户空间），差异通常在微秒级
2. **事件丢失风险**：高负载下事件队列可能溢出
3. **单机部署**：当前不支持分布式部署
4. **Linux专用**：依赖Linux特有的netlink机制
5. **时间戳由插件管理**：框架不强制统一时间源，插件需自行保证时间戳的准确性

## 参考资料

- [libpcap文档](https://www.tcpdump.org/manpages/pcap.3pcap.html)
- [Netlink协议](https://man7.org/linux/man-pages/man7/netlink.7.html)
- [rtnetlink文档](https://man7.org/linux/man-pages/man7/rtnetlink.7.html)
- [dlopen手册](https://man7.org/linux/man-pages/man3/dlopen.3.html)

---

**文档版本**：v0.2
**最后更新**：2026-02-03
**状态**：设计阶段
