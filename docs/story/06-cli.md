# Story: CLI - 命令行接口模块

## 模块概述

提供用户友好的命令行界面，整合所有模块功能，支持子命令、交互式操作、进度显示等功能。

## 优先级

**P1 - 中优先级**（用户交互层）

## 功能需求

### 1. 命令行架构

#### 1.1 主命令
- 主命令：`ns-replay`
- 版本信息：`--version`
- 帮助信息：`--help`
- 全局选项：日志级别、配置文件等

#### 1.2 子命令
- `capture` - 流量抓取
- `replay` - 流量回放
- `namespace` - namespace管理
- `network` - 网络信息查询
- `run` - 完整流程执行
- `config` - 配置管理

### 2. Capture子命令

#### 2.1 基础功能
```bash
ns-replay capture [OPTIONS]
```

#### 2.2 选项
- `-i, --interface` - 指定网卡
- `-o, --output` - 输出文件
- `-f, --filter` - BPF过滤器
- `-d, --duration` - 抓取时长
- `-c, --count` - 包数量限制
- `--snaplen` - 抓包长度
- `--promisc/--no-promisc` - 混杂模式
- `--namespace` - 指定namespace

#### 2.3 示例
```bash
# 基础抓包
ns-replay capture -i eth0 -o traffic.pcap

# 带过滤器
ns-replay capture -i eth0 -f "tcp port 80" -o http.pcap

# 多网卡
ns-replay capture -i eth0 -i eth1 -o traffic_{interface}.pcap

# 在namespace中抓包
ns-replay capture -i veth0 --namespace test_ns -o ns_traffic.pcap
```

### 3. Replay子命令

#### 3.1 基础功能
```bash
ns-replay replay [OPTIONS] PCAP_FILE
```

#### 3.2 选项
- `-n, --namespace` - 目标namespace
- `-m, --mapping` - 接口映射
- `-s, --speed` - 速度因子
- `-l, --loop` - 循环次数
- `-f, --filter` - 包过滤器
- `--modify-mac` - MAC地址重写
- `--modify-ip` - IP地址重写

#### 3.3 示例
```bash
# 基础回放
ns-replay replay -n replay_ns -m eth0:veth0 traffic.pcap

# 加速回放
ns-replay replay -n replay_ns -m eth0:veth0 -s 2.0 traffic.pcap

# 循环回放
ns-replay replay -n replay_ns -m eth0:veth0 -l 3 traffic.pcap

# 多网卡映射
ns-replay replay -n replay_ns -m eth0:veth0 -m eth1:veth1 traffic.pcap
```

### 4. Namespace子命令

#### 4.1 子命令列表
```bash
ns-replay namespace create NAME
ns-replay namespace delete NAME
ns-replay namespace list
ns-replay namespace info NAME
ns-replay namespace exec NAME COMMAND
```

#### 4.2 示例
```bash
# 创建namespace
ns-replay namespace create test_ns

# 列举namespace
ns-replay namespace list

# 查看信息
ns-replay namespace info test_ns

# 执行命令
ns-replay namespace exec test_ns "ip addr show"

# 删除namespace
ns-replay namespace delete test_ns
```

### 5. Network子命令

#### 5.1 子命令列表
```bash
ns-replay network list
ns-replay network info INTERFACE
ns-replay network export [OPTIONS]
```

#### 5.2 示例
```bash
# 列举网卡
ns-replay network list

# 查看网卡详情
ns-replay network info eth0

# 导出配置
ns-replay network export -o network_config.yaml

# 导出指定namespace的配置
ns-replay network export --namespace test_ns -o ns_config.yaml
```

### 6. Run子命令

#### 6.1 完整流程
```bash
ns-replay run [OPTIONS]
```

#### 6.2 功能
- 根据配置文件执行完整流程
- 自动创建namespace
- 抓取流量（可选）
- 回放流量
- 清理资源

#### 6.3 示例
```bash
# 使用配置文件
ns-replay run -c scenario.yaml

# 交互式模式
ns-replay run --interactive
```

### 7. Config子命令

#### 7.1 子命令列表
```bash
ns-replay config validate FILE
ns-replay config template [SCENARIO]
ns-replay config show
```

