# gin-vue-admin 示例业务（example）子系统

> 本文档基于 gin-vue-admin v2.9.2 源码分析，覆盖示例业务模块的核心能力与边界。

## 1. 概述

示例业务（example）子系统是 gin-vue-admin 内置的演示模块，位于后端 `server/` 下 `example` 域与前端 `web/src/view/example/` 下，用于展示如何基于 GVA 的分层架构（Router → API → Service → Model）开发一个完整的业务 CRUD 模块，同时演示文件上传下载、附件分类、断点续传等能力的前端集成方式，以及 @form-create 表单设计器的集成用法。

本单元聚焦：

- **后端完整链路**：客户管理（exa_customer）的 Router → API → Service → Model → GORM 全链路，含数据权限过滤
- **前端示例页**：客户管理页面、上传页面、断点续传页面
- **前端 API 封装**：customer、attachmentCategory、breakpoint、fileUploadAndDownload 等请求模块
- **@form-create 集成**：表单设计器的引入与 Vue 模板导出

文件上传下载、附件分类、断点续传的后端核心逻辑由 storage 单元负责，本单元仅描述前端集成方式与路由挂载点。

核心代码路径：

- 后端路由：`server/router/example/`（`enter.go` 组合各子路由）
- 后端 API：`server/api/v1/example/`（`enter.go` 组合各 API）
- 后端 Service：`server/service/example/`（`enter.go` 组合各 Service）
- 后端 Model：`server/model/example/`（含 `request/`、`response/` 子目录）
- 前端页面：`web/src/view/example/`
- 前端 API：`web/src/api/customer.js`、`web/src/api/attachmentCategory.js`、`web/src/api/breakpoint.js`、`web/src/api/fileUploadAndDownload.js`
- 表单设计器：`web/src/view/systemTools/formCreate/index.vue`

## 2. 功能清单

| 功能 | 说明 | 源码路径 |
|------|------|----------|
| 客户创建 | 绑定 JSON 请求体，校验客户名/手机号，自动注入当前登录用户 ID 与角色 ID | `server/api/v1/example/exa_customer.go`、`server/service/example/exa_customer.go`、`server/model/example/exa_customer.go` |
| 客户更新 | 校验客户 ID，全字段 Save 更新 | `server/api/v1/example/exa_customer.go`、`server/service/example/exa_customer.go` |
| 客户删除 | 按 ID 删除客户记录 | `server/api/v1/example/exa_customer.go`、`server/service/example/exa_customer.go` |
| 客户详情查询 | 按 ID 获取单个客户，关联返回管理用户信息 | `server/api/v1/example/exa_customer.go`、`server/service/example/exa_customer.go` |
| 客户分页列表（数据权限） | 根据当前用户角色的数据权限范围，过滤可见客户列表，Preload 关联 SysUser | `server/service/example/exa_customer.go`、`server/api/v1/example/exa_customer.go` |
| 客户路由注册 | POST/PUT/DELETE 挂载 OperationRecord 中间件，GET 不记录；路由前缀 `/customer` | `server/router/example/exa_customer.go` |
| 前端客户管理页面 | Element Plus 表格 + 分页 + 抽屉表单，调用 `web/src/api/customer.js` | `web/src/view/example/customer/customer.vue`、`web/src/api/customer.js` |
| 前端上传示例页 | 文件上传与扫码上传两个演示页面 | `web/src/view/example/upload/upload.vue`、`web/src/view/example/upload/scanUpload.vue` |
| 前端断点续传示例页 | 分块上传与断点续传前端交互 | `web/src/view/example/breakpoint/breakpoint.vue` |
| 前端附件分类 API | 分类列表查询、添加/编辑、删除 | `web/src/api/attachmentCategory.js` |
| 前端断点续传 API | 文件查找、分块续传、合并完成、分块移除 | `web/src/api/breakpoint.js` |
| 前端文件上传下载 API | 文件分页列表、删除、编辑文件名、上传文件 | `web/src/api/fileUploadAndDownload.js` |
| @form-create 表单设计器集成 | 引入 `@form-create/designer`，拖拽生成表单规则，支持导出为 Vue 原生模板代码 | `web/src/view/systemTools/formCreate/index.vue` |

## 3. 解决的问题

| 用户痛点 | 解法 |
|----------|------|
| 新开发者不知道如何基于 GVA 写一个标准业务 CRUD | exa_customer 提供完整分层示例：路由注册 → API 参数绑定与校验 → Service GORM 操作 → Model 结构体定义 |
| 多角色下数据可见范围隔离 | Service 层查询时调用 `AuthorityServiceApp.GetAuthorityInfo` 获取数据权限 ID 列表，按 `sys_user_authority_id in ?` 过滤 |
| 文件上传能力需要演示 | 提供上传与断点续传前端示例页，配合 `web/src/api/breakpoint.js` 分块上传接口 |
| 表单页面开发效率低 | 集成 @form-create 设计器，拖拽生成表单规则并可导出为 Vue 模板代码 |
| 附件需要分类管理 | 提供附件分类前端 API 封装，支持分类增删查 |

## 4. 系统边界

### In-Scope（本仓库实现）

- 客户管理（exa_customer）后端完整链路：路由、API、Service、Model、数据权限过滤
- 前端客户管理页面（表格、分页、抽屉表单）
- 前端上传/断点续传/客户管理示例页面
- 前端 API 封装模块（customer、attachmentCategory、breakpoint、fileUploadAndDownload）
- @form-create 表单设计器集成页（Vue 模板导出逻辑）
- 示例模块路由注册与中间件挂载策略（写操作记录操作日志，读操作不记录）

### Out-of-Scope（外部系统/第三方组件，不在本仓库源码内）

- **数据库**：MySQL / PostgreSQL / SQLServer / SQLite（通过 GORM，数据库服务本身不在本仓库源码内）
- **@form-create / FcDesigner**：拖拽式表单设计器组件库（第三方 npm 包，不在本仓库源码内）
- **Element Plus**：前端 UI 组件库（第三方 npm 包，不在本仓库源码内）
- **七牛云 / 阿里云 / 腾讯云 / 华为 OBS / AWS S3 / MinIO / Cloudflare R2**：对象存储（文件上传下载的后端核心逻辑由 storage 单元负责，云服务本身不在本仓库源码内）
- **附件分类后端与断点续传后端**：`server/service/example/exa_attachment_category.go`、`server/service/example/exa_breakpoint_continue.go` 的后端核心由 storage 单元负责，本单元仅描述前端集成

## 5. 架构图

![示例业务子系统架构图](example-architecture.html)

上图展示 example 子系统的前后端分层：前端示例页面（客户管理、上传、断点续传）通过封装的 API 模块发起 HTTP 请求，经 Gin 路由层（JWT 鉴权 + 操作记录中间件）到达 example API 层；API 层绑定参数并校验后调用 Service 层；Service 层通过 GORM 操作数据库，客户列表查询时还会调用 system 域的 AuthorityService 做数据权限过滤。@form-create 设计器作为独立前端页面存在，不与后端 API 直接交互。

## 6. 时序图

![exa_customer CRUD 时序图](example-sequence.html)

上图展示客户列表查询的完整调用链：前端页面发起分页请求 → 前端 API 封装 → Gin 路由（GET 不经过 OperationRecord 中间件）→ API 层绑定 PageInfo → Service 层先调用 AuthorityService 获取当前角色数据权限 → GORM 按权限范围分页查询客户并 Preload 关联用户 → 统一分页响应返回前端。
