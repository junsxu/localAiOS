# 阶段 5：编排层 · 系统编排

> **核心目标**：不是工具组合，而是"操作系统"
>
> 将前几个阶段的能力组合成可维护、可扩展、组件可替换的系统架构。

---

## 知识体系

| 主题 | 详细文档 |
|------|---------|
| 微服务设计与服务边界 | [services.md](./services.md) |
| Docker Compose 完整配置 | [docker_compose.md](./docker_compose.md) |
| Nginx API 网关与路由 | [api_gateway.md](./api_gateway.md) |

---

## 目标架构

```
                    [ Nginx API Gateway :80 ]
                             │
        ─────────────────────────────────────────────
        │          │            │            │
    /api/chat   /api/rag      /api/agent   /api/memory
        │          │            │            │
     Ollama      Qdrant        Agent SVC    Memory SVC
     :11434      :6333         :8001         :8002
        │
     GPU推理
```

---

## 核心知识速览

### 微服务原则

| 原则 | 说明 | 实践 |
|------|------|------|
| 单一职责 | 每个服务只做一件事 | 推理/RAG/Agent/Memory 独立部署 |
| 独立部署 | 单个服务可单独更新 | Docker 容器化 |
| API 通信 | 服务间通过 HTTP 通信 | RESTful API |
| 数据隔离 | 服务不共享数据库 | 各自独立存储 |

### 服务边界

| 服务 | 端口 | 职责 |
|------|------|------|
| Qdrant | 6333 | 向量存储 |
| RAG Service | 8001 | 文档管理 + 语义检索 |
| Agent Service | 8002 | 任务规划 + 工具调用 |
| Memory Service | 8003 | 记忆读写 |
| Nginx Gateway | 80 | 统一入口 + 路由 |

---

## 实操任务清单

### 任务 1：启动全部服务

```bash
# 克隆服务代码
git clone <repo> && cd localAiOS

# 启动所有服务
docker compose up -d

# 验证状态
docker compose ps
curl http://localhost/health
```

> Docker Compose 完整配置见 [docker_compose.md](./docker_compose.md)。

- [ ] `docker compose ps` 所有服务显示 healthy

### 任务 2：验证路由

```bash
# 测试各路径是否路由正确
curl http://localhost/api/rag/health      # → RAG Service
curl http://localhost/api/agent/health    # → Agent Service
curl http://localhost/api/memory/health   # → Memory Service
```

> 路由配置详见 [api_gateway.md](./api_gateway.md)。

- [ ] 所有 `/health` 端点返回 200

### 任务 3：服务解耦验证

```bash
# 重启单个服务，不影响其他
docker compose restart rag-service
sleep 5
curl http://localhost/api/agent/health    # 应该仍然 200
```

- [ ] 重启 RAG 服务后 Agent 服务不受影响

### 任务 4：替换模型（只改环境变量）

```bash
# 修改 docker-compose.yml 中的环境变量
# CHAT_MODEL=qwen2.5:14b → CHAT_MODEL=qwen2.5:32b
docker compose up -d agent-service   # 只重启 Agent
```

- [ ] 换模型不需要修改任何代码

---

## 验收标准

- [ ] ✅ 任意组件可单独替换（如：换模型只改环境变量）
- [ ] ✅ `docker compose restart rag-service` 不影响 Agent
- [ ] ✅ 所有服务通过 Nginx 统一对外暴露
- [ ] ✅ `/health` 端点可检测各服务状态

---

> ⬅️ [上一阶段：Memory](../04-memory/README.md) | ➡️ [下一阶段：性能优化](../06-performance/README.md)
