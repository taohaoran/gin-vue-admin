# gin-vue-admin 系统架构总览

> 基于 gin-vue-admin 源码（`github.com/flipped-aurora/gin-vue-admin/server`，Go 1.24.0，v2.9.2）深度分析产出。
> 全仓 412 个 Go 文件 / 约 4.6 万行代码，前端 Vue 3 + Vite + Pinia + Element Plus。
> 所有图表由 archify 渲染为自包含交互式 HTML（showcase 档，0 校验错误）。

## 1. 项目概述

gin-vue-admin 是一个基于 Vue 与 Gin 开发的全栈前后端分离开发基础平台，集成 JWT 鉴权、动态路由、动态菜单、Casbin 鉴权、表单生成器、代码生成器、插件体系与 MCP（Model Context Protocol）AI 集成能力，为快速搭建中小型后台管理系统提供一整套开箱即用的架构底座。

### 1.1 代码规模

| 维度 | 数值 | 说明 |
|------|------|------|
| Go 文件总数 | 412 | 不含第三方依赖 |
| Go 代码行数 | 46,013 | 实测 `find -name "*.go"` 汇总 |
| 后端分层 | Router → API → Service → Model | `server/` 下 api/v1、router、service、model 分 system / example 两域 |
| 前端结构 | Vue 3 + Vite + Pinia + Element Plus | `web/src/` 下 api、pinia、router、view、utils、plugin 等 |

### 1.2 后端目录分布（Go 文件数）

| 目录 | 文件数 | 职责 |
|------|--------|------|
| `server/model` | 69 | 持久化模型与请求/响应模型（system / example / common） |
| `server/utils` | 67 | 工具：jwt、claims、casbin、upload、autocode、ast、timer、verify 等 |
| `server/plugin` | 49 | 插件：announcement、email、auto、plugin-tool |
| `server/service` | 48 | 业务逻辑（system / example） |
| `server/api/v1` | 35 | HTTP 处理器（参数绑定、校验、响应） |
| `server/router` | 28 | 路由注册与中间件挂载 |
| `server/initialize` | 26 | 初始化：GORM 多库、Redis、Mongo、定时器、路由、插件、建表 |
| `server/mcp` | 26 | MCP 服务器与工具（api_creator、menu_creator、codegen 等） |
| `server/config` | 27 | 配置结构定义（jwt、redis、db、oss、mcp、auto_code 等） |
| 其余（core/global/middleware/source/task/cmd） | 41 | 启动、全局、中间件、初始化数据、定时任务、命令 |

## 2. 功能总览

| 领域 | 功能模块 | 核心代码路径 |
|------|----------|--------------|
| 认证与授权 | JWT 鉴权、Casbin RBAC、多点登录、API Token | `server/middleware/jwt.go`、`server/middleware/casbin_rbac.go`、`server/service/system/sys_casbin.go` |
| 用户与权限 | 用户、角色、菜单、API、按钮权限管理 | `server/api/v1/system/sys_user.go`、`server/api/v1/system/sys_authority.go`、`server/api/v1/system/sys_menu.go` |
| 动态路由/菜单 | 按角色动态下发菜单与路由 | `server/service/system/sys_base_menu.go`、前端 `web/src/permission.js` |
| 系统管理 | 字典、参数、操作日志、登录日志、系统信息、版本、导出模板 | `server/api/v1/system/sys_dictionary.go`、`sys_params.go`、`sys_operation_record.go` 等 |
| 代码生成器 | 模板/包/插件代码生成、多数据库 SQL 生成、SSE、AI 工作流 | `server/api/v1/system/sys_auto_code*.go`、`server/service/system/auto_code_llm.go`、`server/utils/ast/` |
| 文件存储 | 多云对象存储上传下载、分片上传、断点续传 | `server/utils/upload/`、`server/api/v1/example/exa_file_upload_download.go` |
| 插件体系 | announcement（公告）、email（邮件）、auto（自动化）、插件框架 | `server/plugin/`、`server/initialize/plugin*.go` |
| MCP 集成 | MCP Streamable HTTP 服务，API/菜单/字典/代码生成工具 | `server/mcp/` |
| 定时任务 | 数据库表清理等定时任务 | `server/task/clearTable.go`、`server/initialize/timer.go` |
| 基础设施 | 配置（Viper）、日志（Zap）、多数据库（GORM）、Swagger | `server/core/`、`server/config/`、`server/initialize/gorm*.go` |

