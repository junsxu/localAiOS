# Docker Compose 完整配置

## 完整 docker-compose.yml

```yaml
# docker-compose.yml
version: "3.9"

networks:
  aios-net:
    driver: bridge

volumes:
  qdrant_data:
  memory_data:

services:

  # ── 向量数据库 ─────────────────────────────────────────
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    networks: [aios-net]
    ports:
      - "6333:6333"
      - "6334:6334"      # gRPC（可选）
    volumes:
      - qdrant_data:/qdrant/storage
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # ── RAG 服务 ───────────────────────────────────────────
  rag-service:
    build:
      context: ./services/rag
      dockerfile: Dockerfile
    container_name: rag-service
    networks: [aios-net]
    ports:
      - "8001:8001"
    environment:
      - OLLAMA_URL=http://host.docker.internal:11434
      - QDRANT_URL=http://qdrant:6333
      - EMBED_MODEL=${EMBED_MODEL:-bge-m3}
      - PORT=8001
    extra_hosts:
      - "host.docker.internal:host-gateway"   # Linux 下需要此配置
    depends_on:
      qdrant:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # ── Agent 服务 ─────────────────────────────────────────
  agent-service:
    build:
      context: ./services/agent
      dockerfile: Dockerfile
    container_name: agent-service
    networks: [aios-net]
    ports:
      - "8002:8002"
    environment:
      - OLLAMA_URL=http://host.docker.internal:11434
      - RAG_URL=http://rag-service:8001
      - MEMORY_URL=http://memory-service:8003
      - CHAT_MODEL=${CHAT_MODEL:-qwen2.5:14b}
      - PORT=8002
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      rag-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8002/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # ── Memory 服务 ────────────────────────────────────────
  memory-service:
    build:
      context: ./services/memory
      dockerfile: Dockerfile
    container_name: memory-service
    networks: [aios-net]
    ports:
      - "8003:8003"
    environment:
      - OLLAMA_URL=http://host.docker.internal:11434
      - QDRANT_URL=http://qdrant:6333
      - EMBED_MODEL=${EMBED_MODEL:-bge-m3}
      - DATA_DIR=/app/data
      - PORT=8003
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - memory_data:/app/data
    depends_on:
      qdrant:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8003/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # ── API 网关 ───────────────────────────────────────────
  nginx:
    image: nginx:alpine
    container_name: aios-gateway
    networks: [aios-net]
    ports:
      - "80:80"
    volumes:
      - ./nginx/aios.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      rag-service:
        condition: service_healthy
      agent-service:
        condition: service_healthy
      memory-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
```

---

## 环境变量文件 (.env)

```bash
# .env（不要提交到 git）

# 模型配置（只改这里即可换模型）
CHAT_MODEL=qwen2.5:14b
EMBED_MODEL=bge-m3

# Ollama 地址（若 Ollama 不在宿主机，修改此处）
OLLAMA_HOST=http://host.docker.internal:11434

# Qdrant 认证（生产环境开启）
# QDRANT_API_KEY=your_secret_key
```

---

## 常用操作命令

### 启动与停止

```bash
# 后台启动所有服务
docker compose up -d

# 查看所有服务状态
docker compose ps

# 查看实时日志
docker compose logs -f

# 查看某个服务日志
docker compose logs -f rag-service

# 停止所有服务（保留数据卷）
docker compose down

# 停止并清除数据卷（慎用！）
docker compose down -v
```

### 更新单个服务

```bash
# 重新构建并启动（代码更新后）
docker compose up -d --build rag-service

# 仅重启（不重新构建）
docker compose restart agent-service

# 只拉取最新镜像
docker compose pull qdrant && docker compose up -d qdrant
```

### 数据卷管理

```bash
# 查看所有卷
docker volume ls

# 查看卷详情（存储路径等）
docker volume inspect localaios_qdrant_data

# 备份 Qdrant 数据
docker run --rm \
  -v localaios_qdrant_data:/source \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/qdrant_$(date +%Y%m%d).tar.gz -C /source .
```

---

## 服务健康检查脚本

```python
# check_services.py
import requests

SERVICES = {
    "Nginx Gateway":  "http://localhost/health",
    "RAG Service":    "http://localhost:8001/health",
    "Agent Service":  "http://localhost:8002/health",
    "Memory Service": "http://localhost:8003/health",
    "Qdrant":         "http://localhost:6333/healthz",
}

print("服务健康状态检查\n" + "="*40)
all_ok = True
for name, url in SERVICES.items():
    try:
        resp = requests.get(url, timeout=5)
        status = "✅ OK" if resp.ok else f"❌ {resp.status_code}"
    except Exception as e:
        status = f"❌ {type(e).__name__}"
        all_ok = False
    print(f"  {name:<20} {status}")

print("="*40)
print(f"整体状态：{'✅ 全部正常' if all_ok else '❌ 有服务异常'}")
```

---

## 扩容与负载均衡

如果单个服务成为性能瓶颈，可以水平扩容：

```bash
# 将 rag-service 扩展到 3 个实例
docker compose up -d --scale rag-service=3
```

对应的 Nginx 配置会自动将流量分发到多个实例（需使用服务名作为 upstream）：

```nginx
upstream rag_service {
    server rag-service:8001;   # Docker Compose 会自动处理多实例
    # 如果手动扩容，需要分别列出：
    # server rag-service_1:8001;
    # server rag-service_2:8001;
    # server rag-service_3:8001;
}
```

---

## 参考链接

- [微服务设计与服务边界](./services.md)
- [Nginx API 网关](./api_gateway.md)
- [Qdrant 部署](../02-rag/qdrant.md)
