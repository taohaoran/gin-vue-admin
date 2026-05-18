# Docker 部署后数据库未初始化问题排查

> 2026-05-19 实战排查记录，涉及 `docker compose up -d` 后 MySQL 无表、登录 panic 的完整链路。

## 根因

**docker-compose 只启动容器，不会自动初始化数据库。** gin-vue-admin 的初始化流程需要手动触发——server 启动时 MySQL 配置全为空，仅注册 `/init/checkdb` 和 `/init/initdb` 路由等待调用，直到收到初始化请求才会建表写数据。

## 问题表现

| 现象 | 实际含义 |
|------|----------|
| `docker compose ps` 四容器全 healthy | 容器进程正常，不等于数据库有表 |
| 浏览器访问 `localhost:8080` 能看到页面 | Web 容器正常，nginx 代理到 server |
| 登录时 server 抛空指针 panic | `sys_users` 表不存在，gorm 写入登录日志时崩溃 |
| `SHOW TABLES` 返回空 | MySQL 库 `qmPlus` 存在但零张表 |
| `config.docker.yaml` 中 `mysql.{path,port,db-name}` 全为 `""` | 系统尚未初始化，数据库连接信息未写入配置 |

## 排查链路

### 1. 确认容器状态

```bash
docker ps --filter "name=gva"
# 输出：四个容器全 Up + healthy，不是容器问题
```

### 2. 确认端口可达

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/  # 200
curl -s -o /dev/null -w "%{http_code}" http://localhost:8888/  # 404（正常，无根路由）
```

### 3. 确认数据库是否为空

```bash
docker exec gva-mysql mysql -u gva -p'Aa@6447985' qmPlus -e "SHOW TABLES;"
# 输出为空 → 表未创建
```

### 4. 确认 server 日志中的错误

```bash
docker logs gva-server --tail 50
# 发现 POST /base/login → nil pointer dereference
# 堆栈：LoginLogService.CreateLoginLog → gorm.Create → panic
# 根因：sys_users 表不存在
```

### 5. 寻找初始化入口

- 查 `server/initialize/router.go:78` → `systemRouter.InitInitRouter(PublicGroup)`
- 查 `server/router/system/sys_initdb.go:12` → `initRouter.POST("initdb", ...)`
- 路由路径：`POST /init/initdb`（无 `/api` 前缀，`router-prefix` 为空）

### 6. 确认是否需要初始化

```bash
curl -s -X POST http://localhost:8888/init/checkdb
# {"code":0,"data":{"needInit":true},"msg":"前往初始化数据库"}
```

### 7. 确认初始化参数

查 `server/model/system/request/sys_init.go`，必填字段：`adminPassword`、`dbType`、`host`、`port`、`userName`、`password`、`dbName`。

Docker 网络内 MySQL 地址：

| 参数 | 值 | 来源 |
|------|-----|------|
| host | `177.7.0.13` 或 `gva-mysql` | docker-compose.yaml networks 配置 |
| port | `3306` | 容器内端口，非宿主机映射端口 13306 |
| userName | `gva` | docker-compose.yaml MYSQL_USER |
| password | `Aa@6447985` | docker-compose.yaml MYSQL_PASSWORD |
| dbName | `qmPlus` | docker-compose.yaml MYSQL_DATABASE |

### 8. 执行初始化

```bash
curl -s -X POST http://localhost:8888/init/initdb \
  -H "Content-Type: application/json" \
  -d '{
    "adminPassword": "123456",
    "dbType": "mysql",
    "host": "177.7.0.13",
    "port": "3306",
    "userName": "gva",
    "password": "Aa@6447985",
    "dbName": "qmPlus"
  }'
# {"code":0,"data":{},"msg":"自动创建数据库成功"}
```

### 9. 验证结果

```bash
docker exec gva-mysql mysql -u gva -p'Aa@6447985' qmPlus -e "SHOW TABLES;" | wc -l
# 60+ 张表
docker exec gva-mysql mysql -u gva -p'Aa@6447985' qmPlus -e "SELECT id, username FROM sys_users;"
# admin 用户已创建
```

## 关键认知

### 两个端口，两个网络视角

| 访问方 | 连接 MySQL 用 | 原因 |
|--------|--------------|------|
| 宿主机（DataGrip、mysql CLI） | `localhost:13306` | docker-compose ports 映射 |
| Docker 容器内（gva-server） | `gva-mysql:3306` 或 `177.7.0.13:3306` | 同 Docker network，走容器内端口 |

初始化 API 由 server 容器执行，所以填 `3306` 而非 `13306`。

### 登录为什么 panic 而不是返回错误

`LoginLogService.CreateLoginLog` 在登录验证失败时仍尝试往 `sys_login_logs` 写记录，但表不存在 → gorm nil pointer。这本质上是代码缺陷：数据库未初始化时缺少防御性判断。

### 初始化是幂等的吗

`POST /init/checkdb` 检测逻辑是判断 `sys_users` 表是否存在。一旦初始化完成，再次调用 `initdb` 会返回 `"数据库已存在"` 的提示（code=7），不会重复建表。

## 正常流程 vs 本次实际流程

```
正常：docker compose up → 浏览器打开 localhost:8080 → 自动跳到初始化页面 → 填写DB信息 → 完成
本次：docker compose up → 直接尝试登录 → panic → 查日志 → 查路由 → 调API初始化 → 完成
```

## 相关文件

- [[docker-deploy-notes]] — Docker Compose 部署配置修改记录
- `server/config.docker.yaml` — 服务端 Docker 环境配置
- `server/initialize/router.go:78` — init 路由注册
- `server/router/system/sys_initdb.go` — initdb API 路由
- `server/model/system/request/sys_init.go` — 初始化请求参数定义
