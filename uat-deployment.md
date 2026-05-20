# UAT 环境局域网部署与 VS Code 联动指南

本指南说明如何在同一局域网的 Ubuntu 机器上部署本项目作为 UAT（User Acceptance Testing）环境，并实现与 Windows VS Code 开发环境的高效联动。

---

## 一、方案总览

| 项目 | 说明 |
|------|------|
| **UAT 机器** | 局域网内一台 Ubuntu（建议 22.04/24.04 LTS） |
| **部署方式** | Docker Compose 简化版（`compose.uat.yml`），直接端口映射，无需 Traefik 和公网域名 |
| **访问方式** | 通过局域网 IP + 端口直接访问（如 `http://192.168.1.100`） |
| **VS Code 联动** | ① Remote-SSH 远程开发 ② 本地前端 → UAT 后端联调 ③ 数据库直连调试 ④ 远程容器日志监控 |

---

## 二、Ubuntu UAT 机器准备

### 1. 基础系统更新

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ca-certificates gnupg
```

### 2. 安装 Docker & Docker Compose

```bash
# 安装 Docker
curl -fsSL https://get.docker.com | sudo sh

# 将当前用户加入 docker 组（免 sudo）
sudo usermod -aG docker $USER

# 验证
docker --version
docker compose version
```

> ⚠️ 执行 `usermod` 后需要 **重新登录 SSH** 或执行 `newgrp docker` 生效。

### 3. 配置防火墙（UFW）

```bash
sudo ufw allow 22/tcp      # SSH（必须）
sudo ufw allow 80/tcp      # 前端 Nginx
sudo ufw allow 8000/tcp    # 后端 API
sudo ufw allow 8080/tcp    # Adminer
sudo ufw allow 5432/tcp    # PostgreSQL（调试用）
sudo ufw allow 1080/tcp    # Mailcatcher Web
sudo ufw allow 1025/tcp    # Mailcatcher SMTP
sudo ufw enable
```

### 4. 确认 SSH 可用

确保你能从当前 Windows 机器通过 SSH 登录：

```bash
ssh <ubuntu用户名>@<UAT_IP地址>
```

---

## 三、项目部署

### 1. 克隆项目到 Ubuntu

```bash
cd ~
git clone <你的仓库地址> full-stack-uat
cd full-stack-uat
```

### 2. 配置 `.env` 文件（关键步骤）

复制现有的 `.env` 并进行以下**针对性修改**：

```bash
cp .env .env.backup
nano .env
```

**必须修改的项：**

| 变量 | 建议值 | 说明 |
|------|--------|------|
| `UAT_HOST_IP` | `192.168.x.x` | **新增**。Ubuntu 的局域网 IP，前端构建时会嵌入此地址指向后端 |
| `ENVIRONMENT` | `staging` | 非 `local`，会启用严格的安全密钥检查 |
| `DOMAIN` | `192.168.x.x` | 局域网 IP 或内部域名（如 `uat.local`） |
| `FRONTEND_HOST` | `http://192.168.x.x` | UAT 前端访问地址 |
| `BACKEND_CORS_ORIGINS` | `http://localhost:5173, http://192.168.x.x, http://192.168.x.x:80` | **关键！** 必须包含你本地 VS Code 开发时前端的地址 `http://localhost:5173`，否则本地前端无法调用 UAT 后端 |
| `SECRET_KEY` | 随机字符串 | 在 Ubuntu 上执行 `openssl rand -hex 32` 生成，**不能**保留 `changethis` |
| `POSTGRES_PASSWORD` | 强密码 | **不能**保留 `changethis` |
| `FIRST_SUPERUSER_PASSWORD` | 强密码 | **不能**保留 `changethis` |
| `FIRST_SUPERUSER` | `admin@example.com` | UAT 超级管理员邮箱 |
| `PROJECT_NAME` | 你的项目名 | — |

**可简化/保持默认的项：**

- `SMTP_HOST` / `SMTP_USER` / `SMTP_PASSWORD`：留空即可，`compose.uat.yml` 已内置 Mailcatcher 替代
- `SENTRY_DSN`：UAT 环境可留空
- `STACK_NAME` / `DOCKER_IMAGE_BACKEND` / `DOCKER_IMAGE_FRONTEND` / `TAG`：UAT 方案不依赖这些变量

