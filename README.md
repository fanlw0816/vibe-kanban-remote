# Vibe Kanban Remote - Self-hosted Deployment

自托管部署 Vibe Kanban Remote 服务。

## 架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Vibe Kanban Remote                      │
├─────────────────────────────────────────────────────────────┤
│  remote-server (Axum, port 8081)                             │
│    ├── /v1/*         REST API (CRUD + auth + webhooks)      │
│    ├── /shape/*      ElectricSQL proxy (实时同步)            │
│    └── /srv/static   React SPA 前端                          │
├─────────────────────────────────────────────────────────────┤
│  PostgreSQL (port 5432)                                      │
│    └── wal_level=logical (ElectricSQL 逻辑复制)             │
├─────────────────────────────────────────────────────────────┤
│  ElectricSQL (port 3000, 内部)                               │
│    └── Postgres → Electric → 客户端 HTTP shapes             │
└─────────────────────────────────────────────────────────────┘
```

## 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/fanlw/vibe-kanban-remote.git
cd vibe-kanban-remote
```

### 2. 配置环境变量

```bash
cp .env.example .env
```

编辑 `.env` 文件，设置以下必需变量：

```bash
# 生成 JWT Secret (至少 32 字节)
openssl rand -base64 32

# 设置必需变量
VIBEKANBAN_REMOTE_JWT_SECRET=<生成的secret>
PUBLIC_BASE_URL=http://localhost:8081  # 或你的公网地址

# 配置认证方式 (至少一种)
SELF_HOST_LOCAL_AUTH_EMAIL=admin@example.com
SELF_HOST_LOCAL_AUTH_PASSWORD=your-password
```

### 3. 启动服务

```bash
docker-compose up -d
```

### 4. 访问服务

打开浏览器访问 `http://localhost:8081`

## 配置说明

### 必需配置

| 变量 | 说明 |
|------|------|
| `VIBEKANBAN_REMOTE_JWT_SECRET` | JWT 密钥，Base64 编码，至少 32 字节 |
| `PUBLIC_BASE_URL` | 服务公网地址，用于 OAuth 回调 |

### 认证配置 (至少配置一种)

**方式 1: 本地认证 (简单)**
```bash
SELF_HOST_LOCAL_AUTH_EMAIL=admin@example.com
SELF_HOST_LOCAL_AUTH_PASSWORD=admin123
```

**方式 2: GitHub OAuth**
```bash
GITHUB_OAUTH_CLIENT_ID=<your-client-id>
GITHUB_OAUTH_CLIENT_SECRET=<your-client-secret>
```

**方式 3: Google OAuth**
```bash
GOOGLE_OAUTH_CLIENT_ID=<your-client-id>
GOOGLE_OAUTH_CLIENT_SECRET=<your-client-secret>
```

**方式 4: GitLab OAuth**
```bash
GITLAB_OAUTH_CLIENT_ID=<your-client-id>
GITLAB_OAUTH_CLIENT_SECRET=<your-client-secret>
GITLAB_HOST=https://gitlab.com  # 或私有 GitLab 地址，如 https://jihulab.com
```

### 可选配置

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `IMAGE_TAG` | Docker 镜像版本 | `v0.1.44` |
| `SERVER_PORT` | 服务端口 | `8081` |
| `POSTGRES_PORT` | Postgres 外部端口 | `5433` |
| `POSTGRES_PASSWORD` | Postgres 密码 | `remote` |
| `RUST_LOG` | 日志级别 | `info,remote=info` |

### 附件存储 (可选)

**本地 Azurite (开发测试)**
```bash
docker-compose --profile attachments up -d
```

**Azure Blob Storage (生产)**
```bash
AZURE_STORAGE_ACCOUNT_NAME=<account-name>
AZURE_STORAGE_ACCOUNT_KEY=<account-key>
AZURE_STORAGE_CONTAINER_NAME=issue-attachments
```

**Cloudflare R2 (生产)**
```bash
R2_ACCESS_KEY_ID=<access-key>
R2_SECRET_ACCESS_KEY=<secret-key>
R2_REVIEW_ENDPOINT=<endpoint>
R2_REVIEW_BUCKET=<bucket>
```

## 常用命令

```bash
# 启动服务
docker-compose up -d

# 停止服务
docker-compose down

# 查看日志
docker-compose logs -f

# 查看特定服务日志
docker-compose logs -f remote-server
docker-compose logs -f electric

# 重启服务
docker-compose restart

# 查看服务状态
docker-compose ps

# 停止并清除数据
docker-compose down -v
```

## 更新版本

```bash
# 编辑 .env 文件，修改 IMAGE_TAG
IMAGE_TAG=v0.1.44

# 重新部署
docker-compose pull remote-server
docker-compose up -d
```

## 使用 Vibe Kanban 客户端连接 Remote Server

部署 Remote Server 后，可以通过以下方式连接：

### 方式 1: NPX 启动 (推荐)

```bash
# 设置 Remote Server 地址
export VK_SHARED_API_BASE=http://localhost:8081

# 启动 Vibe Kanban
npx vibe-kanban
```

### 方式 2: 桌面应用

1. 下载桌面应用: https://github.com/vibe-kanban/vibe-kanban/releases
2. 启动时设置环境变量:

**macOS/Linux:**
```bash
VK_SHARED_API_BASE=http://localhost:8081 ./vibe-kanban
```

**Windows (PowerShell):**
```powershell
$env:VK_SHARED_API_BASE="http://localhost:8081"
.\vibe-kanban.exe
```

**Windows (CMD):**
```cmd
set VK_SHARED_API_BASE=http://localhost:8081
vibe-kanban.exe
```

### 方式 3: 直接访问 Web 界面

Remote Server 本身包含 Web 前端，可直接访问：

```
http://localhost:8081
```

### 生产环境配置

将 `localhost:8081` 替换为你的公网地址：

```bash
# NPX
export VK_SHARED_API_BASE=https://kanban.your-domain.com

# 桌面应用
VK_SHARED_API_BASE=https://kanban.your-domain.com ./vibe-kanban
```

> **注意**: 确保 `PUBLIC_BASE_URL` 在 `.env` 中正确配置，OAuth 回调需要使用此地址。

## 数据持久化

数据存储在 `./data/` 目录：

```
./data/
├── postgres/    # PostgreSQL 数据
├── electric/    # ElectricSQL 数据
└── azurite/     # Azurite 数据 (可选)
```

## 故障排查

### Electric sync timeout

首次启动时 ElectricSQL 需要初始化，可能出现短暂超时。等待几秒后刷新页面即可。

### 服务无法启动

```bash
# 检查日志
docker-compose logs remote-server

# 检查数据库连接
docker-compose logs postgres
```

### 数据库迁移问题

```bash
# 重置数据库
docker-compose down -v
docker-compose up -d
```

## Docker Hub

镜像地址: `fanlw0816/vibe-kanban-remote`

可用版本: `v0.1.44`, `v0.1.43`, `latest`

## 许可证

参见上游项目 [vibe-kanban](https://github.com/vibe-kanban/vibe-kanban)