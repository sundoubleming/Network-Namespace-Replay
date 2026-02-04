# Story: Replay - 流量回放模块

## 模块概述

负责读取pcap文件并在指定的Network Namespace中回放网络流量，支持多网卡分发、速度控制、包修改等功能。

## 优先级

**P0 - 最高优先级**（核心功能模块）

## 功能需求

### 1. Pcap文件处理

#### 1.1 文件读取
- 读取pcap格式文件
- 读取pcapng格式文件
- 支持压缩文件（.gz）
- 文件格式验证
- 文件信息提取

#### 1.2 包解析
- 解析以太网帧
- 解析IP层
- 解析传输层（TCP/UDP）
- 提取时间戳信息
- 提取接口信息

### 2. 流量回放

#### 2.1 基础回放
- 按时间戳顺序回放
- 保持原始时序
- 指定回放网卡
- 回放到namespace

#### 2.2 速度控制
- 原速回放（1x）
- 加速回放（2x, 5x, 10x等）
- 减速回放（0.5x, 0.1x等）
- 最快速度回放（忽略时间戳）
- 自定义速度因子

#### 2.3 循环回放
- 单次回放
- 循环回放
- 指定循环次数
- 循环间隔设置

### 3. 多网卡分发

#### 3.1 接口映射
- 原始接口到目标接口的映射
- 支持多对多映射
- 映射配置验证
- 未映射接口的处理策略

#### 3.2 流量分发
- 根据原始接口分发
- 根据MAC地址分发
- 根据IP地址分发
- 自定义分发规则

### 4. 包修改

#### 4.1 链路层修改
- MAC地址重写
- VLAN标签修改
- 以太网类型修改

#### 4.2 网络层修改
- IP地址重写
- TTL修改
- IP校验和重算

#### 4.3 传输层修改
- 端口重写
- TCP/UDP校验和重算
- TCP序列号调整（可选）

### 5. 过滤和选择

#### 5.1 包过滤
- BPF过滤器支持
- 只回放特定协议
- 只回放特定流
- 时间范围过滤

#### 5.2 包选择
- 跳过前N个包
- 只回放前N个包
- 采样回放

### 6. 实时统计

#### 6.1 回放统计
- 已回放包数
- 已回放字节数
- 回放速率
- 回放进度
- 错误统计

#### 6.2 统计展示
- 实时更新
- 按接口统计
- 统计导出

### 7. 回放控制

#### 7.1 控制操作
- 启动回放
- 停止回放
- 暂停回放
- 恢复回放
- 跳转到指定位置

#### 7.2 事件回调
- 回放开始事件
- 回放结束事件
- 错误事件
- 进度更新事件

## 技术要求

### 依赖库
- scapy - 包处理和发送
- dpkt - 包解析（可选）
- tcpreplay - 高性能回放（可选）

### 权限要求
- 需要root权限或CAP_NET_RAW capability
- 需要CAP_NET_ADMIN capability（namespace操作）

### 性能要求
- 支持高速回放（1Gbps+）
- 时序精度 < 1ms
- 内存使用合理

### 准确性要求
- 时间戳精度保持
- 包内容完整性
- 校验和正确性

## 接口定义

```python
# replay/replayer.py
class TrafficReplayer:
    def __init__(self, pcap_file: str, namespace: str = None):
        """初始化回放器"""
        self.pcap_file = pcap_file
        self.namespace = namespace
        self.is_replaying = False
        self.statistics = {}

    def replay(self,
               interface_mapping: dict,
               speed_factor: float = 1.0,
               loop: bool = False,
               loop_count: int = 0,
               packet_filter: str = "",
               modify_config: dict = None) -> bool:
        """开始回放"""
        pass

    def stop(self) -> bool:
        """停止回放"""
        pass

    def pause(self) -> bool:
        """暂停回放"""
        pass

    def resume(self) -> bool:
        """恢复回放"""
        pass

    def get_statistics(self) -> dict:
        """获取统计信息"""
        pass

    def get_progress(self) -> float:
        """获取回放进度（0-100）"""
        pass

# replay/pcap_reader.py
class PcapReader:
    def __init__(self, pcap_file: str):
        """初始化pcap读取器"""
        self.pcap_file = pcap_file

    def read_packets(self, count: int = 0) -> list:
        """读取包"""
        pass

    def get_info(self) -> dict:
        """获取pcap文件信息"""
        pass

    def get_interfaces(self) -> list[str]:
        """获取pcap中的接口列表"""
        pass

    def get_packet_count(self) -> int:
        """获取包总数"""
        pass

    def get_duration(self) -> float:
        """获取时长（秒）"""
        pass

    def filter_packets(self, filter_expr: str) -> list:
        """过滤包"""
        pass

# replay/packet_modifier.py
class PacketModifier:
    """包修改器"""

    def __init__(self, config: dict):
        """从配置初始化"""
        self.config = config

    def modify(self, packet) -> object:
        """修改包"""
        pass

    def rewrite_mac(self, packet, src_mac: str = None, dst_mac: str = None):
        """重写MAC地址"""
        pass

    def rewrite_ip(self, packet, src_ip: str = None, dst_ip: str = None):
        """重写IP地址"""
        pass

    def rewrite_port(self, packet, src_port: int = None, dst_port: int = None):
        """重写端口"""
        pass

    def recalculate_checksums(self, packet):
        """重算校验和"""
        pass

# replay/distributor.py
class PacketDistributor:
    """包分发器"""

    def __init__(self, interface_mapping: dict):
        """初始化分发器"""
        self.interface_mapping = interface_mapping

    def get_target_interface(self, packet, original_interface: str) -> str:
        """获取目标接口"""
        pass

    def distribute(self, packets: list) -> dict:
        """分发包到不同接口，返回{interface: [packets]}"""
        pass

# replay/sender.py
class PacketSender:
    """包发送器"""

    def __init__(self, interface: str, namespace: str = None):
        """初始化发送器"""
        self.interface = interface
        self.namespace = namespace

    def send(self, packet) -> bool:
        """发送单个包"""
        pass

    def send_batch(self, packets: list) -> int:
        """批量发送，返回成功发送的数量"""
        pass

    def send_with_timing(self, packets: list, speed_factor: float = 1.0):
        """按时间戳发送"""
        pass

# replay/statistics.py
class ReplayStatistics:
    def __init__(self):
        self.packets_sent = 0
        self.packets_failed = 0
        self.bytes_sent = 0
        self.start_time = None
        self.end_time = None
        self.interface_stats = {}

    def update(self, interface: str, packet, success: bool):
        """更新统计"""
        pass

    def get_rate(self) -> dict:
        """获取速率"""
        pass

    def to_dict(self) -> dict:
        """转换为字典"""
        pass

    def export(self, output_file: str):
        """导出统计"""
        pass
```