> 💡 **CORS 配置是联动的核心**：`BACKEND_CORS_ORIGINS` 中的 `http://localhost:5173` 允许你在 Windows 本上运行 `bun run dev`，直接调用 UAT 后端 API 进行调试。

### 3. 构建并启动 UAT 环境

```bash
# 设置环境变量并启动（将 192.168.x.x 替换为实际 IP）
UAT_HOST_IP=192.168.x.x docker compose -f compose.uat.yml up -d --build
```

### 4. 验证部署状态

```bash
# 查看所有容器状态
docker compose -f compose.uat.yml ps

# 查看启动日志
docker compose -f compose.uat.yml logs -f

# 特别检查后端健康检查是否通过
docker compose -f compose.uat.yml logs -f backend
```

### 5. 访问 UAT 服务

在同一局域网内的任何设备上，通过 Ubuntu 的 IP 访问：

| 服务 | 地址 | 说明 |
|------|------|------|
| 前端 | `http://<UAT_IP>` | Nginx serving React 生产构建 |
| 后端 API | `http://<UAT_IP>:8000` | FastAPI |
| Swagger 文档 | `http://<UAT_IP>:8000/docs` | API 调试 |
| Adminer | `http://<UAT_IP>:8080` | 数据库 Web 管理 |
| Mailcatcher | `http://<UAT_IP>:1080` | 查看测试邮件 |

---

## 四、VS Code 开发环境联动

### 方案 1：Remote-SSH 远程开发（推荐用于运维和查错）

在 Windows 本地的 VS Code 中：

1. **安装扩展**：`Remote - SSH`（Microsoft 官方）
2. **配置 SSH**：`Ctrl+Shift+P` → `Remote-SSH: Open SSH Configuration File` → 选择 `C:\Users\<你的用户>\.ssh\config`，添加：
   ```
   Host uat-server
       HostName 192.168.x.x
       User <ubuntu用户名>
       ForwardAgent yes
   ```
3. **连接远程**：`Ctrl+Shift+P` → `Remote-SSH: Connect to Host...` → 选择 `uat-server`
4. **打开项目**：连接成功后，`File` → `Open Folder...` → 输入 `~/full-stack-uat`
5. **打开终端**：`` Ctrl+` `` 打开远程终端，直接执行 `docker compose` 命令

**联动效果：**

- 你在 VS Code 里编辑的是 UAT 机器上的代码
- 保存后直接在远程终端执行 `docker compose -f compose.uat.yml up -d --build` 即可更新
- **端口自动转发**：当你在远程 VS Code 中点击 `http://localhost:8000` 链接时，VS Code 会自动将远程 8000 端口转发到你 Windows 本地的某个端口（如 `localhost:59312`），你用本地浏览器即可访问

### 方案 2：本地前端 + UAT 后端（推荐用于前端开发和 API 联调）

这是最实用的日常开发模式：

1. **在 Windows 本地**，确保前端依赖已安装：
   ```bash
   cd frontend
   bun install
   ```

2. **修改本地前端环境变量**（如果本地 `.env` 文件存在）：
   ```env
   VITE_API_URL=http://<UAT_IP>:8000
   ```
   如果没有 `.env`，也可以在 `frontend/.env` 或运行时指定。

3. **本地启动前端开发服务器**：
   ```bash
   cd frontend
   bun run dev
   ```

4. **浏览器访问** `http://localhost:5173`

**联动效果：**

- 你看到的是本地 Vite 热重载的前端（修改前端代码即时刷新）
- 所有 API 请求（登录、CRUD 等）实际打到 UAT 机器的后端
- UAT 后端使用的是 UAT 数据库（包含真实/测试数据）
- **调试网络请求**：在浏览器 DevTools 的 Network 面板中可直接看到与 UAT 后端的交互

### 方案 3：数据库直连调试

在 Windows 本地通过工具连接 UAT 的 PostgreSQL：

- **VS Code 插件**：安装 `PostgreSQL`（Chris Kolkman）或 `SQLTools`，配置连接：
  - Host: `192.168.x.x`
  - Port: `5432`
  - Database: `<POSTGRES_DB>`
  - Username: `<POSTGRES_USER>`
  - Password: `<POSTGRES_PASSWORD>`

- **DBeaver / pgAdmin**：同样使用上述参数连接

**联动效果：**

- 本地直接查看/修改 UAT 数据库中的数据
- 调试时快速验证后端写入是否正确
- 可手动插入测试数据辅助前端开发

