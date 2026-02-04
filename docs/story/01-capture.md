# Story: Capture - 流量抓取模块

## 模块概述

负责网络流量的抓取和保存，支持所有网卡抓取、BPF过滤器、实时统计等功能，将流量保存为pcap格式文件。

### 核心特性

* **独立部署**：作为独立模块，可单独部署运行（Docker容器，host网络模式）
* **一次性任务**：运行完成后容器自动退出，符合Job/CronJob模式
* **明确输出**：输出pcap文件及统计信息YAML文件
* **无状态设计**：不依赖外部服务，配置通过命令行参数或环境变量传入

### 使用场景

1. **生产环境流量采集**：在生产服务器上定时抓取流量样本
2. **问题排查**：出现问题时快速抓取现场流量
3. **性能测试准备**：为性能测试准备真实流量数据
4. **安全审计**：定期采集网络流量用于安全分析

## 功能需求

### 1. 流量抓取

#### 1.1 基础抓取
- 仅支持所有网卡同时抓取
- 设置抓包时长
- 设置抓包数量限制
- 实时抓包和离线分析

#### 1.2 过滤器支持
- 支持BPF（Berkeley Packet Filter）语法
- 常用过滤器模板（HTTP, HTTPS, DNS等）
- 过滤器验证
- 过滤器组合

#### 1.3 抓包参数
- Snaplen（抓包长度）设置
- 混杂模式开关
- 缓冲区大小设置
- 超时设置

### 2. 文件管理

#### 2.1 输出文件
- 保存为pcap格式
- 文件大小限制
- 文件轮转（按大小或时间）
- 文件压缩

#### 2.2 文件元信息
- 自动生成YAML元信息文件
- 记录抓包时间（开始/结束）
- 记录抓包网卡列表
- 记录BPF过滤器
- 记录统计信息（包数、字节数、丢包等）
- 记录系统信息（主机名、内核版本等）
- 记录命令行参数
- 记录输出文件路径和大小

### 4. 多网卡支持

#### 4.1 并发抓取
- 同时在所有网卡上抓取
- 所有网卡使用相同的过滤器
- 输出到同一个pcap文件
- 统一的时间戳
- 支持按网卡分别统计流量

## 命令行接口设计

### 主命令

```bash
ns-capture [OPTIONS]
```

### 命令行参数

#### 必需参数
```
-o, --output <path>        输出文件路径（支持模板变量）
```

#### 可选参数
```
# 抓包控制
-d, --duration <seconds>   抓包时长（秒），0表示不限制（默认：0）
-c, --count <number>       抓包数量限制，0表示不限制（默认：0）
-s, --snaplen <bytes>      抓包长度（字节）（默认：65535）
-p, --promisc              启用混杂模式（默认：true）
-b, --buffer <MB>          缓冲区大小（MB）（默认：10）

# 过滤器
-f, --filter <expr>        BPF过滤表达式（默认：空，抓取所有）
--filter-file <path>       从文件读取BPF过滤器

# 文件管理
--rotate-size <MB>         文件轮转大小（MB），0表示不轮转（默认：0）
--rotate-count <number>    保留的轮转文件数量（默认：5）
--compress                 压缩输出文件（gzip）

# 元信息输出
--meta-file <path>         元信息YAML文件路径（默认：自动生成为 <output>.meta.yaml）
--no-meta                  不生成元信息文件

# 其他
--version                  显示版本信息
--help                     显示帮助信息
```

### 输出文件模板变量

支持在输出路径中使用以下变量：

```
{timestamp}    - 时间戳（格式：20260130_150405）
{hostname}     - 主机名
```

示例：
```bash
-o "/data/capture_{timestamp}.pcap"
# 生成：/data/capture_20260130_150405.pcap
```

## Docker部署

### Dockerfile

```dockerfile
# 构建阶段
FROM alpine:latest AS builder

# 安装编译依赖
RUN apk add --no-cache \
    gcc \
    musl-dev \
    make \
    libpcap-dev \
    yaml-dev \
    zlib-dev \
    linux-headers

WORKDIR /build

# 复制源码
COPY . .

# 编译（静态链接）
RUN make clean && \
    make STATIC=1 && \
    strip ns-capture

# 运行阶段
FROM alpine:latest

# 安装最小运行时依赖（如果完全静态链接可以用scratch）
RUN apk add --no-cache libpcap

# 复制二进制文件
COPY --from=builder /build/ns-capture /usr/local/bin/

# 创建输出目录
RUN mkdir -p /data

WORKDIR /data

ENTRYPOINT ["ns-capture"]
CMD ["--help"]
```

