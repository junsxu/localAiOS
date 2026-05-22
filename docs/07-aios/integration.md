# 全系统集成与自检清单

## 集成顺序

按以下顺序验证，每一步确认正常后再进行下一步：

```
基础设施层
  └── Ollama（WSL dGPU + Windows iGPU）
  └── Nginx（反向代理）
  └── Qdrant（向量数据库）
        │
推理 + RAG 层
  └── 模型加载 + Embedding 验证
  └── RAG 管道（文档入库 + 检索）
        │
Agent + Memory 层
  └── Tool Calling 验证
  └── Memory 读写验证
        │
微服务编排层
  └── Docker Compose（RAG/Agent/Memory 服务）
  └── Nginx 路由验证
        │
路由 + 优化层
  └── Router Service 验证
  └── 性能基准测试
        │
全系统集成测试
```

---

## 逐层自检脚本

```python
# integration_check.py
import requests
import subprocess
import sys

CHECKS = []

def check(name: str, fn) -> bool:
    try:
        ok = fn()
        status = "✅" if ok else "❌"
        print(f"  {status} {name}")
        CHECKS.append((name, ok))
        return ok
    except Exception as e:
        print(f"  ❌ {name} ({type(e).__name__}: {e})")
        CHECKS.append((name, False))
        return False


# ── 层 1：基础服务 ────────────────────────────────────────
def check_ollama_wsl():
    r = requests.get("http://localhost:11434/api/tags", timeout=5)
    models = [m["name"] for m in r.json().get("models", [])]
    return len(models) > 0

def check_ollama_win():
    r = requests.get("http://localhost:11435/api/tags", timeout=5)
    return r.ok

def check_qdrant():
    r = requests.get("http://localhost:6333/healthz", timeout=5)
    return r.ok

def check_nginx():
    r = requests.get("http://localhost/health", timeout=5)
    return r.ok


# ── 层 2：推理 + Embedding ────────────────────────────────
def check_inference():
    r = requests.post("http://localhost:11434/api/generate", json={
        "model": "qwen2.5:14b",
        "prompt": "reply with the word 'ok' only",
        "stream": False,
        "options": {"num_predict": 5}
    }, timeout=30)
    return r.ok and "response" in r.json()

def check_embedding():
    r = requests.post("http://localhost:11434/api/embeddings", json={
        "model": "bge-m3",
        "prompt": "test"
    }, timeout=30)
    vec = r.json().get("embedding", [])
    return len(vec) > 100


# ── 层 3：RAG ─────────────────────────────────────────────
def check_rag_collection():
    r = requests.get("http://localhost:6333/collections", timeout=5)
    cols = [c["name"] for c in r.json().get("result", {}).get("collections", [])]
    return len(cols) > 0

def check_rag_service():
    r = requests.get("http://localhost:8001/health", timeout=5)
    return r.ok


# ── 层 4：Memory ─────────────────────────────────────────
def check_memory_service():
    r = requests.get("http://localhost:8003/health", timeout=5)
    return r.ok


# ── 层 5：Agent ──────────────────────────────────────────
def check_agent_service():
    r = requests.get("http://localhost:8002/health", timeout=5)
    return r.ok


# ── 层 6：路由服务 ───────────────────────────────────────
def check_router():
    r = requests.get("http://localhost:8000/health", timeout=5)
    return r.ok

def check_router_routing():
    r = requests.post("http://localhost:8000/chat", json={
        "prompt": "翻译：hello",
        "force_model": "simple"
    }, timeout=30)
    return r.ok and "response" in r.json()


# ── 全系统端对端测试 ─────────────────────────────────────
def check_e2e():
    """端对端：通过 Nginx 路由，调用 Agent，使用 RAG + Memory"""
    r = requests.post("http://localhost/api/agent/run", json={
        "task": "你好，请告诉我你是什么系统",
        "user_id": "integration_test"
    }, timeout=60)
    return r.ok and "answer" in r.json()


def run_all_checks():
    print("\n" + "="*55)
    print("  Local AI OS 全系统自检")
    print("="*55 + "\n")

    print("[ 基础服务 ]")
    check("Ollama WSL（dGPU 推理）",   check_ollama_wsl)
    check("Ollama Windows（iGPU）",     check_ollama_win)
    check("Qdrant 向量数据库",          check_qdrant)
    check("Nginx API 网关",             check_nginx)

    print("\n[ 推理 + Embedding ]")
    check("LLM 推理（qwen2.5:14b）",    check_inference)
    check("Embedding（bge-m3）",        check_embedding)

    print("\n[ RAG 知识库 ]")
    check("Qdrant Collection 存在",     check_rag_collection)
    check("RAG Service 健康",           check_rag_service)

    print("\n[ Memory 记忆层 ]")
    check("Memory Service 健康",        check_memory_service)

    print("\n[ Agent 执行层 ]")
    check("Agent Service 健康",         check_agent_service)

    print("\n[ 路由层 ]")
    check("Router Service 健康",        check_router)
    check("路由决策正常",               check_router_routing)

    print("\n[ 端对端集成 ]")
    check("全链路 E2E 测试",            check_e2e)

    # 汇总
    total = len(CHECKS)
    passed = sum(1 for _, ok in CHECKS if ok)
    print(f"\n{'='*55}")
    print(f"结果：{passed}/{total} 项通过")
    if passed == total:
        print("✅ 全系统运行正常！")
    else:
        failed = [name for name, ok in CHECKS if not ok]
        print(f"❌ 以下检查未通过：")
        for name in failed:
            print(f"   - {name}")
    print("="*55 + "\n")
    return passed == total


if __name__ == "__main__":
    ok = run_all_checks()
    sys.exit(0 if ok else 1)
```

