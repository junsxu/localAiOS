# 微服务设计与服务边界

## 服务拆分原则

每个服务应满足：
1. **单一职责** - 只做一件事，只暴露与该职责相关的接口
2. **独立部署** - 单独构建、启动、扩容、回滚
3. **故障隔离** - 一个服务崩溃不影响其他服务
4. **数据独立** - 不直接访问其他服务的数据库

---

## 服务目录结构

```
localAiOS/
├── services/
│   ├── rag/
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── main.py
│   ├── agent/
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── main.py
│   └── memory/
│       ├── Dockerfile
│       ├── requirements.txt
│       └── main.py
├── nginx/
│   └── aios.conf
├── docker-compose.yml
└── .env
```

---

## Dockerfile 模板

所有 Python 服务共用相同模板：

```dockerfile
# services/<name>/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 先复制依赖文件（利用缓存层）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY . .

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:${PORT:-8001}/health || exit 1

CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001"]
```

---

## RAG Service

```python
# services/rag/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import requests, os

app = FastAPI(title="RAG Service", version="1.0.0")

OLLAMA_URL  = os.getenv("OLLAMA_URL",  "http://host.docker.internal:11434")
QDRANT_URL  = os.getenv("QDRANT_URL",  "http://qdrant:6333")
EMBED_MODEL = os.getenv("EMBED_MODEL", "bge-m3")


class IngestRequest(BaseModel):
    collection: str
    text: str
    source: str = ""
    metadata: dict = {}


class QueryRequest(BaseModel):
    collection: str
    query: str
    top_k: int = 5
    score_threshold: float = 0.5


@app.post("/ingest")
async def ingest(req: IngestRequest):
    # 获取 embedding
    embed_resp = requests.post(f"{OLLAMA_URL}/api/embeddings", json={
        "model": EMBED_MODEL, "prompt": req.text
    })
    if embed_resp.status_code != 200:
        raise HTTPException(500, "Embedding failed")
    vector = embed_resp.json()["embedding"]

    # 写入 Qdrant
    import hashlib, time
    point_id = int(hashlib.md5(f"{req.source}:{time.time()}".encode())
                   .hexdigest()[:8], 16)
    qdrant_resp = requests.put(f"{QDRANT_URL}/collections/{req.collection}/points", json={
        "points": [{
            "id": point_id,
            "vector": vector,
            "payload": {"text": req.text, "source": req.source, **req.metadata}
        }]
    })
    return {"status": "ok", "point_id": point_id, "collection": req.collection}


@app.post("/query")
async def query(req: QueryRequest):
    embed_resp = requests.post(f"{OLLAMA_URL}/api/embeddings", json={
        "model": EMBED_MODEL, "prompt": req.query
    })
    vector = embed_resp.json()["embedding"]

    search_resp = requests.post(
        f"{QDRANT_URL}/collections/{req.collection}/points/search",
        json={"vector": vector, "limit": req.top_k, "with_payload": True,
              "score_threshold": req.score_threshold}
    )
    results = search_resp.json().get("result", [])
    return {
        "chunks": [{"text": r["payload"]["text"],
                    "score": r["score"],
                    "source": r["payload"].get("source", "")}
                   for r in results],
        "count": len(results)
    }


@app.get("/health")
async def health():
    return {"status": "ok", "service": "rag"}
```

```
# services/rag/requirements.txt
fastapi>=0.110.0
uvicorn>=0.29.0
pydantic>=2.0.0
requests>=2.31.0
```

---

## Agent Service

```python
# services/agent/main.py
from fastapi import FastAPI
from pydantic import BaseModel
import requests, os, json

app = FastAPI(title="Agent Service", version="1.0.0")

OLLAMA_URL = os.getenv("OLLAMA_URL", "http://host.docker.internal:11434")
RAG_URL    = os.getenv("RAG_URL",    "http://rag-service:8001")
MEMORY_URL = os.getenv("MEMORY_URL", "http://memory-service:8003")
CHAT_MODEL = os.getenv("CHAT_MODEL", "qwen2.5:14b")


class AgentRequest(BaseModel):
    task: str
    user_id: str = "default"
    collection: str = "documents"
    max_steps: int = 5


@app.post("/run")
async def run_agent(req: AgentRequest):
    steps = []

    # ① 获取用户记忆
    mem_resp = requests.get(f"{MEMORY_URL}/recall",
                            params={"user_id": req.user_id, "query": req.task, "k": 3})
    memories = mem_resp.json().get("memories", []) if mem_resp.ok else []

    # ② 检索相关知识
    rag_resp = requests.post(f"{RAG_URL}/query", json={
        "collection": req.collection, "query": req.task, "top_k": 3
    })
    chunks = rag_resp.json().get("chunks", []) if rag_resp.ok else []
    context = "\n".join(c["text"] for c in chunks)

    # ③ 构建 System Prompt
    memory_block = "\n".join(f"- {m}" for m in memories) or "暂无"
    system = f"""你是一个 AI 助手，请完成用户任务。

相关知识：
{context or '暂无'}

用户记忆：
{memory_block}

使用 ReAct 格式：
思考：...
行动：...
观察：...
结论：..."""

    # ④ 调用 LLM
    messages = [
        {"role": "system", "content": system},
        {"role": "user",   "content": req.task}
    ]
    resp = requests.post(f"{OLLAMA_URL}/api/chat", json={
        "model": CHAT_MODEL,
        "messages": messages,
        "stream": False
    })
    answer = resp.json()["message"]["content"]

    return {
        "task": req.task,
        "answer": answer,
        "steps": steps,
        "context_chunks": len(chunks),
        "memories_used": len(memories)
    }


@app.get("/health")
async def health():
    return {"status": "ok", "service": "agent"}
```

