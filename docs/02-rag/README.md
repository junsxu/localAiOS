# 阶段 2：知识检索层 · RAG

> **核心目标**：让模型"有记忆"，但不是幻觉
>
> RAG = Retrieval-Augmented Generation：检索增强生成，让模型回答只来自你的知识库。

---

## 知识体系

| 主题 | 详细文档 |
|------|---------|
| Qdrant 向量数据库部署与管理 | [qdrant.md](./qdrant.md) |
| 文档分块（Chunking）策略 | [rag_pipeline.md](./rag_pipeline.md) |
| 完整 RAG Pipeline 实现 | [rag_pipeline.md](./rag_pipeline.md) |
| 检索质量优化（Rerank / 混合检索） | [retrieval.md](./retrieval.md) |

---

## 系统全景

```
               ┌──────── 构建阶段（离线）────────┐
               │                                │
  原始文档 → 切块(Chunk) → Embedding → Qdrant 向量库
  (NAS ai-docs)                     (WSL :6333)
                                                │
               ┌──────── 查询阶段（在线）────────┤
               │                                │
  用户问题 → Embedding → 相似度检索 → Top-K 片段
               │                                │
               └────────→ 拼 Prompt → LLM → 回答
```

---

## 核心知识速览

### Chunking 策略

切块是 RAG 质量的核心决定因素：

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| 固定大小 | 每 N 个字符一组 | 简单快速，质量一般 |
| **递归切块** | 按段落→句子→字符逐级细分 | **推荐，语义完整** |
| 滑动窗口 | 块之间有重叠（overlap） | 防止关键信息被截断 |
| 语义切块 | 按 embedding 相似度分组 | 精度高，计算慢 |

**推荐参数**：`chunk_size=500字符，chunk_overlap=50字符`

> 详细对比和实验见 [rag_pipeline.md](./rag_pipeline.md)。

### 检索策略

```
查询 → Embedding → 向量检索（TopK=5~10）→ [Rerank] → 送入 LLM
```

> 混合检索（向量 + BM25）和 Rerank 详见 [retrieval.md](./retrieval.md)。

### RAG Prompt 设计原则

System Prompt 需明确要求模型"只基于提供的资料回答"，防止模型混入训练记忆（幻觉）。资料不足时应明确回答"无法从现有资料得出结论"，而不是猜测。

---

## 实操任务

### 任务 1：部署 Qdrant

通过 Docker Compose 在 WSL 中启动 Qdrant，将存储目录挂载到本地路径，确保数据持久化。

> 完整配置、systemd 服务、集合管理见 [qdrant.md](./qdrant.md)。

- [ ] Qdrant Dashboard 可访问：`http://localhost:6333/dashboard`

---

### 任务 2：运行完整 RAG Pipeline

实现最小可运行 RAG：读取本地文档 → 切块 → Embedding → 存入 Qdrant → 自然语言提问 → 检索上下文 → LLM 生成答案。

使用 Windows Ollama（核显）负责 Embedding，WSL Ollama（独显）负责推理，体现双实例分工设计。

> 完整 Pipeline 代码（批量索引、增量更新、缓存策略）见 [rag_pipeline.md](./rag_pipeline.md)。

- [ ] 回答来自文档内容，无明显幻觉
- [ ] 查询响应 < 5 秒

---

### 任务 3：数据分层（NAS 冷热分离）

建立冷热分离存储策略：
- **热数据**：本地 SSD 存放 Qdrant 向量索引（快速检索）
- **冷数据**：NAS 存放 PDF/MD/TXT 原始文件（持久备份）

定期将 NAS 新文件增量索引到 Qdrant，保持知识库同步。

---

## 验收标准

- [ ] ✅ 回答"只来自你的知识库"，无明显幻觉
- [ ] ✅ chunk 策略合理，不破坏句子/段落完整性
- [ ] ✅ 查询延迟 < 2 秒（1000 个片段以内）
- [ ] ✅ Embedding 缓存生效，相同文本不重复计算
- [ ] ✅ 能解释：为什么某次检索结果相关性低

---

> ⬅️ [上一阶段：推理层](../01-inference/README.md) | ➡️ [下一阶段：Agent](../03-agent/README.md)
