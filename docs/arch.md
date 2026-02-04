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

## 插件接口

### 插件API定义

每个插件必须实现 `plugin_api_t` 结构体并导出 `get_plugin_api()` 函数：

```c
typedef struct plugin_api {
    // 插件元信息
    const char *name;           // 插件名称
    const char *version;        // 插件版本
    const char *description;    // 插件描述

    // 生命周期函数
    int (*init)(framework_context_t *ctx, const char *config_file);
    int (*start)(void);
    int (*stop)(void);
    int (*cleanup)(void);

    // 可选函数
    int (*reload_config)(const char *config_file);
    int (*get_status)(char *buf, size_t size);
} plugin_api_t;

// 插件入口函数
plugin_api_t* get_plugin_api(void);
```

### 框架上下文

框架通过 `framework_context_t` 向插件提供服务：

```c
typedef struct framework_context {
    // 序列号管理（由Event Bus提供）
    uint64_t (*get_sequence)(void);

    // 事件发送
    int (*emit_event)(event_t *event);

    // 日志接口
    void (*log_debug)(const char *plugin_name, const char *fmt, ...);
    void (*log_info)(const char *plugin_name, const char *fmt, ...);
    void (*log_warn)(const char *plugin_name, const char *fmt, ...);
    void (*log_error)(const char *plugin_name, const char *fmt, ...);

    // 配置读取
    const char* (*get_config_value)(const char *key);
} framework_context_t;
```

**注意**：时间戳由各插件自己负责获取和管理。框架在 Common Library 中提供时间戳工具函数供插件使用。

### 事件结构

```c
typedef struct event {
    uint64_t timestamp_ns;      // 纳秒级时间戳（由插件提供）
    uint64_t sequence;          // 全局序列号（由框架提供）
    char source[64];            // 插件名称
    char type[64];              // 事件类型
    void *data;                 // 事件数据
    size_t data_size;           // 数据大小
    void (*free_data)(void*);   // 数据释放函数
} event_t;
```

### Common Library 时间戳工具

框架在 Common Library 中提供时间戳相关的工具函数：

```c
// 获取当前时间戳（CLOCK_MONOTONIC）
uint64_t get_monotonic_timestamp_ns(void);

// 获取当前时间戳（CLOCK_REALTIME）
uint64_t get_realtime_timestamp_ns(void);

// 时间戳格式转换
uint64_t timeval_to_ns(struct timeval *tv);
uint64_t timespec_to_ns(struct timespec *ts);

// 时间戳格式化为ISO 8601字符串
int format_timestamp_iso8601(uint64_t ts_ns, char *buf, size_t size);
```

插件可以选择：
- 使用自己的时间戳源（如 libpcap 的内核时间戳）
- 使用 Common Library 提供的工具函数获取时间戳

## 内置插件

### 1. Capture插件

**功能**：网络流量抓取

**技术实现**：
- 使用 libpcap 进行数据包捕获
- 基于 `pcap_loop()` 回调机制
- 使用 libpcap 提供的内核时间戳

**事件类型**：
- `packet`：捕获到的数据包

**配置示例**：
```yaml
# config/plugins/capture.yaml
filter: "tcp port 80 or tcp port 443"
snaplen: 65535
promisc: true
buffer_size_mb: 10
```

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

**配置示例**：
```yaml
# config/plugins/network.yaml
monitor_routes: true
monitor_links: true
monitor_neighbors: true
monitor_addresses: true
```

## 配置文件

### 框架配置

```yaml
# config/framework.yaml
framework:
  name: "network-monitor"
  version: "1.0.0"
  output_dir: "/data/output"
  log_level: "info"

  # 运行控制
  duration_seconds: 0        # 0表示持续运行

  # 事件总线配置
  event_queue_size: 10000

  # 持久化配置
  persister:
    output_format: "session"  # session目录模式 或 numbered文件名模式

    # 文件轮转配置（满足任一条件即触发轮转）
    rotation:
      max_size_mb: 100        # 任一插件文件达到100MB时轮转
      max_duration_sec: 300   # 或每300秒轮转一次
      max_events: 10000       # 或任一插件达到10000个事件时轮转

    # 压缩配置
    compression:
      enabled: false          # 是否启用压缩
      algorithm: "gzip"       # 压缩算法

plugins:
  - name: "capture"
    path: "./plugins/capture.so"
    config: "./config/plugins/capture.yaml"
    autostart: true

  - name: "network"
    path: "./plugins/network.so"
    config: "./config/plugins/network.yaml"
    autostart: true
```

