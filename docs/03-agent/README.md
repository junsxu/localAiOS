# 阶段 3：执行层 · Agent

> **核心目标**：从"问答" → "执行任务"
>
> Agent 不只是回答问题，而是能自主规划、调用工具、修正错误，完成复杂任务。

---

## 知识体系

| 主题 | 详细文档 |
|------|---------|
| Open-WebUI 部署与配置（聊天 + RAG 界面） | [open_webui.md](./open_webui.md) |
| OpenHands 自主编码 Agent 本地部署 | [openhands.md](./openhands.md) |
| ReAct 模式与原生 Tool Calling 实现 | [tool_calling.md](./tool_calling.md) |
| RAG + Agent 联动 | [tool_calling.md](./tool_calling.md) |

---

## 核心思维模型：ReAct

```
用户任务
   │
   ▼
┌─────────────────────────────────────────────────────────┐
│  LOOP: 思考(Thought) → 行动(Action) → 观察(Observation)  │
│                                                         │
│  Thought: "我需要先查看文件结构"                          │
│  Action:  list_files("./src")                           │
│  Observation: ["main.py", "utils.py", ...]              │
│                                                         │
│  Thought: "接下来读取 main.py"                           │
│  Action:  read_file("main.py")                          │
│  Observation: "# Main entry point..."                   │
│                                                         │
│  Thought: "已获取足够信息，可以汇总"                      │
│  Final Answer: "这个项目是..."                           │
└─────────────────────────────────────────────────────────┘
```

---

## 三种实现路径

| 路径 | 工具 | 适用场景 |
|------|------|---------|
| **Open-WebUI** | Docker + Web UI | 聊天 + RAG，立即上手，图形界面 |
| **OpenHands** | Docker + 自主 Agent | 让 AI 自动写代码 / 调试 / 提交，任务驱动 |
| **原生 Python** | Ollama Tool Calling API | 深度理解底层，自定义工具，可扩展 |

> 推荐学习顺序：Open-WebUI 体验对话 Agent → 原生 Python 理解 ReAct 原理 → OpenHands 实践自主任务执行。

---

## 实操任务

### 任务 1：部署 Open-WebUI

通过 Docker Compose 在 WSL 中启动 Open-WebUI，配置 Ollama 作为推理后端、`nomic-embed-text` 作为 RAG Embedding 引擎，创建账户后可直接在 Web 界面对话并上传文档做 RAG。

> 完整 docker-compose 配置、RAG 集成、Tools 启用、systemd 服务见 [open_webui.md](./open_webui.md)。

- [ ] Open-WebUI 可访问，模型可对话

---

### 任务 2：OpenHands 自主 Agent（进阶）

通过 Docker Compose 启动 OpenHands，配置本地 Ollama 作为 LLM 后端，输入一个完整的编程任务（如"创建一个 FastAPI 健康检查服务并附上测试"），观察 Agent 自动规划、写代码、运行测试、修复错误的完整流程。

> 完整 docker-compose 配置、Open-WebUI vs OpenHands 选择指南见 [openhands.md](./openhands.md)。

- [ ] OpenHands 可访问，能完成一个完整编程任务

---

### 任务 3：原生 Tool Calling 实现

用 Python 直接调用 Ollama `/api/chat` 接口，手动定义工具（`read_file`、`list_files`），实现一个最小 ReAct 循环：模型决策调用哪个工具 → 执行工具 → 将结果追加到消息历史 → 继续推理，直到得出最终答案。

理解 Tool Calling 在底层如何工作，是使用更高层框架（LangChain、LlamaIndex）的基础。

> 完整工具库、ReAct 模式、失败重试策略见 [tool_calling.md](./tool_calling.md)。

- [ ] Agent 能自动选择工具，完成"分析目录结构"任务

---

### 任务 4：RAG + Agent 联动

在 Agent 的工具列表中添加 `search_knowledge` 工具，调用 RAG Pipeline 查询 Qdrant。Agent 在需要专业知识时会自动决策查询知识库，而非依赖训练记忆。

> 完整联动实现见 [tool_calling.md](./tool_calling.md)。

- [ ] Agent 能在回答问题时自动调用知识库搜索

---

## 验收标准

- [ ] ✅ 输入"分析这个目录结构" → Agent 自动 list/read/汇总
- [ ] ✅ 工具调用失败时 Agent 能感知并换策略
- [ ] ✅ RAG + Agent 联动：Agent 能调用知识库搜索
- [ ] ✅ 多轮对话上下文不丢失

---

> ⬅️ [上一阶段：RAG](../02-rag/README.md) | ➡️ [下一阶段：Memory](../04-memory/README.md)