## 使用示例

```python
from replay import TrafficReplayer, PcapReader, PacketModifier

# 读取pcap信息
reader = PcapReader("traffic.pcap")
info = reader.get_info()
print(f"Packets: {info['packet_count']}, Duration: {info['duration']}s")

# 基础回放
replayer = TrafficReplayer("traffic.pcap", namespace="replay_ns")
replayer.replay(
    interface_mapping={
        "eth0": "veth0",
        "eth1": "veth1"
    },
    speed_factor=1.0
)

# 加速回放
replayer.replay(
    interface_mapping={"eth0": "veth0"},
    speed_factor=2.0,  # 2倍速
    loop=True,
    loop_count=3
)

# 带包修改的回放
modify_config = {
    "rewrite_mac": True,
    "mac_mapping": {
        "00:11:22:33:44:55": "02:00:00:00:00:01"
    },
    "rewrite_ip": False
}

replayer.replay(
    interface_mapping={"eth0": "veth0"},
    modify_config=modify_config
)

# 获取统计
stats = replayer.get_statistics()
print(f"Sent: {stats['packets_sent']} packets")
print(f"Progress: {replayer.get_progress()}%")
```

## 配置示例

```yaml
replay:
  pcap_file: "traffic.pcap"
  namespace: "replay_ns"

  speed_factor: 1.0
  loop: false
  loop_count: 0
  loop_interval: 1  # 秒

  interface_mapping:
    eth0: veth0
    eth1: veth1

  packet_filter: ""  # BPF过滤器

  packet_modification:
    rewrite_mac: true
    mac_mapping:
      "00:11:22:33:44:55": "02:00:00:00:00:01"

    rewrite_ip: false
    ip_mapping: {}

    recalculate_checksums: true

  statistics:
    enabled: true
    interval: 5
    export_file: "replay_stats.json"
```

## 验收标准

### 功能验收
- [ ] 能够正确读取和解析pcap文件
- [ ] 回放时序准确
- [ ] 多网卡分发正常
- [ ] 速度控制准确
- [ ] 包修改功能正确
- [ ] 统计信息准确

### 测试覆盖率
- [ ] 单元测试覆盖率 > 80%
- [ ] 集成测试覆盖主要场景
- [ ] 性能测试通过

### 性能要求
- [ ] 支持1Gbps+回放速度
- [ ] 时序精度 < 1ms
- [ ] 内存使用合理

### 文档要求
- [ ] API文档完整
- [ ] 配置说明清晰
- [ ] 提供使用示例

## 依赖关系

### 被依赖模块
- cli（回放命令）

### 依赖模块
- utils（日志、异常、格式化）
- config（配置管理）
- namespace（namespace操作）
- network（网卡信息）

## 开发计划

### 第一阶段（基础功能）
- Pcap文件读取
- 基础回放功能
- 接口映射

### 第二阶段（增强功能）
- 速度控制
- 包修改
- 多网卡分发

### 第三阶段（优化完善）
- 性能优化
- 高级统计
- 完整测试

## 风险和挑战

1. **时序精度**：高精度时间戳的保持
2. **性能问题**：高速回放的性能优化
3. **TCP状态**：TCP连接状态的处理
4. **校验和**：包修改后的校验和计算
5. **内存管理**：大文件回放的内存使用

## 参考资料

- scapy文档
- tcpreplay文档
- pcap文件格式规范
- 网络协议栈文档
