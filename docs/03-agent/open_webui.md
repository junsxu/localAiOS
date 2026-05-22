# Open-WebUI 部署详解

Open-WebUI 是功能完整的 AI 前端界面，支持多模型切换、RAG 知识库、Tool 调用和 Agent 工作流，部署在 WSL 中通过 Docker 运行。

---

## 一、前置条件

- WSL Ubuntu，已启用 systemd
- Ollama 已部署并运行（`http://localhost:11434`）
- Qdrant 已部署（`http://localhost:6333`，可选）
- Docker 已安装

---

## 二、安装 Docker

```bash
# 官方脚本安装
curl -fsSL https://get.docker.com | sh

# 将当前用户加入 docker 组（免 sudo）
sudo usermod -aG docker $USER
newgrp docker   # 立即生效（或重新登录）

# 验证
docker --version
docker run --rm hello-world
```

---

## 三、部署 Open-WebUI

### 3.1 完整 docker-compose.yml

```bash
mkdir -p ~/open-webui/data

cat > ~/open-webui/docker-compose.yml << 'EOF'
version: '3.8'
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      # Ollama 连接（使用 host.docker.internal 访问宿主机服务）
      - OLLAMA_BASE_URL=http://host.docker.internal:11434

      # RAG 配置（使用 Ollama 的 Embedding）
      - RAG_EMBEDDING_ENGINE=ollama
      - RAG_OLLAMA_BASE_URL=http://host.docker.internal:11434
      - RAG_EMBEDDING_MODEL=nomic-embed-text

      # 启用联网搜索（Tools 功能）
      - ENABLE_RAG_WEB_SEARCH=true
      - RAG_WEB_SEARCH_ENGINE=duckduckgo

      # 安全密钥（务必修改为随机字符串）
      - WEBUI_SECRET_KEY=please-change-this-to-a-random-secret

      # 可选：禁用匿名注册（上线后建议开启）
      # - WEBUI_AUTH=true
      # - DEFAULT_USER_ROLE=user

    volumes:
      - ./data:/app/backend/data

    # 允许容器访问宿主机（WSL）的服务
    extra_hosts:
      - "host.docker.internal:host-gateway"
EOF
```

### 3.2 启动与验证

```bash
cd ~/open-webui
docker compose up -d

# 查看启动日志（首次拉取镜像需要几分钟）
docker compose logs -f open-webui

# 验证
curl http://localhost:8080/health
```

浏览器访问 `http://localhost:8080` 或通过 Nginx `http://<Windows-IP>/open-webui/`，首次访问创建管理员账户。

---

## 四、连接 Ollama 模型

### 4.1 验证 Ollama 连接

`Settings（齿轮图标）→ Connections → Ollama API`：
- URL 确认为 `http://host.docker.internal:11434`
- 点击 **Verify Connection**，显示绿色勾号

### 4.2 选择默认模型

`Settings → Models`（或对话框左上角下拉菜单）：
- 主力：`qwen2.5-coder:14b`（代码生成）
- 快速：`qwen2.5-coder:7b`（快速对话）

---

## 五、RAG 知识库配置

### 5.1 上传文档

`Workspace → Documents → 上传（+）`：
- 支持：PDF、Word、Markdown、TXT、CSV、HTML
- 单文件大小限制：默认 50MB（可在环境变量 `MAX_FILE_SIZE` 中修改）

### 5.2 创建知识库集合

1. `Workspace → Knowledge → + New Knowledge`
2. 命名（如 `技术文档库`），选择上传的文档
3. 保存后等待 Embedding 处理完成

### 5.3 在对话中使用知识库

```
对话框输入：#技术文档库 如何配置 Nginx 反向代理？
```

或在对话设置中启用知识库，后续所有对话自动检索。

### 5.4 RAG 参数调整

`Settings → RAG`：

| 参数 | 说明 | 建议值 |
|------|------|--------|
| Top K | 检索片段数量 | 5 |
| Chunk Size | 文档切块大小（字符） | 500 |
| Chunk Overlap | 切块重叠 | 100 |
| Relevance Threshold | 相关性阈值（低于此值的片段不使用） | 0.5 |

---

## 六、Tools 配置

### 6.1 内置 Tools

`Workspace → Tools`：

| Tool | 说明 | 是否需要额外配置 |
|------|------|----------------|
| Web Search | 联网搜索（DuckDuckGo） | 无需 API Key |
| Code Interpreter | 在沙盒中执行 Python | 无 |
| Image Generation | 连接 AUTOMATIC1111 等 | 需要 SD WebUI |

