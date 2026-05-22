# 采样参数详解

采样参数决定了模型如何从概率分布中选择下一个 Token，直接影响输出的确定性、创意性和质量。

---

## 一、参数总览

| 参数 | Ollama 默认值 | 作用范围 | 说明 |
|------|-------------|---------|------|
| `temperature` | 0.8 | 输出随机性 | 核心参数，最常调整 |
| `top_p` | 0.9 | 候选 Token 集合 | 配合 temperature |
| `top_k` | 40 | 候选 Token 数量 | 通常不需要调整 |
| `repeat_penalty` | 1.1 | 抑制重复 | 长文本生成时有效 |
| `num_predict` | -1（无限） | 输出长度 | 限制最大 Token 数 |
| `num_ctx` | 模型决定 | 上下文窗口 | 影响显存用量 |
| `seed` | 随机 | 随机数种子 | 设置后可复现输出 |

---

## 二、temperature：最重要的参数

### 原理

```
原始 Logits:  [3.2, 1.1, 2.8, 0.5]  （模型对每个候选词的原始评分）
                          ÷ temperature
temperature=0.1: [32, 11, 28, 5]   → Softmax → 最高分几乎独占概率 → 确定性
temperature=1.0: [3.2, 1.1, 2.8, 0.5] → 维持原始分布
temperature=2.0: [1.6, 0.55, 1.4, 0.25] → 概率更均匀 → 随机性高
```

### 实践规律

```
temperature = 0     → 完全确定性（每次输出相同）⚠️ 可能陷入重复循环
temperature = 0.1   → 高度确定，适合代码生成
temperature = 0.3   → 略有变化，适合技术文档
temperature = 0.7   → 均衡，适合通用对话
temperature = 1.0   → 较高创意，适合写作、头脑风暴
temperature = 1.5+  → 高随机性，可能出现不连贯输出
```

### 实验验证

```bash
# 同一 prompt，不同 temperature，运行多次观察稳定性
for temp in 0.1 0.5 1.0 1.5; do
  echo "=== temperature=$temp ==="
  curl -s http://localhost:11434/api/generate -d "{
    \"model\": \"qwen2.5:7b\",
    \"prompt\": \"写一首四行小诗\",
    \"options\": {\"temperature\": $temp, \"seed\": 42},
    \"stream\": false
  }" | python3 -c "import sys,json; print(json.load(sys.stdin)['response'])"
done
```

---

## 三、top_p（核采样）

### 原理

将所有候选 Token 按概率从高到低排序，取累积概率达到 `top_p` 的最小集合，只从这个集合中采样。

```
候选 Token 概率（排序后）:
  Token A: 40%
  Token B: 25%  ← top_p=0.7 时，取到这里（累积 65%，再加下一个超过 70%）
  Token C: 20%
  Token D: 10%
  Token E: 5%

top_p=0.7: 只从 {A, B} 中采样
top_p=0.9: 从 {A, B, C, D} 中采样
top_p=1.0: 从所有候选中采样
```

### 与 temperature 的关系

- 两者同时生效
- **建议只调一个**：调 temperature 时保持 top_p=0.9，或固定 temperature=1.0 只调 top_p
- 同时调小两个会导致输出过于保守

```bash
# 推荐组合
{"temperature": 0.1, "top_p": 0.9}   # 代码生成
{"temperature": 0.7, "top_p": 0.9}   # 通用对话
{"temperature": 1.0, "top_p": 0.8}   # 创意写作（收紧 top_p 防止乱飞）
```

---

## 四、repeat_penalty（重复惩罚）

### 作用

对已经出现过的 Token 降低其被再次选择的概率，防止模型陷入"..., ..., ..."循环。

```
repeat_penalty = 1.0  → 不惩罚（默认行为）
repeat_penalty = 1.1  → 轻微惩罚，适合代码（关键词会重复）
repeat_penalty = 1.3  → 较强惩罚，适合长文本叙述
repeat_penalty = 1.5+ → 强惩罚，可能导致输出不连贯
```

### 注意事项

- 代码生成中，变量名、关键字本身就会重复，惩罚过强会影响质量
- 推荐代码场景：`1.05~1.1`
- 推荐文章场景：`1.1~1.2`

---

## 五、seed：可复现的输出

设置相同 seed 后，在相同 temperature 下每次输出完全一致，用于调试和对比：

```bash
# 设置 seed=42，输出可复现
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:7b",
  "prompt": "用一句话描述 Python",
  "options": {"temperature": 0.7, "seed": 42},
  "stream": false
}'
# 多次运行，输出完全相同
```

---

## 六、场景配置推荐

### 代码补全（Tab Autocomplete）

```json
{
  "temperature": 0.1,
  "top_p": 0.9,
  "repeat_penalty": 1.05,
  "num_predict": 256,
  "num_ctx": 4096
}
```

### 代码生成（Chat）

```json
{
  "temperature": 0.2,
  "top_p": 0.9,
  "repeat_penalty": 1.1,
  "num_predict": 2048,
  "num_ctx": 8192
}
```

### 通用对话

```json
{
  "temperature": 0.7,
  "top_p": 0.9,
  "repeat_penalty": 1.1,
  "num_predict": 1024
}
```

### 创意写作 / 头脑风暴

```json
{
  "temperature": 1.0,
  "top_p": 0.85,
  "repeat_penalty": 1.15,
  "num_predict": 2048
}
```

---

## 七、在 Ollama 中设置参数

### API 请求中设置

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5-coder:14b",
  "prompt": "用 Python 实现二分查找",
  "options": {
    "temperature": 0.1,
    "top_p": 0.9,
    "repeat_penalty": 1.05,
    "num_predict": 1024,
    "num_ctx": 8192
  },
  "stream": false
}'
```

### Modelfile 中永久设置

```bash
cat > /tmp/Modelfile << 'EOF'
FROM qwen2.5-coder:14b

# 代码场景优化参数
PARAMETER temperature 0.1
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.05
PARAMETER num_ctx 8192
PARAMETER num_predict 4096

# 系统提示词
SYSTEM """你是一个代码助手，专注于 Python 和系统编程。
回答简洁、准确，代码可直接运行。"""
EOF

ollama create qwen2.5-coder:14b-code -f /tmp/Modelfile

# 使用自定义模型
ollama run qwen2.5-coder:14b-code "实现一个 LRU 缓存"
```

---

## 八、参数调试工作流

```
1. 从 temperature=0.7, top_p=0.9 开始（通用默认值）
2. 如果输出太随机/不可控 → 降低 temperature（0.3~0.5）
3. 如果输出太保守/重复 → 提高 temperature（0.9~1.1）
4. 如果出现重复循环 → 提高 repeat_penalty（1.15~1.2）
5. 如果输出过长/截断 → 增加 num_predict
6. 确定参数后 → 设置 seed 固定，验证可复现性
7. 满意后写入 Modelfile 持久化
```

---

> ⬅️ [返回阶段 1 概览](./README.md)
