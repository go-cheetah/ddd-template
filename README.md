# ddd

项目启动

```bash
go install github.com/swaggo/swag/cmd/swag@latest
go mod tidy
go get -u github.com/swaggo/swag@latest
swag init -g cmd/app/main.go
go run cmd/app/main.go
```

> 个人理解：DDD（领域驱动设计）强调领域模型的重要性，通过分层架构实现业务逻辑与技术实现的解耦。简化版 DDD 保留核心思想：聚合根、实体、仓储接口，舍弃复杂的值对象和领域事件，适合中小型项目。

## 代码目录结构

```bash
.
├── bin                                   # 生成的二进制文件
├── build                                 # 构建相关的脚本或者内容
│   └── Dockerfile                        # Docker 构建
├── cmd                                   # 程序的入口
│   └── app                               # app1 取名例如前台就叫 front，后台叫 backend，一个目录一个入口
│       ├── app                           # app 固定词
│       │   ├── options                   # 处理选项
│       │   │   └── options.go            # 处理选项，处理一些初始化操作（配置/日志/DB）
│       │   └── server.go                 # cobra 命令入口，组装 Server 并启动
│       └── main.go                       # 入口函数，具体的实现都在上面的 server 之中
├── config                                # 默认配置文件
│   └── config.yaml                       # 默认配置文件
├── docs                                  # 接口文档，主要是 swagger
├── go.mod                                # go.mod
├── internal                              # 内部资源，外部不允许调用  请查看 https://golang.org/doc/go1.4#internalpackages
│   ├── config                            # 配置文件，主要就是设置一下配置的相关结构体之类
│   │   ├── config.go                     # 总配置
│   │   ├── db.go                         # 数据库的配置项
│   │   ├── log.go                        # 日志的配置项
│   │   └── server.go                     # 服务器配置项
│   ├── domain                            # 领域层（对应 Kratos 的 biz）
│   │   └── user                          # 案例
│   │       ├── entity.go                 # 领域实体（聚合根）
│   │       ├── repository.go             # 仓储接口（依赖倒置）
│   │       └── service.go                # 领域服务/用例
│   ├── infrastructure                    # 基础设施层（对应 Kratos 的 data）
│   │   └── persistence                   # 持久化
│   │       ├── db.go                     # 数据库连接初始化（可复用）
│   │       ├── model                     # GORM 模型
│   │       │   └── user.go
│   │       └── user                      # 案例
│   │           ├── mapper.go             # domain ↔ model 映射
│   │           ├── migrate.go            # 迁移注册
│   │           └── repository.go         # 仓储实现
│   ├── interfaces                        # 接口层（对应 Kratos 的 service）
│   │   └── http                          # HTTP 接口
│   │       ├── dto                       # 数据传输对象
│   │       │   ├── mapper.go             # entity ↔ dto 映射
│   │       │   └── user_dto.go           # 请求/响应 DTO
│   │       ├── handler                   # Handler 函数
│   │       │   └── user_handler.go       # Gin Handler，调用领域服务
│   │       └── router.go                 # 路由注册
│   ├── pkg                               # 内部公共包
│   │   ├── migration                     # 迁移系统
│   │   │   ├── migration.go              # migrate 的初始化操作
│   │   │   └── model.go                  # model 层
│   │   └── response                      # 响应内容
│   │       ├── common.go                 # 响应内容
│   │       └── response.go               # 响应内容
│   └── server                            # web 框架入口，负责中间件/迁移/路由编排与依赖注入
│       ├── migrate.go                    # 处理 infrastructure 层的 migrate
│       ├── router.go                     # 依赖注入并委托 interfaces/http 注册路由
│       └── server.go                     # web 框架入口
├── pkg                                   # 外部公共包，其他代码库可调用
│   ├── cors                              # 跨域
│   │   ├── config.go
│   │   ├── cors.go
│   │   └── utils.go
│   ├── database                          # orm，数据库相关，例如 MySQL
│   │   ├── gorm.go
│   │   ├── mysql.go
│   │   └── sqlite.go
│   ├── generator                         # id 生成器
│   │   └── id.go
│   ├── log                               # 日志库
│   │   └── log.go
│   └── pkg.go                            # pkg 空文件，参考 https://github.com/golang-standards/project-layout/issues/10
├── Makefile                              # 构建脚本
└── README.md                             # readme 内容
```

## 分层职责

| 层级 | 职责 | 对应 MVC |
|------|------|----------|
| interfaces | HTTP Handler 处理、请求校验、DTO 转换 | handler |
| domain | 聚合根、实体、仓储接口、业务逻辑 | service + model 接口 |
| infrastructure | 仓储实现、GORM 模型、数据库操作 | repository + model 实现 |
| server | 中间件装配、迁移、依赖注入、启动 HTTP 服务 | server |

**依赖方向**：interfaces → domain ← infrastructure（依赖倒置，infrastructure 实现 domain 定义的仓储接口）

## 启动流程

`cmd/app/main.go` → `cmd/app/app/server.go (cobra)` → `cmd/app/app/options/options.go (NewServer)` → `internal/server/server.go (Run)`：

1. `options.NewServer()` 加载配置、初始化 gin、日志、数据库连接
2. `server.Run()` 装配 CORS、日志、Recovery 中间件
3. `server.migrate()` 调用 `internal/infrastructure/persistence/<aggregate>.RegisterMigrations`
4. `server.SetupRouter()` 完成 repository → service → handler 的依赖注入，并委托 `internal/interfaces/http.RegisterRoutes`

## 使用模块

| 功能 | 工具 |
|------|------|
| web 框架 | `github.com/gin-gonic/gin` |
| 数据库 | `gorm.io/gorm` |
| 日志 | `go.uber.org/zap` + `gopkg.in/natefinch/lumberjack.v2` |
| 命令行 | `github.com/spf13/cobra` |
| 配置 | `github.com/spf13/viper` |
| id 生成器 | `github.com/sony/sonyflake` |
| swagger | `github.com/swaggo/gin-swagger` |

## 与 MVC 脚手架对比

| 方面 | MVC 脚手架 | DDD 脚手架 |
|------|-----------|-----------|
| 分层 | handler → service → repository | interfaces → domain ← infrastructure |
| 领域模型 | Model (GORM) | Entity (domain 层) |
| 仓储接口 | 无 | domain 层定义 |
| 业务逻辑 | Service 层 | domain/service |
| DTO 转换 | Handler 层 | interfaces/http/dto |
| 依赖方向 | 单向依赖 | 依赖倒置 |

## 核心设计模式

### 依赖倒置

仓储接口在 domain 层定义，infrastructure 层实现，使得 domain 层不依赖具体实现。

### 聚合根

实体封装业务行为，如 `User.Activate()` 包含业务规则。

### Mapper

domain ↔ model 双向转换，隔离领域模型与数据库模型。

## Makefile 常用命令

```bash
make build        # 构建
make run          # 运行
make test         # 测试
make swag         # 生成 Swagger 文档
make docker-build # Docker 构建
make docker-run   # Docker 运行
```
