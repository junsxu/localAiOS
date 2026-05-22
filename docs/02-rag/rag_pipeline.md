# RAG Pipeline 完整实现

本文档提供从文档到问答的完整 RAG Pipeline 实现，涵盖文档分块策略、批量索引、查询和质量评估。

---

## 一、Chunking 策略详解

### 1.1 策略对比

```
原始文档（2000 字）

固定大小切块（chunk=500, overlap=0）：
  块1: 第1~500字    块2: 第501~1000字    块3: 第1001~1500字    块4: 第1501~2000字
  问题：可能在句子中间截断 ❌

递归切块（chunk=500, overlap=50）：
  先按 \n\n（段落）分，再按 \n（换行）分，再按句号分...
  块1: "完整段落..."（490字）
  块2: "...重叠50字...下一段落..."（500字）
  优点：语义完整，有重叠防止截断 ✅

语义切块（按 Embedding 相似度）：
  计算相邻句子的 embedding 相似度，在相似度低的地方切块
  优点：语义最完整，质量最高 ✅
  缺点：需要为每个句子计算 embedding，速度慢 3~5 倍
```

### 1.2 推荐参数

| 文档类型 | chunk_size | chunk_overlap | 分隔符优先级 |
|---------|-----------|--------------|------------|
| 中文文章/笔记 | 500 字符 | 50 字符 | `\n\n` > `\n` > `。` > `，` |
| 英文文档 | 1000 字符 | 100 字符 | `\n\n` > `\n` > `.` > ` ` |
| 代码文件 | 1500 字符 | 150 字符 | `\n\n` > `\ndef ` > `\nclass ` > `\n` |
| PDF 扫描件 | 800 字符 | 80 字符 | `\n` > `。` > `，` |

---

## 二、批量索引器（生产版）

