# 演进路线与未来规划

## 当前系统状态（阶段 0-7 完成后）

```
Local AI OS v1.0
├── ✅ 基础设施：WSL + Nginx + Ollama + Qdrant
├── ✅ 推理：双 Ollama 实例（iGPU embedding + dGPU 推理）
├── ✅ RAG：文档知识库 + 语义检索 + 混合检索
├── ✅ Agent：ReAct + Tool Calling + 安全沙箱
├── ✅ Memory：对话摘要 + 长期 Qdrant + 用户画像
├── ✅ 编排：Docker 微服务 + Nginx API 网关
└── ✅ 路由：规则/LLM/自适应三策略路由
```

---

## 近期演进（1-3 个月）

### 1. Memory 系统完善

**目标**：AI 真正"认识"你，主动利用历史

```python
# 待实现：知识图谱式记忆
class KnowledgeGraph:
    """
    将分散的记忆连接成图结构
    例如：用户 → 项目 → 技术栈 → 最佳实践
    """
    def add_relation(self, subject: str, relation: str, object: str):
        ...

    def query_related(self, entity: str, hops: int = 2) -> list[str]:
        ...
```

**具体任务**：
- [ ] 记忆重要度衰减（时间加权，旧记忆权重降低）
- [ ] 跨会话记忆融合（合并重复/矛盾的记忆）
- [ ] 主动记忆召回（AI 主动询问上次未解决的问题）

---

### 2. RAG 质量提升

**目标**：检索准确率从 70% → 90%+

```
当前：基础向量检索
    │
    ├── 混合检索（BM25 + Vector + RRF）  ← 已实现
    ├── Rerank（Cross-Encoder 精排）     ← 下一步
    └── HyDE（假设文档扩展查询）         ← 规划中
```

**具体任务**：
- [ ] 集成 BGE-Reranker 进行精排
- [ ] 实现 HyDE（Hypothetical Document Embeddings）
- [ ] 构建评估数据集，量化检索质量

---

### 3. 路由系统优化

**目标**：路由准确率提升，延迟降低

```python
# 待实现：缓存路由决策
from functools import lru_cache
import hashlib

@lru_cache(maxsize=1000)
def cached_classify(prompt_hash: str) -> str:
    ...

def classify_with_cache(prompt: str) -> str:
    # 对相似的 prompt 复用路由决策
    prompt_hash = hashlib.md5(prompt[:100].encode()).hexdigest()
    return cached_classify(prompt_hash)
```

- [ ] 路由决策缓存（相似 prompt 复用决策）
- [ ] A/B 测试框架（对比不同路由策略效果）
- [ ] 基于用户反馈的路由自调优

---

### 4. 可观测性

**目标**：所有关键指标可视化

```yaml
# docker-compose 添加监控组件
services:
  prometheus:
    image: prom/prometheus
    ports: ["9090:9090"]
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports: ["3001:3000"]    # 宿主机 3001（3000 已被 OpenHands 占用）
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

**Grafana 看板指标**：
- 各模型请求量 / 延迟 / 错误率
- RAG 检索命中率
- Memory 写入/召回频率
- 系统资源（GPU 利用率、内存）

---

## 中期演进（Mac Studio 落地后）

### 架构迁移

```
当前（WSL + Windows 分裂架构）
  Windows iGPU  →  Embedding（bge-m3）
  WSL dGPU      →  推理（14B/32B）
  NAS           →  存储

目标（Mac Studio 256GB 统一架构）
  Apple Silicon →  Embedding + 推理（统一内存）
  7B   常驻（~14GB）  →  快速响应
  14B  常驻（~28GB）  →  日常任务
  32B  常驻（~64GB）  →  复杂推理
  70B  按需（~45GB Q4）→  深度任务
```

### Mac Studio 迁移步骤

```bash
# 1. 在 Mac Studio 安装 Ollama（macOS 原生）
brew install ollama

# 2. 拉取常驻模型
ollama pull qwen2.5:7b
ollama pull qwen2.5:14b
ollama pull qwen2.5:32b
ollama pull bge-m3

# 3. 迁移 Qdrant 数据（从 NAS 恢复快照）
curl -X POST "http://localhost:6333/collections/documents/snapshots/recover" \
  -H "Content-Type: application/json" \
  -d '{"location": "file:///data/backups/qdrant_full.snapshot"}'