## 输出格式

### 事件日志 (events.jsonl)

JSON Lines格式，每行一个事件：

```json
{"timestamp_ns":1706601845123456789,"sequence":1,"source":"network","type":"route_add","data":{"dst":"192.168.1.0/24","gateway":"192.168.1.1","dev":"eth0"}}
{"timestamp_ns":1706601845234567890,"sequence":2,"source":"capture","type":"packet","data":{"len":1500,"proto":"TCP","src":"192.168.1.100:45678","dst":"8.8.8.8:443"}}
{"timestamp_ns":1706601846345678901,"sequence":3,"source":"network","type":"neigh_add","data":{"ip":"192.168.1.1","mac":"00:11:22:33:44:55","dev":"eth0","state":"REACHABLE"}}
```

### 元数据文件 (metadata.yaml)

```yaml
framework:
  name: "network-monitor"
  version: "1.0.0"
  start_time: "2026-01-30T15:04:05.123456789Z"
  end_time: "2026-01-30T15:05:05.987654321Z"
  duration_seconds: 60.864197532

system:
  hostname: "test-server"
  kernel_version: "5.15.0-91-generic"
  os_release: "Ubuntu 22.04.3 LTS"

plugins:
  - name: "capture"
    version: "1.0.0"
    config: "./config/plugins/capture.yaml"
    events_count: 15234

  - name: "network"
    version: "1.0.0"
    config: "./config/plugins/network.yaml"
    events_count: 42

output_files:
  - path: "./output/events.jsonl"
    size_bytes: 16357824
    event_count: 15276

  - path: "./output/capture.pcap"
    size_bytes: 15600000
    packet_count: 15234
```

## 编译和运行

### 编译

```bash
# 编译框架核心
make framework

# 编译所有插件
make plugins

# 编译全部
make all
```

### 运行

```bash
# 使用默认配置运行
sudo ./network-monitor

# 指定配置文件
sudo ./network-monitor -c config/framework.yaml

# 指定运行时长（秒）
sudo ./network-monitor -d 60

# 指定输出目录
sudo ./network-monitor -o /data/output
```

### 插件管理

```bash
# 运行时加载插件（通过信号或控制接口）
# TODO: 实现运行时控制接口

# 卸载插件
# TODO: 实现运行时控制接口
```

## 技术要点

### 1. 时间戳管理

**设计原则**：
- 时间戳由各插件自己负责获取和管理
- 框架提供 Common Library 工具函数辅助
- 框架提供全局序列号保证事件顺序

**Capture插件**：
- 使用 libpcap 提供的**内核时间戳**（最精确）
- 时间戳来自 `struct pcap_pkthdr.ts`

**Network插件**：
- 在收到 netlink 事件时立即调用 `get_monotonic_timestamp_ns()` 获取时间戳
- 尽量减少用户空间延迟

**序列号**：
- 由 Event Bus 维护全局序列号
- 插件在发送事件时通过 `get_sequence()` 获取
- 用于处理同一纳秒内的多个事件排序

### 2. 零轮询设计

**Capture插件**：
- 使用 `pcap_loop()` 阻塞等待数据包
- 内核在收到包时立即回调

**Network插件**：
- 使用 `recv()` 阻塞等待 netlink 消息
- 内核在配置变化时主动发送通知

### 3. 线程模型

```
Main Thread
  ├─> Plugin Manager Thread (监听插件管理请求)
  ├─> Event Bus Thread (处理事件队列)
  ├─> Plugin 1 Thread (capture插件)
  ├─> Plugin 2 Thread (network插件)
  └─> Signal Handler Thread (SIGINT/SIGTERM)
```

