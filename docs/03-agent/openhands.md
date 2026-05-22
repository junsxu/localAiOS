# OpenHands：自主编码 Agent 本地部署

> **OpenHands**（原名 OpenDevin）是开源自主编码 Agent 框架，支持自动编写代码、在沙盒中执行、浏览网页、提交 Pull Request。
> 与 Open-WebUI 的定位不同：Open-WebUI 是聊天 + RAG 界面，OpenHands 是能真正"干活"的自主 Agent。

---

## Open-WebUI vs OpenHands 选择指南

| 维度 | Open-WebUI | OpenHands |
|------|-----------|-----------|
| **定位** | 聊天界面 + RAG 知识库 | 自主编码 Agent（写代码 / 执行 / 调试） |
| **交互方式** | 对话式，手动驱动 | 任务式，自主多步执行 |
| **工具调用** | 内置 Web 搜索、文档上传 | 文件读写、终端命令、浏览器、Git |
| **沙箱安全** | 无沙箱 | Docker 沙箱隔离，不影响宿主机 |
| **本地 Ollama** | ✅ 原生支持，开箱即用 | ⚠️ 支持，但需配置，复杂任务依赖模型能力 |
| **推荐模型** | 7B 及以上均可 | 建议 14B+，复杂任务推荐 32B+ |
| **GitHub Stars** | ~65k | ~70k |

**结论**：两者互补，不是替代关系。
- 日常问答、RAG 检索 → **Open-WebUI**
- 让 AI 自动完成编程任务（写代码 → 运行 → 修 bug）→ **OpenHands**

---

## 前置条件

- WSL Ubuntu + Docker（已在 [open_webui.md](./open_webui.md) 中安装）
- Ollama 运行中（`http://localhost:11434`），已拉取 `qwen2.5-coder:14b`
- 可用内存 ≥ 16GB（OpenHands 容器 + Ollama 同时运行）

---

## 一、部署 OpenHands

### 1.1 创建目录与配置

```bash
mkdir -p ~/openhands/workspace
```

### 1.2 docker-compose.yml

```bash
cat > ~/openhands/docker-compose.yml << 'EOF'
version: '3.8'

services:
  openhands:
    image: docker.all-hands.dev/all-hands-ai/openhands:latest
    container_name: openhands
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      # 使用本地 Ollama（通过 LiteLLM 路由）
      - LLM_MODEL=ollama/qwen2.5-coder:14b
      - LLM_BASE_URL=http://host.docker.internal:11434
      - LLM_API_KEY=ollama
      # 沙箱运行时（Docker-in-Docker）
      - SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:latest
    volumes:
      - ./workspace:/opt/workspace_base
      - /var/run/docker.sock:/var/run/docker.sock
    extra_hosts:
      - "host.docker.internal:host-gateway"
EOF
```

### 1.3 启动

```bash
cd ~/openhands
docker compose pull
docker compose up -d

# 查看启动日志（首次启动需拉取 runtime 镜像，约 1-2 分钟）
docker compose logs -f openhands
```

访问 `http://localhost:3000`，等待"Agent Ready"提示。

---

## 二、连接 Ollama 模型