### Docker Compose示例

```yaml
version: '3.8'

services:
  capture:
    build: .
    image: ns-capture:latest
    container_name: ns-capture
    network_mode: host
    privileged: true  # 或使用 cap_add
    # cap_add:
    #   - NET_RAW
    #   - NET_ADMIN
    volumes:
      - ./output:/data
    environment:
      - NS_CAPTURE_LOG_LEVEL=info
    command: >
      -o /data/traffic_{timestamp}.pcap
      -d 60
      -f "tcp port 80 or tcp port 443"
      --stats-file /data/stats.yaml
```

### Kubernetes Job示例

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: network-capture
spec:
  template:
    spec:
      hostNetwork: true
      containers:
      - name: capture
        image: ns-capture:latest
        args:
          - "-o"
          - "/data/traffic.pcap"
          - "-d"
          - "300"
          - "-f"
          - "tcp port 80"
        securityContext:
          privileged: true
        volumeMounts:
        - name: output
          mountPath: /data
      volumes:
      - name: output
        hostPath:
          path: /var/capture
          type: DirectoryOrCreate
      restartPolicy: Never
  backoffLimit: 3
```

### Kubernetes CronJob示例

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: network-capture-hourly
spec:
  schedule: "0 * * * *"  # 每小时执行一次
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          containers:
          - name: capture
            image: ns-capture:latest
            args:
              - "-o"
              - "/data/traffic_{timestamp}.pcap"
              - "-d"
              - "300"
              - "--compress"
            securityContext:
              privileged: true
            volumeMounts:
            - name: output
              mountPath: /data
          volumes:
          - name: output
            persistentVolumeClaim:
              claimName: capture-pvc
          restartPolicy: OnFailure
```

## 使用示例

### 基础使用

```bash
# 1. 构建镜像
docker build -t ns-capture:latest .

# 2. 简单抓包（60秒）
docker run --rm --network host --privileged \
  -v $(pwd)/output:/data \
  ns-capture:latest \
  -o /data/traffic.pcap -d 60

# 3. 带过滤器抓包
docker run --rm --network host --privileged \
  -v $(pwd)/output:/data \
  ns-capture:latest \
  -o /data/http.pcap -d 60 \
  -f "tcp port 80 or tcp port 443"

# 4. 带统计信息和元信息
docker run --rm --network host --privileged \
  -v $(pwd)/output:/data \
  ns-capture:latest \
  -o /data/traffic.pcap -d 60 \
  --stats-file /data/stats.yaml

# 输出文件：
# - /data/traffic.pcap              (pcap文件)
# - /data/traffic.pcap.meta.yaml    (元信息，自动生成)
# - /data/stats.yaml                (统计信息)

# 5. 自定义元信息文件路径
docker run --rm --network host --privileged \
  -v $(pwd)/output:/data \
  ns-capture:latest \
  -o /data/traffic.pcap -d 60 \
  --meta-file /data/capture_metadata.yaml

# 6. 不生成元信息文件
docker run --rm --network host --privileged \
  -v $(pwd)/output:/data \
  ns-capture:latest \
  -o /data/traffic.pcap -d 60 \
  --no-meta

# 7. 文件轮转和压缩
docker run --rm --network host --privileged \
  -v $(pwd)/output:/data \
  ns-capture:latest \
  -o /data/traffic.pcap -d 3600 \
  --rotate-size 100 \
  --rotate-count 10 \
  --compress

# 输出文件：
# - /data/traffic.pcap.meta.yaml     (元信息)
# - /data/traffic_001.pcap.gz        (轮转文件1)
# - /data/traffic_002.pcap.gz        (轮转文件2)
# - ...
```

### 生产环境示例

```bash
# 使用环境变量配置
docker run -d \
  --name capture-prod \
  --network host \
  --privileged \
  -v /var/capture:/data \
  -e NS_CAPTURE_FILTER="tcp port 80 or tcp port 443" \
  -e NS_CAPTURE_LOG_LEVEL=warn \
  ns-capture:latest \
  -o /data/prod_{timestamp}.pcap \
  -d 3600 \
  --rotate-size 500 \
  --compress \
  --stats-file /data/stats.yaml
```

### 输出示例

