# DDD Go 脚手架设计文档

## 1. 项目背景

基于现有 MVC 手架 (`cheetah-template/mvc`) 的成功经验，结合 Kratos Layout 的分层思想，创建一个简化版 DDD 手架。

**参考项目**：
- MVC 脚手架：`/home/hank/code/cheetah-template/mvc`
- Kratos Layout：`/home/hank/code/cheetah-template/kratos-layout`

---

## 2. 技术决策

| 决策项 | 选择 | 理由 |
|--------|------|------|
| API 定义 | Gin 原生路由 | 轻量，适合单体应用 |
| 依赖注入 | 手动构造 | 简单直接，无额外编译步骤 |
| DDD 深度 | 简化版 | 聚合根、实体、仓储，无值对象/领域事件 |
| 分层架构 | 三层架构 | interfaces → domain → infrastructure |
| 事件总线 | 无 | 保持简单，聚合间通过服务协调 |

---

## 3. 目录结构

```
{{ .ProjectName }}/
├── cmd/
│   └── server/
│       └── main.go                    # 程序入口，手动组装依赖
├── config/
│   └── config.yaml                    # 默认配置文件
├── internal/
│   ├── domain/                        # 领域层（对应 Kratos 的 biz）
│   │   ├── user/
│   │   │   ├── entity.go              # 领域实体
│   │   │   ├── repository.go          # 仓储接口
│   │   │   └── service.go             # 领域服务/用例
│   │   └── ...
│   ├── infrastructure/                # 基础设施层（对应 Kratos 的 data）
│   │   ├── persistence/
│   │   │   ├── user/
│   │   │   │   ├── model.go           # GORM 模型
│   │   │   │   ├── mapper.go          # domain ↔ model 映射
│   │   │   │   ├── repository.go      # 仓储实现
│   │   │   │   └── migrate.go         # 迁移注册
│   │   │   └── db.go                  # 数据库连接初始化
│   │   └── ...
│   ├── interfaces/                    # 接口层（对应 Kratos 的 service）
│   │   ├── http/
│   │   │   ├── handler/
│   │   │   │   └── user_handler.go    # Gin Handler，调用领域服务
│   │   │   ├── dto/
│   │   │   │   └── user_dto.go        # 请求/响应 DTO
│   │   │   ├── router.go              # 路由注册
│   │   │   └── middleware/            # 中间件
│   │   └── ...
│   ├── config/                        # 配置结构定义
│   │   ├── config.go                  # 总配置
│   │   ├── db.go                      # 数据库配置
│   │   ├── log.go                     # 日志配置
│   │   └── server.go                  # 服务配置
│   └── pkg/                           # 内部公共包
│       ├── migration/                 # 迁移系统
│       ├── response/                  # 响应工具
│       └── error/                     # 错误定义
├── pkg/                               # 外部公共包
│   ├── cors/                          # CORS 中间件
│   ├── database/                      # 数据库抽象
│   ├── generator/                     # ID 生成器
│   └── log/                           # 日志工具
├── build/
│   └── Dockerfile                     # Docker 构建
├── go.mod                             # Go 模块定义
└── Makefile                           # 构建脚本
```

---

## 4. 分层职责

### 4.1 interfaces 层（接口层）

**职责**：
- HTTP Handler 处理
- 请求参数校验
- DTO ↔ 领域对象转换
- 调用 domain 层服务
- 响应封装

**依赖**：domain 层

### 4.2 domain 层（领域层）

**职责**：
- 聚合根、实体定义
- 仓储接口定义（依赖倒置）
- 业务逻辑/用例编排

**依赖**：无外部依赖，仅定义仓储接口

### 4.3 infrastructure 层（基础设施层）

**职责**：
- 仓储接口实现
- GORM 模型定义
- domain ↔ model 映射
- 数据库操作

**依赖**：domain 层（实现其接口）

---

## 5. 依赖流

```
                    HTTP Request
                         │
                         ▼
┌────────────────────────────────────┐
│      interfaces/http/router        │
│          (路由注册)                 │
└─────────────────┬──────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│   interfaces/http/handler          │
│      (HTTP Handler + DTO)          │
└─────────────────┬──────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│        domain/user                 │
│   (聚合根 + 仓储接口 + 服务)        │
│                                    │
│   UserRepository (interface)       │◄──── 接口定义
└─────────────────┬──────────────────┘
                  │
                  ▼
┌────────────────────────────────────┐
│  infrastructure/persistence/user   │
│   (仓储实现 + GORM模型 + 映射)      │
│                                    │
│   implements UserRepository        │◄──── 接口实现
└────────────────────────────────────┘
                  │
                  ▼
               Database
```

