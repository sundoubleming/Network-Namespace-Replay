# Story: Network - 网络信息管理模块

## 模块概述

负责获取和长时间监听Linux系统的网络接口信息，包括网卡列表、IP配置、路由表、ARP表等，并支持网络配置的导出。

## 优先级

**P0 - 高优先级**（namespace模块的前置依赖）

## 功能需求

### 1. 网卡信息获取

#### 1.1 网卡列表
- 列举系统所有网络接口
- 过滤虚拟网卡（可选）
- 按类型分类（物理网卡、虚拟网卡、loopback等）
- 获取网卡状态（UP/DOWN）

#### 1.2 网卡详细信息
- 接口名称
- MAC地址
- IP地址（IPv4/IPv6）
- 子网掩码/前缀长度
- MTU值
- 接口索引
- 接口类型
- 驱动信息
- 统计信息（收发包数、字节数、错误数）

### 2. 路由信息获取

#### 2.1 路由表
- 获取主路由表
- 获取所有路由表
- 路由条目详细信息：
  - 目标网络
  - 网关
  - 出接口
  - 度量值
  - 路由类型

#### 2.2 默认路由
- 获取默认网关
- 获取默认出接口

### 3. ARP表信息

#### 3.1 ARP缓存
- 获取ARP表所有条目
- IP到MAC的映射
- ARP条目状态（REACHABLE, STALE等）

### 4. Namespace网络信息

#### 4.1 指定Namespace查询
- 查询指定namespace中的网卡信息
- 查询指定namespace中的路由信息
- 查询指定namespace中的ARP表

### 5. 网络配置导出