```python
#!/usr/bin/env python3
"""
rag_indexer.py - 批量文档索引器
支持：PDF、Markdown、TXT，增量索引（跳过已处理文件）
"""
import os
import hashlib
import json
import time
import logging
import asyncio
import aiohttp
from pathlib import Path
from datetime import datetime
from typing import Optional

from langchain_text_splitters import RecursiveCharacterTextSplitter
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct, Filter, FieldCondition, MatchValue
)

# ── 配置 ────────────────────────────────────────────────────────
OLLAMA_EMBED_URL = "http://localhost:11435"    # Windows 核显 Embedding
EMBED_MODEL      = "nomic-embed-text"          # 768 维；中文用 bge-m3（1024 维）
VECTOR_SIZE      = 768
QDRANT_URL       = "http://localhost:6333"
COLLECTION       = "ai-docs"
DOCS_DIR         = "/mnt/nas/ai-docs"
PROGRESS_FILE    = "/tmp/rag_index_progress.json"

logging.basicConfig(level=logging.INFO,
                    format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger(__name__)

client = QdrantClient(url=QDRANT_URL)

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    # 中文文档优化分隔符
    separators=["\n\n", "\n", "。", "！", "？", ".", "!", "?", "，", " ", ""],
    length_function=len,
)

# ── Embedding ────────────────────────────────────────────────────
async def embed_async(session: aiohttp.ClientSession, text: str) -> list[float]:
    async with session.post(
        f"{OLLAMA_EMBED_URL}/api/embeddings",
        json={"model": EMBED_MODEL, "prompt": text},
        timeout=aiohttp.ClientTimeout(total=60)
    ) as resp:
        data = await resp.json()
        return data["embedding"]

async def batch_embed(texts: list[str], concurrency: int = 4) -> list[list[float]]:
    """并发批量 Embedding"""
    sem = asyncio.Semaphore(concurrency)
    async with aiohttp.ClientSession() as session:
        async def limited(text):
            async with sem:
                return await embed_async(session, text)
        return await asyncio.gather(*[limited(t) for t in texts])

# ── 进度跟踪（增量索引）────────────────────────────────────────
def load_progress() -> dict:
    if Path(PROGRESS_FILE).exists():
        return json.loads(Path(PROGRESS_FILE).read_text())
    return {}

def save_progress(progress: dict):
    Path(PROGRESS_FILE).write_text(json.dumps(progress, indent=2))

def file_hash(path: Path) -> str:
    """用文件路径 + 修改时间作为 hash key（避免完整读取大文件）"""
    stat = path.stat()
    return hashlib.md5(f"{path}:{stat.st_size}:{stat.st_mtime}".encode()).hexdigest()

# ── 文档加载 ────────────────────────────────────────────────────
def load_text(path: Path) -> Optional[str]:
    suffix = path.suffix.lower()
    try:
        if suffix == ".pdf":
            from pypdf import PdfReader
            reader = PdfReader(str(path))
            return "\n".join(page.extract_text() or "" for page in reader.pages)
        elif suffix in (".md", ".markdown"):
            return path.read_text(encoding="utf-8")
        elif suffix in (".txt", ".rst"):
            return path.read_text(encoding="utf-8")
        else:
            return None
    except Exception as e:
        logger.warning(f"无法读取 {path}: {e}")
        return None

# ── 核心索引函数 ────────────────────────────────────────────────
def ensure_collection():
    existing = {c.name for c in client.get_collections().collections}
    if COLLECTION not in existing:
        client.create_collection(
            collection_name=COLLECTION,
            vectors_config=VectorParams(size=VECTOR_SIZE, distance=Distance.COSINE)
        )
        logger.info(f"✅ 创建集合 '{COLLECTION}'（{VECTOR_SIZE} 维）")

async def index_file(path: Path, progress: dict) -> int:
    """索引单个文件，返回写入的点数"""
    key = str(path)
    current_hash = file_hash(path)

    # 增量跳过：已索引且文件未变化
    if progress.get(key) == current_hash:
        logger.debug(f"跳过（未变化）: {path.name}")
        return 0

    text = load_text(path)
    if not text or len(text.strip()) < 50:
        logger.warning(f"跳过（内容为空或过短）: {path.name}")
        return 0

    chunks = splitter.split_text(text)
    if not chunks:
        return 0

    logger.info(f"索引: {path.name}  ({len(chunks)} 块)")

    # 批量 Embedding
    vectors = await batch_embed(chunks, concurrency=4)

    # 构造 Points
    base_id = int(hashlib.md5(key.encode()).hexdigest()[:8], 16)
    points = []
    for i, (chunk, vector) in enumerate(zip(chunks, vectors)):
        points.append(PointStruct(
            id=(base_id + i) % (2**31),  # Qdrant 要求 uint64
            vector=vector,
            payload={
                "text": chunk,
                "source": str(path),
                "file_name": path.name,
                "chunk_index": i,
                "total_chunks": len(chunks),
                "indexed_at": datetime.now().isoformat(),
                "doc_type": path.parent.name,  # 用目录名作为文档类别
            }
        ))

    # 分批写入（每批 100 条）
    for i in range(0, len(points), 100):
        client.upsert(COLLECTION, points[i:i+100])

    progress[key] = current_hash
    save_progress(progress)
    logger.info(f"  → 完成 {len(points)} 个片段")
    return len(points)

async def index_directory(root: str):
    """递归扫描目录并索引所有支持的文件"""
    ensure_collection()
    progress = load_progress()
    supported = {".pdf", ".md", ".markdown", ".txt", ".rst"}

    files = [p for p in Path(root).rglob("*")
             if p.suffix.lower() in supported and p.is_file()]
    logger.info(f"发现 {len(files)} 个文件待处理")

    total = 0
    for path in files:
        count = await index_file(path, progress)
        total += count

    logger.info(f"✅ 索引完成，共写入 {total} 个片段")

# ── 入口 ─────────────────────────────────────────────────────────
if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--source", default=DOCS_DIR)
    parser.add_argument("--file", help="只索引单个文件")
    args = parser.parse_args()

    if args.file:
        progress = load_progress()
        asyncio.run(index_file(Path(args.file), progress))
    else:
        asyncio.run(index_directory(args.source))
```

---

## 三、查询 Pipeline