#### 控制台输出
```
2026-01-30 15:04:05 [INFO] Starting capture on all interfaces
2026-01-30 15:04:05 [INFO] Detected interfaces: eth0, eth1, lo
2026-01-30 15:04:05 [INFO] BPF Filter: tcp port 80 or tcp port 443
2026-01-30 15:04:05 [INFO] Output file: /data/traffic_20260130_150405.pcap
2026-01-30 15:04:05 [INFO] Duration: 60s
2026-01-30 15:04:10 [INFO] Stats: packets=1234, bytes=1.2MB, rate=0.2Mbps, dropped=0
2026-01-30 15:04:15 [INFO] Stats: packets=2456, bytes=2.5MB, rate=0.3Mbps, dropped=0
...
2026-01-30 15:05:05 [INFO] Capture completed
2026-01-30 15:05:05 [INFO] Total packets: 15234, Total bytes: 15.6MB, Dropped: 0
2026-01-30 15:05:05 [INFO] Output file: /data/traffic_20260130_150405.pcap (15.6MB)
2026-01-30 15:05:05 [INFO] Metadata file: /data/traffic_20260130_150405.pcap.meta.yaml
2026-01-30 15:05:05 [INFO] Statistics file: /data/stats.yaml
```

#### 统计信息YAML（stats.yaml）
```yaml
# 抓包统计信息
# 自动生成于抓包结束时

capture:
  interfaces:
    - eth0
    - eth1
  packets_captured: 15234
  packets_dropped: 0
  bytes_captured: 16357824
  start_time: 2026-01-30T15:04:05Z
  end_time: 2026-01-30T15:05:05Z
  duration_seconds: 60.0
  rate_mbps: 2.18
  filter: "tcp port 80 or tcp port 443"
  output_file: "/data/traffic_20260130_150405.pcap"

per_interface:
  eth0:
    packets: 12000
    bytes: 13000000
    dropped: 0
  eth1:
    packets: 3234
    bytes: 3357824
    dropped: 0

summary:
  total_packets: 15234
  total_bytes: 16357824
  total_dropped: 0
  success: true
```

#### 元信息YAML（traffic.pcap.meta.yaml）
```yaml
# 抓包元信息文件
# 自动生成于抓包结束时，用于后续分析和调试

version: "1.0.0"
capture_id: "550e8400-e29b-41d4-a716-446655440000"

# 时间信息
start_time: 2026-01-30T15:04:05Z
end_time: 2026-01-30T15:05:05Z
duration_seconds: 60.0

# 系统信息
hostname: "prod-server-01"
kernel_version: "5.15.0-91-generic"
os_release: "Ubuntu 22.04.3 LTS"

# 抓包配置
config:
  filter: "tcp port 80 or tcp port 443"
  snaplen: 65535
  promisc: true
  buffer_size_mb: 10
  duration_seconds: 60

# 网卡信息
interfaces:
  - name: eth0
    mac_address: "00:0c:29:12:34:56"
    ip_addresses:
      - "192.168.1.100/24"
      - "fe80::20c:29ff:fe12:3456/64"
    mtu: 1500
    state: "UP"

# 输出文件
output_files:
  - path: "/data/traffic_20260130_150405.pcap"
    size_bytes: 16357824
    size_human: "15.6 MB"
    compressed: false
    md5: "5d41402abc4b2a76b9719d911017c592"

# 命令行
command_line: "ns-capture -o /data/traffic.pcap -d 60 -f 'tcp port 80 or tcp port 443'"
```

## 技术栈

### 编程语言
- **C (C11标准)**
  - 极致性能，零运行时开销
  - 直接调用libpcap，无中间层损耗
  - 精确的内存控制，适合高速网络（10Gbps+）
  - 静态编译后镜像极小（< 10MB）
  - 适合高性能网络场景

### 核心依赖
- **libpcap** - 底层抓包库（直接使用C API）
  - `pcap_open_live()` - 打开网卡
  - `pcap_compile()` / `pcap_setfilter()` - BPF过滤器
  - `pcap_loop()` / `pcap_next_ex()` - 抓包循环
  - `pcap_dump_open()` / `pcap_dump()` - 写入pcap文件
- **libyaml** - YAML文件生成（C库）
- **pthread** - POSIX线程库（多网卡并发）
- **zlib** - gzip压缩支持

### 可选依赖
- **openssl/libcrypto** - MD5计算（或使用自实现）
- **libuuid** - UUID生成（capture_id）

### 部署环境
- **Docker** - 容器化部署
- **Kubernetes** - 支持Job/CronJob模式

## 技术要求

### 权限要求
- 需要root权限或CAP_NET_RAW capability
- 需要CAP_NET_ADMIN capability（某些操作）
- Docker运行时需要`--cap-add=NET_RAW --cap-add=NET_ADMIN`或`--privileged`

### 性能要求
- 支持高速网络（10Gbps+）
- 极低丢包率（1Gbps: < 0.01%, 10Gbps: < 0.1%）
- 内存使用极小（< 200MB for 10Gbps）
- CPU使用率 < 30%（单核，1Gbps）

