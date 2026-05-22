# GitLab CE 部署详解

GitLab Community Edition 部署在 QNAP NAS 上，作为整个 AI 系统的私有代码仓库，管理配置文件、脚本与项目代码。

---

## 一、前置条件

- QNAP QTS ≥ 5.1
- Container Station 已安装（App Center 搜索安装）
- NAS 可用内存 ≥ 4GB（GitLab 推荐最低配置）
- `gitlab-data` 共享文件夹已创建（见 [nas_storage.md](./nas_storage.md)）

---

## 二、部署 GitLab

> 以下步骤全程在 QNAP 网页管理界面中以鼠标操作完成，无需 SSH 命令行。

### 2.1 创建目录结构（File Station）

1. 浏览器登录 QNAP 管理页面（`http://<NAS-IP>:8080`）
2. 打开 **File Station**
3. 进入 `gitlab-data` 共享文件夹，依次新建三个子文件夹：
   - `config`
   - `logs`
   - `data`

### 2.2 部署容器（Container Station）

1. 在 QNAP 主页打开 **Container Station**
2. 左侧菜单点击 **应用程序（Applications）** → 右上角 **创建（Create）**
3. 在编辑框中粘贴以下 `docker-compose.yml` 内容：

```yaml
version: '3.8'

services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: gitlab.local
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        # 外部访问 URL
        external_url 'http://<NAS-IP>:8929'

        # SSH 端口（避免与 NAS 自身的 22 端口冲突）
        gitlab_rails['gitlab_shell_ssh_port'] = 2224

        # ── 内存优化（QNAP 453Bmini 内存有限）────────────────
        # Puma Web 服务器（减少 worker 数量）
        puma['worker_processes'] = 2
        puma['min_threads'] = 1
        puma['max_threads'] = 4

        # Sidekiq 后台任务
        sidekiq['max_concurrency'] = 5

        # 禁用监控组件（节省 1~2GB 内存）
        prometheus_monitoring['enable'] = false
        grafana['enable'] = false
        alertmanager['enable'] = false
        node_exporter['enable'] = false

        # 禁用 Pages（若不需要）
        pages_enabled = false

    ports:
      - "8929:8929"   # HTTP
      - "2224:22"     # SSH

    volumes:
      - /share/gitlab-data/config:/etc/gitlab
      - /share/gitlab-data/logs:/var/log/gitlab
      - /share/gitlab-data/data:/var/opt/gitlab

    # 共享内存（默认 64MB 不够）
    shm_size: '256m'
```

4. 点击 **创建（Create）** 按钮，Container Station 会自动拉取镜像并启动容器
5. 在应用程序列表中可以看到容器状态变为 **运行中（Running）**（首次启动需 3～5 分钟初始化）

### 2.3 获取初始 root 密码（Container Station 终端）

1. 在 Container Station 的容器列表中，点击 **gitlab** 容器
2. 点击 **终端（Terminal）** 标签，进入容器内置 Shell
3. 在终端中执行：

```bash
cat /etc/gitlab/initial_root_password
# 输出类似：
# Password: xxxxxxxxxxxxxxxxx
# （24 小时后此文件会被自动删除）
```

---

## 三、初始配置

### 3.1 登录 Web UI

浏览器访问 `http://<NAS-IP>:8929`，使用 `root` 和初始密码登录。

### 3.2 修改 root 密码

登录后立即：`头像 → Edit profile → Password → 修改`。

### 3.3 创建个人用户

`Admin → Users → New user`，填写用户名、邮箱，设置密码。

### 3.4 创建项目组和仓库

1. `Groups → New group`，创建 `localAiOS` 组
2. 在组内 `New project → Create blank project`
3. 建议创建：
   - `localAiOS/configs`：各服务配置文件
   - `localAiOS/scripts`：自动化脚本
   - `localAiOS/rag-pipeline`：RAG 相关代码

---

## 四、SSH 访问配置

### 4.1 生成 SSH Key（各客户端执行）

```bash
# WSL / MBP / Linux
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519_gitlab
cat ~/.ssh/id_ed25519_gitlab.pub   # 复制此内容
```

### 4.2 添加公钥到 GitLab

GitLab Web UI → 头像 → `Preferences → SSH Keys → Add new key`，粘贴公钥。

### 4.3 配置 SSH config

编辑 `~/.ssh/config`：

```
Host nas-gitlab
    HostName <NAS-IP>
    User git
    Port 2224
    IdentityFile ~/.ssh/id_ed25519_gitlab
    StrictHostKeyChecking no
```

