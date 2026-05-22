# WSL 网络配置详解

WSL2 提供三种主要网络模式，本文档逐一说明原理、配置步骤和适用场景。

---

## 模式对比

| 特性 | NAT（默认） | Mirrored（推荐） | Bridge |
|------|------------|----------------|--------|
| 系统要求 | 所有 Windows | Windows 11 22H2+ / WSL ≥ 2.0 | 手动配置 |
| WSL IP | 动态虚拟 IP | 与 Windows 共享 | 自定义 |
| Windows → WSL | 需知道 WSL IP | `127.0.0.1` 直达 | 自定义路由 |
| 局域网 → WSL | 需端口转发 | 直接访问 Windows IP | 需路由器配置 |
| Nginx 配置 | 需动态更新 WSL IP | 固定 `127.0.0.1` | 固定 IP |
| 推荐场景 | Windows 10 / 旧机器 | Windows 11（本项目推荐） | 特殊网络需求 |

---

## 一、Mirrored 模式（推荐）

### 原理

WSL 与 Windows 共享同一网络命名空间。WSL 内的服务监听在 `0.0.0.0` 时，Windows 可通过 `127.0.0.1` 直接访问；局域网其他设备通过 Windows 的局域网 IP 访问，无需任何端口转发。

```
局域网设备  →  Windows IP:80  →  Nginx  →  127.0.0.1:11434 (WSL Ollama)
                                        →  127.0.0.1:6333  (WSL Qdrant)
                                        →  127.0.0.1:8080  (WSL Open-WebUI)
                                        →  127.0.0.1:3000  (WSL OpenHands)
```

### 系统要求

- Windows 11 22H2（Build 22621）或更高
- WSL 版本 ≥ 2.0（`wsl --version` 查看）

```powershell
# 更新 WSL 到最新版本
wsl --update
wsl --version
```

### 配置步骤

**1. 编辑 `%USERPROFILE%\.wslconfig`**

```ini
[wsl2]
# 启用 Mirrored 网络
networkingMode=mirrored

# 可选：自动代理（继承 Windows 代理设置）
autoProxy=true

# 可选：DNS 隧道（解决某些企业网络 DNS 问题）
dnsTunneling=true

# 可选：防火墙集成（WSL 与 Windows 共用 Defender 规则）
firewall=true

# 资源限制（按实际硬件调整）
memory=20GB
processors=12
swap=8GB
```

**2. 编辑 WSL 内的 `/etc/wsl.conf`**

```bash
sudo tee /etc/wsl.conf <<'EOF'
[boot]
# 启用 systemd（Ollama、Qdrant 需要）
systemd=true

[network]
hostname=wsl-ubuntu
generateResolvConf=true

[interop]
enabled=true
appendWindowsPath=true

[automount]
enabled=true
root=/mnt/
options="metadata,umask=22,fmask=11"
mountFsTab=true
EOF
```

**3. 重启 WSL**

```powershell
wsl --shutdown
# 等待约 5 秒后重新打开 WSL
wsl
```

### 验证

```bash
# 在 WSL 内查看 IP，应与 Windows 局域网 IP 相同
ip addr show eth0

# 从 Windows 访问 WSL 服务（不需要知道 WSL IP）
curl http://127.0.0.1:11434/api/tags    # 假设 WSL Ollama 已启动
```

```powershell
# 从 Windows PowerShell 访问 WSL 服务
Invoke-RestMethod "http://127.0.0.1:11434/api/tags"
```

### 已知限制

- 某些 VPN 软件（如 Cisco AnyConnect）可能与 Mirrored 模式冲突，表现为 WSL 网络断开。临时解决：断开 VPN 后 `wsl --shutdown` 再重启。
- Windows 10 不支持，需降级到 NAT 模式。

---

## 二、NAT 模式（备选）

### 原理

WSL2 默认使用虚拟以太网（vEthernet WSL）进行 NAT。WSL 拥有独立的虚拟 IP（通常 `172.x.x.x`），每次重启可能变化。Windows 访问 WSL 需要知道当前 WSL IP，局域网设备访问 WSL 需要配置端口转发。

```
局域网设备  →  Windows IP:11434 (端口转发)  →  172.x.x.x:11434 (WSL)
```

### 获取 WSL IP

```powershell
# 方法 1：从 Windows PowerShell
wsl hostname -I
# 输出: 172.20.x.x

# 方法 2：获取 WSL 虚拟网卡 IP
Get-NetIPAddress -InterfaceAlias "vEthernet (WSL*)" -AddressFamily IPv4 | Select-Object IPAddress

# 方法 3：从 WSL 内部获取
ip route show | grep -i default | awk '{print $3}'  # 获取 Windows 侧 IP（网关）
```