### 兼容性要求
- 支持各种网卡类型（物理网卡、虚拟网卡）
- 支持不同的链路层协议（Ethernet, VLAN等）
- 支持Linux kernel 3.10+
- 支持主流Linux发行版（Ubuntu, CentOS, Debian等）

## 验收标准

### 功能验收
- [ ] 能够在所有网卡上成功抓包
- [ ] BPF过滤器正常工作
- [ ] 多网卡并发抓包正常
- [ ] 统计信息准确
- [ ] 文件轮转功能正常
- [ ] 文件压缩功能正常
- [ ] 输出文件模板变量正确替换
- [ ] 容器运行完成后正常退出
- [ ] 元信息YAML文件自动生成
- [ ] 元信息内容完整准确（包含系统信息、配置、统计等）
- [ ] 元信息文件可被正确解析

### 测试覆盖率
- [ ] 单元测试覆盖率 > 75%
- [ ] 集成测试覆盖主要场景
- [ ] 性能测试通过（1Gbps网络）
- [ ] Docker镜像构建成功
- [ ] Kubernetes Job/CronJob测试通过

### 性能要求
- [ ] 1Gbps网络下丢包率 < 0.01%
- [ ] 10Gbps网络下丢包率 < 0.1%
- [ ] CPU使用率 < 30%（单核，1Gbps）
- [ ] 内存使用 < 200MB（10Gbps）
- [ ] 镜像大小 < 10MB

### 文档要求
- [ ] README完整（包含快速开始）
- [ ] 命令行参数文档完整
- [ ] Docker部署文档完整
- [ ] Kubernetes部署示例完整
- [ ] BPF过滤器使用说明
- [ ] 性能调优指南

## Task 拆分

### Task 1: 项目初始化和基础设施
**目标**: 搭建项目基础框架和开发环境

**子任务**:
- [ ] 初始化 C 项目结构（src/, include/, tests/, examples/）
- [ ] 编写 Makefile（支持静态/动态编译）
  - 编译目标：ns-capture
  - 测试目标：test
  - 清理目标：clean
  - 安装目标：install
- [ ] 配置依赖检测（libpcap, libyaml, pthread, zlib）
- [ ] 设置基础日志框架（log.h/log.c）
  - 支持日志级别（DEBUG, INFO, WARN, ERROR）
  - 支持时间戳格式化
- [ ] 设置配置管理（config.h/config.c）
  - 命令行参数解析（getopt_long）
  - 环境变量读取
- [ ] 编写项目头文件（ns-capture.h）

**验收标准**:
- 项目结构清晰，符合 C 项目规范
- Makefile 可以成功编译
- 基础日志输出正常
- 依赖检测正确

---

### Task 2: 核心抓包功能实现
**目标**: 实现单网卡抓包的核心功能

**子任务**:
- [ ] 实现 Capturer 结构体定义（include/capturer.h, src/capturer.c）
- [ ] 实现基础抓包器（使用 libpcap C API）
- [ ] 实现 BPF 过滤器支持和验证
- [ ] 实现抓包参数配置（snaplen, promisc, buffer, timeout）
- [ ] 实现抓包控制（按时长、按数量）
- [ ] 实现基础统计信息收集（包数、字节数、丢包率）
- [ ] 实现 pcap 文件写入

**核心数据结构定义**:

```c
// include/capturer.h
#ifndef CAPTURER_H
#define CAPTURER_H

#include <pcap.h>
#include <stdint.h>
#include <time.h>

// 抓包配置
typedef struct {
    char *filter;           // BPF过滤器表达式
    int snaplen;            // 抓包长度
    int promisc;            // 混杂模式
    int timeout_ms;         // 超时（毫秒）
    int buffer_size_mb;     // 缓冲区大小（MB）
    char *output_file;      // 输出文件路径
    int duration_sec;       // 抓包时长（秒），0表示不限制
    int64_t packet_count;   // 抓包数量限制，0表示不限制
} capture_config_t;

// 统计信息
typedef struct {
    int64_t packets_captured;
    int64_t bytes_captured;
    int64_t packets_dropped;
    char interface[64];
    time_t start_time;
    time_t end_time;
} capture_stats_t;

// 抓包器结构
typedef struct {
    pcap_t *handle;              // pcap句柄
    pcap_dumper_t *dumper;       // pcap文件写入器
    capture_config_t *config;    // 配置
    capture_stats_t stats;       // 统计信息
    volatile int should_stop;    // 停止标志
} capturer_t;

// 函数声明
capturer_t* capturer_create(const char *interface, capture_config_t *config);
int capturer_start(capturer_t *cap);
void capturer_stop(capturer_t *cap);
void capturer_destroy(capturer_t *cap);
capture_stats_t* capturer_get_stats(capturer_t *cap);

#endif // CAPTURER_H
```

