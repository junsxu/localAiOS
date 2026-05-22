# 验证环境版本

> 文档中所有命令和配置均基于以下版本验证。  
> 如果你使用了不同版本，行为可能有差异，欢迎[提 Issue](https://github.com/your-repo/localAiOS/issues) 补充。

---

## 主机环境

| 软件 | 版本 | 验证日期 | 备注 |
|------|------|---------|------|
| Windows 11 | 24H2 | - | 需要支持 WSL2 Mirrored 网络模式 |
| WSL2 Ubuntu | 22.04 LTS | - | |
| NVIDIA 驱动 | - | - | 需支持 CUDA 12.x |
| Docker Desktop | - | - | 或直接在 WSL 内安装 Docker Engine |
| Python | 3.11.x | - | 文档代码示例使用 3.11 |
| Nginx | - | - | Windows 版，部署在 Windows 主机 |

---

## AI 组件

| 软件 | 版本 | 验证日期 | 备注 |
|------|------|---------|------|
| Ollama | - | - | Windows + WSL 各一套 |
| Qdrant | - | - | Docker 部署于 WSL |
| Open-WebUI | - | - | Docker 部署于 WSL |
| OpenHands | - | - | Docker 部署于 WSL |

---

## Python 依赖（代码示例）

| 包 | 版本 | 用途 |
|----|------|------|
| `qdrant-client` | - | Qdrant Python 客户端 |
| `ollama` | - | Ollama Python 客户端 |
| `fastapi` | - | 微服务 API 框架 |
| `uvicorn` | - | ASGI 服务器 |
| `httpx` | - | 异步 HTTP 客户端 |
| `pydantic` | - | 数据校验 |

---

## 模型

| 模型 | 来源 | 运行位置 | 用途 |
|------|------|---------|------|
| `qwen2.5-coder:7b` | Ollama 官方 | WSL（dGPU） | 日常代码生成 |
| `qwen2.5-coder:14b` | Ollama 官方 | WSL（dGPU） | 复杂推理 |
| `nomic-embed-text` | Ollama 官方 | Windows（iGPU） | 文本 Embedding |
| `bge-m3` | Ollama 官方 | Windows（iGPU） | 多语言 Embedding（备选） |

---

## 填写说明

验证通过后，请将版本号填入上表，格式示例：

```bash
# 查询各组件版本
ollama --version
docker --version
qdrant --version  # 或查看 Docker Hub 镜像 tag
python --version
pip show qdrant-client ollama fastapi uvicorn
```

完整验证流程见 [docs/07-aios/integration.md](./docs/07-aios/integration.md)。
