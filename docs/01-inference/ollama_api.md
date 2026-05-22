# Ollama API 完整参考

Ollama 提供两套 API：原生 API 和 OpenAI 兼容 API。本文档涵盖本项目实际使用的所有端点。

---

## 一、基础信息

| 实例 | 地址 | 用途 |
|------|------|------|
| Windows（核显） | `http://localhost:11435` 或 `http://<IP>/ollama-win/` | Embedding |
| WSL（独显） | `http://localhost:11434` 或 `http://<IP>/ollama-wsl/` | 推理 |

---

## 二、原生 API

### 2.1 文本生成（非流式）

```bash
POST /api/generate
```

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5-coder:14b",
  "prompt": "用 Python 写一个二分查找",
  "stream": false,
  "options": {
    "temperature": 0.1,
    "top_p": 0.9,
    "num_ctx": 8192,
    "num_predict": 2048
  }
}'
```

**响应字段**：

```json
{
  "model": "qwen2.5-coder:14b",
  "response": "def binary_search(arr, target):\n    ...",
  "done": true,
  "total_duration": 5432000000,      // 总耗时（纳秒）
  "load_duration": 12000000,         // 模型加载耗时
  "prompt_eval_count": 15,           // Prompt Token 数
  "prompt_eval_duration": 800000000, // Prompt 处理耗时
  "eval_count": 120,                 // 生成 Token 数
  "eval_duration": 4600000000        // 生成耗时
}
```

**计算推理速度**：

```python
speed = data["eval_count"] / data["eval_duration"] * 1e9
print(f"{speed:.1f} tok/s")
```

### 2.2 流式生成

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5-coder:14b",
  "prompt": "解释 Python 的 GIL",
  "stream": true
}'
# 每行返回一个 JSON，包含 "response" 字段（单个 Token）
```

```python
import requests

def stream_generate(model: str, prompt: str):
    resp = requests.post("http://localhost:11434/api/generate",
                         json={"model": model, "prompt": prompt, "stream": True},
                         stream=True)
    for line in resp.iter_lines():
        if line:
            import json
            data = json.loads(line)
            print(data["response"], end="", flush=True)
            if data.get("done"):
                break
```

### 2.3 对话（Chat）

```bash
POST /api/chat
```

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "qwen2.5-coder:14b",
  "messages": [
    {"role": "system", "content": "你是一个代码助手"},
    {"role": "user", "content": "写一个冒泡排序"},
    {"role": "assistant", "content": "def bubble_sort(arr):\n    ..."},
    {"role": "user", "content": "现在优化它的性能"}
  ],
  "stream": false
}'
```

### 2.4 Embedding

```bash
POST /api/embeddings
```

```bash
curl http://localhost:11435/api/embeddings -d '{
  "model": "nomic-embed-text",
  "prompt": "需要向量化的文本"
}'
# 响应: {"embedding": [0.123, -0.456, ...]}  // 768 个浮点数
```

### 2.5 Tool Calling（工具调用）

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "qwen2.5-coder:14b",
  "messages": [
    {"role": "user", "content": "今天北京的天气怎么样？"}
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "获取指定城市的天气",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {"type": "string", "description": "城市名称"}
          },
          "required": ["city"]
        }
      }
    }
  ],
  "stream": false
}'
```

---

## 三、OpenAI 兼容 API

Ollama 实现了 OpenAI API 的主要端点，可直接使用 OpenAI SDK：

### 3.1 Chat Completions

```bash
POST /v1/chat/completions
```

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"  # 任意字符串
)

response = client.chat.completions.create(
    model="qwen2.5-coder:14b",
    messages=[
        {"role": "system", "content": "你是一个代码助手"},
        {"role": "user", "content": "实现快速排序"}
    ],
    temperature=0.1,
    max_tokens=2048
)
print(response.choices[0].message.content)
```

### 3.2 Embeddings（OpenAI 格式）

```python
client_win = OpenAI(
    base_url="http://localhost:11435/v1",
    api_key="ollama"
)

embedding = client_win.embeddings.create(
    model="nomic-embed-text",
    input="需要向量化的文本"
)
vector = embedding.data[0].embedding
print(f"维度: {len(vector)}")   # 768
```

---

## 四、模型管理 API

### 4.1 列出已安装模型

```bash
GET /api/tags

curl http://localhost:11434/api/tags | python3 -m json.tool
```

### 4.2 查看已加载模型（显存状态）

```bash
GET /api/ps

curl http://localhost:11434/api/ps
# 返回当前加载在显存中的模型及占用信息
```

### 4.3 拉取模型

```bash
POST /api/pull

curl http://localhost:11434/api/pull -d '{"name": "qwen2.5-coder:7b"}'
# 流式返回进度信息
```

### 4.4 删除模型

```bash
DELETE /api/delete

curl -X DELETE http://localhost:11434/api/delete -d '{"name": "qwen2.5:32b"}'
```

### 4.5 模型信息

```bash
POST /api/show

curl http://localhost:11434/api/show -d '{"name": "qwen2.5-coder:14b"}'
# 返回模型详情：参数量、量化方式、上下文窗口等
```

---

## 五、通过 Nginx 访问

所有 API 通过 Nginx 统一入口访问时，路径前缀不变，只需修改 base URL：

```bash
# 直接访问（局域网）
curl http://localhost:11434/api/generate

# 通过 Nginx（局域网其他设备）
curl http://<主笔记本IP>/ollama-wsl/api/generate

# MBP 访问 Embedding（Windows 核显）
curl http://<主笔记本IP>/ollama-win/api/embeddings
```

---

## 六、常用管理命令（CLI）

```bash
ollama list                          # 列出已安装模型
ollama pull qwen2.5-coder:14b        # 拉取模型
ollama rm qwen2.5:32b               # 删除模型
ollama run qwen2.5-coder:14b        # 交互式对话
ollama show qwen2.5-coder:14b       # 查看模型信息
ollama ps                            # 查看正在运行的模型
ollama serve                         # 启动服务（调试用）
```

---

> ⬅️ [返回阶段 1 概览](./README.md)
