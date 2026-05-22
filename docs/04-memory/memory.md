# Memory 架构详解

## 整体架构

```
┌────────────────────────────────────────────────────────────┐
│                     Memory Service                         │
│                                                            │
│  ┌─────────────┐   ┌──────────────┐   ┌────────────────┐   │
│  │  短期记忆    │   │  长期记忆    │    │   用户画像     │   │
│  │ (In-Memory) │   │  (Qdrant)    │   │   (SQLite)     │   │
│  │             │   │              │   │                │   │
│  │ 当前对话轮次 │   │ 对话摘要向量  │   │ preferences    │   │
│  │ 当前任务状态 │   │ 重要决策记录  │   │ projects       │   │
│  │ 临时变量     │   │ 习惯偏好摘要  │   │ expertise      │   │
│  └─────────────┘   └──────────────┘   └────────────────┘   │
│         │                                                  │
│         └──── 对话结束时 ────→ 摘要 ──→ 长期记忆             │
└────────────────────────────────────────────────────────────┘
```

---

## 存储方案选型

| 方案 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| Qdrant | 语义记忆（按含义检索） | 语义相似度搜索 | 需要 embedding 步骤 |
| SQLite | 结构化画像（精确查询） | 简单快速，无需服务 | 不支持语义搜索 |
| Redis | 高频会话缓存 | 超快读写 | 内存限制，无持久化保证 |
| 文件系统 | 原始对话日志 | 最简单 | 无搜索能力 |

**推荐组合**：Qdrant（语义记忆）+ SQLite（用户画像）

---

## Qdrant Collection 设计

### memory Collection

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams

client = QdrantClient("http://localhost:6333")

client.create_collection(
    collection_name="user_memory",
    vectors_config=VectorParams(
        size=1024,           # bge-m3 维度
        distance=Distance.COSINE
    )
)
```

### Payload 设计

```json
{
  "user_id":      "default",
  "content":      "用户决定使用 PostgreSQL 而非 MySQL，原因是需要 JSONB 支持",
  "memory_type":  "decision",
  "session_id":   "sess_20260331_001",
  "timestamp":    "2026-03-31T10:00:00",
  "importance":   0.9,
  "tags":         ["tech_decision", "database"]
}
```

**memory_type 取值**：
- `conversation_summary` - 对话摘要（最常见）
- `preference` - 用户偏好（"喜欢简洁代码"）
- `decision` - 重要决策（"选择了 PostgreSQL"）
- `fact` - 用户事实（"住在上海"）
- `task_result` - 任务完成记录

---

## SQLite 用户画像 Schema

```python
import sqlite3

conn = sqlite3.connect("user_profile.db")

conn.executescript("""
CREATE TABLE IF NOT EXISTS profiles (
    user_id    TEXT PRIMARY KEY,
    profile_json TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS memory_log (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id    TEXT NOT NULL,
    session_id TEXT NOT NULL,
    memory_type TEXT NOT NULL,
    content    TEXT NOT NULL,
    created_at TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_memory_user ON memory_log(user_id);
CREATE INDEX IF NOT EXISTS idx_memory_session ON memory_log(session_id);
""")
conn.commit()
conn.close()
```

---

## 完整 Memory 服务

```python
# memory_service.py
import json
import uuid
import sqlite3
import hashlib
import requests
from datetime import datetime
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct, Filter, FieldCondition, MatchValue

OLLAMA_URL  = "http://localhost:11434"
QDRANT_URL  = "http://localhost:6333"
EMBED_MODEL = "bge-m3"
MEMORY_COL  = "user_memory"
VECTOR_DIM  = 1024
DB_PATH     = "user_profile.db"

qdrant = QdrantClient(url=QDRANT_URL)


