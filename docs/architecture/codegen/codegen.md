# gin-vue-admin 代码生成器与 AI 工作流子系统（codegen）

> 本文档基于 gin-vue-admin v2.9.2 源码（后端模块 `github.com/flipped-aurora/gin-vue-admin/server`，Go 1.24.0）分析，覆盖代码生成器、AI 辅助生成（LLM 代理 / SSE 流式）、AI 工作流会话持久化、模板与插件管理、MCP 工具生成及生成历史回滚的核心能力与边界。

## 1. 概述

codegen 是 gin-vue-admin 的核心亮点子系统，定位为"低代码脚手架"：运营人员在前端页面选择数据库表、字段、包名与模板后，系统自动生成一整套符合 GVA 分层规范（Router → API → Service → Model）的 Go 后端代码与 Vue 前端代码，并通过 Go AST 解析器自动把新生成的分组注册进 `enter.go`、路由初始化、GORM 初始化等组合入口，实现"一键生成可运行业务模块"。

在此基础上，本子系统还封装了两类 AI 能力：

- **LLM 代理层**：把前端发起的大模型请求转发到插件市场配置的上游 AI 服务（`AiPath`），支持普通 JSON 与 SSE 流式两种模式，用于 AI 自动生成表结构 / 代码。
- **AI 工作流会话**：把"需求分析 / 工作流"两类对话会话（含消息、表单、结果、当前节点）按用户持久化到数据库，支持列表、详情、删除与 Markdown 落盘。

核心代码路径：

- API 层：`server/api/v1/system/sys_auto_code.go`、`sys_auto_code_sse.go`、`auto_code_package.go`、`auto_code_template.go`、`auto_code_plugin.go`、`auto_code_history.go`、`auto_code_mcp.go`、`ai_workflow_session.go`
- Service 层：`server/service/system/auto_code_package.go`（生成管线核心）、`sys_auto_code_interface.go`（Database 多方言接口）、`sys_auto_code_{mysql,pgsql,mssql,oracle,sqlite}.go`、`auto_code_template.go`、`auto_code_plugin.go`、`auto_code_history.go`、`auto_code_mcp.go`、`auto_code_llm.go`、`ai_workflow_session.go`、`ai_workflow_markdown.go`
- Model 层：`server/model/system/sys_auto_code_package.go`、`sys_auto_code_history.go`、`sys_ai_workflow_session.go`
- 工具层：`server/utils/autocode/template_funcs.go`（模板函数）、`server/utils/ast/`（Go AST 注入/回滚，32 个文件）
- 路由层：`server/router/system/sys_auto_code.go`、`sys_auto_code_history.go`
- 配置：`server/config/auto_code.go`

## 2. 功能清单