**验收标准**:
- 能够在单个网卡上成功抓包
- BPF 过滤器正常工作
- 统计信息准确
- 输出的 pcap 文件可被 tcpdump/wireshark 正常读取

---

### Task 3: 多网卡并发抓包
**目标**: 支持同时在所有网卡上抓包

**子任务**:
- [ ] 实现 MultiCapturer 多网卡抓包器（include/multi_capturer.h, src/multi_capturer.c）
- [ ] 实现网卡自动发现和过滤（排除 lo 等）
- [ ] 实现并发抓包逻辑（pthread 多线程）
- [ ] 实现统一输出到单个 pcap 文件（线程安全的写入）
- [ ] 实现按网卡分别统计功能
- [ ] 实现优雅退出和资源清理（信号处理 SIGINT/SIGTERM）

**核心数据结构定义**:

```c
// include/multi_capturer.h
#ifndef MULTI_CAPTURER_H
#define MULTI_CAPTURER_H

#include "capturer.h"
#include <pthread.h>

#define MAX_INTERFACES 16

// 多网卡抓包器
typedef struct {
    capturer_t *capturers[MAX_INTERFACES];  // 抓包器数组
    pthread_t threads[MAX_INTERFACES];      // 线程数组
    int num_interfaces;                     // 网卡数量
    pcap_dumper_t *shared_dumper;          // 共享的pcap写入器
    pthread_mutex_t write_mutex;            // 写入互斥锁
    capture_config_t *config;               // 共享配置
    volatile int should_stop;               // 全局停止标志
} multi_capturer_t;

// 线程参数
typedef struct {
    multi_capturer_t *mc;
    int index;
    char interface[64];
} thread_arg_t;

// 函数声明
multi_capturer_t* multi_capturer_create(capture_config_t *config);
int multi_capturer_start(multi_capturer_t *mc);
void multi_capturer_stop(multi_capturer_t *mc);
void multi_capturer_destroy(multi_capturer_t *mc);
int multi_capturer_get_all_stats(multi_capturer_t *mc, capture_stats_t **stats);

// 辅助函数
int discover_interfaces(char interfaces[][64], int max_count);
void* capture_thread(void *arg);

#endif // MULTI_CAPTURER_H
```

**实现要点**:
- 使用 pthread_mutex 保护共享的 pcap_dumper
- 使用 pthread_cond 或信号量协调线程
- 正确处理 SIGINT/SIGTERM 信号
- 确保所有线程正确退出和资源释放

**验收标准**:
- 能够同时在多个网卡上抓包
- 所有网卡的包输出到同一个文件
- 按网卡统计信息准确
- 优雅退出时资源正确释放
- 无内存泄漏（使用 valgrind 检测）

---

### Task 4: 文件管理功能
**目标**: 实现文件轮转、压缩等高级文件管理功能

**子任务**:
- [ ] 实现文件大小监控（fstat）
- [ ] 实现文件轮转功能（按大小触发）
- [ ] 实现轮转文件命名（_001, _002 等）
- [ ] 实现轮转文件数量限制（自动删除旧文件）
- [ ] 实现文件压缩功能（使用 zlib 或调用 gzip）
- [ ] 实现文件 MD5 计算（使用 openssl 或自实现）

**实现要点**:
- 使用 stat() 监控文件大小
- 文件轮转时需要原子操作，避免数据丢失
- 压缩可以在后台线程进行，不阻塞抓包

**验收标准**:
- 文件轮转功能正常，无数据丢失
- 文件压缩正常，压缩后可正常解压
- 轮转文件数量限制生效
- 文件 MD5 计算正确

---

### Task 5: 元信息和统计输出
**目标**: 自动生成元信息和统计信息 YAML 文件

**子任务**:
- [ ] 实现 CaptureMetadata 结构体（include/metadata.h, src/metadata.c）
- [ ] 实现系统信息收集（hostname, kernel version, OS release）
  - 使用 uname() 获取内核信息
  - 读取 /etc/os-release 获取系统信息
- [ ] 实现网卡信息收集（name, MAC, IP, MTU, state）
  - 使用 getifaddrs() 或 ioctl() 获取网卡信息