# 4. 更新 docker-compose.yml
# 修改 OLLAMA_URL 和 OLLAMA_WIN 为 Mac Studio 的 IP 地址
```

### Mac 特有优化

```bash
# 使用 Metal 加速（Apple Silicon 原生）
export OLLAMA_NUM_GPU=1      # 使用 GPU
export OLLAMA_METAL=1        # 启用 Metal

# 利用统一内存的大上下文窗口
# Mac Studio 256GB 可以运行 70B 模型的 128K ctx
```

---

## 远期演进（AI OS v2.0）

### 感知层扩展

```
v1.0 感知层
  └── 文本输入（REST API / CLI）

v2.0 感知层
  ├── 文本输入（REST API / CLI）
  ├── 语音输入（Whisper 本地转录）
  ├── 图像理解（多模态模型）
  ├── 屏幕截图分析（OCR + 理解）
  └── 定时任务（APScheduler 周期执行）
```

```python
# 规划：语音输入集成
import whisper  # pip install openai-whisper

def transcribe_audio(audio_path: str) -> str:
    model = whisper.load_model("base")
    result = model.transcribe(audio_path, language="zh")
    return result["text"]
```

### 主动推送能力

```python
# 规划：AI 主动提醒
from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler()

@scheduler.scheduled_job("cron", hour=9, minute=0)
async def morning_briefing():
    """每天早 9 点主动推送：昨天的总结 + 今天的建议"""
    summary = await get_yesterday_summary()
    suggestions = await generate_daily_plan()
    await send_notification(f"早安！\n{summary}\n\n今日建议：\n{suggestions}")
```

### 跨设备同步

```
目标：手机 / iPad / Mac / Windows 统一记忆

实现思路：
  1. 记忆存储在 NAS（中心化）
  2. 各设备通过 API 读写
  3. 本地缓存最近 N 条记忆（离线可用）
  4. 同步冲突解决（时间戳 + 内容哈希）
```

### 自我优化

```python
# 规划：根据使用反馈自动调整行为
class SelfOptimizer:
    """
    收集用户反馈（👍/👎），用于：
    1. 调整路由阈值（哪类问题用哪个模型）
    2. 调整 RAG 参数（chunk size、top_k）
    3. 更新用户画像（自动学习新偏好）
    """
    def record_feedback(self, request_id: str, rating: int, comment: str = ""):
        ...

    def analyze_patterns(self) -> dict:
        """定期分析反馈，输出优化建议"""
        ...

    def apply_optimizations(self, suggestions: dict):
        ...
```

---

## 开放能力（生态扩展）

### 插件系统设计

```python
# 规划：标准化工具插件接口
from abc import ABC, abstractmethod

class AIToolPlugin(ABC):
    name: str
    description: str

    @abstractmethod
    def execute(self, params: dict) -> dict:
        """执行工具，返回结果"""
        ...

    @abstractmethod
    def get_schema(self) -> dict:
        """返回 Ollama Tool Calling 格式的 schema"""
        ...

# 示例插件
class GitPlugin(AIToolPlugin):
    name = "git_operations"
    description = "Execute git operations on a repository"

    def execute(self, params: dict) -> dict:
        command = params.get("command")
        repo = params.get("repo_path", ".")
        # 安全验证 + 执行
        ...

    def get_schema(self) -> dict:
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": {
                    "type": "object",
                    "properties": {
                        "command": {"type": "string"},
                        "repo_path": {"type": "string"}
                    }
                }
            }
        }
```

---

## 里程碑汇总

| 里程碑 | 目标 | 时间线 |
|--------|------|--------|
| v1.0 | 完成阶段 0-7，全系统运行 | 当前 |
| v1.1 | Memory 知识图谱 + RAG Rerank | 1 个月 |
| v1.2 | Grafana 监控 :3001 + A/B 路由测试 | 2 个月 |
| v1.3 | 语音输入 + 定时任务 | 3 个月 |
| v2.0 | Mac Studio 迁移，统一内存架构 | Mac 到货后 |
| v2.1 | 主动推送 + 跨设备同步 | v2.0 后 2 个月 |
| v3.0 | 多模态 + 自我优化 + 插件生态 | 远期 |

---

## 参考链接

- [全系统集成与自检清单](./integration.md)
- [阶段 6 性能优化](../06-performance/README.md)
- [阶段 4 Memory 系统](../04-memory/README.md)
- [阶段 2 RAG](../02-rag/README.md)
