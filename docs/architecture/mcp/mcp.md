# gin-vue-admin MCP 服务器与客户端子系统（mcp）

> 本文档基于 gin-vue-admin 源码分析，覆盖 GVA 在 mark3labs/mcp-go 之上封装的 MCP 工具集、Streamable HTTP 服务、上游代理机制与独立进程托管模式。

## 1. 概述

GVA 的 MCP（Model Context Protocol）子系统把后台的代码生成、API/菜单/字典/权限管理能力以标准 MCP 工具的形式暴露给 AI 编辑器（Cursor、Claude 等），使大模型可以直接在 GVA 平台上完成"分析需求 → 创建包/模块 → 生成代码 → 挂权限 → 审查"的闭环。

整体设计上有三点值得注意：

1. **不重新实现业务逻辑，而是做 HTTP 代理**。MCP 工具本身不直接调用 Service 层，而是把请求参数序列化成 JSON，连同调用方的 `x-token` 一起，以 HTTP 请求转发到 GVA 主服务（默认 `http://127.0.0.1:8888`）的既有 REST API 上，再把 `{code, data, msg}` 解包后返回给 MCP 客户端。这样主服务既有的 JWT + Casbin 鉴权、RBAC、统一响应格式全部复用，权限边界不被打破。
2. **工具自注册**。每个工具一个文件，在 `init()` 中调用 `RegisterTool(&Xxx{})`；`NewMCPServer()` 启动时遍历工具注册表，一次性 `AddTool` 到 mcp-go 的 `MCPServer` 实例。
3. **支持两种部署形态**。既可由 `server/cmd/mcp/main.go` 独立编译为二进制（默认监听 `:8889`，路径 `/mcp`），也可在 GVA 主服务进程内由 `standalone_manager` 通过页面按钮触发 `go build ./cmd/mcp` 并 `exec` 拉起独立进程，PID、日志、配置元信息写入项目根 `.tmp/mcp/` 目录。

核心源码路径：

- MCP 服务与工具注册：`server/mcp/server.go`、`server/mcp/enter.go`
- 鉴权上下文：`server/mcp/context.go`
- 上游 HTTP 代理：`server/mcp/http_client.go`、`server/mcp/autocode_http.go`、`server/mcp/dictionary_http.go`
- 业务工具：`server/mcp/api_creator.go`、`server/mcp/api_lister.go`、`server/mcp/menu_creator.go`、`server/mcp/menu_lister.go`、`server/mcp/role_api_assigner.go`、`server/mcp/dictionary_generator.go`、`server/mcp/dictionary_query.go`、`server/mcp/requirement_analyzer.go`、`server/mcp/gva_analyze.go`、`server/mcp/gva_execute.go`、`server/mcp/gva_review.go`
- 独立进程托管：`server/mcp/standalone_manager.go`、`server/mcp/process_utils_unix.go`、`server/mcp/process_utils_windows.go`
- MCP 客户端封装：`server/mcp/client/client.go`
- 独立进程入口：`server/cmd/mcp/main.go`、`server/cmd/mcp/config.go`、`server/cmd/mcp/logger.go`
- 配置结构：`server/config/mcp.go`
- 全局句柄：`server/global/global.go`（`GVA_MCP_SERVER`）

## 2. 功能清单

