# WSL 环境下 Hermes 连接 Windows Ollama 配置指南

## 问题描述

当 Hermes 安装在 WSL (Windows Subsystem for Linux) 的 Ubuntu 环境中，而 Ollama 安装在 Windows 主机环境时，直接使用 `http://localhost:11434` 无法访问 Ollama。

这是因为 WSL 和 Windows 是两个独立的网络环境，`localhost` 在 WSL 中指向的是 Ubuntu 本身，而非 Windows 主机。

## 解决方案

使用 **Windows 主机的 WSL 虚拟网卡 IP 地址** 代替 `localhost` 来访问 Ollama。

---

## 配置步骤

### 1. 获取 Windows 主机 IP 地址

在 WSL 终端中执行以下命令：

```bash
ip route | grep default | awk '{print $3}'
```

典型输出示例：
```
172.XX.XX.1
```

> 注意：这是示例IP格式，请替换为你实际获取的IP地址。

> **注意**：每次重启 WSL 后这个 IP 通常是稳定的，但如果网络配置发生变化可能需要重新确认。

### 2. 验证 Ollama 可访问性

使用获取到的 IP 地址测试连接：

```bash
curl http://YOUR_WINDOWS_IP:11434/api/tags
```

将 `YOUR_WINDOWS_IP` 替换为上一步获取的实际 IP 地址。

**成功响应示例：**
```json
{"models":[{"name":"gemma3:4b","modified_at":"2025-04-10T10:30:00Z","size":3000000000}]}
```

如果命令超时或无响应，请继续完成以下防火墙和 Ollama 配置步骤。

---

## Windows 防火墙配置

Windows 默认会阻止来自 WSL 的网络连接，需要放行 11434 端口。

### 方案一：PowerShell 命令（推荐）

以**管理员身份**打开 PowerShell，执行：

```powershell
New-NetFirewallRule -DisplayName "Ollama WSL" -Direction Inbound -Protocol TCP -LocalPort 11434 -Action Allow
```

### 方案二：图形界面操作

1. 打开 **Windows 设置** → **更新与安全** → **Windows 安全中心** → **防火墙和网络保护**
2. 点击 **高级设置**
3. 选择 **入站规则** → **新建规则**
4. 选择 **端口** → 下一步
5. 选择 **TCP**，特定本地端口填入 `11434`
6. 选择 **允许连接**
7. 配置文件全选（域、专用、公用）
8. 名称填写 `Ollama WSL`，完成

---

## Ollama 监听配置

默认情况下 Ollama 只监听 `localhost`，需要配置为监听所有网络接口。

### 设置环境变量

在 Windows 系统中添加环境变量：

| 变量名 | 变量值 |
|--------|--------|
| `OLLAMA_HOST` | `0.0.0.0` |

**操作步骤：**
1. 右键点击 **此电脑** → **属性** → **高级系统设置**
2. 点击 **环境变量**
3. 在 **系统变量** 区域点击 **新建**
4. 填入上述变量名和变量值
5. 点击确定保存

### 重启 Ollama

1. 在任务栏找到 Ollama 图标，右键选择 **退出**
2. 重新启动 Ollama 应用程序

---

## Hermes 配置

### 配置文件示例

在项目配置文件中设置 Ollama URL：

```json
{
  "ollama_url": "http://YOUR_WINDOWS_IP:11434",
  "model": "gemma3:4b"
}
```

### 环境变量方式

```bash
export OLLAMA_URL="http://YOUR_WINDOWS_IP:11434"
```

### Python 代码示例

```python
import os

OLLAMA_URL = os.getenv("OLLAMA_URL", "http://YOUR_WINDOWS_IP:11434")

# 或者在配置文件中读取
config = {
    "ollama_url": "http://YOUR_WINDOWS_IP:11434",
    "model": "gemma3:4b"
}
```

---

## 完整验证流程

```bash
# 1. 获取 Windows IP
export WINDOWS_IP=$(ip route | grep default | awk '{print $3}')
echo "Windows IP: $WINDOWS_IP"

# 2. 测试 Ollama 连接
curl http://${WINDOWS_IP}:11434/api/tags

# 3. 配置 Hermes 环境变量
export OLLAMA_URL="http://${WINDOWS_IP}:11434"

# 4. 启动 Hermes 或相关应用
python app.py
```

---

## 常见问题

### Q: IP 地址会变化吗？
**A:** 在正常使用过程中，WSL 虚拟网卡的 IP 地址通常是稳定的。但如果重启 WSL 或修改网络配置，可能需要重新确认。

### Q: 每次都要手动查 IP 吗？
**A:** 可以在启动脚本中动态获取：
```bash
export OLLAMA_URL="http://$(ip route | grep default | awk '{print $3}'):11434"
```

### Q: 防火墙配置后仍然连不上？
**A:** 请检查：
1. Ollama 是否正确设置了 `OLLAMA_HOST=0.0.0.0`
2. Ollama 是否已重启
3. Windows 防火墙规则是否配置正确（入站规则、TCP、端口 11434）
4. 使用 `curl -v` 查看详细连接信息

### Q: 可以直接在 WSL 里安装 Ollama 吗？
**A:** 可以，那样就不需要这些配置。但 Windows 版本的 Ollama 通常性能更好（GPU 支持更完善），推荐保持 Windows 安装并使用本指南进行配置。

---

## 网络架构说明

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     Windows 主机                             │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                 WSL (Ubuntu)                         │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │              Hermes                          │   │   │
│  │  │   OLLAMA_URL=http://172.XX.XX.1:11434       │   │   │
│  │  └───────────────────────┼─────────────────────┘   │   │
│  │                        │                           │   │
│  │         WSL 虚拟网卡   │  172.XX.XX.2            │   │
│  └───────────────────────────┼─────────────────────────────────┘   │
│                          │                                 │
│  虚拟交换机              │  172.XX.XX.1                   │
│                          │                                 │
│  ┌───────────────────────────┼────────────────────────────────────────────┐ │
│  │                    Ollama Server                      │ │
│  │              监听 0.0.0.0:11434                       │ │
│  └─────────────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 相关链接

- [Ollama 官方文档](https://github.com/ollama/ollama)
- [WSL 网络文档](https://docs.microsoft.com/en-us/windows/wsl/networking)

---

**作者:** Hermes  
**创建时间:** 2025-04-14  
**适用环境:** WSL2 + Windows 10/11 + Ollama
