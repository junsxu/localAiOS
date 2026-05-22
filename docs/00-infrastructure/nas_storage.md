# NAS 文档存储详解

QNAP 作为整个 AI 系统的持久化文件存储层，存放知识库原始文档，并通过 SMB/NFS 共享挂载到 Windows 和 WSL。

---

## 一、硬件配置

| 组件 | 规格 | 配置建议 |
|------|------|---------|
| CPU | Celeron J4125 | 足够运行 Docker 服务 |
| 内存 | 4GB × 2 = 8GB | GitLab 至少需要 4GB |
| 机械硬盘 | 8TB × 2 | RAID 1（镜像，数据安全优先） |
| SSD | 240GB | 设为 SSD 读写缓存 |

---

## 二、存储结构规划

### 2.1 RAID 与卷配置

1. 打开 `Storage & Snapshots`
2. 创建存储池：选择两块 8TB 机械盘，RAID 类型选 **RAID 1**
3. 在存储池上创建厚卷，分配 4TB 给文档存储
4. SSD 240GB 设为**读写 SSD 缓存**，加速频繁访问的热数据

### 2.2 共享文件夹规划

| 共享文件夹 | NAS 路径 | 用途 |
|-----------|---------|------|
| `ai-docs` | `/share/ai-docs` | 知识库原始文档（PDF/Word/MD） |
| `ai-models` | `/share/ai-models` | Ollama 模型文件（大文件，可选） |
| `ai-code` | `/share/ai-code` | 代码项目归档 |
| `gitlab-data` | `/share/gitlab-data` | GitLab 数据持久化卷 |
| `backups` | `/share/backups` | 系统备份 |

**创建步骤**：`控制台 → 共享文件夹 → 新增共享文件夹`，选择目标卷，权限设为局域网可访问。

### 2.3 知识库目录结构建议

```
/share/ai-docs/
├── technical/              # 技术文档、白皮书
│   ├── ai/                 # AI 相关论文、教程
│   ├── devops/
│   └── networking/
├── projects/               # 项目文档
│   └── localAiOS/
├── books/                  # 电子书 PDF
├── notes/                  # 个人笔记 Markdown
└── .indexed/               # 已建立向量索引的文件（软链接或标记）
```

---

## 三、SMB 配置（Windows 挂载）

### 3.1 QNAP 启用 SMB

`控制台 → 网络服务 → Win/Mac/NFS → Microsoft 网络 → 启用`，协议版本选择 **SMB 3**。

### 3.2 Windows 挂载为网络驱动器

```powershell
# 挂载 ai-docs 为 Z 盘（持久化）
net use Z: \\<NAS-IP>\ai-docs /user:<NAS用户名> <密码> /persistent:yes

# 验证
Get-PSDrive Z
dir Z:\

# 断开挂载
net use Z: /delete
```

**资源管理器方式**：`此电脑 → 映射网络驱动器 → 填写 \\<NAS-IP>\ai-docs`。

### 3.3 凭据管理

```powershell
# 保存凭据（避免每次输入密码）
cmdkey /add:<NAS-IP> /user:<用户名> /pass:<密码>

# 查看已保存凭据
cmdkey /list

# 删除凭据
cmdkey /delete:<NAS-IP>
```

---

## 四、NFS 配置（WSL 挂载）

### 4.1 QNAP 启用 NFS

1. `控制台 → 网络服务 → Win/Mac/NFS → NFS 服务 → 启用 NFS v4`
2. 在 `ai-docs` 共享文件夹属性中，勾选 **NFS 访问权限**
3. 允许访问的 IP：填写主笔记本 IP 或子网（如 `192.168.1.0/24`）

### 4.2 WSL 安装 NFS 客户端

```bash
sudo apt update && sudo apt install -y nfs-common
```

### 4.3 手动挂载测试

```bash
sudo mkdir -p /mnt/nas/ai-docs
sudo mount -t nfs <NAS-IP>:/share/ai-docs /mnt/nas/ai-docs

# 验证
ls /mnt/nas/ai-docs
df -h /mnt/nas/ai-docs    # 查看挂载的空间信息

# 卸载
sudo umount /mnt/nas/ai-docs
```

