# gin-vue-admin 插件体系（extension）子系统

> 本文档基于 gin-vue-admin 源码分析，覆盖插件机制、两个插件代际接口、三个示例插件与插件元数据自举安装流程。

## 1. 概述

gin-vue-admin（下文简称 GVA）的插件体系允许在不改动主框架业务代码的前提下，以独立目录（`server/plugin/<name>/`）挂载新业务模块，并在首次安装时自动写入 API 权限、前端菜单、字典与数据表。体系内同时存在两代插件接口：

- **v1 插件接口**（`server/utils/plugin/plugin.go`）：要求实现 `Register(*gin.RouterGroup)` 与 `RouterPath() string`，由主程序手动构造并挂载到带路由前缀的 `RouterGroup`，以邮件插件为唯一内置实例。
- **v2 插件接口**（`server/utils/plugin/v2/plugin.go`）：只要求实现 `Register(*gin.Engine)`，插件在自身 `init()` 中调用 `interfaces.Register(Plugin)` 自注册到全局注册表，主程序通过 blank import 触发 `init()` 后统一回调。

插件的发现、初始化与路由挂载由 `server/initialize/` 下四个文件协同完成；插件在首次安装时借助 `server/plugin/plugin-tool/utils` 将自身元数据幂等写入 GVA 数据库的 `sys_api` / `sys_base_menu` / `sys_dictionary` 表，从而与系统的 RBAC 权限体系打通。

核心源码路径：

- 插件注册与编排：`server/initialize/plugin.go`、`server/initialize/plugin_biz_v1.go`、`server/initialize/plugin_biz_v2.go`、`server/initialize/plugin_notifier.go`
- 两代插件接口：`server/utils/plugin/plugin.go`、`server/utils/plugin/v2/plugin.go`、`server/utils/plugin/v2/registry.go`
- blank import 聚合点：`server/plugin/register.go`
- 元数据安装工具：`server/plugin/plugin-tool/utils/check.go`
- 示例插件：`server/plugin/announcement/`、`server/plugin/email/`、`server/plugin/auto/`
- 路由挂接点：`server/initialize/router.go`（`InstallPlugin(PrivateGroup, PublicGroup, Router)`）

## 2. 功能清单

| 功能 | 说明 | 源码路径 |
|------|------|----------|
| v1 插件接口 | 定义 `Register(*gin.RouterGroup)` + `RouterPath() string` 两方法契约，插件以路由子路径挂载 | `server/utils/plugin/plugin.go` |
| v2 插件接口 | 定义 `Register(*gin.Engine)` 单方法契约，插件直接持有完整 Gin 引擎 | `server/utils/plugin/v2/plugin.go` |
| v2 全局注册表 | 读写互斥保护的 `[]Plugin` 切片，`Register`/`Registered` 供 init 自注册与启动期遍历 | `server/utils/plugin/v2/registry.go` |
| blank import 聚合 | 仅 blank import announcement、auto 两个包，触发其 `init()` 自注册 | `server/plugin/register.go` |
| 插件编排入口 | `InstallPlugin` 在 Initialize 阶段被调用，按 DB 是否已就绪选择立即或延迟注册 | `server/initialize/plugin.go` |
| v1 插件手动装载 | `bizPluginV1` 用 `CreateEmailPlug(...)` 构造邮件插件，按 `RouterPath()` 建子路由组后 `Register` | `server/initialize/plugin_biz_v1.go` |
| v2 插件批量装载 | `bizPluginV2` 调 `PluginInitV2(engine, plugin.Registered()...)` 遍历回调 | `server/initialize/plugin_biz_v2.go` |
| 数据库就绪通知器 | 单例 `DBReadyNotifier`，订阅/通知模式解耦"首次初始化建库"与"插件注册"时序 | `server/initialize/plugin_notifier.go` |
| API 权限幂等注册 | `RegisterApis` 事务内按 `path+method+api_group` FirstOrCreate 写入 `sys_api` | `server/plugin/plugin-tool/utils/check.go` |
| 菜单幂等注册 | `RegisterMenus` 首项作为父菜单，其余以其 ID 为 `ParentId` 建子菜单 | `server/plugin/plugin-tool/utils/check.go` |
| 字典幂等注册 | `RegisterDictionaries` 先建字典主表，再按 `sys_dictionary_id+value` 建详情 | `server/plugin/plugin-tool/utils/check.go` |
| 按插件名回溯元数据 | `ApiMap/MenuMap/DictMap` 记录每个插件注册的元数据，供 `GetPluginData` 查询 | `server/plugin/plugin-tool/utils/check.go` |
| 公告插件（v2 示例） | `init()` 自注册；`Register` 内依次 `Api/Menu/Dictionary/Gorm/Router` 完成自举 | `server/plugin/announcement/plugin.go`、`server/plugin/announcement/initialize/` |
| 邮件插件（v1 示例） | 通过 `CreateEmailPlug` 注入 SMTP 配置，挂载 `/email` 路由组 | `server/plugin/email/main.go`、`server/plugin/email/router/`、`server/plugin/email/service/` |
| 自动代码插件（v2） | 承载 AI 代码生成相关路由（AutoCode / History / Skills） | `server/plugin/auto/plugin.go`、`server/plugin/auto/initialize/router.go` |
| 插件私有路由鉴权 | 插件内 `private.Use(middleware.JWTAuth()).Use(middleware.CasbinHandler())` 复用主框架中间件 | `server/plugin/announcement/initialize/router.go`、`server/plugin/auto/initialize/router.go` |

