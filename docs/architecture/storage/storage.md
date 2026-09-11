# gin-vue-admin 文件存储与上传子系统（storage）

> 本文档基于 gin-vue-admin v2.9.2 源码（后端模块 `github.com/flipped-aurora/gin-vue-admin/server`，Go 1.24.0）分析，覆盖统一对象存储抽象（本地 + 七云厂商）、文件上传下载与附件分类管理、大文件断点续传（分片上传）的核心能力与边界。

## 1. 概述

storage 子系统负责"文件二进制内容往哪放、上传记录怎么管、大文件怎么传"三件事。其架构核心是**策略模式 + 接口抽象**：在 `server/utils/upload/upload.go` 定义唯一的 `OSS` 接口（`UploadFile` / `DeleteFile`），由工厂函数 `NewOss()` 根据 `config.yaml` 的 `system.oss-type` 在本地磁盘与七家云厂商对象存储之间切换，业务 Service 层只依赖 `OSS` 接口，完全不感知底层存储实现。

在此之上，`example` 域提供三类业务能力：
- **普通文件上传下载**：文件记录 CRUD、附件分类（树形）、URL 批量导入；
- **断点续传（分片上传）**：基于文件 MD5 的秒传/续传，分片落地、合并、清理；
- **多驱动可切换**：改一行配置即可在 local / qiniu / aliyun-oss / tencent-cos / huawei-obs / aws-s3 / cloudflare-r2 / minio 间切换。

核心代码路径：
- 存储抽象：`server/utils/upload/upload.go`（OSS 接口 + NewOss 工厂）、`local.go`、`aliyun_oss.go`、`aws_s3.go`、`cloudflare_r2.go`、`minio_oss.go`、`obs.go`、`qiniu.go`、`tencent_cos.go`
- 配置：`server/config/oss_{local,aliyun,aws,cloudflare,huawei,minio,qiniu,tencent}.go`、`server/config/disk.go`
- API 层：`server/api/v1/example/exa_file_upload_download.go`、`exa_attachment_category.go`、`exa_breakpoint_continue.go`
- Service 层：`server/service/example/exa_file_upload_download.go`、`exa_attachment_category.go`、`exa_breakpoint_continue.go`
- Model 层：`server/model/example/exa_file_upload_download.go`、`exa_attachment_category.go`、`exa_breakpoint_continue.go`
- 路由层：`server/router/example/exa_file_upload_and_download.go`、`exa_attachment_category.go`
- 工具：`server/utils/zip.go`（打包）、`server/utils/directory.go`

## 2. 功能清单

| 功能 | 说明 | 源码路径 |
|------|------|----------|
| 统一 OSS 接口抽象 | 定义 `OSS` 接口（UploadFile/DeleteFile），业务层面向接口编程 | `server/utils/upload/upload.go`（OSS interface） |
| 存储驱动工厂 | 按 `system.oss-type` 实例化对应驱动；minio 初始化失败直接 panic 提醒配置错误 | `server/utils/upload/upload.go`（NewOss） |
| 本地磁盘存储 | MD5 加密文件名 + 时间戳防重名，按 `local.store-path` 落盘；删除时校验 key 防路径穿越并用文件锁 | `server/utils/upload/local.go`；配置 `server/config/oss_local.go` |
| 七牛云存储 | 走七牛 SDK 上传/删除对象 | `server/utils/upload/qiniu.go`；配置 `server/config/oss_qiniu.go` |
| 阿里云 OSS | `oss.Bucket.PutObject` 流式上传，按日期分目录 | `server/utils/upload/aliyun_oss.go`；配置 `server/config/oss_aliyun.go` |
| 腾讯云 COS | 腾讯云 COS SDK 上传/删除 | `server/utils/upload/tencent_cos.go`；配置 `server/config/oss_tencent.go` |
| 华为云 OBS | 华为 OBS SDK 上传/删除 | `server/utils/upload/obs.go`；配置 `server/config/oss_huawei.go` |
| AWS S3 | AWS S3 SDK 上传/删除 | `server/utils/upload/aws_s3.go`；配置 `server/config/oss_aws.go` |
| Cloudflare R2 | R2（S3 兼容）SDK 上传/删除 | `server/utils/upload/cloudflare_r2.go`；配置 `server/config/oss_cloudflare.go` |
| MinIO | 按 endpoint/ak/sk/bucket/useSSL 初始化 MinIO 客户端 | `server/utils/upload/minio_oss.go`；配置 `server/config/oss_minio.go` |
| 普通文件上传 | 接收 multipart 文件 → 选驱动上传 → 记录 url/name/tag/key/classId，支持 noSave 不落库与同 key 去重 | `server/service/example/exa_file_upload_download.go`（UploadFile）；API `server/api/v1/example/exa_file_upload_download.go` |
| 文件删除 | 先查库拿 key → 调 OSS 驱动删对象 → 硬删数据库记录 | `server/service/example/exa_file_upload_download.go`（DeleteFile） |
| 文件记录管理 | 编辑文件名、分页列表（关键字/分类过滤）、URL 批量导入 | 同上；API UploadFile/EditFileName/GetFileList/ImportURL |
| 附件分类（树形） | 分类名 + 父节点 pid，支持树形列表、增删 | `server/model/example/exa_attachment_category.go`；Service `server/service/example/exa_attachment_category.go`；API `server/api/v1/example/exa_attachment_category.go` |
| 断点续传：分片校验 | 接收分片 fileMd5/chunkMd5/chunkNumber/chunkTotal，校验分片 MD5 | `server/api/v1/example/exa_breakpoint_continue.go`（BreakpointContinue） |
| 断点续传：秒传/续传 | 按 fileMd5+fileName 查库：已完成则秒传返回路径，否则 Preload 已有切片续传 | `server/service/example/exa_breakpoint_continue.go`（FindOrCreateFile） |
| 断点续传：分片落地 | 分片字节写入临时目录并登记 ExaFileChunk | 同上（CreateFileChunk）；API 调 `utils.BreakPointContinue` |
| 断点续传：合并完成 | 全部分片到齐后 `utils.MakeFile` 合并为最终文件 | API `BreakpointContinueFinish` |
| 断点续传：清理切片 | 删除临时分片并把主记录置完成，拦截 `..` 等路径穿越 | API `RemoveChunk`；Service `DeleteFileChunk` |
| 磁盘挂载配置 | 磁盘挂载点配置项 | `server/config/disk.go` |