class MemoryService:

    def __init__(self):
        self._ensure_collection()
        self._ensure_db()

    def _ensure_collection(self):
        existing = {c.name for c in qdrant.get_collections().collections}
        if MEMORY_COL not in existing:
            qdrant.create_collection(
                collection_name=MEMORY_COL,
                vectors_config=VectorParams(size=VECTOR_DIM, distance=Distance.COSINE)
            )

    def _ensure_db(self):
        with sqlite3.connect(DB_PATH) as conn:
            conn.executescript("""
                CREATE TABLE IF NOT EXISTS profiles (
                    user_id TEXT PRIMARY KEY,
                    profile_json TEXT,
                    updated_at TEXT
                );
                CREATE TABLE IF NOT EXISTS memory_log (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id TEXT, session_id TEXT,
                    memory_type TEXT, content TEXT, created_at TEXT
                );
                CREATE INDEX IF NOT EXISTS idx_ml_user ON memory_log(user_id);
            """)

    # ── Embedding ─────────────────────────────────────────────
    def _embed(self, text: str) -> list[float]:
        resp = requests.post(f"{OLLAMA_URL}/api/embeddings",
                             json={"model": EMBED_MODEL, "prompt": text})
        return resp.json()["embedding"]

    # ── 写入 ───────────────────────────────────────────────────
    def save(self, user_id: str, content: str,
             memory_type: str = "conversation_summary",
             session_id: str = "", importance: float = 0.7,
             tags: list[str] | None = None):
        vector = self._embed(content)
        pt_id = int(hashlib.md5(f"{user_id}:{content[:50]}:{datetime.now()}".encode())
                    .hexdigest()[:8], 16)

        qdrant.upsert(
            collection_name=MEMORY_COL,
            points=[PointStruct(
                id=pt_id,
                vector=vector,
                payload={
                    "user_id":     user_id,
                    "content":     content,
                    "memory_type": memory_type,
                    "session_id":  session_id,
                    "timestamp":   datetime.now().isoformat(),
                    "importance":  importance,
                    "tags":        tags or []
                }
            )]
        )

        with sqlite3.connect(DB_PATH) as conn:
            conn.execute(
                "INSERT INTO memory_log VALUES (NULL,?,?,?,?,?)",
                (user_id, session_id, memory_type, content, datetime.now().isoformat())
            )

    # ── 检索 ───────────────────────────────────────────────────
    def recall(self, user_id: str, query: str, k: int = 5,
               memory_type: str | None = None) -> list[dict]:
        vector = self._embed(query)
        must = [FieldCondition(key="user_id", match=MatchValue(value=user_id))]
        if memory_type:
            must.append(FieldCondition(key="memory_type",
                                       match=MatchValue(value=memory_type)))

        hits = qdrant.search(
            collection_name=MEMORY_COL,
            query_vector=vector,
            query_filter=Filter(must=must),
            limit=k,
            with_payload=True
        )
        return [{"content": h.payload["content"], "score": h.score,
                 "type": h.payload.get("memory_type"), "ts": h.payload.get("timestamp")}
                for h in hits]

    # ── 最近记忆 ───────────────────────────────────────────────
    def get_recent(self, user_id: str, limit: int = 10) -> list[dict]:
        with sqlite3.connect(DB_PATH) as conn:
            rows = conn.execute(
                "SELECT memory_type, content, created_at FROM memory_log "
                "WHERE user_id=? ORDER BY created_at DESC LIMIT ?",
                (user_id, limit)
            ).fetchall()
        return [{"type": r[0], "content": r[1], "ts": r[2]} for r in rows]

    # ── System Prompt 注入 ────────────────────────────────────
    def build_context(self, user_id: str, current_topic: str) -> str:
        semantic = self.recall(user_id, current_topic, k=5)
        recent   = self.get_recent(user_id, limit=3)

        semantic_block = "\n".join(f"- {m['content']}" for m in semantic) or "暂无"
        recent_block   = "\n".join(f"- [{m['type']}] {m['content'][:80]}"
                                   for m in recent) or "暂无"

        return f"""【语义相关记忆】
{semantic_block}

【最近记录】
{recent_block}"""


# 单例
memory = MemoryService()
```

---

## 记忆质量优化

### 1. 过滤低价值摘要

```python
def is_worth_saving(summary: str) -> bool:
    """过滤无意义的摘要（如纯闲聊）"""
    worthless = ["谢谢", "没事", "好的", "再见", "随便", "你好"]
    if len(summary.strip()) < 20:
        return False
    if all(w in summary for w in worthless):
        return False
    return True
```

### 2. 重要度评分

```python
def score_importance(content: str, memory_type: str) -> float:
    base = {"decision": 0.9, "preference": 0.8,
            "fact": 0.7, "conversation_summary": 0.5}.get(memory_type, 0.5)
    # 包含技术关键词加权
    tech_keywords = ["架构", "选择", "决定", "最终", "确定"]
    bonus = 0.1 if any(k in content for k in tech_keywords) else 0.0
    return min(base + bonus, 1.0)
```

### 3. 记忆去重

```python
def deduplicate(user_id: str, new_content: str, threshold: float = 0.95) -> bool:
    """若与现有记忆相似度过高则跳过保存"""
    existing = memory.recall(user_id, new_content, k=3)
    return any(m["score"] > threshold for m in existing)
```

---

## 性能基准

| 操作 | 典型延迟 | 说明 |
|------|----------|------|
| 单次 embedding (bge-m3) | 50~150ms | GPU 加速 |
| Qdrant 写入 | 5~20ms | 本地部署 |
| 语义检索 (k=5) | 10~50ms | 含 embedding |
| SQLite 写入 | < 5ms | 本地文件 |

**端对端记忆写入总延迟**：~200ms（可异步化至接近 0ms 用户感知）

---

## 异步化记忆写入

```python
import asyncio
import concurrent.futures

executor = concurrent.futures.ThreadPoolExecutor(max_workers=2)

async def save_memory_async(user_id: str, content: str, memory_type: str = "conversation_summary"):
    """后台保存，不阻塞主对话流程"""
    loop = asyncio.get_event_loop()
    await loop.run_in_executor(executor,
        lambda: memory.save(user_id, content, memory_type))

# 使用方式：
# await save_memory_async("default", summary, "conversation_summary")
```

---

## 参考链接

- [对话摘要与触发策略](./conversation.md)
- [用户画像建模](./user_profile.md)
- [Qdrant 部署详情](../02-rag/qdrant.md)
- [Embedding 模型选择](../01-inference/embedding.md)
