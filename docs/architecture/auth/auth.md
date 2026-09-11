# gin-vue-admin 认证与授权（auth）子系统

> 本文档基于 gin-vue-admin v2.9.2 源码（后端模块 `github.com/flipped-aurora/gin-vue-admin/server`）分析，覆盖登录认证、JWT 签发与校验、RBAC 授权、角色/菜单/API 权限管理以及 API Token 管理的核心能力与边界。

## 1. 概述

auth 子系统负责"你是谁"（认证 Authentication）与"你能做什么"（授权 Authorization）两件事，是 gin-vue-admin 私有路由组（`PrivateGroup`）的第一道防线。

整体采用双层中间件模型：

1. **认证层**：`middleware.JWTAuth()` 从请求头 `x-token`（或 Cookie）解析 JWT，校验签名、过期时间与黑名单，并在缓冲期内自动续签。
2. **授权层**：`middleware.CasbinHandler()` 从已注入的 claims 取出 `AuthorityId`，调用 Casbin Enforcer 执行 `sub=角色, obj=路径, act=方法` 的策略匹配。

两层中间件在 `server/initialize/router.go` 中统一挂载到 `PrivateGroup`，所有业务路由均受其保护；登录、健康检查、Swagger 等走 `PublicGroup` 不经过鉴权。

核心代码路径：

- 中间件：`server/middleware/jwt.go`、`server/middleware/casbin_rbac.go`
- JWT 与上下文工具：`server/utils/jwt.go`、`server/utils/claims.go`、`server/utils/casbin_util.go`
- Service：`server/service/system/sys_casbin.go`、`sys_authority.go`、`sys_authority_btn.go`、`sys_base_menu.go`、`sys_menu.go`、`sys_api.go`、`sys_api_token.go`、`jwt_black_list.go`
- API：`server/api/v1/system/sys_casbin.go`、`sys_authority.go`、`sys_authority_btn.go`、`sys_menu.go`、`sys_api.go`、`sys_api_token.go`、`sys_jwt_blacklist.go`
- Model：`server/model/system/sys_authority.go`、`sys_authority_btn.go`、`sys_authority_menu.go`、`sys_base_menu.go`、`sys_menu_btn.go`、`sys_api.go`、`sys_api_token.go`、`sys_jwt_blacklist.go`、`sys_user_authority.go`
- 路由：`server/router/system/sys_casbin.go`、`sys_authority.go`、`sys_authority_btn.go`、`sys_jwt.go`、`sys_menu.go`、`sys_api.go`
- 配置：`server/config/jwt.go`；初始化：`server/initialize/other.go`（BlackCache）、`server/initialize/ensure_tables.go`（CasbinRule 表迁移）

## 2. 功能清单

| 功能 | 说明 | 源码路径 |
|------|------|----------|
| 用户名密码登录 | 校验验证码/密码（bcrypt），签发 JWT，记录登录日志 | `server/api/v1/system/sys_user.go`（`Login`/`TokenNext`）、`server/service/system/sys_user.go`（`Login`） |
| JWT 签发与解析 | HS256 签名，claims 含用户 ID/UUID/昵称/角色，支持缓冲期续签 | `server/utils/jwt.go`、`server/utils/claims.go` |
| JWT 认证中间件 | 取 token、查黑名单、ParseToken、注入 claims、临期自动续签并下发 `new-token` 响应头 | `server/middleware/jwt.go` |
| JWT 黑名单 | 登出/异地登录时将 token 写入 DB + 本地 BlackCache，中间件即时拦截 | `server/service/system/jwt_black_list.go`、`server/api/v1/system/sys_jwt_blacklist.go` |
| 单点登录 | `UseMultipoint` 开启时，Redis 记录用户名最新 token，新登录使旧 token 入黑名单 | `server/api/v1/system/sys_user.go`（`TokenNext`）、`server/utils/jwt.go`（`SetRedisJWT`） |
| Casbin RBAC 鉴权 | 中间件按 `(角色, 路径, 方法)` 执行 Enforce，策略持久化在 casbin_rule 表 | `server/middleware/casbin_rbac.go`、`server/utils/casbin_util.go` |
| 角色 API 权限管理 | 为角色批量增删改 API 策略，支持严格树形权限校验与 API 去重 | `server/service/system/sys_casbin.go`、`server/api/v1/system/sys_casbin.go` |
| 角色管理 | 创建/复制/更新/删除角色，创建时绑定默认菜单与默认 Casbin 策略，删除时级联清理 | `server/service/system/sys_authority.go` |
| 角色-用户绑定 | 全量覆盖角色用户列表，主角色失效时自动切换 | `server/service/system/sys_authority.go`（`SetRoleUsers`） |
| 角色-菜单绑定 | 设置角色可见菜单、数据权限范围（父子角色树） | `server/service/system/sys_authority.go`（`SetMenuAuthority`/`SetDataAuthority`） |
| 按钮级权限 | 角色按钮权限配置与 API 关联维护 | `server/service/system/sys_authority_btn.go`、`server/model/system/sys_authority_btn.go` |
| API 元数据管理 | 注册/维护后端 API 元数据，作为 Casbin 策略与前端动态路由的数据源 | `server/service/system/sys_api.go`、`server/router/system/sys_api.go` |
| API Token（长期 Token） | 为指定用户+角色签发长期 JWT（按天或永久），支持列表查询与作废（入黑名单） | `server/service/system/sys_api_token.go`、`server/api/v1/system/sys_api_token.go` |
| JWT 配置 | 签名密钥、过期时间、缓冲时间、签发者可配置 | `server/config/jwt.go` |
| 种子数据 | 初始化默认角色、菜单、API、Casbin 策略 | `server/source/system/authority.go`、`casbin.go`、`menu.go`、`api.go`、`authorities_menus.go` |

