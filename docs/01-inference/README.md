# 阶段 1：推理层 · 模型理解

> **核心目标**：理解"大模型到底在干嘛"，而不是黑盒调用
>
> 能解释回答为什么变好或变差，能人为稳定输出风格。

---

## 知识体系

| 主题 | 详细文档 |
|------|---------|
| Transformer 推理原理（KV Cache / Attention） | [transformer.md](./transformer.md) |
| Tokenization 与中文 Token 消耗 | [transformer.md](./transformer.md) |
| 采样参数（temperature / top_p / top_k） | [sampling.md](./sampling.md) |
| Embedding vs Generation | [embedding.md](./embedding.md) |
| Ollama API 完整参考 | [ollama_api.md](./ollama_api.md) |

---

## 核心概念速览

### 推理流程

```
输入 Tokens
    │
Embedding 层     ← Token → 向量
    │
N × Attention    ← 理解上下文关系
    │
KV Cache         ← 缓存已计算的 Key/Value，加速续写
    │
Logits → Softmax ← 转为概率分布
    │
Sampling         ← 从概率中采样下一个 Token
    │
输出 Token
```

### 关键采样参数

| 参数 | 作用 | 代码推荐值 | 创意推荐值 |
|------|------|-----------|-----------|
| `temperature` | 随机性（越低越确定） | 0.1 | 0.8 |
| `top_p` | 核采样概率阈值 | 0.9 | 0.95 |
| `num_ctx` | 上下文窗口大小（Token 数） | 4096 | 4096 |
| `repeat_penalty` | 惩罚重复输出 | 1.05 | 1.1 |

> 详细原理和调参实验见 [sampling.md](./sampling.md)。

### Embedding vs Generation

```
Generation（生成）        Embedding（嵌入）
输入 prompt → 输出文字    输入文本 → 输出向量（768/1024 维）
用于对话、代码、写作       用于语义搜索、RAG、聚类
模型大（7B~70B）          模型小（200MB~700MB）
```

> 详细说明和模型选型见 [embedding.md](./embedding.md)。

---

## 实操任务

### 任务 1：对比不同规模模型

拉取 `qwen2.5:7b` 和 `qwen2.5:14b`，用同一 prompt 分别请求，对比输出质量和响应速度。

重点观察：逻辑完整性、专业术语准确度、首 Token 延迟。

> API 调用方式见 [ollama_api.md](./ollama_api.md)。

- [ ] 记录 7B vs 14B 的输出质量和速度差异

---

### 任务 2：采样参数实验

用同一 prompt（如"写一首关于秋天的诗"），分别用 `temperature=0.1` 和 `temperature=1.2` 请求，感受确定性与创意性的差异。

> 完整参数实验方案和预期结果见 [sampling.md](./sampling.md)。

- [ ] 记录两种参数下的输出差异，找到适合自己场景的参数组合

---

### 任务 3：构建最小 Embedding Pipeline

将几段文本向量化后存入 Qdrant，对自然语言问题执行语义检索，验证"语义相似"比"关键词匹配"更精准。

> 完整代码实现和 Qdrant 集合管理见 [embedding.md](./embedding.md)。

- [ ] 成功运行并看到语义匹配结果

---

## 验收标准

- [ ] ✅ 能解释 KV Cache 和 temperature 对输出的影响
- [ ] ✅ 能通过参数调整稳定输出风格（不靠 prompt 碰运气）
- [ ] ✅ 成功将文本转为向量并存入 Qdrant
- [ ] ✅ 能描述 7B / 14B 模型的质量和速度差异

---

## 延伸阅读

- [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) — 最好的 Transformer 图解
- [Ollama Model Library](https://ollama.com/library) — 可用模型列表

---

> ⬅️ [上一阶段：基础设施](../00-infrastructure/README.md) | ➡️ [下一阶段：RAG](../02-rag/README.md)