| 功能 | 说明 | 源码路径 |
|------|------|----------|
| 数据库/表/字段元数据读取 | 按 `businessDB` 别名与 `dbType` 路由到对应方言实现，列出库、表、列元数据供前端选择 | `server/service/system/sys_auto_code_interface.go`、`sys_auto_code_mysql.go`、`sys_auto_code_pgsql.go`、`sys_auto_code_mssql.go`、`sys_auto_code_oracle.go`、`sys_auto_code_sqlite.go`；API `server/api/v1/system/sys_auto_code.go`（GetDB/GetTables/GetColumn） |
| 包信息创建与代码生成（核心管线） | 校验包名/Go 关键字/重名 → 落库 `SysAutoCodePackage` → 遍历模板目录 `server/resource/<template>/{server,web}` → `text/template` 渲染生成 API/Service/Router/Model/Vue 代码 → 用 AST 注入 enter.go 与初始化文件 | `server/service/system/auto_code_package.go`（Create/templates） |
| 包/插件自动发现与同步 | 扫描 `service/` 与 `plugin/` 目录，自动登记包记录；插件校验 v2 标准结构（api/config/initialize/plugin/router/service）；清理数据库中已不存在文件的记录 | `server/service/system/auto_code_package.go`（All） |
| 模板清单列举 | 读取 `server/resource/` 下模板目录，排除 page/function/preview/mcp 等特殊目录 | `server/service/system/auto_code_package.go`（Templates） |
| 模板预览/创建/函数 | 预览生成代码、创建模板、添加自定义模板函数 | `server/service/system/auto_code_template.go`；API `server/api/v1/system/auto_code_template.go` |
| 插件打包/安装/移除/列表 | 把生成产物打包发布为插件、安装插件、移除插件、列举插件市场列表 | `server/service/system/auto_code_plugin.go`；API `server/api/v1/system/auto_code_plugin.go` |
| 生成历史与回滚 | 记录每次生成的请求 meta，支持按 id 回滚（删除已生成 API/菜单/字典/导出模板、AST 回滚文件）、分页查询与删除记录 | `server/service/system/auto_code_history.go`；Model `server/model/system/sys_auto_code_history.go`；API `server/api/v1/system/auto_code_history.go`；AST 回滚 `server/utils/ast/ast_rollback.go` |
| LLM 自动生成（阻塞） | 按 `mode` 拼接上游 `AiPath`（`{FUNC}` 占位替换），POST 调用上游大模型服务，解析统一 `{code,data,msg}` 响应 | `server/service/system/auto_code_llm.go`（LLMAuto）；API `server/api/v1/system/sys_auto_code.go`（LLMAuto） |
| LLM SSE 流式代理 | 识别流式请求（response_mode=streaming/sse 或 Accept: text/event-stream），向上游发起 SSE 请求并逐块转发给前端，处理上游非 SSE 回退、`[DONE]`、错误事件 | `server/service/system/auto_code_llm.go`（LLMAutoStream）；API `server/api/v1/system/sys_auto_code_sse.go`（LLMAutoSSE） |
| AI 工作流会话持久化 | 按用户保存/查询/删除"analysis"与"workflow"两类会话，自动从首条用户消息/表单推导标题，截断 255 字符，记录 Dify conversationId/messageId 与当前节点 | `server/service/system/ai_workflow_session.go`；Model `server/model/system/sys_ai_workflow_session.go`；API `server/api/v1/system/ai_workflow_session.go` |
| AI 工作流 Markdown 落盘 | 把会话结果导出为 Markdown 写入项目目录 | `server/service/system/ai_workflow_markdown.go`；API `DumpMarkdown` |
| MCP 工具生成与生命周期 | 按 `resource/mcp/tools.tpl` 生成 MCP 工具 Go 文件，提供生成、状态查询、启动、停止、列表、路由列表、测试接口 | `server/service/system/auto_code_mcp.go`；API `server/api/v1/system/auto_code_mcp.go` |
| 模板函数库 | 为 `.tpl` 模板注入 `GenerateField/GenerateSearchField/GenerateTableColumn/GenerateFormItem` 等字段渲染函数 | `server/utils/autocode/template_funcs.go` |
| Go AST 注入与回滚 | 对 enter.go、router_biz.go、gorm_biz.go、plugin register.go 等做 import/结构体/方法注入与 gofmt 格式化，支持插件 v2 与 package 两种形态 | `server/utils/ast/`（ast.go、ast_enter.go、package_enter.go、plugin_enter.go、package_initialize_router.go、package_initialize_gorm.go、plugin_initialize_v2.go 等） |

## 3. 解决的问题