---

## 全系统启动顺序

```bash
#!/bin/bash
# start_all.sh

echo "=== 启动 Local AI OS ==="

# 1. 启动 WSL 服务（Ollama + Qdrant systemd 服务应已开机自启）
echo "▶ 检查 WSL 服务..."
wsl systemctl is-active ollama || wsl systemctl start ollama
wsl systemctl is-active qdrant || wsl systemctl start qdrant

# 2. 启动 Windows Ollama（开机自启，验证即可）
echo "▶ 检查 Windows Ollama..."
curl -s http://localhost:11435/api/tags > /dev/null || echo "  ⚠️ Windows Ollama 未运行，请手动启动"

# 3. 启动 Nginx（Windows 服务）
echo "▶ 启动 Nginx..."
sc start nginx

# 4. 启动 Docker Compose 服务
echo "▶ 启动微服务..."
docker compose up -d

# 5. 等待服务就绪
echo "▶ 等待服务就绪..."
sleep 10

# 6. 健康检查
python integration_check.py
```

---

## 全系统关闭顺序

```bash
#!/bin/bash
# stop_all.sh

echo "=== 停止 Local AI OS ==="

# 1. 停止 Docker Compose
docker compose down

# 2. 停止 Nginx
sc stop nginx

# 注意：WSL 的 Ollama/Qdrant 通常保持运行
# 如需停止：wsl systemctl stop ollama qdrant
```

---

## 数据备份脚本

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "=== 备份 Local AI OS 数据 ==="

# 1. Qdrant 快照（向量数据库）
echo "▶ 备份 Qdrant..."
curl -X POST "http://localhost:6333/snapshots" > "$BACKUP_DIR/qdrant_snapshot.json"
# 获取快照名并下载
SNAPSHOT=$(cat "$BACKUP_DIR/qdrant_snapshot.json" | python3 -c "import sys,json;print(json.load(sys.stdin)['result']['name'])")
curl "http://localhost:6333/snapshots/$SNAPSHOT" -o "$BACKUP_DIR/qdrant_full.snapshot"

# 2. SQLite 用户画像
echo "▶ 备份 SQLite..."
docker compose exec memory-service cp /app/data/user_profile.db /tmp/
docker compose cp memory-service:/tmp/user_profile.db "$BACKUP_DIR/"

# 3. 配置文件
echo "▶ 备份配置..."
cp docker-compose.yml "$BACKUP_DIR/"
cp .env "$BACKUP_DIR/.env.bak"

echo "✅ 备份完成：$BACKUP_DIR"
du -sh "$BACKUP_DIR"
```

---

## 常见集成问题排查

| 问题 | 可能原因 | 排查命令 |
|------|----------|---------|
| RAG Service 无法访问 Qdrant | 网络不通 | `docker compose exec rag-service curl http://qdrant:6333/healthz` |
| Agent 无法访问 Ollama | `host.docker.internal` 未配置 | `docker compose exec agent-service curl http://host.docker.internal:11434/api/tags` |
| Memory Service 写入失败 | 数据卷权限问题 | `docker compose exec memory-service ls -la /app/data` |
| Nginx 502 Bad Gateway | 下游服务未就绪 | `docker compose ps && docker compose logs nginx` |
| 路由服务分类错误 | 小模型未加载 | `curl http://localhost:11434/api/tags \| grep qwen2.5:3b` |

---

## 参考链接

- [演进路线与未来规划](./roadmap.md)
- [Docker Compose 配置](../05-orchestration/docker_compose.md)
- [性能基准测试](../06-performance/benchmarks.md)