| 功能 | 说明 | 源码路径 |
|------|------|----------|
| MCP 服务器构造 | 基于 mark3labs/mcp-go `NewMCPServer`，按配置注入 Name/Version，并把全局句柄写入 `global.GVA_MCP_SERVER` | `server/mcp/server.go` |
| Streamable HTTP 挂载 | 在 `http.ServeMux` 上按 `config.MCP.Path`（默认 `/mcp`）挂载 MCP handler，并额外提供 `/health` 健康检查 | `server/mcp/server.go` |
| 工具接口契约 | `McpTool` 接口要求 `New() mcp.Tool` 与 `Handle(ctx, CallToolRequest) (*CallToolResult, error)` | `server/mcp/enter.go` |
| 工具自注册与批量装载 | `toolRegister` map + `RegisterTool`/`RegisterAllTools`，各工具文件 `init()` 自动注册 | `server/mcp/enter.go` |
| 调用方 token 提取 | 从请求头按 `x-token`/`token`/`authorization`(Bearer) 顺序提取鉴权 token 注入 context | `server/mcp/context.go` |
| 上游 HTTP 泛型客户端 | `getUpstream/postUpstream/deleteUpstream` 泛型封装，转发 token、超时控制、`{code,data,msg}` 解包与错误归一化 | `server/mcp/http_client.go` |
| AutoCode 上游封装 | 对 `/autoCode/getPackage`、`/createPackage`、`/createTemp`、`/delPackage`、`/getSysHistory`、`/delSysHistory` 的类型化封装 | `server/mcp/autocode_http.go` |
| 字典上游封装 | 对 `/sysDictionary/*`、`/sysDictionaryDetail/*` 的查询/创建封装 | `server/mcp/dictionary_http.go` |
| 创建 API 工具 `create_api` | 校验后调用 `/api/createApi`，并先查 `/api/getApiList` 去重 | `server/mcp/api_creator.go` |
| 列举 API 工具 `list_all_apis` | 调 `/api/getAllApis` + `/autoCode/mcpRoutes` 返回全量 API 与动态路由 | `server/mcp/api_lister.go` |
| 创建菜单工具 `create_menu` | 调 `/menu/addBaseMenu`，并通过 `/menu/getMenuList` 校验父子关系 | `server/mcp/menu_creator.go` |
| 列举菜单工具 `list_all_menus` | 调 `/menu/getMenuList` 返回菜单树供前端路由编写 | `server/mcp/menu_lister.go` |
| 角色授权工具 `assign_api_to_role` | 先 `/casbin/getPolicyPathByAuthorityId` 取现状，再 `/casbin/updateCasbin` 追加授权（不覆盖） | `server/mcp/role_api_assigner.go` |
| 生成字典工具 `generate_dictionary_options` | 按类型自动生成字典项并创建字典与详情 | `server/mcp/dictionary_generator.go` |
| 查询字典工具 `query_dictionaries` | 按类型精确查询或返回全部字典，供 AI 生成表单时使用 | `server/mcp/dictionary_query.go` |
| 需求分析工具 `requirement_analyzer` | 文档标注为所有 MCP 工具的首选入口，输出模块/包设计建议 | `server/mcp/requirement_analyzer.go` |
| GVA 现状分析 `gva_analyze` | 汇总有效包/模块，判断需求是否需要新建包/模块/字典，并清理空包 | `server/mcp/gva_analyze.go` |
| 代码生成执行 `gva_execute` | 直接调用 `/autoCode/createTemp` 执行代码生成，无需人工确认 | `server/mcp/gva_execute.go` |
| 生成结果审查 `gva_review` | 在 `gva_execute` 之后对产物与用户需求做对照审查 | `server/mcp/gva_review.go` |
| 独立进程托管 | 页面触发时 `go build -o .tmp/mcp/gva-mcp-standalone ./cmd/mcp`，`exec` 拉起，PID 元信息持久化 | `server/mcp/standalone_manager.go` |
| 托管进程健康检查 | `GET /health` 2s 超时；启动等待 20s、停止等待 8s、构建超时 2min | `server/mcp/standalone_manager.go` |
| 跨平台进程工具 | Unix/Windows 两套 `processExists`/`terminateProcess` 实现 | `server/mcp/process_utils_unix.go`、`server/mcp/process_utils_windows.go` |
| MCP 客户端封装 | `NewStreamableHttpClient` + Initialize 握手 + ServerName 校验 | `server/mcp/client/client.go` |
| 独立进程入口 | 加载 `-config` 指定的 yaml，初始化日志，启动 Streamable HTTP 服务 | `server/cmd/mcp/main.go`、`server/cmd/mcp/config.go` |
| MCP 配置结构 | Name/Version/Path/Addr/BaseURL/UpstreamBaseURL/AuthHeader/RequestTimeout，含废弃字段向后兼容 | `server/config/mcp.go` |