| 用户痛点 | 解法 |
|----------|------|
| 手写 CRUD 后端分层代码（Router/API/Service/Model）与 Vue 页面重复劳动大 | 模板引擎 + 模板函数库，一套表元数据批量生成后端五层与前端 api/form/view/table 代码 |
| 新生成模块无法自动挂载到现有组合入口（enter.go、路由组、GORM 注册） | 自研 Go AST 解析注入器，自动改写 enter.go 组合、`router_biz.go`/`gorm_biz.go` 初始化并 gofmt |
| 多数据库方言（MySQL/PG/SQLServer/Oracle/SQLite）元数据查询 SQL 各不相同 | `Database` 接口 + 按 `dbType`/`businessDB` 别名路由到五套方言实现 |
| 直接在源码树生成代码易出错、无法撤回 | 生成历史表记录请求 meta，提供按 id 回滚（删 API/菜单/字典 + AST 文件回滚） |
| 希望用自然语言/AI 描述需求自动产出表结构与代码 | LLM 代理层转发到插件市场上游 AI 服务，支持阻塞与 SSE 流式两种交互 |
| AI 生成的会话（需求分析/工作流）刷新即丢失 | 会话按用户落库，存消息、表单、结果、当前节点，支持列表恢复 |
| 业务包需要与插件两种发布形态，目录结构约定不同 | templates() 按 `template=package/plugin` 分流生成路径与 AST 注入目标 |
| 可被 AI Agent 调用的工具需要标准化 | 基于 `mark3labs/mcp-go`（第三方组件，不在本仓库源码内）的 MCP 工具模板生成与启停 |

## 4. 系统边界

**In-Scope（本仓库实现）：**
- 元数据读取（五套方言）、模板渲染管线、AST 注入/回滚、包与插件的生成/同步/打包/安装、生成历史回滚。
- LLM 阻塞代理与 SSE 流式代理（本仓库只做转发与协议适配，不实现模型推理）。
- AI 工作流会话的 CRUD 与 Markdown 落盘。
- MCP 工具文件生成与生命周期管理接口。

**Out-of-Scope（外部系统 / 第三方组件，不在本仓库源码内）：**
- 业务数据库 MySQL / PostgreSQL / SQLServer / Oracle / SQLite（通过 GORM 驱动连接，仅读取元数据）。
- 上游大模型服务 / Dify 工作流平台（由 `config.yaml` 的 `AutoCode.AiPath` 配置，本仓库仅 HTTP 转发；Dify 为第三方工作流平台）。
- `mark3labs/mcp-go`（MCP 协议 SDK，第三方依赖）。
- `@Variant Form`（前端表单设计器，第三方组件）。
- 七牛云 / 阿里云 / 腾讯云 / 华为 OBS / AWS S3 / MinIO / Cloudflare R2 等文件存储（本仓库通过 SDK 调用，存储服务本身不在源码内）。

## 5. 架构图

![codegen 架构图](codegen-architecture.html)

架构图分层说明：
- **接入层**：前端管理界面（Vue 3）与公开的 LLM 代理接口，分别走鉴权组与公开组。
- **API 层**：`AutoCodeApi`、`AIWorkflowSessionApi` 等处理 HTTP 请求，SSE 接口负责流式转发。
- **Service 层**：`autoCodePackage` 为生成管线核心；`Database` 接口按方言分流；`autoCodeLlm` 转发上游 AI；`aiWorkflowSession` 持久化会话。
- **工具层**：`autocode` 模板函数、`ast` AST 注入器负责源码改写。
- **存储层**：关系库（GORM）保存包/历史/会话元数据；`server/resource/` 为模板仓库；源码树为生成产物落点。
- **外部**：业务数据库（读元数据）与上游大模型/Dify 服务（HTTP/SSE）。

## 6. 数据流图：代码生成管线

![codegen 数据流图](codegen-dataflow.html)

阶段说明：用户在前端选定库表与字段 → 读取数据库元数据 → 由包信息与模板函数渲染 `server/resource/<tpl>` 模板 → 同时生成后端（api/router/service/model）与前端（api/form/view/table）文件 → 经 AST 注入器改写组合入口与初始化文件 → 落盘源码树并写历史记录 → 必要时打包 ZIP / 回滚。

## 7. 时序图：AI 工作流 LLM 调用链

![codegen 时序图](codegen-sequence.html)

分段说明：前端发起 LLM 请求（可指定流式）→ 后端 API 判断流式模式 → Service 拼接 `AiPath`（`{FUNC}` 替换 mode）→ 向后上游大模型/Dify 发起阻塞或 SSE 请求 → 流式时逐块透传 SSE 事件并以 `done` 收尾 → 前端展示结果，会话由 `aiWorkflowSessionService` 异步落库。
