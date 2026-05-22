# Qdrant 向量数据库详解

Qdrant 是高性能向量数据库，部署在 WSL 中，负责存储和检索文档的语义向量，支撑 RAG 知识库。

---

## 一、安装与启动

### 1.1 Docker 部署（推荐）

```bash
mkdir -p ~/qdrant/storage

cat > ~/qdrant/docker-compose.yml << 'EOF'
version: '3.8'
services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    restart: unless-stopped
    ports:
      - "6333:6333"   # REST API / Dashboard
      - "6334:6334"   # gRPC API（高性能场景）
    volumes:
      - ./storage:/qdrant/storage
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
      # 可选：API Key 认证
      # - QDRANT__SERVICE__API_KEY=your-secret-key
EOF

cd ~/qdrant
docker compose up -d

# 验证
curl http://localhost:6333/
# {"title":"qdrant - vector search engine","version":"..."}
```

### 1.2 注册为 systemd 服务

```bash
# 替换 <username> 为实际用户名（如 ubuntu）
sudo tee /etc/systemd/system/qdrant.service << 'EOF'
[Unit]
Description=Qdrant Vector Database
After=network.target docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/<username>/qdrant
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
User=<username>

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now qdrant
sudo systemctl status qdrant
```

### 1.3 Web Dashboard

浏览器访问 `http://localhost:6333/dashboard`：
- 浏览所有 Collection 及数据量
- 执行语义搜索测试
- 查看集合配置和统计信息

---

## 二、Collection 管理

### 2.1 创建 Collection

```bash
# nomic-embed-text → 768 维
curl -X PUT http://localhost:6333/collections/ai-docs \
  -H "Content-Type: application/json" \
  -d '{
    "vectors": {
      "size": 768,
      "distance": "Cosine"
    },
    "optimizers_config": {
      "default_segment_number": 2,
      "memmap_threshold": 20000
    },
    "hnsw_config": {
      "m": 16,
      "ef_construct": 100
    }
  }'
```

**参数说明**：

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `size` | 向量维度，必须与 Embedding 模型一致 | nomic: 768, bge-m3: 1024 |
| `distance` | 相似度计算方式 | `Cosine`（推荐）/ `Dot` / `Euclid` |
| `m` | HNSW 图每节点连接数，越大精度越高但更占内存 | 16（默认） |
| `ef_construct` | 构建索引时的搜索宽度，越大精度越高但构建慢 | 100（默认） |

### 2.2 Python 创建（idempotent）

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams

client = QdrantClient("http://localhost:6333")

def ensure_collection(name: str, size: int = 768):
    """创建集合（已存在则跳过）"""
    existing = {c.name for c in client.get_collections().collections}
    if name not in existing:
        client.create_collection(
            collection_name=name,
            vectors_config=VectorParams(size=size, distance=Distance.COSINE)
        )
        print(f"✅ Collection '{name}' 已创建（{size} 维）")
    else:
        info = client.get_collection(name)
        print(f"ℹ️  Collection '{name}' 已存在，包含 {info.points_count} 条向量")
```

### 2.3 常用管理命令

```bash
# 列出所有 Collection
curl http://localhost:6333/collections

# 查看 Collection 详情（点数量、配置）
curl http://localhost:6333/collections/ai-docs

# 删除 Collection（谨慎！）
curl -X DELETE http://localhost:6333/collections/ai-docs

# 清空 Collection 内所有点（保留配置）
curl -X POST http://localhost:6333/collections/ai-docs/points/delete \
  -H "Content-Type: application/json" \
  -d '{"filter": {}}'
```

---

## 三、数据写入（Upsert）

### 3.1 基本 Upsert

```python
from qdrant_client.models import PointStruct

def upsert_points(collection: str, points: list[dict]):
    """
    points: [{"id": int, "vector": [...], "payload": {...}}, ...]
    """
    qdrant_points = [
        PointStruct(
            id=p["id"],
            vector=p["vector"],
            payload=p["payload"]
        )
        for p in points
    ]
    client.upsert(collection_name=collection, points=qdrant_points)
```

### 3.2 Payload 设计建议

```python
payload = {
    "text": chunk_text,           # 原始文本片段（检索后返回给 LLM）
    "source": "/mnt/nas/ai-docs/technical/ai/paper.pdf",  # 来源文件
    "file_name": "paper.pdf",     # 文件名
    "chunk_index": 3,             # 在文件中的第几块
    "page": 5,                    # 页码（PDF 用）
    "created_at": "2024-01-15",   # 索引时间
    "doc_type": "technical",      # 文档类别（用于过滤）
}
```

### 3.3 批量写入（分批）

```python
def upsert_batch(collection: str, all_points: list, batch_size: int = 100):
    """分批写入，避免内存压力"""
    for i in range(0, len(all_points), batch_size):
        batch = all_points[i:i+batch_size]
        client.upsert(collection_name=collection, points=batch)
        print(f"  写入 {min(i+batch_size, len(all_points))}/{len(all_points)}")