---

## 6. 核心设计模式

### 6.1 依赖倒置

仓储接口在 domain 层定义，infrastructure 层实现：

```go
// domain/user/repository.go
type UserRepository interface {
    Save(ctx context.Context, user *User) (*User, error)
    FindByID(ctx context.Context, id int64) (*User, error)
    ...
}

// infrastructure/persistence/user/repository.go
type userRepo struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) domain.UserRepository {
    return &userRepo{db: db}
}
```

### 6.2 聚合根

User 作为聚合根，封装业务行为：

```go
// domain/user/entity.go
type User struct {
    ID        int64
    Name      string
    Email     string
    Status    Status
    CreatedAt time.Time
    UpdatedAt time.Time
}

// 业务方法
func (u *User) Activate() error {
    if u.Status == StatusDeleted {
        return errors.New("cannot activate deleted user")
    }
    u.Status = StatusActive
    return nil
}
```

### 6.3 领域服务

封装业务逻辑编排：

```go
// domain/user/service.go
type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) Create(ctx context.Context, name, email string) (*User, error) {
    user := &User{Name: name, Email: email}
    // 业务规则检查
    if err := user.Validate(); err != nil {
        return nil, err
    }
    return s.repo.Save(ctx, user)
}
```

### 6.4 映射器（Mapper）

domain ↔ model 双向转换：

```go
// infrastructure/persistence/user/mapper.go
func toModel(user *domain.User) *UserModel {
    return &UserModel{
        ID:     user.ID,
        Name:   user.Name,
        Email:  user.Email,
        Status: int(user.Status),
    }
}

func toDomain(model *UserModel) *domain.User {
    return &domain.User{
        ID:     model.ID,
        Name:   model.Name,
        Email:  model.Email,
        Status: domain.Status(model.Status),
    }
}
```

---

## 7. 技术栈

从 MVC 手架保留：

| 组件 | 技术 | 版本 |
|------|------|------|
| Web 框架 | Gin | v1.9+ |
| ORM | GORM | v1.25+ |
| 数据库 | MySQL / SQLite | - |
| 配置 | Viper | v1.18+ |
| CLI | Cobra | v1.8+ |
| 日志 | Zap + Lumberjack | - |
| Swagger | swaggo/gin-swagger | - |
| ID 生成 | Sonyflake | - |

---

## 8. 初始化流程

```go
// cmd/server/main.go
func main() {
    // 1. 加载配置
    cfg := config.Load()

    // 2. 初始化基础设施
    db := database.New(cfg.DB)
    logger := log.New(cfg.Log)

    // 3. 初始化基础设施层（仓储实现）
    userRepo := persistence.NewUserRepository(db)

    // 4. 初始化领域层（依赖注入）
    userService := domain.NewUserService(userRepo)

    // 5. 初始化接口层（Handler）
    userHandler := handler.NewUserHandler(userService)

    // 6. 启动 HTTP 服务
    router := http.NewRouter(userHandler)
    server := http.NewServer(cfg.Server, logger, router)
    server.Run()
}
```

---

## 9. 与 MVC 脚手架对比

| 方面 | MVC 脚手架 | DDD 脚手架 |
|------|-----------|-----------|
| 分层 | handler → service → repository | interfaces → domain → infrastructure |
| 领域模型 | Model (GORM) | Entity (domain 层) |
| 仓储接口 | 无 | domain 层定义 |
| 业务逻辑 | Service 层 | domain/service |
| DTO 转换 | Handler 层 | interfaces/http/dto |
| 依赖方向 | 单向依赖 | 依赖倒置 |

---

## 10. 示例模块设计

脚手架内置 `user` 示例模块，展示完整的 CRUD 流程：

### domain/user/entity.go

```go
package user

import (
    "errors"
    "time"
)

// Status 状态枚举
type Status int

const (
    StatusInactive Status = 0
    StatusActive   Status = 1
    StatusDeleted  Status = 2
)

// User 领域实体（聚合根）
type User struct {
    ID        int64
    Name      string
    Email     string
    Status    Status
    CreatedAt time.Time
    UpdatedAt time.Time
}

// Validate 验证
func (u *User) Validate() error {
    if u.Name == "" {
        return errors.New("name is required")
    }
    if u.Email == "" {
        return errors.New("email is required")
    }
    return nil
}

// Activate 激活
func (u *User) Activate() error {
    if u.Status == StatusDeleted {
        return errors.New("cannot activate deleted user")
    }
    u.Status = StatusActive
    return nil
}
```