- [ ] 实现配置信息记录
- [ ] 实现统计信息汇总（总计 + 按网卡）
- [ ] 实现输出文件信息记录（path, size, MD5）
- [ ] 实现元信息 YAML 自动生成（使用 libyaml）
- [ ] 实现统计信息 YAML 输出

**核心数据结构定义**:

```c
// include/metadata.h
#ifndef METADATA_H
#define METADATA_H

#include <time.h>
#include <stdint.h>

#define MAX_INTERFACES 16
#define MAX_IPS 8
#define MAX_PATH 256

// 配置信息
typedef struct {
    char filter[256];
    int snaplen;
    int promisc;
    int buffer_size_mb;
    int duration_seconds;
    int64_t packet_count_limit;
} config_info_t;

// 网卡信息
typedef struct {
    char name[64];
    char mac_address[18];
    char ip_addresses[MAX_IPS][64];
    int ip_count;
    int mtu;
    char state[16];
} interface_info_t;

// 单个网卡统计
typedef struct {
    int64_t packets;
    int64_t bytes;
    int64_t dropped;
    double rate_mbps;
} interface_stats_t;

// 统计信息
typedef struct {
    int64_t total_packets;
    int64_t total_bytes;
    int64_t dropped_packets;
    double drop_rate_percent;
    double avg_rate_mbps;

    // 按网卡统计
    char interface_names[MAX_INTERFACES][64];
    interface_stats_t per_interface[MAX_INTERFACES];
    int interface_count;
} stats_info_t;

// 输出文件信息
typedef struct {
    char path[MAX_PATH];
    int64_t size_bytes;
    char size_human[32];
    int compressed;
    char md5[33];
} output_file_info_t;

// 抓包元信息
typedef struct {
    char version[32];
    char capture_id[64];

    time_t start_time;
    time_t end_time;
    double duration_seconds;

    char hostname[256];
    char kernel_version[128];
    char os_release[256];

    config_info_t config;

    interface_info_t interfaces[MAX_INTERFACES];
    int interface_count;

    stats_info_t statistics;

    output_file_info_t output_files[16];
    int output_file_count;

    char command_line[512];
} capture_metadata_t;

// 函数声明
int metadata_write_yaml(const capture_metadata_t *metadata, const char *filepath);
int metadata_collect_system_info(char *hostname, char *kernel, char *os_release);
int metadata_collect_interface_info(const char *ifname, interface_info_t *info);
void metadata_generate_uuid(char *uuid_str);

#endif // METADATA_H
```

**实现要点**:
- 使用 libyaml 的 emitter API 生成 YAML
- 正确处理字符串转义和格式化
- 时间戳使用 ISO 8601 格式

**验收标准**:
- 元信息 YAML 文件自动生成
- 元信息内容完整准确
- YAML 格式正确，可被正确解析
- 统计信息准确

---

### Task 6: 命令行接口
**目标**: 实现完整的命令行接口和参数处理

**子任务**:
- [ ] 实现命令行参数解析（使用 getopt_long）
- [ ] 实现所有必需参数（-o/--output）
- [ ] 实现所有可选参数（-d, -c, -s, -p, -b, -f 等）
- [ ] 实现环境变量支持（NS_CAPTURE_*）
- [ ] 实现输出文件模板变量（{timestamp}, {hostname}）
  - 使用 strftime 格式化时间戳
  - 使用 gethostname 获取主机名
- [ ] 实现参数验证和错误提示
- [ ] 实现 --version 和 --help
- [ ] 实现日志级别控制

**实现要点**:
- 使用 getopt_long 解析长短选项
- 环境变量优先级低于命令行参数
- 模板变量替换使用字符串处理函数

**验收标准**:
- 所有命令行参数正常工作
- 环境变量正确读取
- 模板变量正确替换
- 参数验证有效，错误提示清晰
- --help 输出完整准确

---

### Task 7: Docker 和 Kubernetes 部署
**目标**: 实现容器化部署和 Kubernetes 支持

**子任务**:
- [ ] 编写 Dockerfile（多阶段构建）
  - 构建阶段：安装依赖、编译二进制
  - 运行阶段：最小化镜像、安装运行时依赖
- [ ] 优化 Docker 镜像大小（< 50MB）
- [ ] 测试 Docker 镜像构建和运行
- [ ] 编写 Docker Compose 示例（examples/docker-compose.yml）
- [ ] 编写 Kubernetes Job YAML（examples/k8s-job.yaml）
- [ ] 编写 Kubernetes CronJob YAML（examples/k8s-cronjob.yaml）
- [ ] 测试 hostNetwork 模式
- [ ] 测试权限配置（privileged / capabilities）
- [ ] 测试 volume 挂载和数据持久化
- [ ] 编写部署文档

