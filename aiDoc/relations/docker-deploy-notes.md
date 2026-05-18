# Docker Compose 部署注意事项

> 基于 2026-05-19 `docker compose up -d` 全流程排查的实战记录。

## 修改清单

| # | 类型 | 文件 | 修改内容 | 原因 |
|---|------|------|----------|------|
| 1 | 告警 | `deploy/docker-compose/docker-compose.yaml:1` | `version: "3"` → 注释 | Docker Compose V2 不再需要 |
| 2 | 告警 | `deploy/docker-compose/docker-compose.yaml:45-48` | `links` 块 → 注释 | 已废弃，同网络容器可互访 |
| 3 | 告警 | `server/Dockerfile:1` | `as builder` → `AS builder` | BuildKit 要求 FROM/AS 大小写一致 |
| 4 | 告警 | `server/Dockerfile:31` | shell 形式 → `ENTRYPOINT ["./server", "-c", "config.docker.yaml"]` | JSONArgsRecommended，shell 形式 OS 信号异常 |
| 5 | 错误 | `web/Dockerfile:5-6` | `node:20-slim` → `node:22-slim` | pnpm 11.x 依赖 `node:sqlite`，Node 20 不支持 |
| 6 | 错误 | `web/Dockerfile:16` | `pnpm install --prod` → `pnpm install --prod --ignore-scripts` | 解决 `ERR_PNPM_IGNORED_BUILDS` |
| 7 | 错误 | `web/Dockerfile:21` | `pnpm install` → `pnpm install --ignore-scripts` | build 阶段同样需要跳过构建脚本 |
| 8 | 配置 | `web/package.json` | 添加 `"pnpm": {"onlyBuiltDependencies": []}` | 显式声明允许的构建依赖（pnpm 11.x 已不读取此字段，保留供旧版兼容） |
| 9 | 代理 | `deploy/docker-compose/.env.proxy` | 新建，`HTTP_PROXY=http://127.0.0.1:7897` | BuildKit 不走 Docker Desktop 内部代理，需显式注入 |
| 10 | 工具 | `deploy/docker-compose/dc` | 新建包装脚本，自动加载 `.env.proxy` 后执行 `docker compose` | 避免每次手动设置环境变量 |

## 前置条件

### 网络环境

国内网络无法直接访问 Docker Hub（`auth.docker.io`、`registry-1.docker.io`），需确保以下之一：

- **VPN/代理可用**：Docker Desktop → Settings → Resources → Proxies 配置 HTTP/HTTPS 代理（本项目开发机使用 `127.0.0.1:7897`，由 Clash Verge 提供）
- **或预拉取镜像**：在 VPN 连通时先 `docker pull` 所有基础镜像，再执行 `docker compose up -d`

涉及的基础镜像：

| 镜像 | 用途 |
|------|------|
| `golang:alpine` | server 构建阶段 |
| `alpine:latest` | server 运行阶段 |
| `node:22-slim` | web 构建阶段 |
| `nginx:alpine` | web 运行阶段 |
| `mysql/mysql-server:8.0.21` | MySQL 数据库 |
| `redis:6.0.6` | Redis 缓存 |

注意：BuildKit builder 使用 `network: host` 模式，**不走 Docker Desktop 内部代理**（`http.docker.internal:3128`）。以下对比说明：

| 操作 | 代理路径 | 结果 |
|------|----------|------|
| `docker pull` | daemon → `http.docker.internal:3128` → VPN | ✅ |
| `docker build` (BuildKit FROM) | BuildKit containerd → 直连 Docker Hub | ❌ 超时 |
| `HTTP_PROXY=http://127.0.0.1:7897 docker build` | BuildKit → `127.0.0.1:7897` (VPN) | ✅ |

**永久解决方案**：使用项目提供的 `dc` 包装脚本（自动注入代理），或手动设置：