### 4.4 测试连接

```bash
ssh -T nas-gitlab
# Welcome to GitLab, @username!
```

### 4.5 克隆 / 推送

```bash
# 使用 SSH config 别名
git clone nas-gitlab:localAiOS/configs.git

# 或完整 URL
git clone ssh://git@<NAS-IP>:2224/localAiOS/configs.git

# 推送
git remote add nas ssh://git@<NAS-IP>:2224/localAiOS/configs.git
git push -u nas main
```

---

## 五、内存监控与优化

### 5.1 监控内存使用

```bash
# 在 NAS SSH 中查看 GitLab 容器内存
docker stats gitlab --no-stream
# 正常运行约 2~3GB（优化后）

# 查看各进程内存
docker exec gitlab ps aux --sort=-%mem | head -20
```

### 5.2 进一步内存压缩

若 NAS 内存不足，在 `docker-compose.yml` 的 `GITLAB_OMNIBUS_CONFIG` 中继续限制：

```ruby
# 极限内存模式（可能影响性能）
puma['worker_processes'] = 1
puma['min_threads'] = 1
puma['max_threads'] = 2
sidekiq['max_concurrency'] = 2

# 禁用更多功能
gitlab_rails['usage_ping_enabled'] = false
gitlab_rails['sentry_enabled'] = false
```

重新应用配置：

```bash
docker exec gitlab gitlab-ctl reconfigure
```

---

## 六、自动备份

### 6.1 手动触发备份

```bash
# 在容器内执行备份（备份文件保存到 /var/opt/gitlab/backups）
docker exec gitlab gitlab-backup create

# 备份文件位于 NAS 的 /share/gitlab-data/data/backups/
ls /share/gitlab-data/data/backups/
```

### 6.2 自动定期备份（NAS crontab）

```bash
# 编辑 NAS 的 crontab
crontab -e

# 每天凌晨 2 点备份（跳过大型附件，加快速度）
0 2 * * * docker exec gitlab gitlab-backup create SKIP=artifacts,packages >> /share/backups/gitlab-backup.log 2>&1

# 每周日同步备份文件到 /share/backups
0 3 * * 0 cp /share/gitlab-data/data/backups/*.tar /share/backups/gitlab/
```

### 6.3 恢复备份

```bash
# 停止相关服务
docker exec gitlab gitlab-ctl stop puma
docker exec gitlab gitlab-ctl stop sidekiq

# 恢复（替换实际的备份文件名）
docker exec gitlab gitlab-backup restore BACKUP=1234567890_2024_01_15_16.0.0

# 重启
docker exec gitlab gitlab-ctl start
docker exec gitlab gitlab-rake gitlab:check SANITIZE=true
```

---

## 七、CI/CD 配置（可选）

在 NAS 资源有限的情况下，推荐将 CI Runner 注册到主笔记本的 WSL 上而非 NAS：

```bash
# WSL 中安装 GitLab Runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install gitlab-runner

# 注册 Runner（从 GitLab 获取 token：Settings → CI/CD → Runners）
sudo gitlab-runner register \
  --url "http://<NAS-IP>:8929" \
  --registration-token "<TOKEN>" \
  --executor "shell" \
  --description "WSL Runner" \
  --tag-list "wsl,local"
```

---

## 八、常用管理命令

```bash
# 在 NAS SSH 中

# 查看 GitLab 状态
docker exec gitlab gitlab-ctl status

# 重启 GitLab 所有服务
docker exec gitlab gitlab-ctl restart

# 查看日志
docker exec gitlab gitlab-ctl tail

# 重新配置（修改 omnibus config 后执行）
docker exec gitlab gitlab-ctl reconfigure

# 更新镜像
docker-compose pull gitlab
docker-compose up -d --no-deps gitlab
```

---

## 九、故障排查

| 问题 | 解决方法 |
|------|---------|
| 首次启动很慢（>10分钟） | 正常现象，QNAP 机械盘 IO 慢 |
| 内存 OOM 杀死容器 | 检查 puma workers，限制为 1~2 |
| SSH 连接失败 | 确认 `~/.ssh/config` 端口为 2224，密钥已添加 |
| Web UI 504 超时 | NAS 负载过高，等待或重启容器 |
| `git push` 卡住 | 检查 NAS 磁盘空间 `df -h` |
| 忘记 root 密码 | `docker exec gitlab gitlab-rake "gitlab:password:reset[root]"` |

---

> ⬅️ [返回阶段 0 概览](./README.md)
