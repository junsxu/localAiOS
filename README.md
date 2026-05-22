# Local AI OS · 本地 AI 操作系统

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Docs Status](https://img.shields.io/badge/docs-v0.9--draft-blue)](./docs/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](./CONTRIBUTING.md)

> **这不是一个一键部署的 AI 应用。**  
> 这是一份带你从零理解并构建完整本地 AI 系统的学习路径——从 WSL 网络配置，到 RAG、Agent、记忆系统，再到多服务编排。  
> 不是"会用工具"，而是"构建工具"；不是"黑盒调用"，而是"完全掌控"。

---

## 适合谁

- 想在本地运行大模型，但不知道如何把各组件串起来的开发者
- 想理解 RAG / Agent / Memory 原理，而不只是调 API 的工程师
- 有 **Windows 11 + WSL2 + NVIDIA GPU** 环境，想充分利用的人

## 硬件要求

| 最低配置 | 推荐配置（本文档验证环境） |
|----------|--------------------------|
| Windows 11 + WSL2 | Windows 11 24H2 + WSL2 Ubuntu 22.04 |
| NVIDIA GPU 8GB+ | RTX 5070Ti 12GB |
| 内存 16GB | 内存 48GB |
| 存储 512GB SSD | PCIe5 1TB |

> 完整的验证环境版本见 [VERSIONS.md](./VERSIONS.md)

---

## 系统架构

```
                ┌────────────────────┐
                │ MBP M1             │
                │ VSCode + Continue  │
                └─────────┬──────────┘
                          │
        ┌────────────────────────────────┐
        │ 主笔记本 (WSL + Windows)        │
        ├────────────────────────────────┤
        │ Windows                        │
        │  ┌──────────────────────────┐  │
        │  │ Nginx (统一入口 :80)      │  │
        │  └──┬───────────────────────┘  │
        │  ┌──▼────────┐                 │
        │  │ Ollama    │  iGPU · :11435  │
        │  │ embedding │                 │
        │  └───────────┘                 │
        ├────────────────────────────────┤
        │ WSL Ubuntu                     │
        │  ┌──────────────────────────┐  │
        │  │ Open-WebUI  :8080        │  │
        │  │ OpenHands   :3000        │  │  自主编码 Agent
        │  │ Ollama dGPU :11434       │  │  qwen2.5-coder 7B/14B
        │  │ Qdrant      :6333        │  │
        │  └──────────────────────────┘  │
        └────────────────────────────────┘
                          │
        ┌─────────────────▼────────────┐
        │ NAS (QNAP)                   │
        │  GitLab CE :8929             │
        │  文档存储 (SMB/NFS)           │
        └──────────────────────────────┘
```

### 端口规划

| 服务 | 位置 | 端口 |
|------|------|------|
| Nginx（统一入口） | Windows | 80 / 443 |
| Ollama（iGPU · embedding） | Windows | 11435 |
| Ollama（dGPU · 推理） | WSL | 11434 |
| Open-WebUI（聊天 + RAG 界面） | WSL | 8080 |
| OpenHands（自主编码 Agent） | WSL | 3000 |
| Qdrant（向量数据库） | WSL | 6333 |
| GitLab CE | NAS | 8929 |

---

## 学习路径（8 个阶段）

| 阶段 | 模块 | 核心目标 |
|------|------|----------|
| [阶段 0](./docs/00-infrastructure/README.md) | 基础设施 · 环境部署 | WSL 网络 / Nginx / 双实例 Ollama / NAS / GitLab |
| [阶段 1](./docs/01-inference/README.md) | 推理层 · 模型理解 | Transformer 原理 / 采样参数 / Ollama API |
| [阶段 2](./docs/02-rag/README.md) | 知识检索层 · RAG | Qdrant 部署 / Embedding / 混合检索 Pipeline |
| [阶段 3](./docs/03-agent/README.md) | 执行层 · Agent | Open-WebUI / OpenHands / ReAct / Tool Calling |
| [阶段 4](./docs/04-memory/README.md) | 记忆层 · Memory | 对话摘要 / 长期记忆 / 用户画像 |
| [阶段 5](./docs/05-orchestration/README.md) | 编排层 · 系统编排 | Docker 微服务 / API 网关 |
| [阶段 6](./docs/06-performance/README.md) | 性能层 · 分层路由 | 规则 / LLM / 自适应三策略路由 |
| [阶段 7](./docs/07-aios/README.md) | 终态 · AI 操作系统 | 全系统集成 / 演进路线 |

**从这里开始 →** [阶段 0：环境部署](./docs/00-infrastructure/README.md)

---

## 快速验证（全系统运行检查表）

```bash
# 验证所有服务正常运行
curl http://localhost/health                     # Nginx 入口
curl http://localhost:11435/api/tags             # Windows Ollama
curl http://localhost:11434/api/tags             # WSL Ollama
curl http://localhost:6333/healthz               # Qdrant
curl http://localhost:3000                         # OpenHands
```

- [ ] Windows Nginx 启动，`/health` 返回 `ok`
- [ ] Windows Ollama（:11435）`ollama list` 显示 `nomic-embed-text`
- [ ] WSL Ollama（:11434）运行 `qwen2.5-coder:14b`，`nvidia-smi` 显示 GPU 占用
- [ ] Qdrant Dashboard 可访问：`http://localhost:6333/dashboard`
- [ ] Open-WebUI 可登录：`http://localhost:8080`
- [ ] OpenHands 可访问`http://localhost:3000`，Agent 能正常进行任务
- [ ] NAS 文档目录挂载到 WSL `/mnt/nas/ai-docs`
- [ ] MBP Continue 插件连接主机模型正常响应

完整集成自检脚本见 [docs/07-aios/integration.md](./docs/07-aios/integration.md)。

---

## 三个核心突破方向

```
1. RAG 做深       chunk / embedding / 检索质量 → 准确率 90%+
2. Agent 做闭环   输入 → 执行 → 反馈 → 修正
3. 模型路由       小模型 + 大模型协同调度，降本提速
```

---

## 仓库文件结构

```
localAiOS/
├── docs/                    ← 8 个阶段的详细文档
│   ├── 00-infrastructure/   ← 环境部署
│   ├── 01-inference/        ← 推理原理
│   ├── 02-rag/              ← RAG 检索
│   ├── 03-agent/            ← Agent 执行
│   ├── 04-memory/           ← 记忆系统
│   ├── 05-orchestration/    ← 系统编排
│   ├── 06-performance/      ← 性能路由
│   └── 07-aios/             ← AI OS 终态
├── VERSIONS.md              ← 验证环境版本锁定
├── CONTRIBUTING.md          ← 贡献指南
├── CHANGELOG.md             ← 版本历史
└── LICENSE                  ← MIT License
```

---

## 贡献与社区

欢迎提交踩坑记录、文档修正，详见 [CONTRIBUTING.md](./CONTRIBUTING.md)。

- **Bug 报告**：文档命令执行报错 → [提 Issue](../../issues/new?template=bug_report.md)
- **提问**：概念不理解 → [提 Issue](../../issues/new?template=question.md)
- **版本迭代**：演进路线见 [docs/07-aios/roadmap.md](./docs/07-aios/roadmap.md)

---

## License

本项目采用 [MIT License](./LICENSE)。
