# 模型路由器实现详解

## 路由策略 1：基于规则

最简单的实现，零额外推理开销：

```python
# rule_based_router.py

SIMPLE_PATTERNS = [
    "翻译", "总结一下", "是什么", "怎么写", "什么意思",
    "帮我格式化", "简单解释", "是否", "对不对", "几个",
]

COMPLEX_PATTERNS = [
    "分析", "设计方案", "写一个完整", "优化", "对比",
    "系统架构", "深入", "详细说明", "原理", "为什么",
    "实现", "步骤", "流程", "比较",
]


def route_by_rules(prompt: str) -> str:
    """返回 simple / medium / complex"""
    prompt_lower = prompt.lower()

    # 超长 prompt 默认复杂
    if len(prompt) > 500:
        return "complex"

    complex_score = sum(1 for p in COMPLEX_PATTERNS if p in prompt_lower)
    simple_score  = sum(1 for p in SIMPLE_PATTERNS  if p in prompt_lower)

    if complex_score >= 2 or (complex_score > 0 and simple_score == 0):
        return "complex"
    if simple_score > 0 and complex_score == 0:
        return "simple"
    return "medium"
```

---

## 路由策略 2：基于 LLM 分类（推荐）

用极小模型（3B）做分类，100ms 内完成：

```python
# llm_router.py
import requests
import time

OLLAMA_URL   = "http://localhost:11434"
ROUTER_MODEL = "qwen2.5:3b"   # 用最小模型做路由

MODELS = {
    "simple":  {"url": "http://localhost:11435", "model": "qwen2.5:3b"},    # Windows 核显
    "medium":  {"url": "http://localhost:11434", "model": "qwen2.5:7b"},    # WSL 独显
    "complex": {"url": "http://localhost:11434", "model": "qwen2.5:14b"},   # WSL 独显
}

CLASSIFY_PROMPT = """判断以下任务的复杂度，只能回答 simple/medium/complex 三种之一。

复杂度定义：
- simple: 翻译、格式化、简单问答、是非判断、摘要（< 500字原文）
- medium: 写代码（< 50行）、解释概念、生成内容、分析短文本
- complex: 系统设计、多步推理、深度分析、复杂算法、长文本处理

任务：{prompt}

复杂度（只输出一个单词）："""


def classify_complexity(prompt: str, timeout: float = 5.0) -> str:
    start = time.time()
    try:
        resp = requests.post(f"{OLLAMA_URL}/api/generate", json={
            "model": ROUTER_MODEL,
            "prompt": CLASSIFY_PROMPT.format(prompt=prompt[:300]),
            "stream": False,
            "options": {"temperature": 0, "num_predict": 10}
        }, timeout=timeout)
        result = resp.json()["response"].strip().lower()
        elapsed = time.time() - start

        if "complex" in result:
            return "complex"
        elif "medium" in result:
            return "medium"
        else:
            return "simple"
    except Exception:
        # 分类失败时降级到规则路由
        return route_by_rules(prompt)


def routed_generate(prompt: str, force_tier: str | None = None) -> dict:
    tier = force_tier or classify_complexity(prompt)
    target = MODELS[tier]

    resp = requests.post(f"{target['url']}/api/generate", json={
        "model": target["model"],
        "prompt": prompt,
        "stream": False
    }, timeout=300)

    return {
        "response": resp.json()["response"],
        "model_used": target["model"],
        "complexity": tier,
    }
```

---

## 路由策略 3：自适应（负载感知）

在高负载时自动降级，防止大模型被打爆：

```python
# adaptive_router.py
import time
from collections import deque
from threading import Lock


class AdaptiveRouter:
    """根据实时负载动态调整路由决策"""

    def __init__(self, latency_threshold: float = 8.0):
        self.latency_threshold = latency_threshold
        self._latencies: dict[str, deque] = {
            tier: deque(maxlen=20) for tier in ("simple", "medium", "complex")
        }
        self._lock = Lock()

    def record(self, tier: str, latency: float):
        with self._lock:
            self._latencies[tier].append(latency)

    def avg_latency(self, tier: str) -> float:
        with self._lock:
            hist = self._latencies[tier]
            return sum(hist) / len(hist) if hist else 0.0

    def route(self, base_tier: str) -> str:
        """在高负载时自动降级"""
        if base_tier == "complex":
            avg = self.avg_latency("complex")
            if avg > self.latency_threshold:
                print(f"⚠️ 复杂模型平均延迟 {avg:.1f}s，降级到 medium")
                return "medium"
        if base_tier == "medium":
            avg = self.avg_latency("medium")
            if avg > self.latency_threshold * 0.6:
                print(f"⚠️ 中等模型平均延迟 {avg:.1f}s，降级到 simple")
                return "simple"
        return base_tier

    def stats(self) -> dict:
        return {tier: round(self.avg_latency(tier), 2)
                for tier in ("simple", "medium", "complex")}


adaptive_router = AdaptiveRouter(latency_threshold=8.0)
```