**验收标准**:
- Docker 镜像成功构建，大小 < 50MB
- Docker 容器能够正常抓包并退出
- Docker Compose 示例可正常运行
- Kubernetes Job 可正常运行并完成
- Kubernetes CronJob 可按计划执行
- 输出文件正确保存到 volume
- 部署文档完整清晰

---

### Task 8: 测试和质量保证
**目标**: 确保代码质量和功能正确性

**子任务**:
- [ ] 编写单元测试
  - capture 包测试
  - metadata 包测试
  - 文件管理功能测试
- [ ] 编写集成测试
  - 端到端抓包测试
  - 多网卡测试
  - 文件轮转测试
- [ ] 编写性能测试
  - 1Gbps 网络测试
  - 丢包率测试
  - CPU/内存使用测试
- [ ] 代码覆盖率检查（> 75%）
- [ ] 代码质量检查（golint, go vet）

**验收标准**:
- 单元测试覆盖率 > 75%
- 所有测试通过
- 1Gbps 网络下丢包率 < 0.1%
- CPU 使用率 < 50%（单核）
- 内存使用 < 500MB

---

### Task 9: 文档和示例
**目标**: 提供完整的使用文档和示例

**子任务**:
- [ ] 编写 README.md
  - 项目介绍
  - 快速开始
  - 功能特性
  - 安装说明
- [ ] 编写命令行参数文档（docs/cli.md）
- [ ] 编写 Docker 部署文档（docs/docker.md）
- [ ] 编写 Kubernetes 部署文档（docs/kubernetes.md）
- [ ] 编写 BPF 过滤器使用说明（docs/filters.md）
- [ ] 编写性能调优指南（docs/performance.md）
- [ ] 编写故障排查指南（docs/troubleshooting.md）
- [ ] 提供使用示例（examples/）

**验收标准**:
- README 完整，包含快速开始
- 所有文档清晰易懂
- 示例可直接运行
- 文档与实际功能一致

---

## 开发计划

### Phase 1: 核心功能（1周）
- [ ] Task 1: 项目初始化和基础设施
- [ ] Task 2: 核心抓包功能实现
- [ ] Task 6: 命令行接口（基础部分）

### Phase 2: 增强功能（1周）
- [ ] Task 3: 多网卡并发抓包
- [ ] Task 4: 文件管理功能
- [ ] Task 5: 元信息和统计输出
- [ ] Task 6: 命令行接口（完整）

### Phase 3: 容器化和部署（3-5天）
- [ ] Task 7: Docker 和 Kubernetes 部署

### Phase 4: 测试和文档（3天）
- [ ] Task 8: 测试和质量保证
- [ ] Task 9: 文档和示例

## 依赖关系

### 外部依赖
- **无**（作为独立模块，不依赖其他模块）

### 被依赖模块
- **replay模块**：使用capture生成的pcap文件
- **用户/运维**：直接使用Docker/K8s部署

### 系统依赖
- Linux kernel 3.10+
- libpcap库
- Docker或Kubernetes（部署环境）

## 风险和挑战

### 技术风险

1. **性能问题**
   - 高速网络（10Gbps+）下的丢包问题
   - 解决方案：使用零拷贝技术、增大缓冲区、优化goroutine数量

2. **权限问题**
   - 生产环境可能限制privileged容器
   - 解决方案：使用最小权限集（CAP_NET_RAW + CAP_NET_ADMIN）

3. **内存管理**
   - 长时间抓包的内存泄漏风险
   - 解决方案：定期GC、限制缓冲区大小、文件轮转

4. **并发安全**
   - 多网卡并发抓包的goroutine安全
   - 解决方案：使用channel通信、sync包同步、context控制

### 实现挑战

1. **静态编译**
   - CGO依赖libpcap，静态编译较复杂
   - 解决方案：使用musl-gcc、Alpine Linux基础镜像

2. **文件轮转**
   - 抓包过程中的文件切换需要无缝
   - 解决方案：使用双缓冲、原子操作

3. **信号处理**
   - 优雅退出和资源清理
   - 解决方案：监听SIGINT/SIGTERM、使用context.WithCancel

4. **跨平台兼容**
   - 不同Linux发行版的libpcap版本差异
   - 解决方案：使用容器统一环境、最小化系统调用

### 运维挑战

1. **存储管理**
   - pcap文件快速增长，存储空间管理
   - 解决方案：文件轮转、自动压缩、定期清理策略

2. **监控告警**
   - 抓包失败、丢包率高的告警
   - 解决方案：输出统计信息YAML、容器退出码

