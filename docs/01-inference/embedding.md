# Embedding 详解

Embedding（嵌入）将文本转为高维向量，是 RAG 系统的核心基础。理解 Embedding 的原理和选型，直接决定知识库检索质量。

---

## 一、Embedding vs Generation 对比

| 维度 | Generation（生成） | Embedding（嵌入） |
|------|------------------|-----------------|
| 输入 | 文本 Prompt | 任意文本 |
| 输出 | 新的文本 Token 序列 | 固定长度浮点向量 |
| 用途 | 对话、代码、写作 | 语义搜索、RAG、聚类、去重 |
| 模型大小 | 7B~70B（几GB~几十GB） | 100MB~700MB |
| 推理速度 | 慢（逐 Token 生成） | 快（一次前向传播） |
| 本项目分工 | WSL Ollama（独显） | Windows Ollama（核显） |

---

## 二、向量的本质

Embedding 将语义相似的文本映射到向量空间中距离相近的位置：

```
"机器学习"  → [0.12, -0.34, 0.78, ..., 0.23]  (768 维)
"深度学习"  → [0.15, -0.31, 0.81, ..., 0.21]  ← 距离近（语义相近）
"天气预报"  → [-0.45, 0.62, -0.12, ..., 0.55] ← 距离远（语义无关）
```

**余弦相似度**（Cosine Similarity）是最常用的相似度计算方式：

```python
import numpy as np

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# 值域：[-1, 1]，越接近 1 越相似
# Qdrant 的 Cosine 距离 = 1 - cosine_similarity
```

---

## 三、模型选型

| 模型 | 向量维度 | 大小 | 语言 | 推荐场景 |
|------|---------|------|------|---------|
| `nomic-embed-text` | 768 | ~270MB | 英文优先 | 英文文档，速度快 |
| `mxbai-embed-large` | 1024 | ~670MB | 英文 | 高精度英文 |
| `bge-m3` | 1024 | ~570MB | **中英双语** | **中文文档（推荐）** |
| `shaw/dmeta-embedding-zh` | 768 | ~400MB | 中文专用 | 纯中文场景 |

```bash
# 拉取推荐模型
ollama pull nomic-embed-text   # 英文/快速
ollama pull bge-m3             # 中英双语
```

**选型建议**：
- 知识库以中文为主 → 使用 `bge-m3`（在 Windows 端或 WSL 端均可）
- 知识库以英文为主 → 使用 `nomic-embed-text`（更轻量）
- 注意：**Embedding 模型和 Qdrant Collection 的向量维度必须一致**

---

## 四、调用方式

### Ollama 原生 API

```bash
# nomic-embed-text（768 维）
curl -X POST http://localhost:11435/api/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model": "nomic-embed-text", "prompt": "你好世界"}'

# 响应
{
  "embedding": [0.123, -0.456, 0.789, ...]  # 768 个浮点数
}
```

### OpenAI 兼容 API

```bash
curl -X POST http://localhost:11435/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nomic-embed-text",
    "input": "你好世界"
  }'

# 响应（兼容 OpenAI 格式）
{
  "data": [{"embedding": [0.123, ...], "index": 0}],
  "model": "nomic-embed-text",
  "usage": {"prompt_tokens": 4, "total_tokens": 4}
}
```

### Python 封装

```python
import requests
import numpy as np
from typing import Union

EMBED_URL = "http://localhost:11435/api/embeddings"  # Windows Ollama
EMBED_MODEL = "nomic-embed-text"

def embed(text: Union[str, list[str]]) -> Union[list[float], list[list[float]]]:
    """
    获取文本 embedding，支持单条或批量（串行调用）。
    返回 768 维浮点向量或向量列表。
    """
    if isinstance(text, str):
        resp = requests.post(EMBED_URL, json={"model": EMBED_MODEL, "prompt": text})
        resp.raise_for_status()
        return resp.json()["embedding"]
    else:
        return [embed(t) for t in text]

def cosine_sim(a: list[float], b: list[float]) -> float:
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

# 使用示例
v1 = embed("机器学习")
v2 = embed("深度学习")
v3 = embed("天气预报")
print(f"机器学习 vs 深度学习: {cosine_sim(v1, v2):.3f}")   # ~0.85
print(f"机器学习 vs 天气预报: {cosine_sim(v1, v3):.3f}")   # ~0.40
```

---

## 五、批量处理与性能优化

### 批量 Embedding（并发）

```python
import asyncio
import aiohttp

async def embed_async(session, text: str) -> list[float]:
    async with session.post(
        "http://localhost:11435/api/embeddings",
        json={"model": "nomic-embed-text", "prompt": text}
    ) as resp:
        data = await resp.json()
        return data["embedding"]

async def batch_embed(texts: list[str], concurrency: int = 4) -> list[list[float]]:
    semaphore = asyncio.Semaphore(concurrency)
    async with aiohttp.ClientSession() as session:
        async def limited_embed(text):
            async with semaphore:
                return await embed_async(session, text)
        return await asyncio.gather(*[limited_embed(t) for t in texts])

# 使用
texts = ["文档1内容...", "文档2内容...", "文档3内容..."]
vectors = asyncio.run(batch_embed(texts, concurrency=4))
```

### 缓存机制（避免重复计算）

```python
import hashlib
import shelve
import json

CACHE_FILE = "embed_cache.db"

def embed_cached(text: str, model: str = "nomic-embed-text") -> list[float]:
    """带缓存的 embedding，相同文本不重复调用 API"""
    key = hashlib.sha256(f"{model}:{text}".encode()).hexdigest()
    with shelve.open(CACHE_FILE) as cache:
        if key in cache:
            return cache[key]
        vector = embed(text)  # 调用原始 embed 函数
        cache[key] = vector
        return vector
```

### 性能基准

在 Intel 13900 核显 + `nomic-embed-text` 上的参考速度：

| 文本长度 | 单次耗时 | 吞吐量（并发4） |
|---------|---------|--------------|
| 100 字符 | ~50ms | ~60 条/秒 |
| 500 字符 | ~80ms | ~40 条/秒 |
| 1000 字符 | ~120ms | ~25 条/秒 |

---

## 六、向量归一化

部分应用场景需要确保向量已归一化（单位长度），以便直接用点积替代余弦相似度：

```python
def normalize(v: list[float]) -> list[float]:
    arr = np.array(v)
    norm = np.linalg.norm(arr)
    return (arr / norm).tolist() if norm > 0 else v

# Qdrant 使用 Cosine 距离时会自动归一化，一般不需要手动处理
```

---

## 七、常见问题

**Q：为什么两段意思相近的文本相似度只有 0.6？**

可能原因：
1. 一段是中文，一段是英文，使用了仅支持英文的 `nomic-embed-text`，语义未正确对齐 → 换用 `bge-m3`
2. 文本包含大量无关词汇（如文档标题、页码）→ 预处理清洗
3. `chunk_size` 过大，语义被稀释 → 减小切块大小

**Q：更大维度（1024 vs 768）一定更好吗？**

不一定。维度越大需要越多存储和计算资源。在本项目规模（万级文档）内，768 维已经足够。只有在精度要求极高时才有必要升级到 1024 维。

**Q：Embedding 模型可以放在 WSL 吗？**

可以，但本项目设计上将 Embedding 放在 Windows 核显，原因是：
1. 独显（RTX5070Ti）优先保留给推理，避免两类工作互相竞争显存
2. Embedding 模型小，核显足够运行，且 embedding 请求密集（批量索引时），分离更高效

---

> ⬅️ [返回阶段 1 概览](./README.md)