```

---

## 四、向量检索

### 4.1 基本搜索

```python
def search(collection: str, query_vector: list[float],
           top_k: int = 5, score_threshold: float = 0.5) -> list[dict]:
    results = client.search(
        collection_name=collection,
        query_vector=query_vector,
        limit=top_k,
        score_threshold=score_threshold,   # 过滤低相关性结果
        with_payload=True
    )
    return [
        {
            "score": r.score,
            "text": r.payload["text"],
            "source": r.payload.get("source", ""),
        }
        for r in results
    ]
```

### 4.2 带过滤条件的检索

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue

# 只搜索 technical 类型的文档
results = client.search(
    collection_name="ai-docs",
    query_vector=embed("机器学习基础"),
    query_filter=Filter(
        must=[
            FieldCondition(
                key="doc_type",
                match=MatchValue(value="technical")
            )
        ]
    ),
    limit=5
)
```

### 4.3 REST API 检索

```bash
curl -X POST http://localhost:6333/collections/ai-docs/points/search \
  -H "Content-Type: application/json" \
  -d '{
    "vector": [0.123, -0.456, ...],
    "limit": 5,
    "score_threshold": 0.5,
    "with_payload": true
  }'
```

---

## 五、存储管理

### 5.1 存储位置

```bash
# 查看 Qdrant 数据目录大小
du -sh ~/qdrant/storage/

# 内部结构
~/qdrant/storage/
├── collections/
│   └── ai-docs/
│       ├── segments/       # 向量索引分片
│       └── wal/            # Write-Ahead Log
└── meta.json
```

### 5.2 迁移到 NAS

当本地 SSD 空间不足时，将 Qdrant 存储迁移到 NAS：

```bash
sudo systemctl stop qdrant

# 修改 docker-compose.yml 中的卷路径
vi ~/qdrant/docker-compose.yml
# 将 ./storage 改为 /mnt/nas/qdrant/storage

# 迁移现有数据
sudo mkdir -p /mnt/nas/qdrant/storage
sudo cp -r ~/qdrant/storage/* /mnt/nas/qdrant/storage/

sudo systemctl start qdrant
```

> **注意**：NAS（机械盘）的随机读写性能远低于 SSD，会影响检索延迟。建议热集合保留在 SSD，冷集合才移到 NAS。

### 5.3 备份与恢复

```bash
# 备份（停止 Qdrant 后）
sudo systemctl stop qdrant
tar -czf ~/qdrant-backup-$(date +%Y%m%d).tar.gz ~/qdrant/storage/
sudo systemctl start qdrant

# 恢复
tar -xzf ~/qdrant-backup-20240115.tar.gz -C ~/
```

---

## 六、性能调优

### 6.1 查询性能参数

```python
# 搜索时设置 ef 参数（越大越精确，越慢）
results = client.search(
    collection_name="ai-docs",
    query_vector=vector,
    limit=5,
    search_params={"hnsw_ef": 128}  # 默认 ef=ef_construct
)
```

### 6.2 监控关键指标

```bash
# 集合统计
curl http://localhost:6333/collections/ai-docs | python3 -c "
import sys, json
d = json.load(sys.stdin)
info = d['result']
print(f'点数量: {info[\"points_count\"]}')
print(f'索引点数: {info[\"indexed_vectors_count\"]}')
print(f'状态: {info[\"status\"]}')
"
```

### 6.3 内存估算

| 向量维度 | 点数量 | 内存占用（约）|
|---------|--------|-------------|
| 768 | 10,000 | ~60MB |
| 768 | 100,000 | ~600MB |
| 768 | 1,000,000 | ~6GB |
| 1024 | 100,000 | ~800MB |

---

## 七、故障排查

| 问题 | 解决方法 |
|------|---------|
| 端口 6333 无法访问 | `docker ps` 确认容器运行 |
| 向量维度不匹配 | Collection `size` 必须等于 Embedding 模型输出维度 |
| 写入后检索不到 | 等待索引构建完成（大量数据时有延迟） |
| 搜索结果相关性差 | 检查 Embedding 模型语言支持，考虑换 `bge-m3` |
| 磁盘空间不足 | 检查 `~/qdrant/storage` 大小，迁移冷集合到 NAS |

---

> ⬅️ [返回阶段 2 概览](./README.md)
