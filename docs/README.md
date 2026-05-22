# 文档导航

> 完整学习路径说明见根目录 [README.md](../README.md)。

---

## 各阶段文档

| 阶段 | 目录 | 核心内容 |
|------|------|----------|
| 阶段 0 | [00-infrastructure/](./00-infrastructure/README.md) | WSL 网络 · Nginx · 双实例 Ollama · NAS · GitLab |
| 阶段 1 | [01-inference/](./01-inference/README.md) | Transformer 原理 · 采样参数 · Embedding · Ollama API |
| 阶段 2 | [02-rag/](./02-rag/README.md) | Qdrant 部署 · RAG Pipeline · 混合检索 · Rerank |
| 阶段 3 | [03-agent/](./03-agent/README.md) | Open-WebUI · OpenHands · ReAct · Tool Calling |
| 阶段 4 | [04-memory/](./04-memory/README.md) | 对话摘要 · 长期记忆（Qdrant）· 用户画像 |
| 阶段 5 | [05-orchestration/](./05-orchestration/README.md) | Docker 微服务 · docker-compose · Nginx API 网关 |
| 阶段 6 | [06-performance/](./06-performance/README.md) | 规则路由 · LLM 路由 · 自适应路由 · 基准测试 |
| 阶段 7 | [07-aios/](./07-aios/README.md) | 全系统集成自检 · 演进路线 v1.0 → v3.0 |

---

## 各阶段子文档索引

### 阶段 0 · 基础设施

| 文件 | 内容 |
|------|------|
| [wsl_network.md](./00-infrastructure/wsl_network.md) | WSL2 NAT vs Mirrored 网络模式配置 |
| [nginx.md](./00-infrastructure/nginx.md) | Windows Nginx 反向代理完整配置 |
| [ollama.md](./00-infrastructure/ollama.md) | 双实例 Ollama 部署（iGPU + dGPU 隔离） |
| [continue.md](./00-infrastructure/continue.md) | MacBook VSCode + Continue 插件配置 |
| [nas_storage.md](./00-infrastructure/nas_storage.md) | QNAP NAS SMB/NFS 挂载与文档目录规划 |
| [gitlab.md](./00-infrastructure/gitlab.md) | GitLab CE Docker 部署（NAS） |

### 阶段 1 · 推理层

| 文件 | 内容 |
|------|------|
| [transformer.md](./01-inference/transformer.md) | Transformer 架构、KV Cache、量化原理 |
| [sampling.md](./01-inference/sampling.md) | 温度、top_p、采样策略与 Modelfile 调参 |
| [embedding.md](./01-inference/embedding.md) | Embedding 模型原理与 Python 工具函数 |
| [ollama_api.md](./01-inference/ollama_api.md) | Ollama REST API 与 OpenAI 兼容接口 |

### 阶段 2 · RAG

| 文件 | 内容 |
|------|------|
| [qdrant.md](./02-rag/qdrant.md) | Qdrant 生产部署、集合管理、备份 |
| [rag_pipeline.md](./02-rag/rag_pipeline.md) | 批量索引脚本 + 查询 Pipeline 实现 |
| [retrieval.md](./02-rag/retrieval.md) | 混合检索（BM25 + Vector + RRF）、Rerank |

### 阶段 3 · Agent

| 文件 | 内容 |
|------|------|
| [open_webui.md](./03-agent/open_webui.md) | Open-WebUI 部署、RAG 集成、SearXNG 搜索 |
| [openhands.md](./03-agent/openhands.md) | OpenHands 自主编码 Agent 本地部署 |
| [tool_calling.md](./03-agent/tool_calling.md) | ReAct 模式 + Tool Calling 原生 Python 实现 |

### 阶段 4 · Memory

| 文件 | 内容 |
|------|------|
| [memory.md](./04-memory/memory.md) | MemoryService 实现（Qdrant + SQLite） |
| [conversation.md](./04-memory/conversation.md) | 对话摘要、滑动窗口上下文管理 |
| [user_profile.md](./04-memory/user_profile.md) | 用户画像提取与偏好学习 |

### 阶段 5 · 编排

| 文件 | 内容 |
|------|------|
| [services.md](./05-orchestration/services.md) | FastAPI 微服务（RAG / Agent / Memory） |
| [docker_compose.md](./05-orchestration/docker_compose.md) | 全栈 docker-compose + .env 模板 |
| [api_gateway.md](./05-orchestration/api_gateway.md) | Nginx API 网关路由规则 |

### 阶段 6 · 性能

| 文件 | 内容 |
|------|------|
| [router.md](./06-performance/router.md) | 三策略路由：规则 / LLM / 自适应 |
| [benchmarks.md](./06-performance/benchmarks.md) | 基准测试脚本与负载测试 |

### 阶段 7 · AI OS 终态

| 文件 | 内容 |
|------|------|
| [integration.md](./07-aios/integration.md) | 全系统集成自检脚本与启停流程 |
| [roadmap.md](./07-aios/roadmap.md) | 演进路线：v1.1 记忆图谱 → v2.0 Mac Studio → v3.0 多模态 |