### 4.4 开机自动挂载（/etc/fstab）

```bash
# 追加到 /etc/fstab
sudo tee -a /etc/fstab <<'EOF'
<NAS-IP>:/share/ai-docs    /mnt/nas/ai-docs    nfs    defaults,_netdev,nfsvers=4,rsize=131072,wsize=131072    0    0
<NAS-IP>:/share/ai-models  /mnt/nas/ai-models  nfs    defaults,_netdev,nfsvers=4    0    0
EOF

# 测试 fstab（不重启）
sudo mount -a
```

> `rsize`/`wsize=131072` 提升大文件传输性能（读取 PDF 等）。

### 4.5 WSL 中 systemd 管理自动挂载

WSL 启用 `systemd=true` 后，`/etc/fstab` 会由 `remote-fs.target` 自动处理：

```bash
sudo systemctl enable remote-fs.target
sudo systemctl start remote-fs.target
```

---

## 五、挂载性能优化

### 5.1 NFS 挂载参数说明

| 参数 | 说明 |
|------|------|
| `nfsvers=4` | 使用 NFS v4，性能和安全性更好 |
| `rsize=131072` | 读块大小 128KB |
| `wsize=131072` | 写块大小 128KB |
| `_netdev` | 等待网络就绪后再挂载 |
| `noatime` | 不更新访问时间，减少写操作 |
| `soft` | 网络断开时命令超时而不是卡死 |

```bash
# 性能优化完整参数
<NAS-IP>:/share/ai-docs /mnt/nas/ai-docs nfs defaults,_netdev,nfsvers=4,rsize=131072,wsize=131072,noatime,soft 0 0
```

### 5.2 测试传输速度

```bash
# 写速度测试
dd if=/dev/zero of=/mnt/nas/ai-docs/test.bin bs=1M count=100 conv=fsync
# 删除测试文件
rm /mnt/nas/ai-docs/test.bin

# 读速度测试（先清缓存）
sudo sh -c "echo 3 > /proc/sys/vm/drop_caches"
dd if=/mnt/nas/ai-docs/<某个大文件> of=/dev/null bs=1M
```

---

## 六、文档同步工作流

### 6.1 扫描新文件触发索引

```bash
#!/bin/bash
# watch-and-index.sh：监控 NAS 新文件并触发 RAG 索引

WATCH_DIR="/mnt/nas/ai-docs"
MARKER="/tmp/last_index_run"

# 找出比上次运行更新的文件
find "$WATCH_DIR" -type f \( -name '*.pdf' -o -name '*.md' -o -name '*.txt' \) \
  -newer "$MARKER" | while read -r file; do
    echo "[$(date)] Queuing for index: $file"
    # 调用 rag_indexer.py（见 02-rag 文档）
    python3 ~/rag_indexer.py --file "$file"
done

touch "$MARKER"
```

```bash
# 每天凌晨 4 点运行
(crontab -l 2>/dev/null; echo "0 4 * * * ~/watch-and-index.sh >> ~/indexer.log 2>&1") | crontab -
```

### 6.2 文件组织规范

为了让 RAG 检索质量更好，文档命名建议：

```
YYYYMMDD_主题_来源.pdf
20240115_transformer-attention-mechanism_arxiv.pdf
20240201_qnap-nas-setup-guide_official.pdf
```

---

## 七、备份策略

### 7.1 QNAP HBS3 自动备份

1. 打开 **Hybrid Backup Sync (HBS3)**
2. 创建备份任务：源 → `ai-docs`，目标 → `backups` 或外接 USB
3. 设置每日凌晨 3 点执行增量备份
4. 启用版本控制，保留最近 30 个版本

### 7.2 手动备份到 Windows

```powershell
# 将 NAS 文档备份到本机
robocopy Z:\ D:\Backup\NAS-AI-Docs\ /E /COPYALL /R:3 /W:5 /LOG:D:\Backup\robocopy.log
```

### 7.3 WSL 中 rsync 增量同步

```bash
# 将 NAS 同步到本地 SSD（热数据缓存）
rsync -avz --delete /mnt/nas/ai-docs/ ~/ai-docs-cache/
```

---

> ⬅️ [返回阶段 0 概览](./README.md)
