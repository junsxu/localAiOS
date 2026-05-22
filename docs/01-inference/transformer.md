# Transformer 推理原理

理解 Transformer 的推理机制，是优化本地 AI 系统性能的基础。本文档聚焦工程实践视角，而非数学推导。

---

## 一、推理流程全图

```
用户输入文本
     │
     ▼
┌─────────────────────────────────────┐
│  Tokenization（分词）                │
│  "你好世界" → [15880, 1533, 1533]   │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  Embedding 层                        │
│  每个 Token ID → 高维向量（如 4096 维）│
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  N × Transformer Block               │
│  ┌─────────────────────────────┐    │
│  │ Self-Attention               │    │
│  │  每个 Token 关注其他所有 Token│    │
│  │  Q × K → Attention Score    │    │
│  │  Score × V → 输出            │    │
│  └──────────────┬──────────────┘    │
│  ┌──────────────▼──────────────┐    │
│  │ Feed-Forward Network         │    │
│  │  进一步变换特征表示            │    │
│  └─────────────────────────────┘    │
└──────────────────┬──────────────────┘
                   │ （重复 N 层，7B ≈ 32层，14B ≈ 48层）
                   ▼
┌─────────────────────────────────────┐
│  输出层（LM Head）                    │
│  最后一个 Token 的隐状态 → Logits    │
│  Logits → Softmax → 概率分布        │
│  采样策略 → 下一个 Token             │
└─────────────────────────────────────┘
```

**关键洞察**：模型每次只预测**下一个 Token**，然后将新 Token 加入输入，循环直到生成结束符或达到长度限制。这就是为什么流式输出（streaming）是逐词出现的。

---

## 二、KV Cache：推理加速的核心

### 问题背景

Self-Attention 中，每个 Token 都需要与**所有历史 Token** 计算 Key（K）和 Value（V）。如果不缓存，生成第 100 个 Token 时需要重新计算前 99 个 Token 的 K/V，极其低效。

### KV Cache 机制

```
生成第 1 个 Token：
  计算 token_1 的 K1, V1
  → 保存到 KV Cache

生成第 2 个 Token：
  从 Cache 取 K1, V1（不重新计算！）
  计算新的 K2, V2
  → 追加到 KV Cache

生成第 N 个 Token：
  从 Cache 取 K1...K(N-1), V1...V(N-1)
  只计算新的 KN, VN
  → 时间复杂度从 O(N²) → O(N)
```

### 对实践的影响

| 影响点 | 说明 |
|--------|------|
| **显存占用** | KV Cache 随上下文长度线性增长。14B 模型，4096 Token 上下文，约占 2~4GB 显存 |
| **冷启动** | 第一次请求（模型未加载/KV Cache 空）速度慢，后续快 |
| **并发限制** | 多个并发请求需要各自的 KV Cache，显存压力倍增 |
| **`num_ctx` 参数** | 设置上下文窗口大小，越大越消耗显存 |

```bash
# 查看当前显存使用（包括 KV Cache）
nvidia-smi --query-gpu=memory.used,memory.free --format=csv

# 控制上下文窗口大小（减少显存压力）
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5-coder:14b",
  "prompt": "...",
  "options": {"num_ctx": 4096}
}'
```

---

## 三、Tokenization：理解"贵"在哪里

### 什么是 Token

Token 是模型处理的最小语言单元，不等于字或词。不同语言的 Token 密度差异很大：

```
英文：每个单词 ≈ 1~1.5 个 Token
"Hello World" → 2 tokens

中文：每个字 ≈ 1.5~2 个 Token
"你好，世界" → 5~7 tokens

代码：
"def hello():" → 5 tokens（关键词通常是单个 Token）
```

### 为什么重要（工程角度）

**1. 影响上下文窗口利用率**

qwen2.5-coder:14b 的 `num_ctx` 默认 4096 Token：
- 存 4096 个英文单词 ≈ 3000+ 词的文章
- 存 4096 个中文字 ≈ 2000 个汉字

**2. 影响 RAG 的 Chunk 大小设计**（见 [02-rag/rag_pipeline.md](../02-rag/rag_pipeline.md)）

```python
# 错误：按字符数切块，不考虑 Token 数
chunk_size = 1000  # 字符

# 更精确：按 Token 数控制（需要 tokenizer）
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-Coder-14B")
tokens = tokenizer.encode(text)
print(f"Token 数: {len(tokens)}")
```

**3. 快速估算**（工程经验值）

| 语言/类型 | 字符:Token 比例 |
|---------|---------------|
| 英文纯文本 | 4:1（4个字符≈1个Token） |
| 中文纯文本 | 2:1（2个汉字≈1个Token） |
| 代码（Python） | 3:1 |
| Markdown | 3.5:1 |

---

## 四、模型规模对比

| 规格 | 7B | 14B | 32B |
|------|-----|-----|-----|
| 参数量 | 70亿 | 140亿 | 320亿 |
| Q4_K_M 大小 | ~4.7GB | ~8.9GB | ~19GB |
| RTX5070Ti 12GB | ✅ 舒适 | ✅ 可运行 | ❌ 不够 |
| 推理速度（本机） | ~30 tok/s | ~15 tok/s | — |
| 代码质量 | 良 | 优 | 极优 |
| 适用场景 | 补全、问答 | 主力代码生成 | 需更大显存 |

**量化版本说明**：

| 量化 | 精度损失 | 大小（14B） | 推荐 |
|------|---------|-----------|------|
| FP16 | 无 | ~28GB | 需大显存 |
| Q8_0 | 极小 | ~14.7GB | 高端显卡 |
| Q5_K_M | 小 | ~11GB | 高精度需求 |
| **Q4_K_M** | 中等 | **~8.9GB** | **推荐** |
| Q3_K_M | 较大 | ~7GB | 显存紧张时 |

---

## 五、推理性能监控

```bash
# 实时 GPU 状态
watch -n 1 nvidia-smi

# 查看 Ollama 已加载模型及显存占用
curl http://localhost:11434/api/ps | python3 -m json.tool

# 查看推理速度（从 Ollama API 响应中读取）
curl -X POST http://localhost:11434/api/generate \
  -d '{"model": "qwen2.5-coder:14b", "prompt": "hello", "stream": false}' \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'速度: {d[\"eval_count\"] / d[\"eval_duration\"] * 1e9:.1f} tok/s')
print(f'总 Token: {d[\"eval_count\"]}')
"
```

---

## 六、冷启动 vs 热推理

```
首次请求（冷启动）：
  加载模型权重到 GPU 显存    →  ~10~30 秒
  首次 Prefill（计算 KV）    →  ~1~5 秒
  逐 Token 生成               →  ~15 tok/s

后续请求（模型已在显存）：
  跳过加载                    →  0 秒
  Prefill（计算新上下文）      →  ~0.5~2 秒
  逐 Token 生成               →  ~15 tok/s
```

**保持模型在显存中**（Ollama `keep_alive`）：

```bash
# 永久保持（直到手动卸载）
curl -X POST http://localhost:11434/api/generate \
  -d '{"model": "qwen2.5-coder:14b", "prompt": "", "keep_alive": -1}'

# 查看当前加载的模型
curl http://localhost:11434/api/ps

# 手动卸载（释放显存）
curl -X DELETE http://localhost:11434/api/models/qwen2.5-coder:14b
```

---

> ⬅️ [返回阶段 1 概览](./README.md)
