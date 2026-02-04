# Story: Namespace - Network Namespace管理模块

## 模块概述

负责Linux Network Namespace的创建、配置和管理，包括虚拟网卡创建、网络配置、namespace间通信等功能。

## 优先级

**P0 - 高优先级**（核心功能模块）

## 功能需求

### 1. Namespace生命周期管理

#### 1.1 Namespace创建
- 创建新的Network Namespace
- 检查namespace是否已存在
- 支持自定义namespace名称
- 创建时的错误处理

#### 1.2 Namespace删除
- 删除指定的namespace
- 清理namespace中的所有资源
- 强制删除选项
- 删除前的安全检查

#### 1.3 Namespace查询
- 列举所有namespace
- 检查namespace是否存在
- 获取namespace详细信息
- 查询namespace中的进程

### 2. 虚拟网卡管理

#### 2.1 Veth Pair创建
- 创建veth pair
- 自定义veth名称
- 设置veth属性（MTU等）
- 将veth一端移入namespace

#### 2.2 网卡配置
- 设置IP地址
- 设置MAC地址
- 设置MTU
- 启用/禁用网卡
- 设置网卡别名

#### 2.3 网卡移动
- 将网卡移入namespace
- 将网卡移出namespace
- 跨namespace网卡移动

### 3. 网络配置

#### 3.1 IP地址配置
- 添加IP地址
- 删除IP地址
- 支持IPv4和IPv6
- 支持多IP地址

#### 3.2 路由配置
- 添加路由
- 删除路由
- 设置默认网关
- 配置策略路由

#### 3.3 ARP配置
- 添加静态ARP条目
- 删除ARP条目
- 清空ARP缓存

### 4. Namespace内命令执行

#### 4.1 命令执行
- 在namespace中执行命令
- 捕获命令输出
- 命令超时控制
- 错误处理

#### 4.2 进程管理
- 在namespace中启动进程
- 监控namespace中的进程
- 终止namespace中的进程

### 5. 网络连接

#### 5.1 Namespace间连接
- 通过veth pair连接两个namespace
- 配置连接的网络参数
- 测试连接性

#### 5.2 与主机连接
- 将namespace连接到主机网络
- 配置NAT
- 配置端口转发

### 6. 配置导入导出

#### 6.1 配置导入
- 根据配置文件创建namespace
- 批量创建网卡和配置
- 配置验证和错误处理

#### 6.2 配置导出
- 导出namespace的完整配置
- 导出为可复用的配置文件

## 技术要求

### 依赖库
- pyroute2 - netlink接口
- subprocess - 命令执行

### 权限要求
- 需要root权限或CAP_NET_ADMIN capability
- 需要CAP_SYS_ADMIN capability（创建namespace）

### 性能要求
- Namespace创建时间 < 500ms
- 网卡配置时间 < 200ms
- 命令执行响应及时

### 安全要求
- 防止namespace名称注入
- 防止命令注入
- 资源清理确保完整

## 接口定义

```python
# namespace/manager.py
class NamespaceManager:
    def __init__(self):
        """初始化namespace管理器"""
        pass

    def create(self, name: str) -> bool:
        """创建namespace"""
        pass

    def delete(self, name: str, force: bool = False) -> bool:
        """删除namespace"""
        pass

    def exists(self, name: str) -> bool:
        """检查namespace是否存在"""
        pass

    def list(self) -> list[str]:
        """列举所有namespace"""
        pass

    def get_info(self, name: str) -> dict:
        """获取namespace信息"""
        pass

    def execute(self, name: str, command: str, timeout: int = 30) -> tuple[int, str, str]:
        """在namespace中执行命令，返回(返回码, stdout, stderr)"""
        pass

    def cleanup(self, name: str):
        """清理namespace中的所有资源"""
        pass

# namespace/interface.py
class NamespaceInterface:
    def __init__(self, namespace: str):
        """初始化，指定namespace"""
        self.namespace = namespace

    def create_veth_pair(self, veth1: str, veth2: str, mtu: int = 1500) -> bool:
        """创建veth pair"""
        pass

    def move_to_namespace(self, interface: str, target_ns: str) -> bool:
        """将网卡移入namespace"""
        pass

    def set_ip(self, interface: str, ip: str, prefix_len: int) -> bool:
        """设置IP地址"""
        pass

    def set_mac(self, interface: str, mac: str) -> bool:
        """设置MAC地址"""
        pass

    def set_mtu(self, interface: str, mtu: int) -> bool:
        """设置MTU"""
        pass

    def set_state(self, interface: str, state: str) -> bool:
        """设置网卡状态（up/down）"""
        pass

    def add_ip(self, interface: str, ip: str, prefix_len: int) -> bool:
        """添加IP地址"""
        pass

    def del_ip(self, interface: str, ip: str, prefix_len: int) -> bool:
        """删除IP地址"""
        pass

# namespace/routing.py
class NamespaceRouting:
    def __init__(self, namespace: str):
        self.namespace = namespace

    def add_route(self, dest: str, gateway: str = None, interface: str = None) -> bool:
        """添加路由"""
        pass

    def del_route(self, dest: str) -> bool:
        """删除路由"""
        pass

    def set_default_gateway(self, gateway: str, interface: str = None) -> bool:
        """设置默认网关"""
        pass

    def get_routes(self) -> list[dict]:
        """获取路由表"""
        pass

# namespace/builder.py
class NamespaceBuilder:
    """用于根据配置构建namespace的Builder类"""

    def __init__(self, config: dict):
        """从配置初始化"""
        self.config = config
        self.namespace = None

    def build(self) -> bool:
        """根据配置构建完整的namespace环境"""
        pass

    def create_namespace(self) -> bool:
        """创建namespace"""
        pass

    def setup_interfaces(self) -> bool:
        """设置所有网卡"""
        pass

    def setup_routing(self) -> bool:
        """设置路由"""
        pass

    def verify(self) -> bool:
        """验证配置是否正确应用"""
        pass

    def export_config(self) -> dict:
        """导出当前配置"""
        pass
```