## 3. 解决的问题

| 用户痛点 | 解法 |
|----------|------|
| 后台系统重复开发成本高 | 提供全栈脚手架：鉴权、权限、菜单、代码生成一应俱全，专注业务开发 |
| 权限模型复杂、难以落地 | JWT 无状态鉴权 + Casbin RBAC，API/菜单/按钮三级权限，权限策略持久化于数据库 |
| 前后端协作契约不统一 | 统一响应结构 `{code, data, msg}` 与分页结构 `{page, pageSize, total, list}` |
| 不同角色菜单不同、需动态配置 | 动态路由/动态菜单：菜单随角色下发，前端按权限注册路由 |
| 重复 CRUD 开发耗时 | 代码生成器一键生成前后端基础代码，并支持 AI 工作流与插件化模板 |
| 文件存储厂商锁定 | 统一 `utils/upload` 接口抽象，适配七牛/阿里/腾讯/AWS/MinIO/华为 OBS/Cloudflare R2 等 |
| 多项目多环境配置管理困难 | Viper 配置中心 + 多环境配置文件 + 数据库多库支持（DBList） |
| AI 编辑器难以操作平台 | 内置 MCP 服务器，AI 可直接调用 API/菜单/字典/代码生成等工具 |

## 4. 系统边界

### 4.1 上边界（用户 / 第三方接入）

- 浏览器用户通过 HTTPS 访问 Vue3 前端（`web/`），前端经 REST 调用后端（携带 `x-token`）。
- AI 编辑器作为 MCP 客户端，通过 MCP Streamable HTTP 协议接入 `server/mcp` 工具。
- Swagger UI（`/swagger/*`）为开发者提供在线 API 文档。

### 4.2 下边界（基础设施，不在本仓库源码内）

| 组件 | 用途 | 标注 |
|------|------|------|
| MySQL / PostgreSQL / SQLServer / Oracle / SQLite | GORM 关系数据库（支持多库） | 外部系统/第三方组件，不在本仓库源码内 |
| Redis | JWT 黑名单、多点登录、缓存 | 外部系统/第三方组件，不在本仓库源码内 |
| MongoDB（qmgo） | 可选文档存储 | 外部系统/第三方组件，不在本仓库源码内 |
| 七牛/阿里/腾讯/AWS S3/MinIO/华为 OBS/Cloudflare R2 | 文件对象存储 | 外部系统/第三方组件，不在本仓库源码内 |

### 4.3 内边界（本仓库实现）

- `server/`：Gin 后端（路由、API、Service、Model、中间件、初始化、插件、MCP、任务）。
- `web/`：Vue3 前端（页面、路由、状态、接口封装、工具、插件）。
- `deploy/`：Docker / Compose 等部署资产。
- `aiDoc/`：AI 协作文档层（关系、模块、示例、记忆）。

### 4.4 侧边界（扩展方式）

- 后端插件：`server/plugin/<name>/`；前端插件：`web/src/plugin/<name>/`，保持前后端对称结构。
- 业务模块按 `Router → API → Service → Model` 分层扩展，`enter.go` 作为分组注册入口。
- 新存储/新能力通过 `utils/` 适配层与 `config/` 配置扩展。

### 4.5 不做什么（Out-of-Scope）

- 不内置业务领域功能（CRM/ERP 等），提供的是底座与示例。
- 不包含 Kubernetes 编排、服务网格等平台级能力（`deploy/` 仅提供 Docker 部署资产）。
- 不提供独立的消息队列中间件，异步能力通过定时任务与插件机制承载。

## 5. 系统架构图说明