### 4. 插件热插拔实现

```c
// 加载插件
void *handle = dlopen("./plugins/capture.so", RTLD_LAZY);
plugin_api_t* (*get_api)(void) = dlsym(handle, "get_plugin_api");
plugin_api_t *api = get_api();
api->init(framework_ctx, config_file);

// 卸载插件
api->stop();
api->cleanup();
dlclose(handle);
```

### 5. 插件时间戳使用示例

**Capture插件**（使用libpcap内核时间戳）：
```c
void packet_handler(u_char *user, const struct pcap_pkthdr *h, const u_char *bytes) {
    // 使用libpcap提供的内核时间戳
    uint64_t ts_ns = timeval_to_ns(&h->ts);
    uint64_t seq = fw_ctx->get_sequence();

    event_t *event = event_create("capture", "packet", data, size);
    event->timestamp_ns = ts_ns;
    event->sequence = seq;
    fw_ctx->emit_event(event);
}
```

**Network插件**（使用Common Library工具函数）：
```c
void handle_netlink_event() {
    // 立即获取时间戳
    uint64_t ts_ns = get_monotonic_timestamp_ns();
    uint64_t seq = fw_ctx->get_sequence();

    event_t *event = event_create("network", "route_add", data, size);
    event->timestamp_ns = ts_ns;
    event->sequence = seq;
    fw_ctx->emit_event(event);
}
```

## 依赖库

- **libpcap**：数据包捕获
- **libyaml**：YAML配置解析
- **pthread**：多线程支持
- **libdl**：动态库加载
- **libnl** (可选)：更高级的netlink操作

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
- [ ] 事件持久化（JSON Lines）
- [ ] 元数据生成
- [ ] 输出格式优化

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

1. 创建插件目录：`src/plugins/myplugin/`
2. 实现 `plugin_api_t` 接口
3. 导出 `get_plugin_api()` 函数
4. 编译为 `.so` 文件
5. 添加配置文件：`config/plugins/myplugin.yaml`
6. 在框架配置中注册插件

### 插件示例场景

- **DNS监控插件**：记录DNS查询和响应
- **连接跟踪插件**：记录TCP连接状态变化
- **性能指标插件**：记录CPU、内存、网络带宽
- **eBPF插件**：记录内核级别的网络事件
- **防火墙规则插件**：监控iptables/nftables变化

## 性能考虑

### 当前设计（Phase 1-3）
- 可接受一定性能损耗
- 优先保证功能完整性和时间戳准确性
- 目标：支持1Gbps网络，丢包率 < 1%

### 未来优化（Phase 4+）
- 零拷贝技术
- 无锁队列
- 批量写入
- 数据压缩
- 目标：支持10Gbps网络，丢包率 < 0.1%

## 已知限制

1. **时间戳精度差异**：libpcap时间戳（内核）vs netlink时间戳（用户空间），差异通常在微秒级，这是设计上的权衡
2. **事件丢失风险**：高负载下事件队列可能溢出
3. **单机部署**：当前不支持分布式部署
4. **Linux专用**：依赖Linux特有的netlink机制
5. **时间戳由插件管理**：框架不强制统一时间源，插件需自行保证时间戳的准确性

## 参考资料

- [libpcap文档](https://www.tcpdump.org/manpages/pcap.3pcap.html)
- [Netlink协议](https://man7.org/linux/man-pages/man7/netlink.7.html)
- [rtnetlink文档](https://man7.org/linux/man-pages/man7/rtnetlink.7.html)
- [dlopen手册](https://man7.org/linux/man-pages/man3/dlopen.3.html)

## TODO列表

- [ ] 实现运行时控制接口（Unix socket或信号）
- [ ] 添加事件过滤功能
- [ ] 支持多namespace监控
- [ ] 实现事件回放工具
- [ ] 性能基准测试
- [ ] 内存泄漏检测（valgrind）
- [ ] 完整的单元测试
- [ ] Docker容器化部署

---

**文档版本**：v0.1
**最后更新**：2026-02-03
**状态**：设计阶段
