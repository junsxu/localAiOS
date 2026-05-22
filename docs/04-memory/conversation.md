# 对话摘要与长期记忆写入

## 对话摘要策略

### 策略对比

| 策略 | 触发时机 | 摘要方式 | 适用场景 |
|------|----------|----------|----------|
| 对话结束摘要 | 用户输入 quit / 会话结束 | LLM 生成摘要 | 主要方案 |
| 滑动窗口摘要 | 对话超过 N 轮 | 每 N 轮摘要一次 | 长对话 |
| 实时片段摘要 | 每轮对话后 | 小模型快速摘要 | 高频使用 |
| 关键词触发 | 检测到偏好/决策关键词 | 直接提取片段 | 精准捕获 |

---

## 对话结束摘要实现

```python
# conversation_memory.py
import requests
from datetime import datetime

OLLAMA_URL  = "http://localhost:11434"
CHAT_MODEL  = "qwen2.5:14b"
ROUTER_MODEL = "qwen2.5:3b"   # 轻量模型用于摘要


def summarize_conversation(messages: list[dict]) -> str:
    """将对话提炼为 3-5 条可用于长期记忆的事实"""
    dialogue = "\n".join(
        f"{m['role'].upper()}: {m['content']}"
        for m in messages
        if m["role"] in ("user", "assistant")
    )

    prompt = f"""请将以下对话提炼为 3~5 条简洁的事实或结论，用于长期记忆存储。
格式：每条一行，以 "- " 开头。只输出事实，不要解释。

对话内容：
{dialogue}

事实摘要："""

    resp = requests.post(f"{OLLAMA_URL}/api/generate", json={
        "model": ROUTER_MODEL,   # 摘要用小模型，节省成本
        "prompt": prompt,
        "stream": False,
        "options": {"temperature": 0.1, "num_predict": 300}
    })
    return resp.json()["response"].strip()
```

---

## 滑动窗口摘要

当对话超过 20 轮时，将最早的 10 轮压缩为摘要，避免上下文窗口溢出：

```python
def sliding_window_compress(messages: list[dict],
                            window_size: int = 20,
                            keep_recent: int = 10) -> list[dict]:
    """
    当对话超过 window_size 轮时，压缩最早的部分。
    保留 system 消息 + 摘要消息 + 最近 keep_recent 条消息。
    """
    if len(messages) <= window_size:
        return messages

    system_msgs = [m for m in messages if m["role"] == "system"]
    chat_msgs   = [m for m in messages if m["role"] != "system"]

    to_compress = chat_msgs[:-keep_recent]
    recent      = chat_msgs[-keep_recent:]

    summary = summarize_conversation(to_compress)
    summary_msg = {
        "role": "system",
        "content": f"[对话历史摘要（{len(to_compress)} 轮）]\n{summary}"
    }

    return system_msgs + [summary_msg] + recent
```

---

## 关键词触发实时写入

```python
# 不等对话结束，遇到关键触发词立即保存
TRIGGER_PATTERNS = {
    "preference": ["我喜欢", "我偏好", "我习惯", "我不喜欢", "以后都用", "我想要"],
    "decision":   ["我们决定", "最终选择", "确定用", "不用了", "改成", "换成"],
    "fact":       ["我是", "我在", "我的项目", "我负责", "我的团队"],
}

def check_trigger(user_input: str) -> tuple[bool, str]:
    """检查是否触发即时记忆写入，返回 (是否触发, 记忆类型)"""
    for memory_type, patterns in TRIGGER_PATTERNS.items():
        if any(p in user_input for p in patterns):
            return True, memory_type
    return False, ""


def extract_memory_snippet(user_input: str, memory_type: str) -> str:
    """从用户输入中提取最关键的记忆片段"""
    prompt = f"""从以下用户输入中提取一条简洁的 {memory_type} 记忆（一句话）：
输入：{user_input}
记忆："""
    resp = requests.post(f"{OLLAMA_URL}/api/generate", json={
        "model": ROUTER_MODEL,
        "prompt": prompt,
        "stream": False,
        "options": {"temperature": 0, "num_predict": 100}
    })
    return resp.json()["response"].strip()
```

---

## 完整对话流程（含 Memory）

