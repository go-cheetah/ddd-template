# ddd

项目启动

```bash
go install github.com/swaggo/swag/cmd/swag@latest
go mod tidy
go get -u github.com/swaggo/swag@latest
swag init -g cmd/server/main.go
go run cmd/server/main.go
```

> 个人理解：DDD（领域驱动设计）强调领域模型的重要性，通过分层架构实现业务逻辑与技术实现的解耦。简化版 DDD 保留核心思想：聚合根、实体、仓储接口，舍弃复杂的值对象和领域事件，适合中小型项目。

## 代码目录结构

```bash
.
├── bin                              # 生成的二进制文件
├── build                            # 构建相关的脚本或者内容
│   └── Dockerfile                   # Docker 构建
├── cmd                              # 程序的入口
│   └── server                       # 服务入口
│       └── main.go                  # 入口函数，手动组装依赖
├── config                           # 默认配置文件
│   └── config.yaml                  # 默认配置文件
├── docs                             # 接口文档,主要是swagger
├── go.mod                           # go.mod
├── internal                         # 内部资源，外部不允许调用  请查看 https://golang.org/doc/go1.4#internalpackages
│   ├── config                       # 配置文件，主要就是设置一下配置的相关结构体之类
│   │   ├── config.go                # 总配置
│   │   ├── db.go                    # 数据库的配置项
│   │   ├── log.go                   # 日志的配置项
│   │   └── server.go                # 服务器配置项
│   ├── domain                       # 领域层（对应 Kratos 的 biz）
│   │   └── user                     # 案例
│   │       ├── entity.go            # 领域实体（聚合根）
│   │       ├── repository.go        # 仓储接口（依赖倒置）
│   │       └── service.go           # 领域服务/用例
│   ├── infrastructure               # 基础设施层（对应 Kratos 的 data）
│   │   └── persistence              # 持久化
│   │       ├── user                 # 案例
│   │       │   ├── model.go         # GORM 模型
│   │       │   ├── mapper.go        # domain ↔ model 映射
│   │       │   ├── repository.go    # 仓储实现
│   │       │   └ migrate.go         # 迁移注册
│   │       └── db.go                # 数据库连接初始化
│   ├── interfaces                   # 接口层（对应 Kratos 的 service）
│   │   └── http                     # HTTP 接口
│   │       ├── handler              # Handler 函数
│   │       │   └ user_handler.go    # Gin Handler，调用领域服务
│   │       ├── dto                  # 数据传输对象
│   │       │   └ user_dto.go        # 请求/响应 DTO
│   │       ├── middleware           # 中间件
│   │       └ router.go              # 路由注册
│   │       └ server.go              # HTTP 服务入口
│   └── pkg                          # 内部公共包
│       ├── migration                # 迁移系统
│       ├── response                 # 响应内容
│       │   ├── common.go            # 响应内容
│       │   └ response.go            # 响应内容
│       └── error                    # 错误定义
├── pkg                              # 外部公共包，其他代码库可调用
│   ├── cors                         # 跨域
│   │   ├── config.go
│   │   ├── cors.go
│   │   └ utils.go
│   ├── database                     # 数据库相关，例如MySQL
│   │   ├── database.go              # 定义接口
│   │   ├── mysql.go                 # 初始化MySQL
│   │   └ sqlite.go                  # 初始化sqlite
│   ├── generator                    # id生成器
│   │   └ id.go
│   └── log                          # 日志
│       └ log.go
├── Makefile                         # 构建脚本
└── README.md                        # readme内容
```

## 分层职责

| 层级 | 职责 | 对应 MVC |
|------|------|----------|
| interfaces | HTTP Handler 处理、请求校验、DTO 转换 | handler |
| domain | 聚合根、实体、仓储接口、业务逻辑 | service + model 接口 |
| infrastructure | 仓储实现、GORM 模型、数据库操作 | repository + model 实现 |

**依赖方向**：interfaces → domain → infrastructure（依赖倒置）

## 使用模块

| 功能 | 工具 |
|------|------|
| web框架 | `github.com/gin-gonic/gin` |
| 数据库 | `gorm.io/gorm` |
| 日志 | `go.uber.org/zap` + `gopkg.in/natefinch/lumberjack.v2` |
| 命令行 | `github.com/spf13/cobra` |
| 配置 | `github.com/spf13/viper` |
| id生成器 | `github.com/sony/sonyflake` |
| swagger | `github.com/swaggo/gin-swagger` |

## 与 MVC 脚手架对比

| 方面 | MVC 脚手架 | DDD 脚手架 |
|------|-----------|-----------|
| 分层 | handler → service → repository | interfaces → domain → infrastructure |
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