### 方案 4：Docker 容器与日志监控

在 VS Code（Remote-SSH 连接到 UAT 后）：

1. 安装扩展 `Docker`（Microsoft 官方）
2. 左侧边栏会出现 Docker 面板，可直接看到：
   - 所有运行中的容器状态
   - 实时日志流（右键容器 → `View Logs`）
   - 容器内文件系统
   - 一键重启/停止容器

---

## 五、日常运维命令

在 UAT 机器或 Remote-SSH 终端执行。

### 更新代码并重新部署

```bash
cd ~/full-stack-uat
git pull origin main

# 完全重建并重启
UAT_HOST_IP=192.168.x.x docker compose -f compose.uat.yml up -d --build

# 如果只有后端代码变更，可只重建 backend
UAT_HOST_IP=192.168.x.x docker compose -f compose.uat.yml up -d --build backend prestart
```

### 查看实时日志

```bash
# 全部服务
docker compose -f compose.uat.yml logs -f

# 仅后端
docker compose -f compose.uat.yml logs -f backend

# 仅前端
docker compose -f compose.uat.yml logs -f frontend
```

### 进入容器内部排查

```bash
# 进入后端容器
docker compose -f compose.uat.yml exec backend bash

# 在容器内手动执行迁移
alembic upgrade head

# 退出
exit
```

### 数据库备份与恢复

```bash
# 备份
docker compose -f compose.uat.yml exec db pg_dump -U ${POSTGRES_USER} ${POSTGRES_DB} > uat-backup-$(date +%Y%m%d).sql

# 恢复
cat uat-backup-20240101.sql | docker compose -f compose.uat.yml exec -T db psql -U ${POSTGRES_USER} -d ${POSTGRES_DB}
```

### 停止/清理 UAT 环境

```bash
# 停止（保留数据）
docker compose -f compose.uat.yml down

# 彻底清理（包括数据库卷！谨慎使用）
docker compose -f compose.uat.yml down -v
```

---

## 六、可选：标准 Traefik 部署（更接近生产）

如果你的局域网内有 DNS 服务器（或愿意在每台开发机上改 `hosts` 文件），可以使用原版生产配置：

```bash
# 1. 在 Ubuntu 上创建网络
docker network create traefik-public

# 2. 启动 Traefik
docker compose -f compose.traefik.yml up -d

# 3. 启动生产栈
docker compose -f compose.yml up -d --build
```

然后在各开发机的 `hosts` 文件中加入：

```
192.168.x.x   dashboard.uat.local
192.168.x.x   api.uat.local
192.168.x.x   adminer.uat.local
192.168.x.x   traefik.uat.local
```

同时 `.env` 中设置 `DOMAIN=uat.local`。

> 此方案更复杂，建议仅在你需要模拟生产域名路由和 HTTPS 重定向行为时使用。日常 UAT 调试用上面的 `compose.uat.yml` 即可。

---

## 七、新增文件说明

本次 UAT 部署方案新增/涉及以下文件：

| 文件 | 说明 |
|------|------|
| `compose.uat.yml` | 局域网 UAT 专用部署配置（直接端口映射、内置 Mailcatcher、暴露 PostgreSQL） |
| `uat-deployment.md` | 本指南 |

---

## 八、常见问题排查

### 前端构建后无法连接后端

检查 `UAT_HOST_IP` 是否正确传入构建参数：

```bash
# 确认 frontend 容器内的 API URL
docker compose -f compose.uat.yml exec frontend sh -c 'grep -r "VITE_API_URL" /usr/share/nginx/html || echo "Not found"'
```

### 本地前端调 UAT 后端报 CORS 错误

确认 `.env` 中的 `BACKEND_CORS_ORIGINS` 包含 `http://localhost:5173`，并且修改后已重新构建后端容器。

### 后端提示 SECRET_KEY 不能为 changethis

确认 `.env` 中 `ENVIRONMENT` 不是 `local`，且 `SECRET_KEY` / `POSTGRES_PASSWORD` / `FIRST_SUPERUSER_PASSWORD` 均已改为非默认值的强密码。

### 数据库连接不上

确认 Ubuntu 防火墙已开放 5432 端口，且 PostgreSQL 容器健康检查已通过：

```bash
docker compose -f compose.uat.yml ps db
```