## 使用示例

```python
# 基础使用
from namespace import NamespaceManager, NamespaceInterface, NamespaceRouting

# 创建namespace
ns_mgr = NamespaceManager()
ns_mgr.create("test_ns")

# 创建veth pair并配置
ns_if = NamespaceInterface("test_ns")
ns_if.create_veth_pair("veth0", "veth0-peer")
ns_if.move_to_namespace("veth0", "test_ns")
ns_if.set_ip("veth0", "192.168.1.10", 24)
ns_if.set_mac("veth0", "02:00:00:00:00:01")
ns_if.set_state("veth0", "up")

# 配置路由
ns_rt = NamespaceRouting("test_ns")
ns_rt.set_default_gateway("192.168.1.1", "veth0")

# 在namespace中执行命令
code, stdout, stderr = ns_mgr.execute("test_ns", "ip addr show")

# 清理
ns_mgr.delete("test_ns")

# 使用Builder从配置构建
config = {
    "name": "replay_ns",
    "interfaces": [
        {
            "name": "veth0",
            "peer": "veth0-peer",
            "ip": "192.168.1.10/24",
            "mac": "02:00:00:00:00:01",
            "mtu": 1500,
            "state": "up",
            "routes": [
                {"dest": "0.0.0.0/0", "gateway": "192.168.1.1"}
            ]
        }
    ]
}

builder = NamespaceBuilder(config)
builder.build()
```

## 验收标准

### 功能验收
- [ ] 能够成功创建和删除namespace
- [ ] 能够创建veth pair并正确配置
- [ ] 网卡配置功能完整（IP、MAC、MTU、状态）
- [ ] 路由配置功能正常
- [ ] 能够在namespace中执行命令
- [ ] Builder能够根据配置构建完整环境

### 测试覆盖率
- [ ] 单元测试覆盖率 > 80%
- [ ] 集成测试覆盖主要场景
- [ ] 错误处理测试完整

### 性能要求
- [ ] 满足性能指标
- [ ] 资源清理及时完整

### 文档要求
- [ ] API文档完整
- [ ] 提供使用示例
- [ ] 错误处理说明清晰

## 依赖关系

### 被依赖模块
- replay（需要在namespace中回放流量）
- cli（namespace管理命令）

### 依赖模块
- utils（日志、权限检查、异常）
- config（配置管理）
- network（网络信息获取）

## 开发计划

### 第一阶段（基础功能）
- Namespace创建和删除
- Veth pair创建
- 基础网卡配置

### 第二阶段（增强功能）
- 路由配置
- 命令执行
- Builder模式实现

### 第三阶段（优化完善）
- 错误处理完善
- 资源清理优化
- 完整测试覆盖

## 风险和挑战

1. **权限问题**：需要高权限，测试和部署需要特别注意
2. **资源泄漏**：namespace和网卡资源需要确保清理
3. **并发安全**：多个namespace操作的并发安全性
4. **错误恢复**：配置失败时的回滚机制

## 参考资料

- Linux Network Namespace文档
- pyroute2 NETNS文档
- iproute2源码
- man ip-netns
