# Nginx 反向代理详解

Nginx 作为整个 Local AI OS 的统一入口，将局域网请求路由到正确的后端服务。本文档涵盖安装、完整配置、服务管理、高级用法和故障排查。

---

## 一、安装

### 1.1 下载

前往 [nginx.org/en/download.html](http://nginx.org/en/download.html)，下载 **Windows 稳定版** zip（如 `nginx-1.26.x.zip`），解压到 `C:\nginx\`。

```powershell
# 验证
cd C:\nginx
.\nginx.exe -v
# nginx version: nginx/1.26.x

# 检查配置语法
.\nginx.exe -t
```

### 1.2 注册为 Windows 服务（NSSM）

使用 [NSSM](https://nssm.cc/)（Non-Sucking Service Manager）将 Nginx 注册为开机自启服务：

```powershell
# 安装 NSSM
winget install nssm

# 管理员 PowerShell
nssm install Nginx "C:\nginx\nginx.exe"
nssm set Nginx AppDirectory "C:\nginx"
nssm set Nginx AppParameters ""
nssm set Nginx Start SERVICE_AUTO_START
nssm set Nginx AppStdout "C:\nginx\logs\service.log"
nssm set Nginx AppStderr "C:\nginx\logs\service-error.log"

Start-Service Nginx
Get-Service Nginx   # 确认状态为 Running
```

---

## 二、完整配置文件

`C:\nginx\conf\nginx.conf`：

```nginx
worker_processes  1;

error_log  logs/error.log warn;
pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent"';

    access_log  logs/access.log  main;

    sendfile        on;
    keepalive_timeout  65;

    # AI 推理超时（生成长文本可能需要数分钟）
    proxy_read_timeout    300s;
    proxy_connect_timeout  30s;
    proxy_send_timeout    300s;

    # 上传文件大小限制（上传知识库文档）
    client_max_body_size  100m;

    # 通用代理头部
    proxy_set_header  Host             $host;
    proxy_set_header  X-Real-IP        $remote_addr;
    proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;

    # ── 上游服务定义 ──────────────────────────────────────────
    # Mirrored 网络模式下，WSL 服务直接用 127.0.0.1
    # NAT 模式下，替换为实际 WSL IP（见 wsl_network.md）

    upstream ollama_win {
        server 127.0.0.1:11435;   # Windows 核显·Embedding
    }

    upstream ollama_wsl {
        server 127.0.0.1:11434;   # WSL 独显·大模型推理
    }

    upstream open_webui {
        server 127.0.0.1:8080;    # WSL Open-WebUI Agent UI
    }

    upstream qdrant {
        server 127.0.0.1:6333;    # WSL Qdrant 向量库
    }

    upstream openhands {
        server 127.0.0.1:3000;    # WSL OpenHands 自主编码 Agent
    }

    # ── HTTP 服务器 ────────────────────────────────────────────
    server {
        listen       80;
        server_name  localhost;

        # ── Windows Ollama（Embedding 服务）─────────────────────
        location /ollama-win/ {
            rewrite ^/ollama-win/(.*) /$1 break;
            proxy_pass http://ollama_win;
        }

        # ── WSL Ollama（推理服务）────────────────────────────────
        location /ollama-wsl/ {
            rewrite ^/ollama-wsl/(.*) /$1 break;
            proxy_pass http://ollama_wsl;
        }

        # ── Open-WebUI Agent UI（需要 WebSocket）──────────────────
        location /open-webui/ {
            rewrite ^/open-webui/(.*) /$1 break;
            proxy_pass http://open_webui;
            # WebSocket 升级
            proxy_http_version 1.1;
            proxy_set_header   Upgrade    $http_upgrade;
            proxy_set_header   Connection "upgrade";
        }

        # ── OpenHands 自主编码 Agent（需要 WebSocket）───────────────
        location /openhands/ {
            rewrite ^/openhands/(.*) /$1 break;
            proxy_pass         http://openhands;
            # WebSocket 支持（Agent 实时流式输出）
            proxy_http_version 1.1;
            proxy_set_header   Upgrade    $http_upgrade;
            proxy_set_header   Connection "upgrade";
            proxy_read_timeout 600s;
        }

        # ── Qdrant REST API ──────────────────────────────────────
        location /qdrant/ {
            rewrite ^/qdrant/(.*) /$1 break;
            proxy_pass http://qdrant;
        }

        # ── 健康检查 ─────────────────────────────────────────────
        location /health {
            return 200 "ok\n";
            add_header Content-Type text/plain;
        }

        # ── 状态页（仅本地访问）──────────────────────────────────
        location /nginx_status {
            stub_status on;
            allow 127.0.0.1;
            deny  all;
        }
    }
}
```

---

## 三、服务管理命令

```powershell
# 启动 / 停止 / 重启
Start-Service Nginx
Stop-Service Nginx
Restart-Service Nginx

# 热更新配置（不中断连接）
C:\nginx\nginx.exe -s reload

# 检查配置语法（更新前必做）
C:\nginx\nginx.exe -t

# 查看进程
Get-Process nginx

# 查看日志
Get-Content C:\nginx\logs\error.log -Tail 50
Get-Content C:\nginx\logs\access.log -Tail 50
```

---

## 四、防火墙配置

```powershell
# 放行 HTTP 80 端口（允许局域网访问）
New-NetFirewallRule -DisplayName "Nginx HTTP" `
    -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow

# 如果需要 HTTPS
New-NetFirewallRule -DisplayName "Nginx HTTPS" `
    -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow

# 查看已有规则
Get-NetFirewallRule -DisplayName "Nginx*" | Select-Object Name, Enabled, Direction
```

---

## 五、验证各路由

```powershell
$IP = "<主笔记本局域网IP>"

# 健康检查
Invoke-RestMethod "http://$IP/health"

# Windows Embedding 路由
Invoke-RestMethod "http://$IP/ollama-win/api/tags"

# WSL 推理路由
Invoke-RestMethod "http://$IP/ollama-wsl/api/tags"

# Qdrant
Invoke-RestMethod "http://$IP/qdrant/collections"
```

```bash
# 从 MBP 验证
curl http://<主笔记本IP>/ollama-wsl/api/tags | python3 -m json.tool
```

---

## 六、高级配置

### 6.1 限速（防止单客户端占用全部带宽）

```nginx
# 在 http{} 块中添加
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

# 在 location 中应用
location /ollama-wsl/ {
    limit_req zone=api burst=20 nodelay;
    # ...
}
```

### 6.2 简单认证（保护 API 访问）

```bash
# 生成密码文件（在 WSL 中）
sudo apt install -y apache2-utils
htpasswd -c /tmp/nginx_passwd admin
# 将生成的文件复制到 C:\nginx\conf\passwd
```

```nginx
# 在需要保护的 location 中添加
location /qdrant/ {
    auth_basic "AI OS API";
    auth_basic_user_file conf/passwd;
    # ...
}
```

### 6.3 NAT 模式下动态更新 WSL IP

当使用 NAT 网络模式时，WSL IP 重启后可能变化，使用脚本自动更新 Nginx 上游配置：

```powershell
# update-nginx-wsl-ip.ps1
$wslIp = (wsl hostname -I).Trim().Split(' ')[0]
$confPath = "C:\nginx\conf\nginx.conf"

$content = Get-Content $confPath -Raw
# 替换 ollama_wsl、open_webui、qdrant、openhands 上游的 IP
$content = $content -replace '(upstream ollama_wsl \{\s+server )[\d.]+(:11434)', "`${1}$wslIp`${2}"
$content = $content -replace '(upstream open_webui \{\s+server )[\d.]+(:8080)', "`${1}$wslIp`${2}"
$content = $content -replace '(upstream qdrant \{\s+server )[\d.]+(:6333)', "`${1}$wslIp`${2}"
$content = $content -replace '(upstream openhands \{\s+server )[\d.]+(:3000)', "`${1}$wslIp`${2}"
Set-Content $confPath $content

& "C:\nginx\nginx.exe" -t  # 验证语法
& "C:\nginx\nginx.exe" -s reload
Write-Host "Nginx updated for WSL IP: $wslIp"
```

### 6.4 HTTPS 配置（使用自签证书）

```powershell
# 生成自签证书（PowerShell）
$cert = New-SelfSignedCertificate `
    -DnsName "localhost", "192.168.1.x" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -NotAfter (Get-Date).AddYears(10)

# 导出 PEM 格式（Nginx 使用）
$certPath = "C:\nginx\conf\cert.pem"
$keyPath  = "C:\nginx\conf\key.pem"
# 使用 openssl 或其他工具导出...
```

```nginx
# nginx.conf 中的 HTTPS server 块
server {
    listen       443 ssl;
    server_name  localhost;

    ssl_certificate      conf/cert.pem;
    ssl_certificate_key  conf/key.pem;
    ssl_protocols        TLSv1.2 TLSv1.3;

    # ... 其余 location 配置同 HTTP
}

# HTTP 重定向到 HTTPS
server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

---

## 七、故障排查

| 症状 | 可能原因 | 解决方法 |
|------|---------|---------|
| 502 Bad Gateway | 后端服务未启动 | 检查 `systemctl status ollama` / `docker ps` |
| 504 Gateway Timeout | 推理超时 | 增大 `proxy_read_timeout`（已设 300s） |
| 403 Forbidden | 路径配置错误 | 检查 `rewrite` 规则，确保路径正确 |
| 端口被占用 | 其他程序占用 80 | `netstat -ano \| findstr :80` 找到并终止 |
| WebSocket 连接失败 | 缺少 Upgrade 头 | 确认 `open-webui` location 有 WebSocket 配置 |
| OpenHands Agent 无响应 | 容器未启动或 Ollama 断连 | `docker compose logs openhands`，确认 `LLM_BASE_URL` 用 `host.docker.internal` |
| 日志报 `connect() failed` | WSL IP 变化（NAT 模式） | 运行 `update-nginx-wsl-ip.ps1` |

---

> ⬅️ [返回阶段 0 概览](./README.md)