## 3. 解决的问题

| 用户痛点 | 解法 |
|----------|------|
| 每次新增业务模块都要改主框架的路由、菜单、权限表，升级易冲突 | 插件以独立目录承载 api/service/model/router/initialize 全套，主框架仅保留两个接口与一个注册聚合点 |
| 插件需要同时提供前端菜单、按钮级 API 权限、字典与数据表 | 插件 `Register` 内调用 `initialize.Api/Menu/Dictionary/Gorm`，由 plugin-tool 统一幂等落库，开箱即有权限菜单 |
| 全新部署时数据库表尚未建好，插件启动期读库会失败 | `DBReadyNotifier` 把插件注册挂到 service 层 DB 就绪回调之后，首启与升级启动走同一套幂等流程 |
| 插件元数据重复安装会产生脏数据 | `FirstOrCreate` 按业务键（path+method、menu name、dict type）幂等写入，多次启动安全 |
| 不同插件对路由粒度诉求不同（子路由组 vs 完整引擎） | v1/v2 两代接口并存：简单 CRUD 用 v1 子路由组，需自定义中间件/公共路由的用 v2 完整引擎 |

## 4. 系统边界

**In-Scope（本仓库 `server/` 内实现）**：

- 两代插件接口定义与 v2 全局注册表（`server/utils/plugin/`）
- 插件编排、DB 就绪通知、v1/v2 装载流程（`server/initialize/plugin*.go`）
- 元数据幂等写入工具（`server/plugin/plugin-tool/utils/check.go`）
- 三个内置示例插件：announcement、email、auto
- 插件路由复用主框架的 JWT / Casbin 中间件

**Out-of-Scope（外部系统 / 第三方组件，不在本仓库源码内）**：

- 关系型数据库（MySQL / PostgreSQL / SQLServer / Oracle / SQLite，由 GORM 接入）——插件元数据与业务表最终落在这里
- Casbin 鉴权引擎（`middleware.CasbinHandler` 底层）——不在本仓库源码内
- JWT 签发与校验库——不在本仓库源码内
- 邮件插件实际依赖的外部 SMTP 服务器（七牛云/阿里云/腾讯云等邮件推送通道属部署侧配置）——不在本仓库源码内
- 插件前端页面（`web/src/` 下对应组件，如 `plugin/announcement/view/info.vue`）不在后端仓库分析范围内

## 5. 架构图

![插件体系架构图](extension-architecture.html)

图中按列从左到右展开：Gin 引擎在启动期调用 `InstallPlugin`；`InstallPlugin` 在数据库未就绪时把注册动作挂到 `DBReadyNotifier`，数据库就绪后再继续；v1 邮件插件由 `bizPluginV1` 手动构造并挂到 `/email` 子路由组；v2 插件通过 blank import 在 `init()` 自注册到全局注册表，`bizPluginV2` 遍历注册表回调各插件 `Register(engine)`；公告/自动代码插件在自身 `Register` 内调用 `plugin-tool` 把 API/菜单/字典幂等写入 GORM 数据库，并通过 `JWT+Casbin` 中间件保护私有路由。

## 6. 时序图

![插件加载时序图](extension-sequence.html)

时序图以"冷启动且数据库尚未初始化"这一最复杂场景为例：阶段一 `InstallPlugin` 发现 `GVA_DB==nil`，将 `PluginParams` 交给 `DBReadyNotifier` 订阅；阶段二 service 层完成建表后触发 `NotifyDBReady`；阶段三回调里先从 v2 注册表取出全部插件，再逐个 `Register(engine)`，插件内部完成元数据入库与路由挂载，最后由编排层重刷 `GVA_ROUTERS`。若数据库已存在，则跳过阶段一/二直接进入阶段三。