## 3. 解决的问题

| 用户痛点 | 解法 |
|----------|------|
| 前后端分离后无法用 Session 维持登录态 | 无状态 JWT：登录签发 token，后续请求带 `x-token`，服务端不存会话 |
| token 无法主动失效（登出/异地登录） | JWT 黑名单：登出时写 DB + 本地 BlackCache，中间件逐请求拦截；重启时 `LoadAll` 从 DB 恢复 |
| 同一账号多处登录互相踢下线 | `UseMultipoint` 模式下用 Redis 存用户名最新 token，新登录自动将旧 token 拉黑 |
| 细粒度 API 授权需要灵活可配 | Casbin RBAC：`(role, path, method)` 三元策略，gorm-adapter 持久化，带缓存（1h） |
| 多级角色树需要权限收敛 | `UseStrictAuth` 严格模式：创建/修改角色时校验目标角色必须在当前管理员的子树内 |
| token 临期导致用户操作中断 | 缓冲期机制：剩余时间小于 BufferTime 时自动签发新 token，通过 `new-token`/`new-expires-at` 响应头下发 |
| 第三方系统对接不想走浏览器登录 | API Token：按用户+角色签发长期 JWT，可随时作废并立即入黑名单 |
| 按钮级操作也要按角色控制 | 角色-按钮关联表（`sys_authority_btns`）+ 前端按权限渲染 |

## 4. 系统边界

**In-Scope（本仓库实现）**

- JWT 的签发、解析、续签、黑名单、单点登录逻辑
- Casbin Enforcer 的初始化（字符串模型 + gorm-adapter）、策略增删改查与刷新
- 角色（Authority）、角色树、角色-菜单、角色-用户、角色-按钮、API 元数据的 CRUD
- API Token 的签发、列表、作废
- 认证与授权两个 Gin 中间件及其在路由组上的统一挂载

**Out-of-Scope（外部系统 / 第三方组件，不在本仓库源码内）**

- 关系型数据库（MySQL / PostgreSQL / SQLServer / Oracle / SQLite）：通过 GORM 访问，存储用户、角色、菜单、casbin_rule、jwt_blacklists 等表
- Redis：用于单点登录最新 token 记录（`global.GVA_REDIS`）
- Casbin（`github.com/casbin/casbin/v3`）与 gorm-adapter：策略引擎与持久化适配器，第三方库
- golang-jwt/v5：JWT 签发与解析库，第三方组件
- 本地缓存 `github.com/songzhibin97/gkit/cache/local_cache`：JWT 黑名单进程内缓存，第三方组件
- 验证码（github.com/mojocn/base64Captcha）：登录验证码，第三方组件
- 前端 Vue3 + Element Plus：token 的存储、请求头携带与权限按钮渲染，在 `web/` 目录（独立前端工程）

## 5. 架构图

![auth 子系统架构图](auth-architecture.html)

图中主链路为横向流水线：前端浏览器 → JWT 鉴权中间件 → Casbin 鉴权中间件 → API/Service 层。下方三个支撑组件分别承担：JWT 黑名单本地缓存（JWTAuth 中间件逐请求查询）、Casbin Enforcer（CasbinHandler 调用 `Enforce`，策略来自数据库 casbin_rule 表）、业务数据库（GORM 读写用户/角色/策略）。右侧 Redis 在 `UseMultipoint` 开启时记录单点登录状态。两个虚线框分别表示 Gin 私有路由组边界与鉴权组件集合。

## 6. 时序图

![登录签发与请求鉴权时序](auth-sequence.html)

时序图分为两个阶段：

- **阶段一：登录与 JWT 签发**。前端提交用户名/密码/验证码，BaseApi 调用 UserService 查询用户并做 bcrypt 校验，随后由 JWT 工具构造 claims 并签发 token；开启单点登录时写入 Redis，最终通过 Set-Cookie 与响应体返回 token。
- **阶段二：携带 Token 访问受保护接口**。前端请求带 `x-token`，JWTAuth 中间件解析并注入 claims 后放行；CasbinHandler 调用 Enforcer 按角色/路径/方法匹配策略，allow 则进入业务，deny 则返回 403 权限不足。

## 7. 备注

- 中间件中"已登录用户被禁用则使 JWT 失效"的逻辑默认注释关闭（每次请求查库开销较大），如需启用可参考 `server/middleware/jwt.go` 中的注释块。
- Casbin 模型为字符串内嵌定义（`server/utils/casbin_util.go`），匹配器使用 `keyMatch2` 支持路径参数占位。
