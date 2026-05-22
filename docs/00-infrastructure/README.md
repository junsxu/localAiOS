# 阶段 0：基础设施 · 环境部署

> **核心目标**：不是"能跑"，而是"完全掌控"
>
> 在进入任何 AI 能力层之前，确保本地环境是可预期、可调试、可切换的。

---

## 知识体系

| 知识点 | 详细文档 |
|--------|---------|
| WSL 网络模型（Mirrored / NAT） | [wsl_network.md](./wsl_network.md) |
| Nginx 反向代理 & 服务注册 | [nginx.md](./nginx.md) |
| Ollama 双实例（核显 + 独显） | [ollama.md](./ollama.md) |
| Continue 插件（MacBook 客户端） | [continue.md](./continue.md) |
| NAS 文档存储 & NFS 挂载 | [nas_storage.md](./nas_storage.md) |
| GitLab CE 私有仓库 | [gitlab.md](./gitlab.md) |

---

## 目标架构

```
                ┌────────────────────┐
                │ MBP M1             │
                │ VSCode + Continue  │
                └─────────┬──────────┘
                          │ 局域网
        ┌─────────────────┼────────────────────────┐
        │ 主笔记本                                  │
        │  ┌────────────────────────────────────┐  │
        │  │   Nginx (统一入口 :80)              │  │
        │  └──┬─────────┬───────────────────────┘  │
        │     │         │                          │
        │  ┌──▼──────┐  │ Windows                  │
        │  │ Ollama  │  │ 核显·nomic embed         │
        │  └─────────┘  │                          │
        │  ─────────────┼─── WSL ───────────────── │
        │  ┌────────────▼──────────────────────┐   │
        │  │ Ollama  :11434  (独显推理)         │   │
        │  │ Qdrant  :6333  (向量库)            │   │
        │  │ Open-WebUI  :8080 （聊天+RAG 界面）│   │
        │  │ OpenHands   :3000 （自主 Agent）   │   │
        │  └───────────────────────────────────┘   │
        └──────────────────┬───────────────────────┘
                           │
        ┌──────────────────▼────────────────┐
        │ NAS (QNAP)                        │
        │  GitLab :8929  |  ai-docs (NFS)   │
        └───────────────────────────────────┘
```

---

## 端口规划

| 服务 | 位置 | 端口 |
|------|------|------|
| Nginx（统一入口） | Windows | 80 |
| Ollama（核显·embedding） | Windows | 11435 |
| Ollama（独显·大模型） | WSL | 11434 |
| Qdrant（向量数据库） | WSL | 6333 |
| Open-WebUI（聊天 + RAG 界面） | WSL | 8080 |
| OpenHands（自主编码 Agent） | WSL | 3000 |
| GitLab CE | NAS | 8929 / SSH 2224 |

---

## 关键设计决策

**WSL 网络：Mirrored 模式优先**
WSL 与 Windows 共享网络命名空间，所有服务通过 `127.0.0.1` 互通，避免 NAT 带来的端口转发复杂度。Windows 10 或不支持时降级到 NAT 方案。详见 [wsl_network.md](./wsl_network.md)。

**Nginx 作为统一入口**
所有服务通过路径前缀路由（`/ollama-win/`、`/ollama-wsl/`、`/qdrant/` 等），外部只需记一个 IP 和端口，后端地址变更只需改 Nginx 配置。详见 [nginx.md](./nginx.md)。

**双 Ollama 实例：GPU 资源隔离**
Windows 侧禁用 CUDA，专跑轻量 Embedding 模型（核显/CPU）；WSL 侧独占独显，运行大模型推理。避免两个实例竞争显存。详见 [ollama.md](./ollama.md)。

**NAS 作为持久化存储**
文档、模型缓存、GitLab 数据统一存放在 NAS，WSL 通过 NFS 挂载访问，防止本地磁盘撑满。详见 [nas_storage.md](./nas_storage.md)。

---

## 部署顺序

各组件存在依赖关系，建议按以下顺序部署：

1. **WSL 网络**（[wsl_network.md](./wsl_network.md)）— 基础，所有服务依赖它互通
2. **双 Ollama 实例**（[ollama.md](./ollama.md)）— 核心推理与 Embedding 服务
3. **Nginx 统一入口**（[nginx.md](./nginx.md)）— 汇聚所有服务，暴露给局域网
4. **NAS 存储挂载**（[nas_storage.md](./nas_storage.md)）— 持久化文档与数据
5. **GitLab CE**（[gitlab.md](./gitlab.md)）— 代码版本管理
6. **Continue 插件**（[continue.md](./continue.md)）— 最终客户端，依赖前面所有服务

---

## 验收标准

- [ ] WSL 与 Windows 通过 `127.0.0.1` 双向互通
- [ ] `http://localhost/health` 返回 `ok`，局域网其他设备同样可达
- [ ] GPU 资源绑定正确：WSL 独显推理，Windows 核显 Embedding
- [ ] NAS 目录挂载到 WSL `/mnt/nas/ai-docs`，重启后自动挂载
- [ ] GitLab 可访问，代码可推送
- [ ] MacBook Continue 插件可正常调用推理与 Embedding

---

## 故障排查

| 问题 | 解决方法 |
|------|---------|
| WSL 中 `nvidia-smi` 报错 | 确认 Windows NVIDIA 驱动 ≥ 527.x |
| Nginx 502 Bad Gateway | 检查对应后端服务是否已启动 |
| Windows Ollama 仍使用独显 | 确认是**系统级**环境变量，重启服务 |
| WSL 重启后 NFS 丢失 | `systemd=true` + `systemctl enable remote-fs.target` |
| GitLab 内存过高 | 见 [gitlab.md](./gitlab.md) 内存优化章节 |

---

> ⬅️ [返回总览](../README.md) | ➡️ [下一阶段：推理层](../01-inference/README.md)