---

## Memory Service

```python
# services/memory/main.py
from fastapi import FastAPI
from pydantic import BaseModel
import requests, os, sqlite3, json, hashlib, time
from datetime import datetime

app = FastAPI(title="Memory Service", version="1.0.0")

OLLAMA_URL  = os.getenv("OLLAMA_URL",  "http://host.docker.internal:11434")
QDRANT_URL  = os.getenv("QDRANT_URL",  "http://qdrant:6333")
EMBED_MODEL = os.getenv("EMBED_MODEL", "bge-m3")
DATA_DIR    = os.getenv("DATA_DIR",    "/app/data")
MEMORY_COL  = "user_memory"

os.makedirs(DATA_DIR, exist_ok=True)
DB_PATH = f"{DATA_DIR}/user_profile.db"


class SaveRequest(BaseModel):
    user_id: str
    content: str
    memory_type: str = "conversation_summary"
    session_id: str = ""


class RecallRequest(BaseModel):
    user_id: str
    query: str
    k: int = 5


def embed(text: str) -> list[float]:
    r = requests.post(f"{OLLAMA_URL}/api/embeddings",
                      json={"model": EMBED_MODEL, "prompt": text})
    return r.json()["embedding"]


@app.post("/save")
async def save(req: SaveRequest):
    vector = embed(req.content)
    pt_id = int(hashlib.md5(
        f"{req.user_id}:{req.content[:30]}:{time.time()}".encode()
    ).hexdigest()[:8], 16)

    requests.put(f"{QDRANT_URL}/collections/{MEMORY_COL}/points", json={
        "points": [{
            "id": pt_id,
            "vector": vector,
            "payload": {
                "user_id": req.user_id, "content": req.content,
                "memory_type": req.memory_type, "session_id": req.session_id,
                "timestamp": datetime.now().isoformat()
            }
        }]
    })
    return {"status": "ok", "point_id": pt_id}


@app.get("/recall")
async def recall(user_id: str, query: str, k: int = 5):
    vector = embed(query)
    resp = requests.post(
        f"{QDRANT_URL}/collections/{MEMORY_COL}/points/search",
        json={
            "vector": vector, "limit": k, "with_payload": True,
            "filter": {"must": [{"key": "user_id", "match": {"value": user_id}}]}
        }
    )
    results = resp.json().get("result", [])
    return {"memories": [r["payload"]["content"] for r in results]}


@app.get("/health")
async def health():
    return {"status": "ok", "service": "memory"}
```

---

## 服务间通信最佳实践

### 超时设置

```python
# 每个对外请求都设置超时，防止级联失败
resp = requests.post(url, json=payload, timeout=(5, 30))
# (connect_timeout, read_timeout)
```

### 服务发现（Docker 内网）

```
# Docker Compose 内，服务名直接作为主机名
# rag-service → http://rag-service:8001
# 宿主机 Ollama → http://host.docker.internal:11434
```

### 优雅降级

```python
def rag_query_with_fallback(query: str, collection: str) -> list[str]:
    try:
        resp = requests.post(f"{RAG_URL}/query",
                             json={"collection": collection, "query": query, "top_k": 3},
                             timeout=(3, 10))
        if resp.ok:
            return [c["text"] for c in resp.json().get("chunks", [])]
    except requests.exceptions.RequestException:
        pass
    return []   # RAG 失败时返回空列表，不阻断主流程
```

---

## 参考链接

- [Docker Compose 配置](./docker_compose.md)
- [Nginx API 网关](./api_gateway.md)
- [RAG 管道实现](../02-rag/rag_pipeline.md)
- [Tool Calling 实现](../03-agent/tool_calling.md)
- [Memory 架构](../04-memory/memory.md)