```python
from memory_service import memory

def chat_with_memory(user_id: str = "default"):
    messages = []
    session_id = f"sess_{datetime.now().strftime('%Y%m%d_%H%M%S')}"
    print("带记忆的对话（输入 'quit' 退出并保存记忆）\n")

    while True:
        user_input = input("你：").strip()
        if not user_input:
            continue
        if user_input.lower() == "quit":
            break

        # ① 检查是否触发即时记忆
        triggered, mem_type = check_trigger(user_input)
        if triggered:
            snippet = extract_memory_snippet(user_input, mem_type)
            memory.save(user_id, snippet, memory_type=mem_type,
                        session_id=session_id, importance=0.85)
            print(f"  💾 即时记忆已保存: {snippet}")

        # ② 构建包含记忆的 System Prompt
        context = memory.build_context(user_id, user_input)
        if not messages:
            messages = [{"role": "system",
                         "content": f"你是用户的个人 AI 助手。\n\n{context}"}]

        # ③ 滑动窗口防止上下文溢出
        messages = sliding_window_compress(messages, window_size=20, keep_recent=10)

        # ④ 添加用户消息并请求 LLM
        messages.append({"role": "user", "content": user_input})
        resp = requests.post(f"{OLLAMA_URL}/api/chat", json={
            "model": CHAT_MODEL,
            "messages": messages,
            "stream": False
        })
        reply = resp.json()["message"]["content"]
        messages.append({"role": "assistant", "content": reply})
        print(f"AI：{reply}\n")

    # ⑤ 对话结束，保存整体摘要
    chat_msgs = [m for m in messages if m["role"] != "system"]
    if len(chat_msgs) >= 4:   # 至少 2 轮对话才值得保存
        summary = summarize_conversation(chat_msgs)
        if len(summary.strip()) > 20:
            memory.save(user_id, summary,
                        memory_type="conversation_summary",
                        session_id=session_id, importance=0.7)
            print(f"\n✅ 对话摘要已保存：\n{summary}")


if __name__ == "__main__":
    chat_with_memory()
```

---

## 摘要质量验证

```python
def evaluate_summary_quality(original_messages: list[dict], summary: str) -> dict:
    """评估摘要是否保留了关键信息"""
    original_text = " ".join(m["content"] for m in original_messages)

    # 1. 长度比（太短说明丢失信息，太长说明未压缩）
    ratio = len(summary) / max(len(original_text), 1)

    # 2. 用 LLM 评分
    prompt = f"""评估以下摘要的质量（0~10 分）：
原文长度：{len(original_text)} 字
摘要：{summary}

评分标准：保留关键决策/偏好/事实 +3，丢失重要信息 -3，简洁 +2，冗余 -2
只输出数字："""
    resp = requests.post(f"{OLLAMA_URL}/api/generate", json={
        "model": ROUTER_MODEL, "prompt": prompt,
        "stream": False, "options": {"temperature": 0}
    })
    try:
        score = float(resp.json()["response"].strip().split()[0])
    except (ValueError, IndexError):
        score = 5.0

    return {"compression_ratio": round(ratio, 3), "quality_score": score}
```

---

## 记忆调试工具

```python
# 查看某用户所有记忆
def dump_memories(user_id: str, limit: int = 20):
    mems = memory.get_recent(user_id, limit=limit)
    print(f"\n=== {user_id} 的记忆（最近 {limit} 条）===")
    for i, m in enumerate(mems, 1):
        print(f"{i:02d}. [{m['type']}] {m['ts'][:19]}: {m['content'][:100]}")

# 测试记忆检索
def test_recall(user_id: str, query: str):
    results = memory.recall(user_id, query, k=5)
    print(f"\n=== 查询: {query} ===")
    for r in results:
        print(f"  [{r['score']:.3f}] {r['content'][:100]}")

# CLI 调试
if __name__ == "__main__":
    import sys
    cmd = sys.argv[1] if len(sys.argv) > 1 else "dump"
    uid = sys.argv[2] if len(sys.argv) > 2 else "default"
    if cmd == "dump":
        dump_memories(uid)
    elif cmd == "test":
        test_recall(uid, " ".join(sys.argv[3:]))
```

---

## 参考链接

- [Memory 架构与存储设计](./memory.md)
- [用户画像建模](./user_profile.md)
- [RAG 管道实现](../02-rag/rag_pipeline.md)
