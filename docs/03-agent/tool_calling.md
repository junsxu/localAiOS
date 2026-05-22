# 原生 Tool Calling 与 ReAct 实现

理解 Agent 的底层实现，是构建自定义 Agent 系统的基础。本文档从 Ollama Tool Calling API 出发，逐步实现完整的 ReAct Agent。

---

## 一、Ollama Tool Calling 机制

### 1.1 工作原理

```
用户输入
    │
    ▼
发送给 LLM（附带工具定义）
    │
    ▼
LLM 决策：
  ┌─────────────────────────────┐
  │ A. 直接回答（无需工具）       │  → 返回文本给用户
  │ B. 调用工具（需要外部信息）   │  → 返回 tool_calls
  └─────────────────────────────┘
    │（如果选 B）
    ▼
应用程序执行对应工具函数
    │
    ▼
将工具结果作为 "role: tool" 消息发回给 LLM
    │
    ▼
LLM 基于工具结果生成最终回答
```

### 1.2 最小示例

```python
import json, requests

resp = requests.post("http://localhost:11434/api/chat", json={
    "model": "qwen2.5-coder:14b",
    "messages": [{"role": "user", "content": "今天几号？"}],
    "tools": [{
        "type": "function",
        "function": {
            "name": "get_date",
            "description": "获取今天的日期",
            "parameters": {"type": "object", "properties": {}}
        }
    }],
    "stream": False
})

msg = resp.json()["message"]
if msg.get("tool_calls"):
    print("模型想调用工具:", msg["tool_calls"])
else:
    print("模型直接回答:", msg["content"])
```

---

## 二、工具库设计

```python
import os
import json
import subprocess
import requests
from pathlib import Path
from typing import Any

# ── 安全白名单 ─────────────────────────────────────────────────
ALLOWED_SHELL_COMMANDS = {"ls", "cat", "grep", "find", "git", "python3", "pip"}
ALLOWED_READ_PATHS = {"/home", "/tmp", "/mnt"}  # 禁止读取系统文件

def _is_safe_path(path: str) -> bool:
    resolved = Path(path).resolve()
    return any(str(resolved).startswith(p) for p in ALLOWED_READ_PATHS)

# ── 工具定义（发送给 LLM）────────────────────────────────────────
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "读取文件内容。适用于读取代码、文档、配置文件。",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "文件的绝对或相对路径"},
                    "max_chars": {"type": "integer", "description": "最多读取字符数", "default": 3000}
                },
                "required": ["path"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_files",
            "description": "列出目录中的文件和子目录。",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "目录路径", "default": "."},
                    "recursive": {"type": "boolean", "description": "是否递归列出", "default": False}
                },
                "required": []
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "run_shell",
            "description": "执行 shell 命令并返回输出。仅支持白名单命令：ls, cat, grep, find, git, python3, pip。",
            "parameters": {
                "type": "object",
                "properties": {
                    "command": {"type": "string", "description": "要执行的命令"}
                },
                "required": ["command"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "http_get",
            "description": "发送 HTTP GET 请求并返回响应内容。",
            "parameters": {
                "type": "object",
                "properties": {
                    "url": {"type": "string"},
                    "headers": {"type": "object", "default": {}}
                },
                "required": ["url"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_knowledge",
            "description": "在本地 RAG 知识库中搜索相关信息。当需要查询文档、技术资料时使用。",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "搜索关键词或问题"}
                },
                "required": ["query"]
            }
        }
    }
]

# ── 工具执行 ─────────────────────────────────────────────────────
def execute_tool(name: str, args: dict) -> str:
    try:
        if name == "read_file":
            path = args["path"]
            if not _is_safe_path(path):
                return f"⛔ 拒绝：路径 '{path}' 不在允许范围内"
            max_chars = args.get("max_chars", 3000)
            content = Path(path).read_text(encoding="utf-8", errors="replace")
            return content[:max_chars] + (f"\n...[截断，共 {len(content)} 字符]" if len(content) > max_chars else "")

        elif name == "list_files":
            path = args.get("path", ".")
            recursive = args.get("recursive", False)
            if recursive:
                files = [str(p) for p in Path(path).rglob("*") if p.is_file()]
            else:
                files = os.listdir(path)
            return json.dumps(sorted(files), ensure_ascii=False)

        elif name == "run_shell":
            cmd = args["command"].strip()
            base_cmd = cmd.split()[0]
            if base_cmd not in ALLOWED_SHELL_COMMANDS:
                return f"⛔ 拒绝：命令 '{base_cmd}' 不在白名单中"
            result = subprocess.run(
                cmd, shell=True, capture_output=True,
                text=True, timeout=30, cwd="/home"
            )
            output = result.stdout or result.stderr
            return output[:2000] + ("...[截断]" if len(output) > 2000 else "")

        elif name == "http_get":
            resp = requests.get(args["url"], headers=args.get("headers", {}), timeout=10)
            return resp.text[:2000]

        elif name == "search_knowledge":
            # 调用 RAG Pipeline（见 02-rag/rag_pipeline.md）
            from rag_query import retrieve
            results = retrieve(args["query"], top_k=3)
            if not results:
                return "知识库中未找到相关内容。"
            return "\n\n".join(
                f"[{r['source']}（相关度:{r['score']}）]\n{r['text']}"
                for r in results
            )

        else:
            return f"❓ 未知工具: {name}"

    except Exception as e:
        return f"⚠️ 工具执行错误: {type(e).__name__}: {e}"
```