3. **网络影响**
   - 抓包对生产网络的性能影响
   - 解决方案：限制抓包时长、使用过滤器减少流量、错峰执行

4. **安全合规**
   - 抓包数据可能包含敏感信息
   - 解决方案：加密存储、访问控制、数据脱敏

## 性能优化建议

### 编译优化
```bash
# 使用优化标志编译
go build -ldflags="-s -w" -o ns-capture ./cmd

# 减小二进制文件大小
go build -ldflags="-s -w" -trimpath -o ns-capture ./cmd
```

### 运行时优化
```bash
# 增大缓冲区
ns-capture -o traffic.pcap -b 100

# 使用更精确的过滤器减少处理量
ns-capture -o traffic.pcap -f "tcp port 80 and host 192.168.1.100"

# 限制snaplen减少内存使用（只抓取包头）
ns-capture -o traffic.pcap -s 128
```

### 系统优化
```bash
# 增大网卡接收缓冲区
ethtool -G eth0 rx 4096

# 调整内核参数
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.rmem_default=134217728
```

## 参考资料

### 技术文档
- **gopacket**: https://pkg.go.dev/github.com/google/gopacket
- **libpcap**: https://www.tcpdump.org/
- **BPF语法**: https://biot.com/capstats/bpf.html
- **pcap文件格式**: https://wiki.wireshark.org/Development/LibpcapFileFormat

### 最佳实践
- **Golang网络编程**: https://golang.org/doc/articles/wiki/
- **Docker最佳实践**: https://docs.docker.com/develop/dev-best-practices/
- **Kubernetes Job模式**: https://kubernetes.io/docs/concepts/workloads/controllers/job/

### 相关工具
- **tcpdump**: 命令行抓包工具
- **Wireshark**: 图形化抓包分析工具
- **tshark**: Wireshark命令行版本

## 快速开始

### 本地开发

```bash
# 1. 克隆代码
git clone https://github.com/yourusername/ns-capture.git
cd ns-capture

# 2. 安装依赖
sudo apt-get install libpcap-dev  # Ubuntu/Debian
# 或
sudo yum install libpcap-devel    # CentOS/RHEL

# 3. 编译
make build

# 4. 运行（需要root权限）
sudo ./ns-capture -o test.pcap -d 10

# 5. 查看结果
tcpdump -r test.pcap | head -20
```

### Docker快速开始

```bash
# 1. 构建镜像
docker build -t ns-capture:latest .

# 2. 运行抓包
docker run --rm --network host --privileged \
  -v $(pwd)/output:/data \
  ns-capture:latest \
  -o /data/test.pcap -d 10

# 3. 查看结果
tcpdump -r output/test.pcap | head -20
```

### Kubernetes快速开始

```bash
# 1. 创建Job
kubectl apply -f examples/k8s-job.yaml

# 2. 查看日志
kubectl logs job/network-capture

# 3. 获取输出文件
kubectl cp <pod-name>:/data/traffic.pcap ./traffic.pcap
```

## 常见问题

### Q1: 为什么需要privileged或特殊capabilities？
A: 抓包需要访问网卡的原始数据包，这需要CAP_NET_RAW和CAP_NET_ADMIN权限。

### Q2: 如何减少抓包对系统性能的影响？
A:
- 使用精确的BPF过滤器，只抓取需要的流量
- 限制snaplen，只抓取包头部分
- 限制抓包时长，避免长时间运行
- 使用文件轮转，避免单个文件过大

### Q3: 如何处理高速网络（10Gbps+）？
A:
- 增大缓冲区大小（-b参数）
- 使用更强大的硬件
- 考虑使用专业抓包工具（如PF_RING）
- 分布式抓包，多台机器分担负载

### Q4: 抓包文件太大怎么办？
A:
- 使用--rotate-size进行文件轮转
- 使用--compress进行压缩
- 使用更精确的过滤器
- 减小snaplen值

### Q5: 容器退出后如何保留数据？
A: 使用volume挂载，将输出目录映射到宿主机或持久化存储。

### Q6: 如何在Kubernetes中定期抓包？
A: 使用CronJob，参考examples/k8s-cronjob.yaml示例。

## 贡献指南

欢迎贡献代码、报告问题或提出建议！

### 开发流程
1. Fork项目
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启Pull Request

### 代码规范
- 遵循Go官方代码规范
- 使用`gofmt`格式化代码
- 添加必要的注释和文档
- 编写单元测试

## 许可证

MIT License

---

**项目状态**: 设计阶段
**最后更新**: 2026-01-30
**维护者**: @yourusername
