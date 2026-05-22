# Continue 插件配置详解

Continue 是 VSCode 的 AI 编程助手插件，连接本地 Ollama 实现代码补全、对话、内联编辑和 Codebase 索引。本文档面向 MacBook M1 客户端配置，但同样适用于任何运行 VSCode 的机器。

---

## 一、安装

1. 打开 VSCode，进入扩展市场（`Cmd+Shift+X`）
2. 搜索 `Continue`，安装 **Continue - Codestral, Claude, and more**
3. 安装后左侧活动栏出现 Continue 图标，点击图标打开面板

---

## 二、配置文件

配置文件位于 `~/.continue/config.yaml`（新版）。点击 Continue 面板右下角齿轮图标，或直接编辑：

```bash
open ~/.continue/config.yaml    # macOS
```

### 完整推荐配置

将 `<主笔记本IP>` 替换为实际局域网 IP（如 `192.168.1.100`）：

```yaml
# ~/.continue/config.yaml

models:
  # 主力对话 / 代码生成（WSL 14B）
  - name: qwen2.5-coder:14b
    provider: ollama
    model: qwen2.5-coder:14b
    apiBase: http://<主笔记本IP>/ollama-wsl/

  # 快速补全（WSL 7B）
  - name: qwen2.5-coder:7b
    provider: ollama
    model: qwen2.5-coder:7b
    apiBase: http://<主笔记本IP>/ollama-wsl/

  # 本地 MBP 离线备用（需在 MBP 安装 Ollama，见第四节）
  # - name: qwen2.5-coder:3b (local)
  #   provider: ollama
  #   model: qwen2.5-coder:3b
  #   apiBase: http://localhost:11434/

# Tab 自动补全模型（用较小模型保证速度）
tabAutocompleteModel:
  name: qwen2.5-coder:7b
  provider: ollama
  model: qwen2.5-coder:7b
  apiBase: http://<主笔记本IP>/ollama-wsl/

# Embedding 模型（用于 @codebase 索引）
embeddingsProvider:
  provider: ollama
  model: nomic-embed-text
  apiBase: http://<主笔记本IP>/ollama-win/

# 上下文提供者
contextProviders:
  - name: code          # 选中代码块
  - name: docs          # 引用文档
  - name: diff          # Git diff
  - name: terminal      # 终端输出
  - name: problems      # 编辑器问题面板
  - name: folder        # 文件夹内容
  - name: codebase      # 整个 codebase（需要先索引）

# Slash 命令
slashCommands:
  - name: edit
    description: Edit highlighted code
  - name: comment
    description: Write comments for the highlighted code
  - name: share
    description: Export the current chat session to markdown
  - name: cmd
    description: Generate a shell command
```

---

## 三、功能使用指南

### 3.1 对话聊天

快捷键 `Cmd+L` 打开聊天面板，直接输入问题。可在对话中引用上下文：

| 引用方式 | 作用 |
|---------|------|
| `@file` | 引用特定文件 |
| `@folder` | 引用整个文件夹 |
| `@codebase` | 引用整个项目（需先索引） |
| `@docs` | 引用已配置的文档站点 |
| `@diff` | 引用当前 Git 改动 |
| `@terminal` | 引用终端最近输出 |

### 3.2 行内编辑（Inline Edit）

选中代码，按 `Cmd+I`，输入修改指令（如"添加错误处理"、"优化性能"），模型直接在编辑器中修改代码，可以 Accept / Reject。

### 3.3 Tab 自动补全

配置 `tabAutocompleteModel` 后，编写代码时自动出现灰色补全建议：

- `Tab`：接受补全
- `Esc`：拒绝
- `Cmd+→`：逐词接受

**提升补全质量的技巧**：

- 在文件顶部写好注释说明文件用途
- 函数名和参数名语义清晰
- 提前 import 需要用到的库

### 3.4 Codebase 索引（@codebase RAG）

Continue 对本地项目建立向量索引，让模型理解整个代码库结构。

**首次索引**：

1. 打开项目文件夹
2. Continue 面板底部点击 **Index** 按钮，或执行命令 `Continue: Index codebase`
3. 等待索引完成（大项目可能需要几分钟）
4. 索引存储于 `~/.continue/index/`

**使用**：在聊天框输入 `@codebase` 后提问：

```
@codebase 这个项目的入口文件在哪里？整体架构是怎样的？
@codebase 找到所有涉及数据库连接的代码
```

### 3.5 文档引用（@docs）

在 `config.yaml` 中添加文档站点：

```yaml
docs:
  - name: Ollama API
    startUrl: https://github.com/ollama/ollama/blob/main/docs/api.md
    rootUrl: https://github.com/ollama/ollama
  - name: Qdrant Docs
    startUrl: https://qdrant.tech/documentation/
    rootUrl: https://qdrant.tech
```

之后对话中可用 `@Ollama API` 引用文档内容。

---

## 四、快捷键参考

| 功能 | 快捷键 |
|------|--------|
| 打开聊天面板 | `Cmd+L` |
| 将选中代码发送到 Continue | `Cmd+Shift+L` |
| 触发行内编辑 | `Cmd+I` |
| 接受 Tab 补全 | `Tab` |
| 拒绝 Tab 补全 | `Esc` |
| 接受行内编辑 | `Cmd+Shift+Enter` |
| 拒绝行内编辑 | `Cmd+Shift+Delete` |

---

## 五、MBP 本地离线小模型（备用）

M1 8GB 内存可运行 3B 以下模型，用于无法连接主机的离线场景：

```bash
# macOS 安装 Ollama
brew install ollama

# 拉取 3B 模型（适合 M1 8GB）
ollama pull qwen2.5-coder:3b    # ~2GB
ollama pull llama3.2:3b          # 备选

# 启动服务
ollama serve
```

在 `config.yaml` 中将本地模型添加为备用（取消注释）：

```yaml
  - name: qwen2.5-coder:3b (local)
    provider: ollama
    model: qwen2.5-coder:3b
    apiBase: http://localhost:11434/
```

**M1 模型速度参考**：

| 模型 | 推理速度 | 内存占用 |
|------|---------|---------|
| qwen2.5-coder:3b | ~20 tok/s | ~2.5GB |
| llama3.2:3b | ~18 tok/s | ~2.5GB |

---

## 六、连接验证

```bash
# 从 MBP 终端验证主机路由可达
curl http://<主笔记本IP>/ollama-wsl/api/tags
curl http://<主笔记本IP>/ollama-win/api/tags

# 验证 Embedding 接口
curl -X POST http://<主笔记本IP>/ollama-win/api/embeddings \
  -d '{"model": "nomic-embed-text", "prompt": "test"}' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'向量维度: {len(d[\"embedding\"])}')"
```

---

## 七、故障排查

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| 模型不响应 | 主机 Nginx 未启动 | 访问 `http://<IP>/health` 验证 |
| 补全极慢 | 7B 模型冷启动 | 在主机预加载模型（`keep_alive: -1`） |
| @codebase 不工作 | 未完成索引 | 重新执行 `Continue: Index codebase` |
| Embedding 失败 | Windows Ollama 未启动 | `Get-Service Ollama` 检查状态 |
| 连接拒绝 | 防火墙未放行 80 端口 | 检查 Windows Defender 防火墙规则 |

---

> ⬅️ [返回阶段 0 概览](./README.md)