```bash
# 方式1: 包装脚本（推荐）
cd deploy/docker-compose
./dc up -d
./dc build

# 方式2: 手动设置环境变量
HTTP_PROXY=http://127.0.0.1:7897 HTTPS_PROXY=http://127.0.0.1:7897 docker compose up -d

# 方式3: source .env.proxy 后执行
source .env.proxy && docker compose up -d
```

## docker-compose.yaml 告警

### `version` 已废弃

Docker Compose V2 不再需要 `version` 字段，应删除：

```yaml
# 删除这行
version: "3"
```

### `links` 已废弃

同一 `networks` 下的容器已可通过容器名互相访问，`links` 多余：

```yaml
# 删除或注释 links 块
# links:
#   - mysql
#   - redis
```

## Dockerfile 告警与错误

### `FROM` 与 `as` 大小写

BuildKit 要求 `FROM ... AS ...` 大小写一致：

```dockerfile
# 错误
FROM golang:alpine as builder

# 正确
FROM golang:alpine AS builder
```

### `ENTRYPOINT` 应使用 JSON 数组格式

shell 形式会导致 OS 信号无法正确传递：

```dockerfile
# 错误（shell 形式）
ENTRYPOINT ./server -c config.docker.yaml

# 正确（exec 形式）
ENTRYPOINT ["./server", "-c", "config.docker.yaml"]
```

关键约束：**`ENTRYPOINT` 行不能有行尾注释**，`#` 会破坏 JSON 数组解析，导致 Docker 回退为 shell 形式：

```dockerfile
# 正确：注释放在上一行
# 使用JSON数组格式，避免OS信号处理异常
ENTRYPOINT ["./server", "-c", "config.docker.yaml"]

# 错误：行尾注释破坏解析
ENTRYPOINT ["./server", "-c", "config.docker.yaml"] # 注释
```

同理，**`FROM` 行也不能有行尾注释**：

```dockerfile
# 正确：注释放在上一行
# 升级至Node.js 22 LTS，pnpm 11.x依赖node:sqlite内置模块
FROM node:22-slim AS base

# 错误：行尾注释会被当作第三个参数
FROM node:22-slim AS base # 注释
```

## Node.js / pnpm 兼容性

### Node.js 版本

`node:20-slim` 与 pnpm 11.x 不兼容。pnpm 11.x 依赖 `node:sqlite` 内置模块，该模块在 Node.js 22 才引入。必须使用 `node:22-slim` 或更高版本：

```dockerfile
# 升级至Node.js 22 LTS，pnpm 11.x依赖node:sqlite内置模块，Node 20不支持
FROM node:22-slim AS base
```

### pnpm 构建脚本策略

pnpm 11.x 默认拒绝未声明 `onlyBuiltDependencies` 的依赖执行构建脚本，抛出 `ERR_PNPM_IGNORED_BUILDS` 错误。

在 Docker build 场景下，最简单的修复是在 `pnpm install` 加 `--ignore-scripts`：

```dockerfile
# prod-deps 阶段
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --prod --ignore-scripts

# build 阶段（也需要加）
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --ignore-scripts && pnpm run build
```

注意：pnpm 11.x 已不再读取 `package.json` 中的 `"pnpm"` 字段（会输出 `WARN: The "pnpm" field in package.json is no longer read by pnpm`），`onlyBuiltDependencies` 需通过 `.npmrc` 或 pnpm workspace 配置，但在 Docker 场景下直接用 `--ignore-scripts` 更简单。

## 快速启动

```bash
# 1. 确保 VPN 已连接
# 2. 启动（使用 dc 包装脚本自动注入代理，解决 BuildKit 网络问题）
cd deploy/docker-compose
./dc up -d

# 3. 验证
./dc ps
# 期望：gva-mysql(healthy) gva-redis(healthy) gva-server(Up) gva-web(Up)
```

## 端口映射

| 服务 | 宿主机端口 | 容器端口 |
|------|-----------|---------|
| web | 8080 | 8080 |
| server | 8888 | 8888 |
| mysql | 13306 | 3306 |
| redis | 16379 | 6379 |
