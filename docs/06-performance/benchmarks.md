# 性能基准测试与监控

## 测试维度

| 维度 | 指标 | 目标 |
|------|------|------|
| 首 token 延迟 (TTFT) | ms | < 500ms（7B） |
| 吞吐量 | tokens/s | > 20 tok/s（7B GPU） |
| 端对端延迟 | ms | < 3s（短回答） |
| 路由开销 | ms | < 200ms |
| 并发处理 | req/s | > 2 req/s（7B） |

---

## 单模型基准测试脚本

```python
# benchmarks/model_benchmark.py
import requests
import time
import statistics
import argparse

OLLAMA_URL = "http://localhost:11434"

TEST_PROMPTS = {
    "simple": [
        "What is 1+1?",
        "Translate to Chinese: Hello World",
        "翻译成英文：你好世界",
    ],
    "medium": [
        "Write a Python function to calculate the Fibonacci sequence",
        "Explain the difference between TCP and UDP in 3 sentences",
        "用 Python 写一个读取 JSON 文件的函数",
    ],
    "complex": [
        "Design a microservices architecture for a real-time chat application with 1M users",
        "Explain how Transformer attention mechanism works mathematically",
        "分析 RAG 系统中 chunk 策略对检索质量的影响，给出优化建议",
    ],
}


def benchmark_model(model: str, tier: str = "medium", num_runs: int = 5) -> dict:
    prompts = TEST_PROMPTS.get(tier, TEST_PROMPTS["medium"])
    results = []

    for i, prompt in enumerate(prompts[:num_runs], 1):
        t0 = time.time()
        resp = requests.post(f"{OLLAMA_URL}/api/generate", json={
            "model": model,
            "prompt": prompt,
            "stream": False,
            "options": {"temperature": 0, "num_predict": 200}
        }, timeout=120)
        elapsed = time.time() - t0

        data = resp.json()
        token_count = data.get("eval_count", 0)
        tokens_per_sec = token_count / elapsed if elapsed > 0 else 0

        results.append({
            "prompt_num": i,
            "elapsed_s": round(elapsed, 2),
            "tokens": token_count,
            "tok_per_s": round(tokens_per_sec, 1),
        })
        print(f"  [{i}/{num_runs}] {elapsed:.2f}s | {token_count} tokens | "
              f"{tokens_per_sec:.1f} tok/s")

    latencies = [r["elapsed_s"] for r in results]
    return {
        "model": model,
        "tier": tier,
        "runs": num_runs,
        "avg_s":    round(statistics.mean(latencies), 2),
        "p50_s":    round(statistics.median(latencies), 2),
        "p95_s":    round(sorted(latencies)[int(0.95 * len(latencies)) - 1], 2),
        "min_s":    round(min(latencies), 2),
        "max_s":    round(max(latencies), 2),
        "avg_tok_s": round(statistics.mean(r["tok_per_s"] for r in results), 1),
        "details":  results,
    }


def run_all_benchmarks(models: list[str], num_runs: int = 5):
    print(f"\n{'='*60}")
    print(f"AI 模型性能基准测试  {time.strftime('%Y-%m-%d %H:%M')}")
    print(f"{'='*60}\n")

    all_results = []
    for model in models:
        for tier in ("simple", "medium", "complex"):
            print(f"\n── {model} [{tier}] ──")
            result = benchmark_model(model, tier, num_runs)
            all_results.append(result)

    # 汇总报告
    print(f"\n{'='*60}")
    print("汇总报告")
    print(f"{'='*60}")
    print(f"{'模型':<20} {'任务':<10} {'均值(s)':<10} {'P95(s)':<10} {'tok/s':<10}")
    print("-"*60)
    for r in all_results:
        print(f"{r['model']:<20} {r['tier']:<10} {r['avg_s']:<10} "
              f"{r['p95_s']:<10} {r['avg_tok_s']:<10}")

    return all_results


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--models", nargs="+",
                        default=["qwen2.5:7b", "qwen2.5:14b"])
    parser.add_argument("--runs", type=int, default=3)
    args = parser.parse_args()

    run_all_benchmarks(args.models, args.runs)
```

---

## 并发负载测试