---

## 三、完整 ReAct Agent

```python
import json
import requests

OLLAMA_URL = "http://localhost:11434"
MODEL = "qwen2.5-coder:14b"

SYSTEM_PROMPT = """你是一个任务执行助手，有以下工具可以使用：
- read_file：读取文件内容
- list_files：列出目录文件
- run_shell：执行 shell 命令（白名单内）
- http_get：发送 HTTP 请求
- search_knowledge：搜索本地知识库

规则：
1. 每次只调用一个工具
2. 等待工具结果后再决定下一步
3. 完成任务后给出完整的最终答案
4. 如果工具返回错误，尝试其他方式或说明无法完成"""


def run_agent(task: str, max_steps: int = 10, verbose: bool = True) -> str:
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": task}
    ]

    for step in range(max_steps):
        if verbose:
            print(f"\n{'─' * 40}")
            print(f"Step {step + 1}/{max_steps}")

        resp = requests.post(f"{OLLAMA_URL}/api/chat", json={
            "model": MODEL,
            "messages": messages,
            "tools": TOOLS,
            "stream": False
        })
        msg = resp.json()["message"]
        messages.append(msg)

        if msg.get("tool_calls"):
            for call in msg["tool_calls"]:
                tool_name = call["function"]["name"]
                tool_args = call["function"]["arguments"]
                if isinstance(tool_args, str):
                    tool_args = json.loads(tool_args)

                if verbose:
                    print(f"🔧 调用: {tool_name}({json.dumps(tool_args, ensure_ascii=False)[:100]})")

                result = execute_tool(tool_name, tool_args)

                if verbose:
                    print(f"📋 结果: {str(result)[:200]}{'...' if len(str(result)) > 200 else ''}")

                messages.append({"role": "tool", "content": str(result)})
        else:
            # 无工具调用 = 任务完成
            answer = msg.get("content", "")
            if verbose:
                print(f"\n✅ 最终答案:\n{answer}")
            return answer

    return f"⚠️ 已达到最大步数 {max_steps}，任务可能未完成"


# ── 带重试的版本 ─────────────────────────────────────────────────
def run_agent_with_retry(task: str, max_retries: int = 3) -> str:
    last_error = None
    for attempt in range(max_retries):
        try:
            return run_agent(task)
        except Exception as e:
            last_error = e
            print(f"⚠️ 第 {attempt + 1} 次失败: {e}，重试中...")
    return f"任务失败（{max_retries} 次尝试后）: {last_error}"


# ── 使用示例 ─────────────────────────────────────────────────────
if __name__ == "__main__":
    examples = [
        "列出当前目录的文件，并告诉我有哪些 Python 文件",
        "搜索知识库中关于 Nginx 配置的内容，并给出配置要点",
        "检查 http://localhost:11434/api/tags 接口，告诉我有哪些可用模型",
    ]
    task = input("输入任务（直接回车使用示例）：").strip() or examples[0]
    run_agent(task)
```

---

## 四、多工具链示例

### 4.1 分析 Git 仓库

```python
result = run_agent("""
分析 /home/user/project 这个项目：
1. 列出主要目录结构
2. 读取 README.md
3. 如果有 requirements.txt 或 pyproject.toml，读取依赖列表
4. 总结这个项目的用途和技术栈
""")
```

### 4.2 RAG + API 联动

```python
result = run_agent("""
我想了解如何优化 RAG 检索质量：
1. 先搜索本地知识库中关于 RAG 的内容
2. 再检查 Qdrant API 当前有哪些集合
3. 综合以上信息给出具体建议
""")
```

### 4.3 代码分析

```python
result = run_agent("""
读取 ~/rag_indexer.py 文件，分析：
1. 代码的主要功能
2. 有没有可以优化的地方
3. 给出具体的改进建议
""")
```

---

## 五、Agent 调试技巧

### 5.1 打印完整消息历史

```python
import json

def run_agent_debug(task: str):
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": task}
    ]
    # ...（运行 Agent）

    # 结束后打印完整对话
    for msg in messages:
        role = msg["role"]
        content = msg.get("content", "")
        tool_calls = msg.get("tool_calls", [])
        print(f"\n[{role.upper()}]")
        if content:
            print(content[:500])
        if tool_calls:
            print(f"工具调用: {json.dumps(tool_calls, ensure_ascii=False, indent=2)}")
```

### 5.2 常见问题

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| 模型不使用工具 | 模型理解问题描述的工具需求不明确 | 在 System Prompt 中明确何时使用工具 |
| 工具调用参数格式错误 | 模型对参数的理解偏差 | 在工具描述中加入示例 |
| 无限循环调用同一工具 | 工具返回错误但模型继续尝试 | 添加最大步数限制和失败退出逻辑 |
| 模型忽略工具结果 | 上下文过长被截断 | 减少工具返回内容，只保留关键信息 |

---

> ⬅️ [返回阶段 3 概览](./README.md)