### domain/user/repository.go

```go
package user

import "context"

// UserRepository 仓储接口
type UserRepository interface {
    Save(ctx context.Context, user *User) (*User, error)
    FindByID(ctx context.Context, id int64) (*User, error)
    FindByEmail(ctx context.Context, email string) (*User, error)
    Update(ctx context.Context, user *User) (*User, error)
    Delete(ctx context.Context, id int64) error
    List(ctx context.Context, page, pageSize int) ([]*User, int64, error)
}
```

### domain/user/service.go

```go
package user

import (
    "context"
    "errors"
)

// UserService 领域服务
type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

// Create 创建用户
func (s *UserService) Create(ctx context.Context, name, email string) (*User, error) {
    // 检查邮箱是否已存在
    existing, _ := s.repo.FindByEmail(ctx, email)
    if existing != nil {
        return nil, errors.New("email already exists")
    }

    user := &User{
        Name:   name,
        Email:  email,
        Status: StatusActive,
    }

    if err := user.Validate(); err != nil {
        return nil, err
    }

    return s.repo.Save(ctx, user)
}

// GetByID 查询用户
func (s *UserService) GetByID(ctx context.Context, id int64) (*User, error) {
    return s.repo.FindByID(ctx, id)
}

// Update 更新用户
func (s *UserService) Update(ctx context.Context, id int64, name, email string) (*User, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }

    user.Name = name
    user.Email = email

    if err := user.Validate(); err != nil {
        return nil, err
    }

    return s.repo.Update(ctx, user)
}

// Delete 删除用户
func (s *UserService) Delete(ctx context.Context, id int64) error {
    return s.repo.Delete(ctx, id)
}

// List 用户列表
func (s *UserService) List(ctx context.Context, page, pageSize int) ([]*User, int64, error) {
    return s.repo.List(ctx, page, pageSize)
}
```

### infrastructure/persistence/user/model.go

```go
package user

import "time"

// UserModel GORM 模型
type UserModel struct {
    ID        int64     `gorm:"primaryKey"`
    Name      string    `gorm:"size:64;not null"`
    Email     string    `gorm:"size:128;not null;uniqueIndex"`
    Status    int       `gorm:"default:1"`
    CreatedAt time.Time `gorm:"autoCreateTime"`
    UpdatedAt time.Time `gorm:"autoUpdateTime"`
}

func (UserModel) TableName() string {
    return "users"
}
```

### infrastructure/persistence/user/mapper.go

```go
package user

import "{{ .ProjectName }}/internal/domain/user"

// toModel 领域对象转 GORM 模型
func toModel(u *user.User) *UserModel {
    return &UserModel{
        ID:        u.ID,
        Name:      u.Name,
        Email:     u.Email,
        Status:    int(u.Status),
        CreatedAt: u.CreatedAt,
        UpdatedAt: u.UpdatedAt,
    }
}

// toDomain GORM 模型转领域对象
func toDomain(m *UserModel) *user.User {
    return &user.User{
        ID:        m.ID,
        Name:      m.Name,
        Email:     m.Email,
        Status:    user.Status(m.Status),
        CreatedAt: m.CreatedAt,
        UpdatedAt: m.UpdatedAt,
    }
}
```

### infrastructure/persistence/user/repository.go

