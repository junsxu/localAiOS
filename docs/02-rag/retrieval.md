# 检索质量优化

基础向量检索已经能解决大多数场景，但在精度要求高或文档量大时，需要更高级的检索策略。

---

## 一、检索质量瓶颈分析

```
向量检索（Vector Search）的局限：
  ✅ 擅长：语义相似性（意思相近）
  ❌ 不擅长：精确关键词匹配（如人名、代码函数名）
  ❌ 不擅长：短词查询（向量维度低，区分度差）

关键词检索（BM25）的局限：
  ✅ 擅长：精确词汇匹配
  ❌ 不擅长：同义词、语义理解

解决方案：混合检索（Hybrid Search）= 向量 + BM25 结果合并
```

---

## 二、混合检索（Hybrid Search）

### 2.1 RRF 融合算法（Reciprocal Rank Fusion）

将两种检索结果通过倒数排名融合，无需调参：

```python
def reciprocal_rank_fusion(rankings: list[list[str]], k: int = 60) -> list[str]:
    """
    rankings: 多个排序列表（文档ID列表），按相关性从高到低
    k: 平滑参数（通常取 60）
    返回：融合后的文档ID排序列表
    """
    scores = {}
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking, start=1):
            scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)
```

### 2.2 BM25 实现（本地）

```python
from rank_bm25 import BM25Okapi
import jieba  # 中文分词

class BM25Index:
    """基于 BM25 的关键词索引"""
    def __init__(self):
        self.docs = []
        self.doc_ids = []
        self.bm25 = None

    def build(self, documents: list[dict]):
        """构建索引：documents = [{"id": ..., "text": ...}]"""
        self.docs = documents
        self.doc_ids = [d["id"] for d in documents]
        # 中文分词
        tokenized = [list(jieba.cut(d["text"])) for d in documents]
        self.bm25 = BM25Okapi(tokenized)
        print(f"BM25 索引构建完成，共 {len(documents)} 条文档")

    def search(self, query: str, top_k: int = 5) -> list[str]:
        """返回最相关的文档 ID 列表"""
        tokens = list(jieba.cut(query))
        scores = self.bm25.get_scores(tokens)
        top_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)[:top_k]
        return [self.doc_ids[i] for i in top_indices]
```

### 2.3 完整混合检索流程

```python
from qdrant_client import QdrantClient

client = QdrantClient("http://localhost:6333")
bm25_index = BM25Index()

def hybrid_search(query: str, collection: str,
                  top_k: int = 5, vector_weight: float = 0.7) -> list[dict]:
    """
    混合检索：向量检索 + BM25 关键词检索，通过 RRF 融合
    vector_weight: 向量检索的权重（0~1）
    """
    # 1. 向量检索
    vector = embed(query)
    vector_results = client.search(
        collection_name=collection,
        query_vector=vector,
        limit=top_k * 2,   # 取更多候选供融合
        with_payload=True
    )
    vector_ids = [str(r.id) for r in vector_results]
    vector_map = {str(r.id): r for r in vector_results}

    # 2. BM25 关键词检索
    bm25_ids = bm25_index.search(query, top_k=top_k * 2)

    # 3. RRF 融合
    fused_ids = reciprocal_rank_fusion([vector_ids, bm25_ids])[:top_k]

    # 4. 取最终结果（优先从向量检索取 payload）
    results = []
    for doc_id in fused_ids:
        if doc_id in vector_map:
            r = vector_map[doc_id]
            results.append({
                "id": doc_id,
                "text": r.payload["text"],
                "source": r.payload.get("source", ""),
            })
    return results
```

---

## 三、Rerank（重排序）

### 3.1 为什么需要 Rerank

向量检索（TopK=10）召回候选，Rerank 用更精确的模型对这 10 条再次排序，最终只送 Top3 给 LLM。

```
向量检索（召回）  →  TopK=10 候选  →  Rerank  →  Top3  →  LLM
     快速                                精准
```

### 3.2 使用 Ollama 做 Rerank

用生成模型评估相关性（轻量方案，无需额外 Rerank 模型）：