#### 5.1 配置导出
- 导出完整网络配置
- 导出为结构化数据(yaml）
- 支持选择性导出（指定网卡）

#### 5.2 配置对比
- 对比两个网络配置的差异
- 生成配置变更报告

#### 5.3 时间戳信息
- 对于抓包中途(story 01)中变化的信息(路由表，ARP表，网卡等)，继续其变更时间

### 6. 网络拓扑分析

#### 6.1 连接关系
- 识别veth pair关系
- 识别bridge连接关系
- 生成网络拓扑图数据

### 7. 监听变化
- 在指定时间内，同时监听网络配置变化(主要是与story01配合)

## 技术要求

### 性能要求
- 网卡列表获取 < 100ms
- 单个网卡详细信息获取 < 50ms
- 路由表获取 < 200ms

### 兼容性要求
- 支持主流Linux发行版（Ubuntu, CentOS, Debian等）
- 支持Linux kernel 3.10+
- 兼容不同的网络配置工具（ifconfig, ip命令）

## 接口定义

```python
# network/interface.py
class NetworkInterface:
    """网络接口信息类"""
    def __init__(self, name: str):
        self.name = name
        self.index = None
        self.mac = None
        self.ips = []  # list of (ip, prefix_len)
        self.mtu = None
        self.state = None
        self.type = None
        self.statistics = {}

    def to_dict(self) -> dict:
        """转换为字典"""
        pass

    @classmethod
    def from_dict(cls, data: dict) -> 'NetworkInterface':
        """从字典创建"""
        pass

# network/manager.py
class NetworkManager:
    def __init__(self, namespace: str = None):
        """初始化，可指定namespace"""
        pass

    def list_interfaces(self, include_virtual: bool = True) -> list[str]:
        """列举所有网络接口名称"""
        pass

    def get_interface(self, name: str) -> NetworkInterface:
        """获取指定网卡的详细信息"""
        pass

    def get_all_interfaces(self) -> list[NetworkInterface]:
        """获取所有网卡的详细信息"""
        pass

    def interface_exists(self, name: str) -> bool:
        """检查网卡是否存在"""
        pass

    def get_interface_state(self, name: str) -> str:
        """获取网卡状态"""
        pass

    def get_statistics(self, name: str) -> dict:
        """获取网卡统计信息"""
        pass

# network/routing.py
class RoutingManager:
    def __init__(self, namespace: str = None):
        pass

    def get_routes(self, table: int = 254) -> list[dict]:
        """获取路由表"""
        pass

    def get_default_route(self) -> dict:
        """获取默认路由"""
        pass

    def get_route_to(self, dest: str) -> dict:
        """获取到指定目标的路由"""
        pass

# network/arp.py
class ARPManager:
    def __init__(self, namespace: str = None):
        pass

    def get_arp_table(self) -> list[dict]:
        """获取ARP表"""
        pass

    def get_mac_for_ip(self, ip: str) -> str:
        """获取指定IP的MAC地址"""
        pass

# network/exporter.py
class NetworkExporter:
    def __init__(self, namespace: str = None):
        pass

    def export_config(self, interfaces: list[str] = None) -> dict:
        """导出网络配置"""
        pass

    def export_to_file(self, output_file: str, interfaces: list[str] = None):
        """导出配置到文件"""
        pass

    def compare_configs(self, config1: dict, config2: dict) -> dict:
        """对比两个配置"""
        pass

# network/topology.py
class NetworkTopology:
    @staticmethod
    def find_veth_peer(interface: str) -> str:
        """查找veth pair的对端"""
        pass

    @staticmethod
    def get_bridge_members(bridge: str) -> list[str]:
        """获取bridge的成员接口"""
        pass

    @staticmethod
    def build_topology() -> dict:
        """构建网络拓扑"""
        pass
```

## 数据结构示例

```python
# 网卡信息
{
    "name": "eth0",
    "index": 2,
    "mac": "00:0c:29:12:34:56",
    "ips": [
        {"address": "192.168.1.100", "prefix_len": 24, "family": "inet"},
        {"address": "fe80::20c:29ff:fe12:3456", "prefix_len": 64, "family": "inet6"}
    ],
    "mtu": 1500,
    "state": "UP",
    "type": "ether",
    "statistics": {
        "rx_packets": 12345,
        "tx_packets": 6789,
        "rx_bytes": 1234567,
        "tx_bytes": 987654,
        "rx_errors": 0,
        "tx_errors": 0
    }
}

# 路由信息
{
    "destination": "0.0.0.0/0",
    "gateway": "192.168.1.1",
    "interface": "eth0",
    "metric": 100,
    "type": "unicast"
}

# ARP条目
{
    "ip": "192.168.1.1",
    "mac": "00:11:22:33:44:55",
    "interface": "eth0",
    "state": "REACHABLE"
}
```

## 验收标准

### 功能验收
- [ ] 能够正确列举所有网络接口
- [ ] 能够获取网卡的完整详细信息
- [ ] 路由表和ARP表获取准确
- [ ] 支持在指定namespace中查询
- [ ] 配置导出功能完整

### 测试覆盖率
- [ ] 单元测试覆盖率 > 80%
- [ ] 包含各种网络配置场景的测试
- [ ] Mock测试不依赖真实网络环境

### 性能要求
- [ ] 满足性能指标要求
- [ ] 大量网卡场景下性能可接受

### 文档要求
- [ ] API文档完整
- [ ] 数据结构说明清晰
- [ ] 提供使用示例

## 依赖关系

### 被依赖模块
- namespace（需要网络信息来配置namespace）
- cli（显示网络信息）

### 依赖模块
- utils（日志、异常、格式化）
- config（配置管理）

## 开发计划

### 第一阶段（基础功能）
- 网卡列表和基本信息获取
- 使用pyroute2实现核心功能
- 基础的路由和ARP信息获取

### 第二阶段（增强功能）
- Namespace支持
- 配置导出功能
- 网络拓扑分析

### 第三阶段（优化完善）
- 性能优化
- 错误处理完善
- 完整测试覆盖

## 风险和挑战

1. **权限问题**：某些网络信息获取需要特定权限
2. **内核版本差异**：不同内核版本的netlink接口可能有差异
3. **性能问题**：大量网卡时的性能优化
4. **Namespace隔离**：正确处理namespace中的网络信息

## 参考资料

- pyroute2文档：https://docs.pyroute2.org/
- Linux netlink文档
- iproute2源码
- /sys/class/net/接口文档
