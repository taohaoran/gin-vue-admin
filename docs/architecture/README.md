# gin-vue-admin 系统架构文档

> 基于 gin-vue-admin 源码（`github.com/flipped-aurora/gin-vue-admin/server`，Go 1.24.0，v2.9.2）深度分析产出，
> 覆盖系统级与 8 个子系统的功能、问题域、系统边界、架构图、时序图与数据流图。
> 全仓 412 个 Go 文件 / 约 4.6 万行代码，前端 Vue 3 + Vite + Pinia + Element Plus。
> 所有图表由 archify 渲染为自包含交互式 HTML。

## 文档导航

### 系统级

| 文档 | 说明 | 图表 |
|------|------|------|
| [system-overview.md](system-overview.md) | 功能总览、解决的问题、系统边界、核心代码映射 | [系统架构图](system-architecture.html) · [登录与鉴权时序图](system-auth-sequence.html) · [系统数据流图](system-dataflow.html) |

### 认证与授权（auth/）

| 子系统 | 文档 | 架构图 | 时序图 | 数据流图 |
|--------|------|--------|--------|----------|
| 认证与授权（JWT + Casbin RBAC + API Token） | [auth.md](auth/auth.md) | [架构图](auth/auth-architecture.html) | [登录签发与请求鉴权时序](auth/auth-sequence.html) | — |

### 系统管理域（admin/）

| 子系统 | 文档 | 架构图 | 时序图 | 数据流图 |
|--------|------|--------|--------|----------|
| 系统管理（用户/字典/参数/日志/系统信息/版本/初始化） | [admin.md](admin/admin.md) | [架构图](admin/admin-architecture.html) | [系统管理时序](admin/admin-sequence.html) | — |

### 代码生成与 AI 工作流（codegen/）

| 子系统 | 文档 | 架构图 | 时序图 | 数据流图 |
|--------|------|--------|--------|----------|
| 代码生成器与 AI 工作流 | [codegen.md](codegen/codegen.md) | [架构图](codegen/codegen-architecture.html) | [代码生成时序](codegen/codegen-sequence.html) | [代码生成数据流](codegen/codegen-dataflow.html) |

### 文件存储与上传（storage/）

| 子系统 | 文档 | 架构图 | 时序图 | 数据流图 |
|--------|------|--------|--------|----------|
| 文件存储与上传（多云 OSS + 断点续传） | [storage.md](storage/storage.md) | [架构图](storage/storage-architecture.html) | [文件上传时序](storage/storage-sequence.html) | — |

### 插件体系（extension/）

| 子系统 | 文档 | 架构图 | 时序图 | 数据流图 |
|--------|------|--------|--------|----------|
| 插件体系（announcement/email/auto） | [extension.md](extension/extension.md) | [架构图](extension/extension-architecture.html) | [插件注册时序](extension/extension-sequence.html) | — |

### MCP 服务器与客户端（mcp/）

| 子系统 | 文档 | 架构图 | 时序图 | 数据流图 |
|--------|------|--------|--------|----------|
| MCP 服务与 AI 集成 | [mcp.md](mcp/mcp.md) | [架构图](mcp/mcp-architecture.html) | [MCP 调用时序](mcp/mcp-sequence.html) | — |

### 定时任务与运行运维（ops/）

| 子系统 | 文档 | 架构图 | 时序图 | 数据流图 |
|--------|------|--------|--------|----------|
| 定时任务与运行时中间件 | [ops.md](ops/ops.md) | [架构图](ops/ops-architecture.html) | — | — |

### 示例业务模块（example/）

| 子系统 | 文档 | 架构图 | 时序图 | 数据流图 |
|--------|------|--------|--------|----------|
| 示例业务（customer CRUD + 表单生成器） | [example.md](example/example.md) | [架构图](example/example-architecture.html) | [示例业务时序](example/example-sequence.html) | — |

## 产出统计

| 板块 | MD | HTML | JSON |
|------|----|------|------|
| 系统级（根目录） | 1 | 3 | 3 |
| auth | 1 | 2 | 2 |
| admin | 1 | 2 | 2 |
| codegen | 1 | 3 | 3 |
| storage | 1 | 2 | 2 |
| extension | 1 | 2 | 2 |
| mcp | 1 | 2 | 2 |
| ops | 1 | 1 | 1 |
| example | 1 | 2 | 2 |
| **合计** | **9** | **19** | **19** |

## 覆盖范围与说明

- **质量档位**：系统级 3 张图与 auth-architecture、ops-architecture、mcp-architecture 共 6 张达到 showcase 档（validate 0 错误）；其余 13 张子系统图因复杂布局约束（多参与者时序、标签间距、dataflow stage 限制等）经多轮迭代后以 standard 档渲染成功，均已在各自文档中如实披露，render 退出码 0，无 schema/逻辑错误。
- **源码可追溯**：所有文档功能行的源码路径均真实存在（全量 331 条引用校验通过）。
- **外部组件标注**：MySQL/PostgreSQL/Redis/MongoDB、Casbin、golang-jwt、base64Captcha、robfig/cron、gopsutil、mark3labs/mcp-go、七云对象存储 SDK、@Variant Form 等第三方组件均已标注"不在本仓库源码内"。
- **命名规范**：全部文件名与目录名使用英文短横线（-），无中文短横线。
- **覆盖缺口**：auth 的菜单/按钮/API 元数据 Service 未逐方法展开；storage 七家云厂商 SDK 具体调用未逐行展开（统一抽象为驱动层）；mcp 12 个工具入参 schema 未逐个展开；codegen 的 auto_code_plugin 与 ai_workflow_markdown 内部细节仅点到。以上缺口不影响主架构结论。

---

**在线访问**：本文档已部署至 GitHub Pages → <https://taohaoran.github.io/gin-vue-admin/>（入口页 index.html，MD 文档在部署时自动渲染为美观 HTML）。
