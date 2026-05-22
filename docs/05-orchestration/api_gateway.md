# Nginx API 网关配置

## 设计目标

Nginx 作为统一入口：
- 对外只暴露 80 端口（或 443）
- 根据 URL 路径路由到不同服务
- 统一处理超时、日志、限流
- 屏蔽内部端口，提升安全性

---

## 完整 nginx/aios.conf

```nginx
# nginx/aios.conf

# ── Upstream 服务定义 ─────────────────────────────────────
upstream ollama {
    server host.docker.internal:11434;
    keepalive 16;
}

upstream rag_service {
    server rag-service:8001;
    keepalive 16;
}

upstream agent_service {
    server agent-service:8002;
    keepalive 16;
}

upstream memory_service {
    server memory-service:8003;
    keepalive 16;
}


server {
    listen 80;
    server_name localhost;

    # ── 全局设置 ─────────────────────────────────────────
    client_max_body_size 50m;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Connection "";

    # ── 直接推理（Ollama）───────────────────────────────
    location /api/chat {
        proxy_pass http://ollama/api/chat;
        proxy_read_timeout 300s;
        proxy_connect_timeout 10s;
    }

    location /api/generate {
        proxy_pass http://ollama/api/generate;
        proxy_read_timeout 300s;
        proxy_buffering off;    # 流式输出需关闭缓冲
    }

    # ── RAG 服务 ─────────────────────────────────────────
    location /api/rag/ {
        proxy_pass http://rag_service/;
        proxy_read_timeout 60s;
    }

    # ── Agent 服务（任务可能耗时较长）─────────────────
    location /api/agent/ {
        proxy_pass http://agent_service/;
        proxy_read_timeout 600s;   # Agent 最多 10 分钟
    }

    # ── Memory 服务 ─────────────────────────────────────
    location /api/memory/ {
        proxy_pass http://memory_service/;
        proxy_read_timeout 30s;
    }

    # ── 网关健康检查 ────────────────────────────────────
    location /health {
        default_type application/json;
        return 200 '{"status":"ok","gateway":"nginx"}';
        access_log off;
    }

    # ── 聚合健康检查（依赖 lua 模块，可选）─────────────
    # 见进阶部分

    # ── 404 处理 ────────────────────────────────────────
    location / {
        return 404 '{"error":"not found","available":["/api/chat","/api/rag/","/api/agent/","/api/memory/","/health"]}';
        default_type application/json;
    }
}
```

---

## 路由规则详解

### URL 路径映射

| 请求路径 | 转发目标 | 说明 |
|---------|---------|------|
| `POST /api/chat` | Ollama `:11434/api/chat` | 直接推理 |
| `POST /api/generate` | Ollama `:11434/api/generate` | 流式生成 |
| `POST /api/rag/ingest` | RAG `:8001/ingest` | 文档入库 |
| `POST /api/rag/query` | RAG `:8001/query` | 知识检索 |
| `POST /api/agent/run` | Agent `:8002/run` | 任务执行 |
| `POST /api/memory/save` | Memory `:8003/save` | 保存记忆 |
| `GET  /api/memory/recall` | Memory `:8003/recall` | 召回记忆 |
| `GET  /health` | Nginx 本地 | 网关状态 |

### 路径重写说明

`location /api/rag/ { proxy_pass http://rag_service/; }` 中的**尾部斜杠**至关重要：
- 请求：`/api/rag/ingest`
- 转发：`http://rag_service/ingest`（自动去掉前缀 `/api/rag`）

---

## 日志配置

```nginx
# 在 nginx.conf 的 http 块中添加日志格式
log_format aios '$remote_addr - $remote_user [$time_local] '
               '"$request" $status $body_bytes_sent '
               '"$http_referer" "$http_user_agent" '
               'rt=$request_time uct=$upstream_connect_time '
               'uht=$upstream_header_time urt=$upstream_response_time';

# 在 server 块中使用
access_log /var/log/nginx/aios.access.log aios;
error_log  /var/log/nginx/aios.error.log warn;
```

---

## 限流配置

```nginx
# 在 http 块中定义限流区域
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=30r/m;
limit_req_zone $binary_remote_addr zone=agent_limit:10m rate=5r/m;

# 在 server 块中应用
location /api/rag/ {
    limit_req zone=api_limit burst=10 nodelay;
    proxy_pass http://rag_service/;
}

location /api/agent/ {
    limit_req zone=agent_limit burst=2 nodelay;
    proxy_read_timeout 600s;
    proxy_pass http://agent_service/;
}
```

---

## 流式输出支持

Ollama 的流式响应需要特殊配置：

```nginx
location /api/generate {
    proxy_pass http://ollama/api/generate;
    proxy_read_timeout 300s;

    # 关键：关闭所有缓冲
    proxy_buffering        off;
    proxy_cache            off;
    proxy_set_header       X-Accel-Buffering no;

    # SSE 支持
    chunked_transfer_encoding on;
    add_header Cache-Control no-cache;
}
```

---

## 基础认证（可选）

为 API 添加 Basic Auth 保护（适合局域网场景）：

```bash
# 创建密码文件（在 WSL 中执行）
sudo apt install -y apache2-utils
htpasswd -c /etc/nginx/.htpasswd aios
```

```nginx
location /api/ {
    auth_basic "AI OS API";
    auth_basic_user_file /etc/nginx/.htpasswd;
    # ... 其他配置
}
```

---

## CORS 配置（跨域）

允许前端应用跨域访问：

```nginx
location /api/ {
    # CORS Headers
    add_header Access-Control-Allow-Origin  "$http_origin" always;
    add_header Access-Control-Allow-Methods "GET, POST, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;

    # 预检请求直接返回
    if ($request_method = OPTIONS) {
        return 204;
    }

    proxy_pass http://rag_service/;
}
```

---

## 聚合健康检查（Shell 脚本版）

```bash
#!/bin/bash
# check_all.sh - 检查所有服务健康状态

SERVICES=(
    "Gateway|http://localhost/health"
    "RAG|http://localhost:8001/health"
    "Agent|http://localhost:8002/health"
    "Memory|http://localhost:8003/health"
    "Qdrant|http://localhost:6333/healthz"
)

ALL_OK=true
echo "=== 服务健康检查 $(date) ==="
for entry in "${SERVICES[@]}"; do
    IFS='|' read -r name url <<< "$entry"
    code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 "$url")
    if [[ "$code" == "200" ]]; then
        echo "  ✅ $name"
    else
        echo "  ❌ $name (HTTP $code)"
        ALL_OK=false
    fi
done

if $ALL_OK; then
    echo "=== 全部正常 ==="
    exit 0
else
    echo "=== 有服务异常 ==="
    exit 1
fi
```

---

## 参考链接

- [微服务设计](./services.md)
- [Docker Compose 配置](./docker_compose.md)
- [Nginx 详细配置参考](../00-infrastructure/nginx.md)