```go
package user

import (
    "context"
    "gorm.io/gorm"
    "{{ .ProjectName }}/internal/domain/user"
)

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) user.UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Save(ctx context.Context, u *user.User) (*user.User, error) {
    m := toModel(u)
    if err := r.db.WithContext(ctx).Create(m).Error; err != nil {
        return nil, err
    }
    return toDomain(m), nil
}

func (r *userRepository) FindByID(ctx context.Context, id int64) (*user.User, error) {
    var m UserModel
    if err := r.db.WithContext(ctx).First(&m, id).Error; err != nil {
        return nil, err
    }
    return toDomain(&m), nil
}

func (r *userRepository) FindByEmail(ctx context.Context, email string) (*user.User, error) {
    var m UserModel
    if err := r.db.WithContext(ctx).Where("email = ?", email).First(&m).Error; err != nil {
        return nil, err
    }
    return toDomain(&m), nil
}

func (r *userRepository) Update(ctx context.Context, u *user.User) (*user.User, error) {
    m := toModel(u)
    if err := r.db.WithContext(ctx).Save(m).Error; err != nil {
        return nil, err
    }
    return toDomain(m), nil
}

func (r *userRepository) Delete(ctx context.Context, id int64) error {
    return r.db.WithContext(ctx).Delete(&UserModel{}, id).Error
}

func (r *userRepository) List(ctx context.Context, page, pageSize int) ([]*user.User, int64, error) {
    var models []UserModel
    var total int64

    offset := (page - 1) * pageSize

    if err := r.db.WithContext(ctx).Model(&UserModel{}).Count(&total).Error; err != nil {
        return nil, 0, err
    }

    if err := r.db.WithContext(ctx).Offset(offset).Limit(pageSize).Find(&models).Error; err != nil {
        return nil, 0, err
    }

    users := make([]*user.User, len(models))
    for i, m := range models {
        users[i] = toDomain(&m)
    }

    return users, total, nil
}
```

### interfaces/http/dto/user_dto.go

```go
package dto

import "time"

// CreateUserRequest 创建用户请求
type CreateUserRequest struct {
    Name  string `json:"name" binding:"required"`
    Email string `json:"email" binding:"required,email"`
}

// UpdateUserRequest 更新用户请求
type UpdateUserRequest struct {
    Name  string `json:"name" binding:"required"`
    Email string `json:"email" binding:"required,email"`
}

// UserResponse 用户响应
type UserResponse struct {
    ID        int64     `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    Status    int       `json:"status"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// ListUserResponse 用户列表响应
type ListUserResponse struct {
    List  []*UserResponse `json:"list"`
    Total int64           `json:"total"`
    Page  int             `json:"page"`
    PageSize int          `json:"page_size"`
}

// ToResponse 领域对象转响应 DTO
func ToResponse(u *domain.User) *UserResponse {
    return &UserResponse{
        ID:        u.ID,
        Name:      u.Name,
        Email:     u.Email,
        Status:    int(u.Status),
        CreatedAt: u.CreatedAt,
        UpdatedAt: u.UpdatedAt,
    }
}
```

### interfaces/http/handler/user_handler.go

```go
package handler

import (
    "github.com/gin-gonic/gin"
    "{{ .ProjectName }}/internal/domain/user"
    "{{ .ProjectName }}/internal/interfaces/http/dto"
    "{{ .ProjectName }}/internal/pkg/response"
)

type UserHandler struct {
    userService *user.UserService
}

func NewUserHandler(userService *user.UserService) *UserHandler {
    return &UserHandler{userService: userService}
}

// Create 创建用户
// @Summary 创建用户
// @Tags user
// @Accept json
// @Produce json
// @Param body body dto.CreateUserRequest true "请求"
// @Success 200 {object} dto.UserResponse
// @Router /api/v1/users [post]
func (h *UserHandler) Create(c *gin.Context) {
    var req dto.CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.BadRequest(c, err.Error())
        return
    }

    result, err := h.userService.Create(c.Request.Context(), req.Name, req.Email)
    if err != nil {
        response.Fail(c, err)
        return
    }

    response.Success(c, dto.ToResponse(result))
}

// GetByID 查询用户
// @Summary 查询用户
// @Tags user
// @Produce json
// @Param id path int true "用户ID"
// @Success 200 {object} dto.UserResponse
// @Router /api/v1/users/{id} [get]
func (h *UserHandler) GetByID(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        response.BadRequest(c, "invalid id")
        return
    }

    result, err := h.userService.GetByID(c.Request.Context(), id)
    if err != nil {
        response.Fail(c, err)
        return
    }

    response.Success(c, dto.ToResponse(result))
}

// List 用户列表
// @Summary 用户列表
// @Tags user
// @Produce json
// @Param page query int false "页码"
// @Param pageSize query int false "每页数量"
// @Success 200 {object} dto.ListUserResponse
// @Router /api/v1/users [get]
func (h *UserHandler) List(c *gin.Context) {
    page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
    pageSize, _ := strconv.Atoi(c.DefaultQuery("pageSize", "10"))

    list, total, err := h.userService.List(c.Request.Context(), page, pageSize)
    if err != nil {
        response.Fail(c, err)
        return
    }

    users := make([]*dto.UserResponse, len(list))
    for i, u := range list {
        users[i] = dto.ToResponse(u)
    }

    response.Success(c, &dto.ListUserResponse{
        List:     users,
        Total:    total,
        Page:     page,
        PageSize: pageSize,
    })
}

// Update 更新用户
// @Summary 更新用户
// @Tags user
// @Accept json
// @Produce json
// @Param id path int true "用户ID"
// @Param body body dto.UpdateUserRequest true "请求"
// @Success 200 {object} dto.UserResponse
// @Router /api/v1/users/{id} [put]
func (h *UserHandler) Update(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        response.BadRequest(c, "invalid id")
        return
    }

    var req dto.UpdateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.BadRequest(c, err.Error())
        return
    }

    result, err := h.userService.Update(c.Request.Context(), id, req.Name, req.Email)
    if err != nil {
        response.Fail(c, err)
        return
    }

    response.Success(c, dto.ToResponse(result))
}

// Delete 删除用户
// @Summary 删除用户
// @Tags user
// @Param id path int true "用户ID"
// @Success 200 {object} response.CommonResponse
// @Router /api/v1/users/{id} [delete]
func (h *UserHandler) Delete(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        response.BadRequest(c, "invalid id")
        return
    }

    if err := h.userService.Delete(c.Request.Context(), id); err != nil {
        response.Fail(c, err)
        return
    }

    response.Success(c, nil)
}
```