### 配置端口转发

```powershell
# 管理员 PowerShell
$wslIp = (wsl hostname -I).Trim().Split(' ')[0]

# 转发 Ollama
netsh interface portproxy add v4tov4 `
    listenport=11434 listenaddress=0.0.0.0 `
    connectport=11434 connectaddress=$wslIp

# 转发 Qdrant
netsh interface portproxy add v4tov4 `
    listenport=6333 listenaddress=0.0.0.0 `
    connectport=6333 connectaddress=$wslIp

# 转发 Open-WebUI
netsh interface portproxy add v4tov4 `
    listenport=8080 listenaddress=0.0.0.0 `
    connectport=8080 connectaddress=$wslIp

# 转发 OpenHands
netsh interface portproxy add v4tov4 `
    listenport=3000 listenaddress=0.0.0.0 `
    connectport=3000 connectaddress=$wslIp

# 查看所有规则
netsh interface portproxy show all

# 删除某条规则
netsh interface portproxy delete v4tov4 listenport=11434 listenaddress=0.0.0.0
```

### 防火墙放行

```powershell
# 放行需要转发的端口
@(11434, 6333, 8080, 3000) | ForEach-Object {
    New-NetFirewallRule -DisplayName "WSL Port $_" `
        -Direction Inbound -Protocol TCP -LocalPort $_ -Action Allow
}
```

### 动态 IP 更新脚本

WSL 重启后 IP 会变化，使用此脚本自动更新转发规则：

```powershell
# update-wsl-ports.ps1
$wslIp = (wsl hostname -I).Trim().Split(' ')[0]
Write-Host "WSL IP: $wslIp"

# 删除旧规则
netsh interface portproxy reset

# 重新添加
@(
    @{port=11434; desc="Ollama"},
    @{port=6333;  desc="Qdrant"},
    @{port=8080;  desc="Open-WebUI"},
    @{port=3000;  desc="OpenHands"}
) | ForEach-Object {
    netsh interface portproxy add v4tov4 `
        listenport=$($_.port) listenaddress=0.0.0.0 `
        connectport=$($_.port) connectaddress=$wslIp
    Write-Host "Forwarded $($_.desc) :$($_.port) -> $wslIp:$($_.port)"
}

# 更新 Nginx 配置中的 WSL IP（如果 Nginx 直接写死了 WSL IP）
$confPath = "C:\nginx\conf\nginx.conf"
if (Test-Path $confPath) {
    (Get-Content $confPath) -replace '172\.\d+\.\d+\.\d+', $wslIp | Set-Content $confPath
    & "C:\nginx\nginx.exe" -s reload
    Write-Host "Nginx config updated and reloaded"
}
```

**开机自动运行**：将脚本注册为任务计划程序触发（登录时运行）：

```powershell
$action = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument "-File C:\scripts\update-wsl-ports.ps1"
$trigger = New-ScheduledTaskTrigger -AtLogOn
Register-ScheduledTask -TaskName "UpdateWSLPorts" -Action $action -Trigger $trigger -RunLevel Highest
```

---

## 三、从 WSL 内访问 Windows 服务

无论哪种网络模式，在 WSL 内访问 Windows 侧服务（如 Windows 上的 Ollama :11435）：

```bash
# Mirrored 模式：直接使用 127.0.0.1
curl http://127.0.0.1:11435/api/tags

# NAT 模式：使用 Windows 网关 IP（DNS nameserver）
WINDOWS_IP=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
curl http://$WINDOWS_IP:11435/api/tags

# 或从路由表获取
WINDOWS_GW=$(ip route show | grep default | awk '{print $3}')
curl http://$WINDOWS_GW:11435/api/tags
```

---

## 四、WSL 服务开机自启

启用 systemd（`/etc/wsl.conf` 中 `systemd=true`）后，用标准 systemctl 管理：

```bash
sudo systemctl enable ollama
sudo systemctl enable qdrant
sudo systemctl enable open-webui
sudo systemctl enable openhands

# 验证
sudo systemctl list-units --type=service --state=enabled | grep -E "ollama|qdrant|open-webui|openhands"
```

---

## 五、网络诊断命令

```bash
# WSL 内查看网络接口
ip addr
ip route

# 测试 WSL → Windows 连通性（NAT 模式）
ping $(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')

# 测试 WSL 内服务监听
ss -tlnp | grep -E "11434|6333|8080|3000"

# 测试从 Windows 访问 WSL 服务
# PowerShell:
Test-NetConnection -ComputerName 127.0.0.1 -Port 11434
```

---

> ⬅️ [返回阶段 0 概览](./README.md)
