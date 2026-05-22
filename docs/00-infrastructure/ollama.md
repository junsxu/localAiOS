# Ollama 部署详解

本项目运行两个独立的 Ollama 实例，实现 GPU 资源的精确分配：

| 实例 | 运行环境 | GPU | 用途 | 端口 |
|------|---------|-----|------|------|
| Ollama-Win | Windows | Intel 核显 | Embedding（nomic-embed-text） | 11435 |
| Ollama-WSL | WSL Ubuntu | RTX5070Ti | 大模型推理（7B/14B） | 11434 |

---

## 一、Windows 实例（核显·Embedding）

### 1.1 安装

前往 [ollama.com/download](https://ollama.com/download) 下载 Windows 安装包（`.exe`），按向导安装。安装完成后 Ollama 自动注册为 Windows 服务。

```powershell
ollama --version
Get-Service -Name "Ollama"   # 状态应为 Running
```

### 1.2 GPU 隔离配置

Ollama 默认优先使用 NVIDIA 独立显卡。为保留 RTX5070Ti 给 WSL 专用，通过系统级环境变量强制使用核显（iGPU）或纯 CPU：

```powershell
# 管理员 PowerShell

# 禁用 CUDA（屏蔽所有 NVIDIA GPU）
[System.Environment]::SetEnvironmentVariable("CUDA_VISIBLE_DEVICES", "", "Machine")

# 强制 Ollama 不使用 GPU 加速层（确保完全使用 CPU/iGPU）
[System.Environment]::SetEnvironmentVariable("OLLAMA_GPU_LAYERS", "0", "Machine")

# 监听地址：11435（与 WSL 的 11434 区分）
[System.Environment]::SetEnvironmentVariable("OLLAMA_HOST", "0.0.0.0:11435", "Machine")

# 重启服务使配置生效
Restart-Service -Name "Ollama"
```

> **注意**：必须设置为**系统级**（"Machine"）而非用户级（"User"），否则服务进程不继承该变量。设置后需重启服务。

### 1.3 验证 GPU 使用情况

```powershell
# 方法 1：任务管理器
# 性能 → GPU → 查看 GPU 0（Intel 核显）有负载，GPU 1（NVIDIA）无负载

# 方法 2：命令行观察
Start-Process -NoNewWindow ollama -ArgumentList "run nomic-embed-text test"
# 同时观察任务管理器 GPU 占用
```

### 1.4 拉取 Embedding 模型

```powershell
ollama pull nomic-embed-text   # 274MB，768 维向量

# 验证
ollama list
# NAME                    ID              SIZE
# nomic-embed-text:latest ...             274 MB
```

### 1.5 防火墙放行

```powershell
New-NetFirewallRule -DisplayName "Ollama Embedding :11435" `
    -Direction Inbound -Protocol TCP -LocalPort 11435 -Action Allow
```

### 1.6 测试 Embedding API

```powershell
$body = @{ model = "nomic-embed-text"; prompt = "Hello, world!" } | ConvertTo-Json
$resp = Invoke-RestMethod -Uri "http://localhost:11435/api/embeddings" `
    -Method POST -ContentType "application/json" -Body $body
$resp.embedding.Count   # 应输出 768
```

### 1.7 Windows 服务管理

```powershell
Get-Service Ollama                          # 查看状态
Restart-Service Ollama                      # 重启
Stop-Service Ollama; Start-Service Ollama   # 重启（等效）

# 查看日志（Event Viewer 或）
Get-EventLog -LogName Application -Source Ollama -Newest 20
```

---

## 二、WSL 实例（独显·大模型推理）

### 2.1 前置条件

**Windows 侧 NVIDIA 驱动**（无需在 WSL 内另装）：

```powershell
# Windows PowerShell 验证
nvidia-smi
# 应显示 RTX5070Ti，CUDA Version: 12.x
```

WSL GPU 直通依赖 Windows NVIDIA 驱动 ≥ 527.x（支持 CUDA 12.x）。

### 2.2 安装 Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh

# 脚本自动完成：
# - 安装二进制到 /usr/local/bin/ollama
# - 创建 ollama 用户
# - 注册 systemd 服务 ollama.service

ollama --version
```

### 2.3 服务配置

```bash
# 创建 systemd override
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo tee /etc/systemd/system/ollama.service.d/override.conf <<'EOF'
[Service]
# 监听所有网卡（Mirrored 模式下 Windows 可通过 127.0.0.1 访问）
Environment="OLLAMA_HOST=0.0.0.0:11434"

# 模型存储路径（可指向 NAS 挂载以节省 SSD 空间）
Environment="OLLAMA_MODELS=/opt/ollama/models"

# 最大并发请求队列
Environment="OLLAMA_MAX_QUEUE=4"

# 显存使用比例上限（保留 10% 给系统）
Environment="OLLAMA_GPU_MEMORY_FRACTION=0.9"

# 开启详细日志（调试用，正常使用可去掉）
# Environment="OLLAMA_DEBUG=1"
EOF

# 创建模型目录
sudo mkdir -p /opt/ollama/models
sudo chown ollama:ollama /opt/ollama/models

# 重载并启动
sudo systemctl daemon-reload
sudo systemctl enable --now ollama

# 验证
sudo systemctl status ollama
```

### 2.4 验证 GPU 直通

```bash
# 应显示 RTX5070Ti 信息
nvidia-smi

# 运行时实时监控
watch -n 1 nvidia-smi

# 安装更友好的监控工具
sudo apt install nvtop
nvtop
```

### 2.5 拉取推理模型

```bash
# 代码专用模型（推荐）
ollama pull qwen2.5-coder:7b    # ~4.7GB，快速补全
ollama pull qwen2.5-coder:14b   # ~8.9GB，主力代码生成

# 通用对话模型（可选）
ollama pull qwen2.5:7b
ollama pull qwen2.5:14b

# 查看已安装模型
ollama list
```

**RTX5070Ti 12GB 显存参考**：

| 模型 | 量化 | 显存占用 | 推理速度 | 建议用途 |
|------|------|---------|---------|---------|
| qwen2.5-coder:7b | Q4_K_M | ~5GB | ~30 tok/s | Tab 补全、快速问答 |
| qwen2.5-coder:14b | Q4_K_M | ~9GB | ~15 tok/s | 主力代码生成、复杂推理 |
| qwen2.5-coder:14b | Q5_K_M | ~11GB | ~12 tok/s | 更高精度（接近满显存） |

### 2.6 推理 API 测试

```bash
# 原生生成接口
curl -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5-coder:14b",
    "prompt": "用 Python 写一个快速排序",
    "stream": false
  }'