```python
#!/usr/bin/env python3
"""rag_query.py - RAG 查询接口"""
import requests
from qdrant_client import QdrantClient

OLLAMA_EMBED = "http://localhost:11435"
OLLAMA_CHAT  = "http://localhost:11434"
EMBED_MODEL  = "nomic-embed-text"
CHAT_MODEL   = "qwen2.5-coder:14b"
COLLECTION   = "ai-docs"

client = QdrantClient("http://localhost:6333")

def embed(text: str) -> list[float]:
    r = requests.post(f"{OLLAMA_EMBED}/api/embeddings",
                      json={"model": EMBED_MODEL, "prompt": text})
    return r.json()["embedding"]

def retrieve(question: str, top_k: int = 5,
             score_threshold: float = 0.5) -> list[dict]:
    """语义检索，返回相关片段"""
    results = client.search(
        collection_name=COLLECTION,
        query_vector=embed(question),
        limit=top_k,
        score_threshold=score_threshold,
        with_payload=True
    )
    return [
        {
            "score": round(r.score, 3),
            "text": r.payload["text"],
            "source": r.payload.get("file_name", ""),
        }
        for r in results
    ]

RAG_PROMPT_TEMPLATE = """你是一个专业助手，请根据以下资料回答问题。
规则：
1. 只使用提供的资料回答，不要添加资料以外的内容
2. 如果资料中没有答案，直接说"根据现有资料，我无法回答这个问题"
3. 在回答末尾注明引用的资料来源（文件名）

资料：
{context}

问题：{question}
"""

def rag_answer(question: str, stream: bool = False) -> str:
    """完整 RAG 问答"""
    chunks = retrieve(question)
    if not chunks:
        return "未在知识库中找到相关内容，请确认问题是否在知识库范围内。"

    context = "\n\n".join(
        f"[来源: {c['source']}（相关度: {c['score']}）]\n{c['text']}"
        for c in chunks
    )
    prompt = RAG_PROMPT_TEMPLATE.format(context=context, question=question)

    resp = requests.post(
        f"{OLLAMA_CHAT}/api/generate",
        json={"model": CHAT_MODEL, "prompt": prompt, "stream": False}
    )
    return resp.json()["response"]

# 交互式使用
if __name__ == "__main__":
    print("RAG 知识库问答（输入 q 退出）\n")
    while True:
        q = input("问题：").strip()
        if q.lower() in ("q", "quit", "exit"):
            break
        if not q:
            continue
        print("\n正在检索和生成...\n")
        answer = rag_answer(q)
        print(f"回答：\n{answer}\n")
        print("-" * 50)
```

---

## 四、Embedding 缓存

```python
import hashlib
import shelve

CACHE_FILE = "/tmp/embed_cache"

def embed_cached(text: str) -> list[float]:
    """带磁盘缓存的 embedding，相同文本不重复调用 API"""
    key = hashlib.sha256(f"{EMBED_MODEL}:{text}".encode()).hexdigest()
    with shelve.open(CACHE_FILE) as cache:
        if key in cache:
            return cache[key]
        vector = embed(text)
        cache[key] = vector
        return vector

# 查看缓存统计
def cache_stats():
    with shelve.open(CACHE_FILE) as cache:
        print(f"缓存条目数: {len(cache)}")
```

---

## 五、定期增量索引（cron）

```bash
# 创建运行脚本
cat > ~/run_indexer.sh << 'EOF'
#!/bin/bash
cd /home/<username>
source .venv/bin/activate 2>/dev/null || true
python3 ~/rag_indexer.py --source /mnt/nas/ai-docs >> ~/indexer.log 2>&1
EOF
chmod +x ~/run_indexer.sh

# 每天凌晨 4 点运行
(crontab -l 2>/dev/null; echo "0 4 * * * ~/run_indexer.sh") | crontab -

# 查看日志
tail -f ~/indexer.log
```

---

## 六、质量评估

### 6.1 检索质量测试

```python
def evaluate_retrieval(test_cases: list[dict]) -> dict:
    """
    test_cases: [{"question": "...", "expected_source": "file.pdf"}]
    """
    hits = 0
    for case in test_cases:
        results = retrieve(case["question"], top_k=5)
        sources = [r["source"] for r in results]
        if case["expected_source"] in sources:
            hits += 1
            print(f"✅ {case['question'][:40]}...")
        else:
            print(f"❌ {case['question'][:40]}... (期望: {case['expected_source']})")
    return {"precision": hits / len(test_cases)}

# 示例测试集
test_cases = [
    {"question": "如何配置 WSL Mirrored 网络", "expected_source": "wsl_network.md"},
    {"question": "Nginx 反向代理配置", "expected_source": "nginx.md"},
]
result = evaluate_retrieval(test_cases)
print(f"检索精度: {result['precision']:.1%}")
```

### 6.2 常见问题诊断

| 症状 | 可能原因 | 解决方法 |
|------|---------|---------|
| 相似度得分普遍 < 0.5 | Embedding 模型语言不匹配 | 中文用 `bge-m3` |
| 检索到不相关片段 | chunk_size 过大，语义被稀释 | 减小到 300~400 字符 |
| 关键信息被截断 | overlap 不够 | 增大 overlap 到 chunk_size 的 15% |
| 同一文档内容重复出现 | Upsert ID 冲突 | 检查 ID 生成逻辑 |

---

> ⬅️ [返回阶段 2 概览](./README.md)