#### 7.2 示例
```bash
# 验证配置
ns-replay config validate scenario.yaml

# 生成模板
ns-replay config template single-interface > config.yaml

# 显示当前配置
ns-replay config show
```

### 8. 交互式功能

#### 8.1 进度显示
- 实时进度条
- 百分比显示
- 速率显示
- 剩余时间估算

#### 8.2 实时统计
- 实时更新统计信息
- 表格化展示
- 颜色高亮

#### 8.3 确认提示
- 危险操作确认
- 资源清理确认
- 覆盖文件确认

### 9. 输出格式

#### 9.1 输出选项
- `--json` - JSON格式输出
- `--yaml` - YAML格式输出
- `--quiet` - 静默模式
- `--verbose` - 详细模式

#### 9.2 颜色支持
- 成功信息（绿色）
- 警告信息（黄色）
- 错误信息（红色）
- 支持`--no-color`选项

## 技术要求

### 依赖库
- click - 命令行框架
- rich - 终端美化（进度条、表格等）
- colorama - 跨平台颜色支持

### 用户体验
- 命令简洁直观
- 帮助信息完整
- 错误提示清晰
- 进度反馈及时

### 兼容性
- 支持bash/zsh自动补全
- 支持管道操作
- 支持脚本调用

## 接口定义

```python
# cli/main.py
import click

@click.group()
@click.version_option()
@click.option('--log-level', default='INFO')
@click.option('--config', type=click.Path())
def cli(log_level, config):
    """Network Namespace Replay Tool"""
    pass

# cli/capture.py
@cli.command()
@click.option('-i', '--interface', multiple=True, required=True)
@click.option('-o', '--output', required=True)
@click.option('-f', '--filter', default='')
@click.option('-d', '--duration', type=int, default=0)
@click.option('-c', '--count', type=int, default=0)
@click.option('--namespace', default=None)
def capture(interface, output, filter, duration, count, namespace):
    """Capture network traffic"""
    pass

# cli/replay.py
@cli.command()
@click.argument('pcap_file')
@click.option('-n', '--namespace', required=True)
@click.option('-m', '--mapping', multiple=True, required=True)
@click.option('-s', '--speed', type=float, default=1.0)
@click.option('-l', '--loop', type=int, default=0)
def replay(pcap_file, namespace, mapping, speed, loop):
    """Replay network traffic"""
    pass

# cli/namespace.py
@cli.group()
def namespace():
    """Manage network namespaces"""
    pass

@namespace.command()
@click.argument('name')
def create(name):
    """Create a namespace"""
    pass

# cli/utils.py
class ProgressDisplay:
    """进度显示"""
    def __init__(self, total: int):
        pass

    def update(self, current: int):
        pass

    def finish(self):
        pass

class TableDisplay:
    """表格显示"""
    def __init__(self, headers: list):
        pass

    def add_row(self, row: list):
        pass

    def render(self):
        pass
```

## 验收标准

### 功能验收
- [ ] 所有子命令正常工作
- [ ] 帮助信息完整准确
- [ ] 参数验证正确
- [ ] 错误处理友好
- [ ] 进度显示正常

### 用户体验
- [ ] 命令直观易用
- [ ] 输出格式清晰
- [ ] 错误提示有帮助
- [ ] 交互流畅

### 文档要求
- [ ] 每个命令都有帮助文档
- [ ] 提供使用示例
- [ ] 编写用户手册

## 依赖关系

### 被依赖模块
- 无（顶层模块）

### 依赖模块
- utils
- config
- network
- namespace
- capture
- replay

## 开发计划

### 第一阶段（基础命令）
- 主命令框架
- capture子命令
- replay子命令

### 第二阶段（管理命令）
- namespace子命令
- network子命令
- config子命令

### 第三阶段（增强功能）
- run子命令
- 进度显示
- 交互式功能

### 第四阶段（优化完善）
- 自动补全
- 用户手册
- 示例脚本

## 风险和挑战

1. **用户体验**：命令设计需要平衡功能和易用性
2. **错误处理**：需要提供清晰的错误信息和解决建议
3. **跨平台**：终端特性的跨平台兼容性
4. **性能**：大量输出时的性能问题

## 参考资料

- click文档
- rich文档
- CLI设计最佳实践
- 其他优秀CLI工具（docker, kubectl等）