```python
# benchmarks/load_test.py
import requests
import time
import concurrent.futures
import statistics
from collections import Counter

ROUTER_URL = "http://localhost:8000"

TEST_PROMPTS = [
    "翻译：Machine Learning",
    "用 Python 写斐波那契数列",
    "解释 Transformer 的注意力机制",
    "What is RAG?",
    "设计一个微服务架构",
] * 10   # 重复以有足够的并发请求


def single_request(prompt: str) -> dict:
    t0 = time.time()
    try:
        resp = requests.post(f"{ROUTER_URL}/chat",
                             json={"prompt": prompt}, timeout=120)
        elapsed = time.time() - t0
        if resp.ok:
            data = resp.json()
            return {"success": True, "elapsed": elapsed,
                    "complexity": data.get("complexity"),
                    "model": data.get("model_used")}
    except Exception as e:
        elapsed = time.time() - t0
        return {"success": False, "elapsed": elapsed, "error": str(e)}
    return {"success": False, "elapsed": time.time() - t0}


def load_test(concurrency: int = 5, total_requests: int = 20):
    prompts = TEST_PROMPTS[:total_requests]
    results = []

    print(f"\n负载测试: {concurrency} 并发, {total_requests} 请求")
    t_start = time.time()

    with concurrent.futures.ThreadPoolExecutor(max_workers=concurrency) as pool:
        futures = [pool.submit(single_request, p) for p in prompts]
        for f in concurrent.futures.as_completed(futures):
            results.append(f.result())

    total_time = time.time() - t_start
    successes = [r for r in results if r["success"]]
    failures  = [r for r in results if not r["success"]]
    latencies = [r["elapsed"] for r in successes]

    complexity_dist = Counter(r.get("complexity") for r in successes)
    model_dist      = Counter(r.get("model") for r in successes)

    print(f"\n{'='*50}")
    print(f"总请求数: {len(results)}")
    print(f"成功: {len(successes)} | 失败: {len(failures)}")
    print(f"总耗时: {total_time:.1f}s")
    print(f"吞吐量: {len(successes)/total_time:.2f} req/s")
    print(f"\n延迟分布:")
    if latencies:
        print(f"  均值: {statistics.mean(latencies):.2f}s")
        print(f"  P50:  {statistics.median(latencies):.2f}s")
        print(f"  P95:  {sorted(latencies)[int(0.95*len(latencies))-1]:.2f}s")
        print(f"  Max:  {max(latencies):.2f}s")
    print(f"\n路由分布: {dict(complexity_dist)}")
    print(f"模型分布: {dict(model_dist)}")
    print('='*50)


if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--concurrency", type=int, default=5)
    parser.add_argument("--requests",    type=int, default=20)
    args = parser.parse_args()
    load_test(args.concurrency, args.requests)
```

---

## 实时监控（Prometheus + Grafana，可选）

如需可视化监控，在路由服务中添加指标暴露：

```python
# 安装：pip install prometheus-client
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from fastapi import Response
import time

REQUEST_COUNT = Counter("aios_requests_total",
                        "Total requests", ["tier", "model"])
LATENCY_HIST  = Histogram("aios_latency_seconds",
                          "Request latency", ["tier"],
                          buckets=[0.5, 1, 2, 5, 10, 30, 60])

# 在路由处理中记录
REQUEST_COUNT.labels(tier=tier, model=target["model"]).inc()
LATENCY_HIST.labels(tier=tier).observe(latency_ms / 1000)

# 暴露指标端点
@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
```

Prometheus 采集配置（`prometheus.yml`）：

```yaml
scrape_configs:
  - job_name: aios_router
    static_configs:
      - targets: ['localhost:8000']
    scrape_interval: 15s
```

---

## 参考硬件基准数据

### WSL + NVIDIA GPU（当前架构）

| 模型 | 参数量 | 量化 | VRAM | 首 token | 生成速度 |
|------|--------|------|------|----------|---------|
| qwen2.5:3b | 3B | Q4_K_M | ~2GB | ~200ms | ~60 tok/s |
| qwen2.5:7b | 7B | Q4_K_M | ~5GB | ~400ms | ~35 tok/s |
| qwen2.5:14b | 14B | Q4_K_M | ~9GB | ~600ms | ~18 tok/s |
| qwen2.5:32b | 32B | Q4_K_M | ~20GB (不可用) | ~1.5s | ~8 tok/s |

### Mac Studio 256GB（规划中）

| 模型 | 参数量 | 量化 | 统一内存 | 首 token | 生成速度 |
|------|--------|------|---------|----------|---------|
| qwen2.5:7b | 7B | F16 | ~14GB | ~100ms | ~80 tok/s |
| qwen2.5:14b | 14B | F16 | ~28GB | ~200ms | ~45 tok/s |
| qwen2.5:32b | 32B | Q8 | ~35GB | ~400ms | ~22 tok/s |
| qwen2.5:72b | 72B | Q4 | ~45GB | ~1s | ~10 tok/s |

> Mac Studio 的统一内存架构消除了 CPU↔GPU 数据传输瓶颈，同量化下速度约为 NVIDIA 独显的 2 倍。

---

## 性能调优清单

```bash
# 1. 预加载模型（避免冷启动）
curl -X POST http://localhost:11434/api/generate \
  -d '{"model":"qwen2.5:14b","prompt":"","keep_alive":"24h"}'

# 2. 调整 Ollama 并发数
export OLLAMA_MAX_QUEUE=4
export OLLAMA_NUM_PARALLEL=2

# 3. 使用适当的量化
# Q4_K_M：速度/质量最佳平衡
# Q8_0：更高质量，VRAM 翻倍
# F16：最高质量，需要足够 VRAM

# 4. 调整 num_ctx（上下文窗口）
# 更小的 ctx 更快，按实际需求设置
ollama run qwen2.5:14b --num-ctx 4096
```

---

## 参考链接

- [模型路由器实现](./router.md)
- [Ollama 部署与调优](../00-infrastructure/ollama.md)
- [采样参数调优](../01-inference/sampling.md)
- [Transformer 推理原理](../01-inference/transformer.md)