---

## FastAPI 路由服务（完整版）

```python
# services/router/main.py
from fastapi import FastAPI
from pydantic import BaseModel
import requests, time, os

app = FastAPI(title="AI Router Service")

OLLAMA_URL   = os.getenv("OLLAMA_URL",   "http://localhost:11434")
OLLAMA_WIN   = os.getenv("OLLAMA_WIN",   "http://localhost:11435")
ROUTER_MODEL = os.getenv("ROUTER_MODEL", "qwen2.5:3b")

MODELS = {
    "simple":  {"url": OLLAMA_WIN, "model": os.getenv("SIMPLE_MODEL",  "qwen2.5:3b")},
    "medium":  {"url": OLLAMA_URL, "model": os.getenv("MEDIUM_MODEL",  "qwen2.5:7b")},
    "complex": {"url": OLLAMA_URL, "model": os.getenv("COMPLEX_MODEL", "qwen2.5:14b")},
}

router = AdaptiveRouter()


class ChatRequest(BaseModel):
    prompt: str
    force_model: str | None = None  # 可强制指定：simple/medium/complex
    stream: bool = False


class ChatResponse(BaseModel):
    response: str
    model_used: str
    complexity: str
    latency_ms: int
    router_stats: dict


@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest):
    t0 = time.time()

    # 1. 分类
    if req.force_model:
        tier = req.force_model
    else:
        tier = classify_complexity(req.prompt)

    # 2. 自适应路由
    tier = router.route(tier)
    target = MODELS[tier]

    # 3. 调用 LLM
    resp = requests.post(f"{target['url']}/api/generate", json={
        "model": target["model"],
        "prompt": req.prompt,
        "stream": False
    }, timeout=300)

    latency_ms = int((time.time() - t0) * 1000)
    router.record(tier, latency_ms / 1000)

    return ChatResponse(
        response   = resp.json()["response"],
        model_used = target["model"],
        complexity = tier,
        latency_ms = latency_ms,
        router_stats = router.stats()
    )


@app.get("/stats")
async def stats():
    return {
        "model_tiers": MODELS,
        "avg_latency": router.stats()
    }


@app.get("/health")
async def health():
    return {"status": "ok", "service": "router"}
```

---

## MoE 类脑结构（进阶）

多专家模型池，按任务类型路由到专业模型：

```python
EXPERT_MODELS = {
    "code":    {"url": OLLAMA_URL, "model": "qwen2.5-coder:7b"},
    "math":    {"url": OLLAMA_URL, "model": "qwen2.5-math:7b"},
    "general": {"url": OLLAMA_URL, "model": "qwen2.5:14b"},
    "embed":   {"url": OLLAMA_WIN, "model": "bge-m3"},
}

TASK_CLASSIFY_PROMPT = """判断以下任务的类型，只能回答 code/math/general 三种之一。

- code: 写代码、debug、代码审查、技术实现
- math: 数学推导、计算、公式推理
- general: 其他（写作、分析、问答等）

任务：{prompt}

类型："""


def classify_task_type(prompt: str) -> str:
    resp = requests.post(f"{OLLAMA_URL}/api/generate", json={
        "model": ROUTER_MODEL,
        "prompt": TASK_CLASSIFY_PROMPT.format(prompt=prompt[:300]),
        "stream": False,
        "options": {"temperature": 0, "num_predict": 5}
    })
    result = resp.json()["response"].strip().lower()
    for t in ("code", "math"):
        if t in result:
            return t
    return "general"
```

---

## 路由决策日志

```python
import logging

logging.basicConfig(level=logging.INFO,
                    format='%(asctime)s [ROUTER] %(message)s')

def log_routing(prompt: str, tier: str, model: str, latency_ms: int):
    logging.info(
        f"tier={tier} model={model} latency={latency_ms}ms "
        f"prompt_len={len(prompt)} "
        f"prompt_preview={prompt[:50].replace(chr(10), ' ')!r}"
    )
```

---

## 参考链接

- [性能基准测试](./benchmarks.md)
- [Ollama API 参考](../01-inference/ollama_api.md)
- [采样参数调优](../01-inference/sampling.md)
