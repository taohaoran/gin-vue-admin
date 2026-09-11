# gin-vue-admin 定时任务与运行运维（ops）子系统

> 本文档基于 gin-vue-admin v2.9.2 源码（后端模块 `github.com/flipped-aurora/gin-vue-admin/server`）分析，覆盖后台定时任务调度与运行时 HTTP 中间件（panic 恢复、请求日志、IP 限流、超时控制、健康检查）的核心能力与边界。本子系统聚焦"平台自身的运行时保障"，不重复 admin 单元的系统信息采集页面。

## 1. 概述

ops 子系统由两部分组成：

1. **后台定时任务调度**：基于 `robfig/cron/v3` 的可扩展定时器 `GVA_Timer`（`server/utils/timer/timed_task.go`），在服务启动时由 `initialize.Timer()` 注册任务。当前内置任务为每日清理操作日志与 JWT 黑名单表。
2. **运行时中间件链**：在 Gin 引擎与路由组上挂载的横切关注点——panic 恢复（GinRecovery）、请求日志（Logger）、IP 限流（DefaultLimit）、超时控制（TimeoutMiddleware），以及 `PublicGroup` 上的 `/health` 健康检查端点。

这些组件不承载业务逻辑，而是保证服务在异常流量、慢查询、panic、磁盘/表膨胀等情况下的可用性与可观测性。

核心代码路径：

- 定时任务：`server/task/clearTable.go`、`server/utils/timer/timed_task.go`、`server/initialize/timer.go`
- 全局定时器实例：`server/global/global.go`（`GVA_Timer timer.Timer = timer.NewTimerTask()`）
- 中间件：`server/middleware/logger.go`、`server/middleware/timeout.go`、`server/middleware/limit_ip.go`、`server/middleware/error.go`（GinRecovery）
- 健康检查：`server/initialize/router.go`（`PublicGroup.GET("/health", ...)`）
- 启动入口：`server/main.go`（调用 `initialize.Timer()`）、`server/initialize/reload.go`（热重载时重启定时器）
- 时长解析工具：`server/utils/human_duration.go`（`ParseDuration`，支持带 `d` 的天数后缀）

## 2. 功能清单

| 功能 | 说明 | 源码路径 |
|------|------|----------|
| 通用定时任务框架 | 封装 robfig/cron，支持按 cronName 分组、按函数/Job 接口添加任务、启停、删除、列出 | `server/utils/timer/timed_task.go` |
| 全局定时器实例 | 进程级单例 `GVA_Timer`，供 initialize 与插件注册任务 | `server/global/global.go` |
| 启动时注册任务 | `main.go` 调用 `initialize.Timer()`，后台 goroutine 中注册每日清理任务 | `server/initialize/timer.go`、`server/main.go` |
| 定时清理数据库 | `@daily` 执行 `ClearTable`：清理 `sys_operation_records`（90 天）与 `jwt_blacklists`（7 天） | `server/task/clearTable.go` |
| 热重载支持 | `initialize/reload.go` 中再次调用 `Timer()`，保证配置热重载后任务仍在 | `server/initialize/reload.go` |
| panic 恢复中间件 | 捕获 handler panic，记录并入库，避免进程崩溃 | `server/middleware/error.go`（GinRecovery，在 `initialize/router.go` 挂载） |
| 请求日志中间件 | 记录路径、Query、Body、IP、UserAgent、耗时、错误；可插拔 Filter/Print/脱敏钩子 | `server/middleware/logger.go` |
| IP 限流中间件 | 基于 Redis 计数器，按 `LimitTimeIP`/`LimitCountIP` 配置限制单位时间内同 IP 请求数 | `server/middleware/limit_ip.go` |
| 请求超时中间件 | 基于 `context.WithTimeout`，超时返回 504 并关闭连接；带 panicChan 透传 panic | `server/middleware/timeout.go` |
| 健康检查端点 | `GET /health` 返回 `ok`，供负载均衡/容器探针使用，挂在免鉴权 PublicGroup | `server/initialize/router.go` |
| 时长解析工具 | 扩展 `time.ParseDuration` 支持 `d`（天）后缀与纯数字秒数，供 JWT 过期/缓冲时间等配置使用 | `server/utils/human_duration.go` |

## 3. 解决的问题

| 用户痛点 | 解法 |
|----------|------|
| 操作日志、JWT 黑名单表随时间无限膨胀 | 每日定时任务按 `created_at` 阈值 DELETE，无需人工清理 |
| 单个慢请求/下游卡死拖垮整个 worker | `TimeoutMiddleware` 用带缓冲 channel 隔离 handler goroutine，超时即返回 504 |
| 恶意 IP 高频刷接口 | Redis 计数器限流：周期内超过 `LimitCountIP` 直接拒绝，返回剩余等待时间 |
| handler panic 导致连接挂起或进程退出 | GinRecovery 统一 recover，记录错误并按状态码响应 |
| 线上问题无法还原现场 | Logger 中间件记录路径/Query/Body/IP/UA/耗时，JSON 输出到 stdout 供 K8s 采集 |
| 多任务需要统一管理与启停 | `Timer` 接口按 cronName 分组封装，支持 Add/Remove/Start/Stop/Close |
| 配置里写 `7d` 这类时长 Go 原生不解析 | `ParseDuration` 扩展支持 `d` 后缀与纯数字秒数 |
| 容器/网关需要存活探针 | `/health` 免鉴权返回 200 ok |

## 4. 系统边界

**In-Scope（本仓库实现）**

- 定时任务框架（对 robfig/cron 的薄封装）与内置 ClearDB 任务
- panic 恢复、请求日志、IP 限流、超时控制四个 Gin 中间件的实现与可配置项
- `/health` 端点
- 时长解析工具

**Out-of-Scope（外部系统 / 第三方组件，不在本仓库源码内）**

- robfig/cron/v3：定时调度内核，第三方组件
- Redis：IP 限流计数器存储（`global.GVA_REDIS`），外部系统
- GORM 连接的业务数据库：定时清理的目标表，外部系统
- 请求日志的采集与存储：Logger 仅输出到 stdout/JSON，实际采集（Filebeat/Loki/ELK）由部署侧完成，不在本仓库源码内
- 系统信息采集（CPU/内存/磁盘）：见 `server/utils/server.go`（gopsutil），本单元不展开，归 admin 子系统
- 进程管理/进程守护（systemd、supervisor、容器编排）：外部运维环境，不在本仓库源码内

## 5. 架构图

![ops 子系统架构图](ops-architecture.html)

上图分为两个区域：

- **运行时中间件链**（上方横向流水线）：请求来源依次经过 GinRecovery（panic 恢复）、请求日志、IP 限流、超时控制后到达业务 Handler；限流中间件通过虚线访问 Redis 做计数。
- **后台定时任务**（下方）：GVA_Timer（robfig/cron）按 `@daily` 触发 ClearTable 任务，由任务执行 DELETE 语句清理业务数据库中的过期记录。

## 6. 时序图 / 数据流图

本子系统不单独绘制时序图，理由如下：

- 中间件链是单向线性管道，其调用顺序与分支（限流拒绝、超时 504、panic 恢复）已在架构图中以流水线形式表达，再画时序图不增加新信息。
- 定时清理任务是"调度器 → 任务函数 → DB"的单跳触发，无跨服务异步交互或多阶段数据管道，不构成有价值的 sequence/dataflow 图。

如需排查具体一次请求经过中间件的行为，可结合 `server/middleware/` 下各 `Next()` 调用顺序与 `server/initialize/router.go` 的中间件挂载点阅读源码。
