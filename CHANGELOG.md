# Changelog

本文件记录 Local AI OS 文档的版本历史，遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/) 格式。

版本号规则：
- **主版本号**：架构发生重大变化（例如迁移到 Mac Studio 统一架构）
- **次版本号**：完成一个新阶段或新功能模块
- **修订号**：文档修正、补充踩坑记录、版本更新

---

## [Unreleased]

### 新增
- `docs/03-agent/openhands.md`：OpenHands 自主编码 Agent 部署文档、含 Open-WebUI 对比选择指南、Docker Compose 配置、Nginx 代理、systemd 服务

### 计划中
- 每个阶段 README 顶部添加验证日期和验证环境
- 填写 `VERSIONS.md` 中所有软件的具体版本
- 补充 GitHub Issue 模板
- 嵌入视频系列链接（Bilibili + YouTube）
- 发布 Release v1.0

---

## [0.9.0] - 2026-05

### 新增
- 阶段 0：基础设施文档（WSL 网络、Nginx、双实例 Ollama、Continue、NAS 存储、GitLab）
- 阶段 1：推理层文档（Transformer 原理、采样参数、Embedding、Ollama API）
- 阶段 2：RAG 文档（Qdrant 部署、RAG Pipeline、混合检索）
- 阶段 3：Agent 文档（Open-WebUI 部署、OpenHands 部署、Tool Calling / ReAct 实现）
- 阶段 4：记忆层文档（对话摘要、长期记忆服务、用户画像）
- 阶段 5：编排层文档（Docker 微服务、docker-compose、Nginx API 网关）
- 阶段 6：性能层文档（分层路由：规则/LLM/自适应、基准测试脚本）
- 阶段 7：AI OS 终态（集成自检、演进路线 v1.0 → v3.0）
- `schedule/` 文件夹：验证计划、视频制作计划、开源发布计划
- 根目录 `README.md`：硬件配置、系统架构图、端口规划、阶段路线图

### 说明
- 当前版本为文档草稿阶段，各阶段命令正在系统验证中
- 代码示例已可运行，但依赖版本以 `VERSIONS.md` 中锁定版本为准

---

## 版本命名约定

| 版本 | 里程碑 |
|------|--------|
| v0.9.x | 文档草稿，验证进行中 |
| v1.0.0 | 阶段 0-7 全部完成端到端验证，配合视频系列正式开源 |
| v1.1.0 | Memory 知识图谱 + RAG Rerank |
| v1.2.0 | Grafana 监控 :3001 + A/B 路由测试 |
| v1.3.0 | 语音输入（Whisper）+ 定时任务 |
| v2.0.0 | Mac Studio 迁移，Apple Silicon 统一内存架构 |
| v3.0.0 | 多模态 + 自我优化 + 插件生态 |

详细演进规划见 [docs/07-aios/roadmap.md](./docs/07-aios/roadmap.md)。