![系统架构图](system-architecture.html)

系统架构遵循"前端 → 网关/中间件 → 业务域 → 存储"的横向分层：

1. **接入层**：浏览器（Vue3 SPA）与 AI 编辑器（MCP 客户端）为两大外部入口；`gva-web` 承担动态路由、状态管理与接口封装。
2. **网关/服务层**：`gin-api`（Gin Engine + 中间件链：Recovery、JWT Auth、CasbinHandler、Operation 等）统一鉴权与权限校验；`mcp-server` 以 Streamable HTTP 暴露工具注册表，工具通过 HTTP 转发执行业务 API。
3. **业务域**：`system-domain`（系统业务域 api/service/model）为核心；`codegen`（代码生成器 + AST）与 `plugin-runtime`（插件体系）作为扩展侧，由 `gin-api` 按请求分发。
4. **存储层**：关系数据库（GORM 多库）承载业务与权限数据，Redis 承载缓存与 JWT 黑名单，MongoDB 为可选文档存储，对象存储承载文件（外部组件，不在本仓库源码内）。

后端平台整体位于 `server/` 区域边界内，保持 `Router → API → Service → Model` 的单向依赖。

## 6. 核心时序图说明

![登录与鉴权时序](system-auth-sequence.html)

时序图分为两个阶段：

- **阶段一「登录签发」**：用户提交账号密码 → `POST /base/login` → Service 查询用户并校验密码 →（启用多点登录时）写入 Redis 会话管理 → 签发 JWT 返回前端，前端保存 token。
- **阶段二「鉴权请求」**：前端携带 `x-token` 访问业务接口 → `middleware/JWTAuth` 校验 token 与 JWT 黑名单（Redis）→ `middleware/CasbinHandler` 通过 GORM Adapter 读取策略做 RBAC 校验 → 通过后进入业务处理并返回统一 JSON。

## 7. 系统数据流图说明

![系统数据流图](system-dataflow.html)

数据流按「来源 → 接入 → 处理 → 存储 → 消费」五阶段组织：

1. **来源**：浏览器（REST 请求）与 AI 编辑器（MCP 调用）。
2. **接入**：Gin API（JWT/Casbin 中间件网关）与 MCP 服务（工具执行经 HTTP 转发业务 API）。
3. **处理**：系统业务域（Router→API→Service→Model）处理主数据；代码生成器接收生成请求。
4. **存储**：业务数据 CRUD 落关系库；缓存/黑名单落 Redis；文件对象上传下载走对象存储。
5. **消费**：关系库查询结果以统一 JSON 返回前端页面渲染。

主请求路径保持 `浏览器 → Gin API → 系统业务域 → 关系库 → 前端页面` 的直线链路；AI 调用路径独立于主路径，通过 MCP 服务接入。

## 8. 核心代码映射

| 能力 | 代码路径 |
|------|----------|
| 启动流程 | `server/main.go` → `core/server.go`（`initializeSystem` → `RunServer`） |
| 路由注册 | `server/initialize/router.go`（PublicGroup / PrivateGroup + 插件安装） |
| 中间件链 | `server/middleware/`（jwt.go、casbin_rbac.go、operation.go、error.go 等） |
| 全局状态 | `server/global/global.go`（GVA_DB、GVA_REDIS、GVA_CONFIG、GVA_MCP_SERVER 等） |
| 初始化数据 | `server/source/system/`（菜单、API、角色、字典等种子数据） |
| 前端入口 | `web/src/main.js`、`web/src/permission.js`（路由守卫）、`web/src/pinia/modules/` |

## 覆盖范围与说明

- 系统级图表全部以 archify showcase 档校验通过并渲染成功（0 错误）。
- 子系统级分析文档位于 `docs/architecture/` 各分组目录（auth / admin / codegen / storage / extension / mcp / ops / example）。
- 本文件中的外部组件（数据库、缓存、对象存储、@Variant Form、mcp-go、cron、gopsutil 等）均为"不在本仓库源码内"的第三方依赖。