### 6.2 在对话中启用 Tool

对话框右侧工具图标 → 勾选需要的 Tool，或在模型设置中默认启用。

### 6.3 自定义 Tool（Python）

在 `Workspace → Tools → + New Tool` 中编写 Python 函数：

```python
"""
tool_rag_search.py - 自定义知识库搜索 Tool
"""
import requests

def search_local_knowledge(query: str) -> str:
    """
    在本地 RAG 知识库中搜索信息
    :param query: 搜索问题或关键词
    :return: 检索到的相关文本片段
    """
    # 获取 Embedding
    embed_resp = requests.post(
        "http://host.docker.internal:11435/api/embeddings",
        json={"model": "nomic-embed-text", "prompt": query}
    )
    vector = embed_resp.json()["embedding"]

    # Qdrant 检索
    search_resp = requests.post(
        "http://host.docker.internal:6333/collections/ai-docs/points/search",
        json={"vector": vector, "limit": 3, "with_payload": True}
    )
    results = search_resp.json().get("result", [])

    if not results:
        return "知识库中未找到相关信息。"

    return "\n\n".join(
        f"[{r['payload'].get('file_name', '未知来源')}]\n{r['payload']['text']}"
        for r in results
    )
```

---

## 七、Pipelines（工作流）

Open-WebUI 支持 Pipelines，可以实现复杂的 Agent 工作流，在请求到达 LLM 之前插入处理逻辑：

```bash
# 启动 Pipelines 服务
cat >> ~/open-webui/docker-compose.yml << 'EOF'

  pipelines:
    image: ghcr.io/open-webui/pipelines:main
    container_name: pipelines
    restart: unless-stopped
    ports:
      - "9099:9099"
    volumes:
      - ./pipelines:/app/pipelines
    extra_hosts:
      - "host.docker.internal:host-gateway"
EOF

docker compose up -d pipelines
```

`Settings → Connections → Pipelines`：URL 设为 `http://host.docker.internal:9099`。

---

## 八、注册为 systemd 服务

```bash
# 替换 <username> 为实际用户名
sudo tee /etc/systemd/system/open-webui.service << 'EOF'
[Unit]
Description=Open-WebUI Agent UI
After=network.target docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/<username>/open-webui
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
User=<username>

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now open-webui
sudo systemctl status open-webui
```

---

## 九、数据备份与恢复

```bash
# 备份聊天记录、文档、配置
tar -czf ~/open-webui-backup-$(date +%Y%m%d).tar.gz ~/open-webui/data/

# 同步到 NAS
cp ~/open-webui-backup-*.tar.gz /mnt/nas/backups/

# 恢复
cd ~/open-webui
tar -xzf ~/open-webui-backup-20240115.tar.gz
docker compose restart open-webui
```

---

## 十、常用管理命令

```bash
# 查看容器状态
docker ps | grep open-webui

# 查看日志
docker compose -f ~/open-webui/docker-compose.yml logs -f open-webui

# 重启
docker compose -f ~/open-webui/docker-compose.yml restart open-webui

# 更新到最新版本
docker compose -f ~/open-webui/docker-compose.yml pull open-webui
docker compose -f ~/open-webui/docker-compose.yml up -d open-webui

# 进入容器
docker exec -it open-webui bash
```

---

## 十一、故障排查

| 问题 | 解决方法 |
|------|---------|
| 无法访问 Ollama | `OLLAMA_BASE_URL` 中使用 `host.docker.internal`，而非 `localhost` |
| 文档上传后 Embedding 卡住 | 检查 Embedding 服务是否运行：`curl http://localhost:11435/api/tags` |
| 端口 8080 被占用 | 修改 `docker-compose.yml` 中的宿主机端口 |
| 容器启动失败 | `docker compose logs open-webui` 查看详细错误 |
| 知识库检索结果不准确 | 调整 RAG 参数（Chunk Size、Top K、Threshold） |
| WebSocket 连接断开 | 确认 Nginx 中 open-webui location 有 WebSocket 升级配置 |

---

## 相关文件

- [openhands.md](./openhands.md)：OpenHands 自主编码 Agent 部署、与 Open-WebUI 对比选择指南、Docker Compose 配置、Nginx 代理
- [README.md](./README.md)：阶段 3 概览（三种实现路径）

> ⬅️ [返回阶段 3 概览](./README.md)