# OpenAI 兼容接口（Continue 插件使用此格式）
curl -X POST http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5-coder:14b",
    "messages": [
      {"role": "system", "content": "你是一个代码助手"},
      {"role": "user", "content": "Hello!"}
    ],
    "stream": false
  }'

# 查看已加载模型及显存占用
curl http://localhost:11434/api/ps
```

### 2.7 性能调优

**预加载模型（消除冷启动延迟）**：

```bash
# keep_alive: -1 表示永久保持在显存中不卸载
curl -X POST http://localhost:11434/api/generate \
  -d '{"model": "qwen2.5-coder:14b", "prompt": "", "keep_alive": -1}'
```

**自定义上下文窗口**：

```bash
cat > /tmp/Modelfile << 'EOF'
FROM qwen2.5-coder:14b

# 扩大上下文窗口（消耗更多显存）
PARAMETER num_ctx 8192

# 限制输出长度
PARAMETER num_predict 4096

# 代码场景推荐参数
PARAMETER temperature 0.1
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.05
EOF

ollama create qwen2.5-coder:14b-code -f /tmp/Modelfile
```

**多模型管理**（同时保留两个模型在显存中）：

```bash
# RTX5070Ti 12GB 可同时载入 7B + 小模型，或单独载入 14B
# 通过 keep_alive 控制是否卸载

# 永久保持 14B 不卸载（占 ~9GB）
curl -X POST http://localhost:11434/api/generate \
  -d '{"model": "qwen2.5-coder:14b", "prompt": "", "keep_alive": -1}'

# 让 7B 在 5 分钟后自动卸载
curl -X POST http://localhost:11434/api/generate \
  -d '{"model": "qwen2.5-coder:7b", "prompt": "", "keep_alive": "5m"}'
```

### 2.8 WSL 服务管理

```bash
sudo systemctl status ollama     # 查看状态
sudo systemctl restart ollama    # 重启
sudo systemctl stop ollama       # 停止
journalctl -u ollama -f          # 实时日志
journalctl -u ollama --since "10 minutes ago"   # 近期日志
```

---

## 三、Embedding 模型选型

| 模型 | 维度 | 大小 | 语言 | 推荐场景 |
|------|------|------|------|---------|
| `nomic-embed-text` | 768 | ~270MB | 英文优先 | 英文文档，轻量快速 |
| `mxbai-embed-large` | 1024 | ~670MB | 英文 | 高精度英文 |
| `bge-m3` | 1024 | ~570MB | 中英双语 | **中文文档（推荐）** |
| `shaw/dmeta-embedding-zh` | 768 | ~400MB | 中文专用 | 纯中文场景 |

```bash
# 切换到中文优化模型
ollama pull bge-m3

# 在 RAG pipeline 中调整向量维度（bge-m3 为 1024）
```

---

## 四、模型存储管理

```bash
# 查看模型存储位置和大小
du -sh /opt/ollama/models/
ollama list

# 删除不需要的模型
ollama rm qwen2.5:32b

# 将模型目录迁移到 NAS（释放 SSD 空间）
sudo systemctl stop ollama
sudo mv /opt/ollama/models/* /mnt/nas/ai-models/
# 修改 override.conf 中的 OLLAMA_MODELS 路径
sudo systemctl start ollama
```

---

## 五、故障排查

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| WSL `nvidia-smi` 不显示 GPU | NVIDIA 驱动版本过低 | Windows 更新驱动到 ≥ 527.x |
| 模型加载失败（OOM） | 显存不足 | 改用更小量化版本（Q3_K_S） |
| Windows Ollama 仍占用独显 | 环境变量为用户级 | 重设为系统级，重启服务 |
| 推理速度极慢 | 模型在 CPU 推理 | `journalctl -u ollama` 检查是否有 `using CPU` 日志 |
| 请求超时 | 模型未预加载，冷启动慢 | 使用 `keep_alive: -1` 预加载 |
| 端口被占用 | 另一 Ollama 实例 | `lsof -i :11434` 或 `netstat -ano \| findstr 11434` |

---

> ⬅️ [返回阶段 0 概览](./README.md)
