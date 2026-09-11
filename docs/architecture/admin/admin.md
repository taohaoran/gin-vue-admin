# gin-vue-admin 系统管理（admin）子系统

> 本文档基于 gin-vue-admin v2.9.2 源码分析，覆盖系统管理域的核心能力与边界。

## 1. 概述

系统管理（admin）子系统是 gin-vue-admin 后端 `server/` 下 `system` 域的核心管理模块，负责用户、字典、参数、日志、系统配置、版本管理、导出模板与数据库初始化等平台级基础能力。后端遵循 Router → API → Service → Model 四层分层架构，通过 `enter.go` 完成分组注册与组合；Service 层不依赖 `gin.Context`，统一响应 `{code, data, msg}`，分页响应 `{page, pageSize, total, list}`。

核心代码路径：

- 路由注册：`server/router/system/`（`enter.go` 组合各子路由）
- API 入口：`server/api/v1/system/`（`enter.go` 组合各 API，持 Service 句柄）
- 业务逻辑：`server/service/system/`（`enter.go` 组合各 Service）
- 数据模型：`server/model/system/`（含 `request/`、`response/` 子目录）
- 中间件：`server/middleware/operation.go`、`server/middleware/error.go`
- 初始化数据：`server/source/system/`（用户、菜单、字典、权限等种子数据）

## 2. 功能清单

| 功能 | 说明 | 源码路径 |
|------|------|----------|
| 用户注册与登录 | 用户名唯一性校验、Bcrypt 密码加密、JWT 签发、默认路由绑定 | `server/service/system/sys_user.go`、`server/api/v1/system/sys_user.go`、`server/model/system/sys_user.go` |
| 用户 CRUD 与冻结 | 用户增删改查、分页搜索、密码修改、启用/冻结切换、多角色关联 | `server/service/system/sys_user.go`、`server/api/v1/system/sys_user.go` |
| 字典管理 | 字典主表 CRUD、type 唯一性校验、级联删除字典详情 | `server/service/system/sys_dictionary.go`、`server/api/v1/system/sys_dictionary.go`、`server/model/system/sys_dictionary.go` |
| 字典详情管理 | 字典项明细 CRUD、按字典 ID 关联查询 | `server/service/system/sys_dictionary_detail.go`、`server/api/v1/system/sys_dictionary_detail.go`、`server/model/system/sys_dictionary_detail.go` |
| 参数配置管理 | 系统参数键值对 CRUD、按名称模糊搜索、时间范围过滤 | `server/service/system/sys_params.go`、`server/api/v1/system/sys_params.go`、`server/model/system/sys_params.go` |
| 操作日志记录 | 中间件自动记录请求 IP/方法/路径/Body/响应/延迟/错误，分页查询与批量删除 | `server/middleware/operation.go`、`server/service/system/sys_operation_record.go`、`server/api/v1/system/sys_operation_record.go`、`server/model/system/sys_operation_record.go` |
| 登录日志 | 登录成功/失败记录、按用户名模糊搜索、按状态过滤、分页查询与删除 | `server/service/system/sys_login_log.go`、`server/api/v1/system/sys_login_log.go`、`server/model/system/sys_login_log.go` |
| 系统信息监控 | 通过 gopsutil 采集 OS/CPU/内存/磁盘信息，前端可视化展示 | `server/service/system/sys_system.go`、`server/api/v1/system/sys_system.go`、`server/model/system/sys_system.go`、`server/utils/server.go` |
| 系统配置读写 | 读取 `global.GVA_CONFIG`，通过 Viper 动态写入并持久化配置文件 | `server/service/system/sys_system.go`、`server/api/v1/system/sys_system.go` |
| 版本管理 | 版本记录 CRUD、菜单/API/字典数据的导入导出（事务递归创建） | `server/service/system/sys_version.go`、`server/api/v1/system/sys_version.go`、`server/model/system/sys_version.go` |
| 错误日志 | Panic 自动捕获并落库、错误日志 CRUD、处理状态跟踪（未处理/处理中/处理完成） | `server/middleware/error.go`、`server/service/system/sys_error.go`、`server/api/v1/system/sys_error.go`、`server/model/system/sys_error.go` |
| 导出模板 | Excel 导出模板配置（表名、SQL、条件、关联表）、基于 excelize 渲染导出 | `server/service/system/sys_export_template.go`、`server/api/v1/system/sys_export_template.go`、`server/model/system/sys_export_template.go`、`server/config/excel.go` |
| 数据库初始化 | 多数据库适配（MySQL/PostgreSQL/SQLServer/SQLite）、子初始化器注册排序、自动建表与种子数据写入 | `server/service/system/sys_initdb.go`、`server/service/system/sys_initdb_mysql.go`、`server/service/system/sys_initdb_pgsql.go`、`server/service/system/sys_initdb_mssql.go`、`server/service/system/sys_initdb_sqlite.go`、`server/api/v1/system/sys_initdb.go`、`server/source/system/` |