---

## 11. 配置文件设计

### config/config.yaml

```yaml
server:
  bind: "0.0.0.0"
  port: 9000
  mode: debug  # debug, release, test
  cors:
    origins:
      - "*"
    methods:
      - GET
      - POST
      - PUT
      - PATCH
      - DELETE
    headers:
      - Content-Type
      - Authorization

db:
  type: sqlite  # mysql or sqlite
  mysql:
    host: 127.0.0.1
    port: 3306
    user: root
    password: root
    database: test
    charset: utf8mb4
  sqlite:
    path: ./data.db

log:
  infoFilename: logs/info.log
  errorFilename: logs/error.log
  maxSize: 10       # MB
  maxBackups: 5
  maxAge: 10        # days
  level: info       # debug, info, warn, error
```

---

## 12. 路由设计

```go
// internal/interfaces/http/router.go
func RegisterRoutes(r *gin.Engine, handlers *Handlers) {
    // 健康检查
    r.GET("/health", func(c *gin.Context) {
        c.JSON(200, gin.H{"status": "ok"})
    })

    // API v1
    v1 := r.Group("/api/v1")
    {
        users := v1.Group("/users")
        {
            users.POST("", handlers.User.Create)
            users.GET("", handlers.User.List)
            users.GET("/:id", handlers.User.GetByID)
            users.PUT("/:id", handlers.User.Update)
            users.DELETE("/:id", handlers.User.Delete)
        }
    }

    // Swagger
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
}
```

---

## 13. Makefile 设计

```makefile
.PHONY: all build run test clean swag

APP_NAME := {{ .ProjectName }}
VERSION := 1.0.0
BUILD_TIME := $(shell date +%Y%m%d%H%M%S)

# 构建
build:
	go build -ldflags "-X main.Version=${VERSION} -X main.BuildTime=${BUILD_TIME}" -o bin/${APP_NAME} cmd/server/main.go

# 运行
run:
	go run cmd/server/main.go

# 测试
test:
	go test -v ./...

# 清理
clean:
	rm -rf bin/
	rm -rf logs/

# Swagger 文档生成
swag:
	swag init -g cmd/server/main.go -o ./docs --parseDependency --parseInternal

# Docker 构建
docker-build:
	docker build -t ${APP_NAME}:${VERSION} -f build/Dockerfile .

docker-run:
	docker run -p 9000:9000 ${APP_NAME}:${VERSION}
```

---

## 14. 待确认事项

请审视以上设计，确认：

1. **目录结构**是否符合预期？
2. **分层职责**划分是否合理？
3. **示例模块**设计是否完整？
4. **配置设计**是否满足需求？
5. **是否需要增加其他功能**？如：
   - 中间件模板（认证、限流）
   - 测试模板
   - 更多示例模块

---

## 15. 下一步

确认设计后，将按照以下顺序生成模板文件：

1. `go.mod.tmpl` + `config.yaml.tmpl`
2. `pkg/` 公共工具模板
3. `internal/config/` 配置结构
4. `internal/domain/` 领域层模板
5. `internal/infrastructure/` 基础设施层模板
6. `internal/interfaces/` 接口层模板
7. `cmd/server/` 入口模板
8. `build/Dockerfile.tmpl` + `Makefile.tmpl`