```python
def rerank_with_llm(query: str, candidates: list[dict],
                    top_k: int = 3) -> list[dict]:
    """用 LLM 评估候选片段与问题的相关性，返回 top_k 个"""
    scores = []
    for c in candidates:
        prompt = f"""请评估以下文本片段与问题的相关性，只回复一个 0~10 的整数。
问题：{query}
文本：{c['text'][:300]}
相关性分数（0=完全无关，10=高度相关）："""

        resp = requests.post("http://localhost:11434/api/generate",
                             json={"model": "qwen2.5:7b",
                                   "prompt": prompt,
                                   "options": {"temperature": 0},
                                   "stream": False})
        try:
            score = int(resp.json()["response"].strip())
        except (ValueError, KeyError):
            score = 5  # 默认中等
        scores.append((score, c))

    scores.sort(key=lambda x: x[0], reverse=True)
    return [c for _, c in scores[:top_k]]
```

### 3.3 使用专用 Rerank 模型

如果有更高精度需求，使用专门的 Cross-Encoder 模型：

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("BAAI/bge-reranker-base")

def rerank_cross_encoder(query: str, candidates: list[dict], top_k: int = 3):
    pairs = [(query, c["text"]) for c in candidates]
    scores = reranker.predict(pairs)
    ranked = sorted(zip(scores, candidates), reverse=True)
    return [c for _, c in ranked[:top_k]]
```

---

## 四、查询扩展（Query Expansion）

通过改写查询来提升召回率：

```python
def expand_query(question: str) -> list[str]:
    """用 LLM 生成查询的多种表述，提升召回率"""
    prompt = f"""请将以下问题改写为 3 个不同的表述方式（每行一个），保持语义相同：
原始问题：{question}
改写："""
    resp = requests.post("http://localhost:11434/api/generate",
                         json={"model": "qwen2.5:7b",
                               "prompt": prompt,
                               "options": {"temperature": 0.3},
                               "stream": False})
    variants = resp.json()["response"].strip().split("\n")
    queries = [question] + [v.strip() for v in variants if v.strip()]
    return queries[:4]

def multi_query_retrieve(question: str, top_k: int = 5) -> list[dict]:
    """多查询检索：对多个查询变体检索后去重合并"""
    queries = expand_query(question)
    all_results = {}

    for q in queries:
        results = retrieve(q, top_k=top_k)
        for r in results:
            # 用文本 hash 去重
            key = hashlib.md5(r["text"].encode()).hexdigest()
            if key not in all_results or r["score"] > all_results[key]["score"]:
                all_results[key] = r

    # 按相关性排序
    merged = sorted(all_results.values(), key=lambda x: x["score"], reverse=True)
    return merged[:top_k]
```

---

## 五、上下文压缩

检索到 5 个片段，但每个片段可能包含大量无关内容。上下文压缩只保留与问题最相关的句子：

```python
def compress_context(question: str, chunk: str, max_length: int = 200) -> str:
    """从 chunk 中提取与 question 最相关的内容"""
    if len(chunk) <= max_length:
        return chunk

    prompt = f"""从以下文本中提取与问题最相关的 1~3 句话，原文引用，不要改写：
问题：{question}
文本：{chunk}
提取："""
    resp = requests.post("http://localhost:11434/api/generate",
                         json={"model": "qwen2.5:7b",
                               "prompt": prompt,
                               "options": {"temperature": 0, "num_predict": 200},
                               "stream": False})
    return resp.json()["response"].strip()
```

---

## 六、完整优化流程

```
用户问题
    │
    ├─→ 查询扩展（生成 3 个变体）
    │
    ├─→ 混合检索（向量 + BM25）×4 次 → 去重 Top20
    │
    ├─→ Rerank（精排）→ Top5
    │
    ├─→ 上下文压缩（可选）
    │
    └─→ 拼接 Prompt → LLM → 回答
```

**何时使用各优化**：

| 优化 | 适用场景 | 代价 |
|------|---------|------|
| 混合检索 | 文档包含大量专有名词/代码 | 需要维护 BM25 索引 |
| Rerank | 要求高精度，可接受 2~3 倍延迟 | 额外 LLM 调用 |
| 查询扩展 | 用户查询模糊、简短 | 额外 LLM 调用 |
| 上下文压缩 | 文档很长，LLM 上下文窗口紧张 | 额外 LLM 调用 |

---

> ⬅️ [返回阶段 2 概览](./README.md)
