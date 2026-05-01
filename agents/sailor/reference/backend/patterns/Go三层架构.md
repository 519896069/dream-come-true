# Go 三层 CRUD 架构规范

基于 `backend/api` 项目结构。

## 目录结构

```
src/
├── controller/          # 控制器层（HTTP入口）
│   └── auth.go
├── app/
│   └── modules/
│       └── auth/
│           ├── entity/     # 实体层（数据结构）
│           │   └── user.go
│           ├── dto/        # 数据传输对象
│           │   └── request.go
│           ├── repository/  # 数据访问层
│           │   └── repository.go
│           └── service.go  # 业务逻辑层
├── middleware/          # 中间件
├── route/               # 路由
└── bootstrap/           # 初始化配置
```

## 分层职责

### Controller（控制器层）

```go
// controller/auth.go
package controller

import (
    "github.com/gin-gonic/gin"
    "xsxd.zepei/api/src/app/modules/auth"
    "xsxd.zepei/api/src/app/modules/auth/dto"
    "xsxd.zepei/api/src/middleware"
)

func GetAuthController() AuthController {
    return AuthController{}
}

type AuthController struct {
    myHttp.Controller  // 嵌入基础控制器
}

// 接收请求 → 参数绑定 → 调用 Service → 返回响应
func (c AuthController) IPromise(ctx *gin.Context) {
    resp := response.Fail(response.CODE_FAIL, errors.New("api error"))
    defer func() {
        c.Json(ctx, resp)
    }()

    // 1. 获取认证信息（从 Token，不是请求参数）
    authInfo, ok := middleware.GetAuthInfo(ctx)
    if !ok || authInfo.Id == 0 {
        ctx.AbortWithStatus(http.StatusUnauthorized)
        return
    }

    // 2. 参数绑定
    var req dto.IPromiseRequest
    if err := ctx.BindJSON(&req); err != nil {
        return
    }

    // 3. 参数校验
    if req.EnterpriseType != 0 && req.EnterpriseType != 1 {
        return
    }

    // 4. 设置用户ID（从 Token 获取，禁止信任请求参数）
    req.UserId = authInfo.Id

    // 5. 调用 Service
    resp = auth.Promise(req)
}
```

**原则：**
- 禁止从请求中获取 user_id、team_id 等身份信息
- 使用 `middleware.GetAuthInfo(ctx)` 获取认证信息
- 统一使用 `defer c.Json(ctx, resp)` 返回响应
- 参数校验在 Controller 层完成

---

### Service（业务逻辑层）

```go
// app/modules/auth/service.go
package auth

const ModuleName = "module_auth"

// Promise 更新企业类型
func Promise(req dto.IPromiseRequest) *response.CommonResponse {
    // 1. 业务规则校验
    if req.UserId == 0 {
        return response.Fail(response.CODE_FAIL, errors.New("user id is required"))
    }

    // 2. 调用 Repository
    err := repository.UserPromise(entity.User{
        Id:               req.UserId,
        EnterpriseType:   req.EnterpriseType,
        EnterpriseStatus: 1,
    })
    if err != nil {
        // 3. 错误日志记录（使用 zap）
        logger.GetLog().Error(err.Error(),
            zap.String("module", ModuleName),
            zap.String("action", "UserPromise"),
            zap.Any("req", req),
            zap.Error(err))
        return response.Fail(response.CODE_FAIL, errors.New("database error"))
    }

    return response.Success(nil)
}
```

**原则：**
- 业务逻辑在 Service 层
- 统一错误处理和日志记录
- 错误日志包含 module、action、请求信息
- 返回 `*response.CommonResponse`

---

### Repository（数据访问层）

```go
// app/modules/auth/repository/repository.go
package repository

import (
    "gorm.io/gorm/clause"
    "xsxd.zepei/api/bootstrap/database"
    "xsxd.zepei/api/src/app/modules/auth/entity"
)

// FindUser 按条件查询用户
func FindUser(enterpriseNumber string) (user entity.User, err error) {
    err = database.GetDB().
        Where("enterprise_number = ?", enterpriseNumber).
        Find(&user).Error
    return
}

// UpsertUser 插入或更新（基于唯一键冲突）
func UpsertUser(user *entity.User) error {
    var err error
    user.Id, err = database.GetUUidSort()
    if err != nil {
        return err
    }
    return database.GetDB().Clauses(clause.OnConflict{
        Columns:   []clause.Column{{Name: "enterprise_number"}},
        DoUpdates: clause.AssignmentColumns([]string{"enterprise_name"}),
    }).Create(user).Error
}

// UserPromise 更新指定字段
func UserPromise(user entity.User) error {
    return database.GetDB().
        Select([]string{"enterprise_type", "enterprise_status"}).
        Where("id = ?", user.Id).
        Updates(&user).Error
}
```

**原则：**
- 禁止在 Repository 层处理业务逻辑
- 使用 GORM 的链式调用
- 复杂更新使用 `Select` 指定字段（防止误更新）
- Upsert 使用 `Clauses(clause.OnConflict{...})`

---

### Entity（实体层）

```go
// app/modules/auth/entity/user.go
package entity

import "time"

const (
    RULE_ENT      = iota // 企业
    RULE_ADMIN           // 管理员
    RULE_PROVINCE        // 省级
)

type User struct {
    Id               uint64    `gorm:"id" json:"id"`
    Name             string    `gorm:"name" json:"name"`
    Email            string    `gorm:"email" json:"email"`
    Password         string    `gorm:"password" json:"-"`
    Salt             string    `gorm:"salt" json:"-"`
    Phone            string    `gorm:"phone" json:"phone"`
    Rule             int8      `gorm:"rule" json:"rule"`
    EnterpriseName   string    `gorm:"enterprise_name" json:"enterprise_name"`
    CreatedAt        time.Time `gorm:"created_at;autoCreateTime" json:"created_at"`
    UpdatedAt        time.Time `gorm:"updated_at;autoUpdateTime" json:"updated_at"`
}

func (User) TableName() string {
    return "users"
}
```

**原则：**
- 使用 `json:"-"` 隐藏敏感字段（Password、Salt）
- `uint64` 用于 ID 类型
- 必须定义 `TableName()` 方法
- 常量定义使用 iota 枚举

---

### DTO（数据传输对象）

```go
// app/modules/auth/dto/request.go
package dto

// AuthRequest 登录请求
type AuthRequest struct {
    Code string `json:"code"`
}

// IPromiseRequest 更新企业类型请求DTO
type IPromiseRequest struct {
    UserId         uint64 `json:"user_id"`      // 从 Token 获取，禁止信任请求
    EnterpriseType int8   `json:"enterprise_type"`
}

// UserInfo 用户信息响应
type UserInfo struct {
    Id             uint64 `json:"id,string"`  // uint64 转 string 避免精度丢失
    Name           string `json:"name"`
    EnterpriseName string `json:"enterprise_name"`
}
```

**原则：**
- 请求 DTO 只包含需要的字段
- user_id、team_id 等身份字段不放在请求 DTO 中（从 Token 获取）
- `json:"id,string"` 用于 uint64 类型避免 JSON 精度问题

---

## 调用流程

```
HTTP Request
    ↓
Controller (参数绑定 + Token 解析)
    ↓
Service (业务逻辑 + 日志记录)
    ↓
Repository (数据库操作)
    ↓
Response
```

## 四层架构（增加 Factory 层）

在复杂业务场景下，可在 Service 和 Repository 之间增加 Factory 层：

```
Controller → Service → Factory → Repository
```

Factory 用于：
- 封装复杂查询逻辑
- 组合多个 Repository 操作
- 事务管理