## 3. 解决的问题

| 用户痛点 | 解法 |
|----------|------|
| 本地磁盘、各家云 OSS API 各不相同，业务代码难切换 | 定义唯一 `OSS` 接口 + `NewOss()` 工厂，业务只依赖接口，改配置即换驱动 |
| 单机部署想用本地盘，上云又不想改业务代码 | 8 种驱动同构实现，`oss-type` 一行配置切换 |
| 上传文件名冲突、覆盖 | 本地驱动对文件名 MD5 + 时间戳重命名；云厂商按日期/唯一 key 组织对象 |
| 删除文件时误删存储路径外文件（路径穿越） | `Local.DeleteFile` 校验 key 不含 `..` 与非法字符，删除加文件锁 |
| 大文件上传中断后要从头重传 | 基于文件 MD5 的分片续传：已传分片登记入库，中断后续传未传分片 |
| 相同文件重复上传浪费带宽 | FindOrCreateFile 检测 `file_md5+is_finish=true`，命中即秒传 |
| 附件需要归类管理 | 树形附件分类表（pid 自关联）+ 文件记录 classId 关联 |
| 删除记录时云对象残留 | 删除记录前先调 `oss.DeleteFile(key)` 删除真实对象 |

## 4. 系统边界

**In-Scope（本仓库实现）：**
- `OSS` 接口与 8 个驱动实现、工厂选择逻辑。
- 文件上传记录 CRUD、附件树形分类、URL 导入。
- 断点续传的分片校验、秒传/续传判断、分片登记、合并与清理。
- 路径穿越防护、文件锁、MD5 校验等安全逻辑。

**Out-of-Scope（外部系统 / 第三方组件，不在本仓库源码内）：**
- 七牛云 / 阿里云 OSS / 腾讯云 COS / 华为云 OBS / AWS S3 / Cloudflare R2 / MinIO 等对象存储服务（本仓库仅通过各家官方 SDK 调用，存储服务本身与账号鉴权不在源码内）。
- 各家云厂商 SDK（aliyun-oss-go-sdk、aws-sdk-go、minio-go、qiniu/go-sdk、tencentcloud-sdk-go 等，第三方依赖）。
- 本地文件系统的实际磁盘与操作系统（仅做文件读写）。
- 业务数据库（GORM），用于保存文件记录、分类、分片元数据。
- 前端分片上传组件与拖拽上传 UI（位于 `web/src/`，本仓库后端仅提供接口）。

## 5. 架构图

![storage 架构图](storage-architecture.html)

架构图说明：
- **接入层**：前端（Vue 3）通过 multipart/form-data 上传文件或分片。
- **API 层**：`FileUploadAndDownloadApi` 与 `AttachmentCategoryApi` 接收请求、参数校验、路径穿越拦截。
- **Service 层**：`FileUploadAndDownloadService` 编排"选驱动 → 上传 → 写记录"；断点续传 Service 维护文件/分片元数据。
- **驱动层（策略模式核心）**：统一 `OSS` 接口，`NewOss()` 按 `oss-type` 返回 Local 或七家云厂商实现。
- **存储层**：本地磁盘（store-path）或各家云对象存储 bucket；关系库存文件记录/分类/分片元数据。

## 6. 时序图：文件上传全流程

![storage 时序图](storage-sequence.html)

分段说明：前端 POST 上传文件 → API 接收 multipart → Service 调 `NewOss()` 按配置选驱动 → 驱动把文件字节写入本地磁盘或云 bucket 并返回 (url, key) → Service 写文件记录到数据库 → API 返回 `{code,data,msg}` 含文件详情。删除时反向：按 id 查 key → 驱动删对象 → 硬删记录。