## 3. 解决的问题

| 用户痛点 | 解法 |
|----------|------|
| 后台管理系统需要一套开箱即用的用户/权限/日志底座 | 内置用户、角色、字典、参数、操作日志、登录日志等完整 CRUD，启动即有可运行的管理后台 |
| 多数据库部署需求 | 初始化层抽象 `TypedDBInitHandler`，按 MySQL/PostgreSQL/SQLServer/SQLite 分别实现建库与建表，`server/source/system/` 提供统一种子数据 |
| 接口调用过程缺乏审计 | `OperationRecord()` 中间件拦截非 GET 请求，自动记录请求体/响应/延迟/错误信息，支持文件上传 Body 截断保护 |
| 线上 Panic 难以定位 | `GinRecovery` 中间件 recover panic 后自动将错误来源/堆栈写入 `sys_error` 表，支持前端跟踪处理状态 |
| 服务器运行状态不可见 | 封装 gopsutil 采集 OS/CPU/内存/磁盘指标，API 直接返回结构化 JSON |
| Excel 导出需求分散 | 导出模板系统将表名、SQL、查询条件、关联表抽象为可配置模板，基于 excelize 统一渲染 |
| 配置需运行时调整 | 通过 Viper 实现配置热读取与持久化写入，无需重启服务 |
| 多环境数据迁移 | 版本管理模块支持菜单/API/字典数据的导出与事务化导入，递归重建菜单树 |

## 4. 系统边界

### In-Scope（本仓库实现）

- 用户、字典、字典详情、参数、操作日志、登录日志、错误日志、版本管理、导出模板、数据库初始化等全部后端业务逻辑
- 操作日志中间件（请求 Body 读取、响应体捕获、延迟计算、记录落库）
- Panic 恢复中间件（错误落库）
- 系统信息采集（gopsutil 封装）
- 配置读写（Viper 封装）
- 种子数据（用户、菜单、权限、字典、API 白名单等）
- 前端管理页面（`web/src/view/superAdmin/` 下用户管理、字典、参数、日志等页面）

### Out-of-Scope（外部系统/第三方组件，不在本仓库源码内）

- **数据库**：MySQL / PostgreSQL / SQLServer / Oracle / SQLite（通过 GORM 驱动连接，数据库服务本身不在本仓库源码内）
- **Redis**：缓存/黑名单存储（第三方组件，不在本仓库源码内）
- **MongoDB**：qmgo 驱动（第三方组件，不在本仓库源码内）
- **gopsutil**：服务器指标采集库（第三方组件，不在本仓库源码内）
- **Viper**：配置管理库（第三方组件，不在本仓库源码内）
- **Bcrypt**：密码哈希（golang.org/x/crypto，第三方组件，不在本仓库源码内）
- **JWT**：令牌签发与校验（github.com/golang-jwt，第三方组件，不在本仓库源码内）
- **excelize**：Excel 文件生成（第三方组件，不在本仓库源码内）
- **Zap**：结构化日志（第三方组件，不在本仓库源码内）

## 5. 架构图

![系统管理子系统架构图](admin-architecture.html)

上图展示 admin 子系统的分层架构：前端管理页面通过 HTTP 请求到达 Gin 路由层，路由层挂载 JWT 鉴权与操作记录中间件后分发到 API 层；API 层调用 Service 层完成业务逻辑，Service 层通过 GORM 操作数据库；系统信息服务通过 gopsutil 采集宿主机指标；数据库初始化模块在启动期自动建表并写入种子数据。

## 6. 时序图

![操作日志记录时序图](admin-sequence.html)

上图展示一次带操作日志记录的写请求生命周期：请求进入 OperationRecord 中间件后先读取并缓存请求 Body，构造操作记录对象，包装 ResponseWriter，再放行到业务处理；业务完成后计算延迟、捕获错误信息，最终将完整记录写入数据库。GET 请求不经过操作日志中间件（路由注册时仅对 POST/PUT/DELETE 挂载）。