## 3. 解决的问题

| 用户痛点 | 解法 |
|----------|------|
| AI 编辑器无法直接读写 GVA 的 API/菜单/字典/代码生成能力 | 把这些能力包装成标准 MCP Tool，Cursor 等客户端按协议自动发现与调用 |
| 在 MCP 层重新实现业务逻辑会与主服务产生两份代码 | 所有工具经 HTTP 代理打主服务既有 REST API，业务逻辑只有一份 |
| MCP 服务本身需要鉴权，但又不能绕过主服务的 RBAC | 透传调用方 `x-token`/Bearer 到主服务，主服务继续走 JWT+Casbin |
| 用户希望在网页上一键启停 MCP 独立进程，而不是手动敲命令行 | `standalone_manager` 自动 `go build` + `exec` + 健康检查 + 元信息落盘 |
| 大模型代码生成需要"分析→生成→审查"三段式工作流 | `requirement_analyzer` / `gva_analyze` / `gva_execute` / `gva_review` 工具按流水线组织，工具描述中显式约定调用顺序 |
| 跨平台部署托管进程 | `process_utils_unix.go` / `process_utils_windows.go` 分别实现进程存在性判断与终止 |

## 4. 系统边界

**In-Scope（本仓库 `server/mcp/`、`server/cmd/mcp/`、`server/config/mcp.go` 内实现）**：

- MCP 服务器构造、Streamable HTTP 挂载、工具自注册与分发
- 调用方 token 提取与 context 传递
- 上游 HTTP 泛型代理与各业务工具对主服务 REST API 的类型化封装
- 独立进程的构建、启停、健康检查与元信息持久化
- MCP 客户端握手封装

**Out-of-Scope（外部系统 / 第三方组件，不在本仓库源码内）**：

- `mark3labs/mcp-go`：MCP 协议框架（server / client / Streamable HTTP transport）——不在本仓库源码内
- GVA 主服务进程（默认 `:8888`）：MCP 工具的所有写操作最终都代理到它的 `/api`、`/menu`、`/autoCode`、`/casbin`、`/sysDictionary` 等路由，其内部 Router->API->Service->Model 链路属于主服务范畴，不在本子系统内
- AI 编辑器 / LLM 客户端（Cursor、Claude Desktop 等）：MCP 协议的发起方，不在本仓库源码内
- JWT / Casbin 中间件：由主服务提供，本子系统仅透传 token
- Go 工具链（`go build`）：`standalone_manager` 在托管构建时依赖本机 Go 环境，不在本仓库源码内

## 5. 架构图

![MCP 子系统架构图](mcp-architecture.html)

图中左侧是 AI 编辑器客户端，通过 Streamable HTTP 调用本仓库 `server/mcp` 包：`mcp-http` 先经 `auth-context` 把调用方 token 注入 context，再由工具注册表把请求按工具名分发到 12 个业务工具；工具统一经 `http-proxy` 把参数序列化为 JSON、带上 token，以 HTTP 调用 GVA 主服务 `:8888` 的既有 REST API，并把 `{code,data,msg}` 解包后回传给 MCP 客户端。右上是独立进程托管形态：`standalone-mgr` 在页面触发时 `go build ./cmd/mcp` 并 `exec` 出 `cmd/mcp` 独立进程，二者共用同一套 `NewStreamableHTTPServer` 代码。

## 6. 时序图

![MCP 工具调用时序图](mcp-sequence.html)

时序图以一次 `create_api` 工具调用为例：阶段一 AI 客户端向 `/mcp` 发起 `CallTool` 并携带 `x-token`；阶段二 MCP 服务按工具名分发到 `create_api` 工具，工具内部通过 `http_client` 调用主服务 `POST /api/createApi`，主服务返回统一响应；阶段三代理解包 `{code,data,msg}`、校验 `code==0`，工具把结果封装为 MCP `CallToolResult` 回传客户端。鉴权始终在主服务侧完成，MCP 自身只做透传。