OpenHands 通过 [LiteLLM](https://docs.litellm.ai/) 转发请求到 Ollama，`LLM_MODEL` 格式为 `ollama/<模型名>`。

### 验证连接

```bash
# 确认 Ollama 已加载目标模型
curl http://localhost:11434/api/tags | python3 -m json.tool | grep "name"

# 在 OpenHands 界面左上角可切换模型
# Settings → LLM → Model: ollama/qwen2.5-coder:14b
```

### 可用模型建议

| 任务类型 | 推荐模型 | 显存需求 |
|---------|---------|---------|
| 简单代码修改 | `qwen2.5-coder:7b` | ~5GB |
| 常规编程任务 | `qwen2.5-coder:14b` | ~9GB |
| 复杂架构设计 | `qwen2.5-coder:32b` | 需更大显存 |

> RTX 5070Ti 12GB 显存运行 14B 模型时，OpenHands 处理中等复杂任务（<5步）表现较好；复杂任务（>10步）可能出现思路偏移，需人工干预。

---

## 三、基础使用

### 3.1 任务示例

在对话框输入任务描述，OpenHands 会自动规划并执行：

```
在 /opt/workspace_base 目录下创建一个 FastAPI 应用，
实现 /health 和 /echo 两个接口，并包含单元测试。
```

OpenHands 会自动：
1. **思考（Think）**：规划文件结构
2. **执行（Act）**：创建文件、写入代码
3. **验证（Observe）**：运行测试，查看输出
4. **修正（Fix）**：如有报错自动修改

### 3.2 工作目录

OpenHands 在 Docker 沙箱中操作 `/opt/workspace_base`，映射到宿主机的 `~/openhands/workspace`：

```bash
# 将你的项目复制进工作目录
cp -r ~/my-project ~/openhands/workspace/

# OpenHands 完成后从这里取出结果
ls ~/openhands/workspace/
```

### 3.3 Git 集成（可选）

在工作目录内 clone 仓库，让 OpenHands 直接操作 Git：

```bash
cd ~/openhands/workspace
git clone ssh://git@<NAS-IP>:2224/<用户名>/my-project.git
```

OpenHands 可以自动完成：读代码 → 修改 → 运行测试 → 提交 commit。

---

## 四、与 Open-WebUI 的协同使用

两者可以**同时运行**，端口不冲突（Open-WebUI `:8080`，OpenHands `:3000`）：

```
工作流建议：

1. Open-WebUI（:8080）
   ├── 日常问答与知识库检索（RAG）
   ├── 代码片段生成（复制粘贴）
   └── 理解文档、解释原理

2. OpenHands（:3000）
   ├── 完整功能的实现（告知需求，自动完成）
   ├── 调试存量代码（给出报错，自动修）
   └── 生成测试用例 + 验证
```

---

## 五、systemd 服务配置

```bash
sudo tee /etc/systemd/system/openhands.service << 'EOF'
[Unit]
Description=OpenHands Agent Service
After=network.target docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/<username>/openhands
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
User=<username>

[Install]
WantedBy=multi-user.target
EOF

# 将 <username> 替换为实际用户名
sudo sed -i "s/<username>/$USER/g" /etc/systemd/system/openhands.service

sudo systemctl enable --now openhands
```

---

## 六、Nginx 反向代理（可选）

在 Windows Nginx 配置中添加 OpenHands 路由（参见 [docs/00-infrastructure/nginx.md](../00-infrastructure/nginx.md)）：

```nginx
upstream openhands {
    server 127.0.0.1:3000;
}

# 在 server {} 块中添加
location /openhands/ {
    rewrite ^/openhands/(.*) /$1 break;
    proxy_pass         http://openhands;
    proxy_set_header   Host $host;
    proxy_set_header   X-Real-IP $remote_addr;
    # WebSocket 支持（Agent 实时流式输出需要）
    proxy_http_version 1.1;
    proxy_set_header   Upgrade $http_upgrade;
    proxy_set_header   Connection "upgrade";
    proxy_read_timeout 600s;
}
```

---

## 七、故障排查

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| "Initializing agent" 卡住不动 | Ollama 连接失败 | 检查 `LLM_BASE_URL`，确认用 `host.docker.internal` 而非 `localhost` |
| 任务开始后无响应 | 模型太小或上下文超限 | 换用 14B 模型，检查 `ollama ps` 显存占用 |
| 沙箱容器无法拉取 | 网络问题 | 提前手动 `docker pull docker.all-hands.dev/all-hands-ai/runtime:latest` |
| 写出的代码不正确 | 本地 14B 能力限制 | 拆分任务为更小步骤，或升级到 32B |
| 端口 3000 冲突 | 其他服务占用 | 修改 `docker-compose.yml` 中宿主机端口映射 |

---

## 参考

- [OpenHands 官方文档](https://docs.all-hands.dev/)
- [OpenHands GitHub](https://github.com/All-Hands-AI/OpenHands)
- [LiteLLM Ollama 配置](https://docs.litellm.ai/docs/providers/ollama)
