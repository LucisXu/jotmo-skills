# Skill：Go/Gin 新接口实现模板（v0.3）

## 目标
参数校验清晰、错误码一致、日志可追踪、可灰度可回滚。

## 项目结构规范

Jotmo 后端服务遵循统一的项目结构：

```
{service-name}/
├── main.go                 # 程序入口
├── gin/
│   ├── api/                # HTTP 路由和处理器
│   │   ├── router.go       # 路由注册
│   │   └── {module}.go     # 模块接口
│   ├── middlewares/        # 中间件
│   │   ├── auth.go         # JWT 认证
│   │   └── sentry.go       # Sentry 集成
│   ├── compound/           # 复合业务逻辑
│   └── response/           # 响应封装
├── internal/               # 业务模块
│   └── {module}/
│       ├── models.go       # 数据模型
│       ├── repository.go   # 数据查询
│       ├── mutation.go     # 数据变更
│       └── enums.go        # 枚举定义
├── pkg/
│   ├── config/             # 配置加载
│   ├── datastore/          # 数据存储连接
│   ├── mq/                 # RabbitMQ 封装
│   ├── validate/           # JWT 验证
│   └── log/                # 日志工具
├── task/                   # 异步任务/定时任务
├── cmd/                    # 命令行工具
└── assets/
    └── config/             # 配置文件目录
```

## 分层职责

### 1. gin/api (Handler 层)
- 参数解析与校验
- 调用 service 层
- 返回响应

```go
// gin/api/user.go
func CreateUser(c *gin.Context) {
    var req CreateUserReq
    if err := c.ShouldBindJSON(&req); err != nil {
        response.Error(c, errors.InvalidParams)
        return
    }

    // 调用业务逻辑
    user, err := compound.CreateUser(c.Request.Context(), req)
    if err != nil {
        response.Error(c, err)
        return
    }

    response.Success(c, user)
}
```

### 2. gin/compound (Service 层)
- 业务逻辑编排
- 可单元测试
- 跨模块调用协调

```go
// gin/compound/user.go
func CreateUser(ctx context.Context, req CreateUserReq) (*User, error) {
    // 业务逻辑
    // 可以调用多个 repository
    return users.Create(ctx, ...)
}
```

### 3. internal/{module} (Repository 层)
- 数据访问
- 可 Mock
- **重要**：所有实体的增删改查必须限制在自身模块的 `mutation.go` 和 `repository.go` 中

```go
// internal/users/repository.go - 查询操作
func GetByID(ctx context.Context, id string) (*User, error) {
    var user User
    err := datastore.MongoDB.Collection("users").FindOne(ctx, bson.M{"_id": id}).Decode(&user)
    if err != nil {
        return nil, errors.WithStack(err)  // 底层包 WithStack
    }
    return &user, nil
}

// internal/users/mutation.go - 变更操作
func Create(ctx context.Context, user *User) error {
    _, err := datastore.MongoDB.Collection("users").InsertOne(ctx, user)
    if err != nil {
        return errors.WithStack(err)  // 底层包 WithStack
    }
    return nil
}
```

> ⚠️ **CRUD 隔离原则**：禁止在其他模块或 compound 层直接操作某实体的数据库。所有数据操作必须通过该实体模块的 repository.go（查询）和 mutation.go（变更）暴露的方法。这样改动时只需修改一处，避免"满天飞"。

## 枚举定义规范

枚举值**必须从 1 开始**，不要从 0 开始。原因：
- 0 值容易与"未设置/默认值"混淆
- 方便判断字段是否被显式设置
- 避免 JSON 序列化时的歧义

```go
// internal/users/enums.go

// ❌ 错误示例：从 0 开始
type UserStatus int
const (
    UserStatusUnknown UserStatus = iota  // 0 - 容易混淆
    UserStatusActive                      // 1
    UserStatusInactive                    // 2
)

// ✅ 正确示例：从 1 开始
type UserStatus int
const (
    UserStatusActive   UserStatus = iota + 1  // 1
    UserStatusInactive                        // 2
    UserStatusBanned                          // 3
)

// 或者显式定义
type OrderStatus int
const (
    OrderStatusPending   OrderStatus = 1
    OrderStatusPaid      OrderStatus = 2
    OrderStatusShipped   OrderStatus = 3
    OrderStatusCompleted OrderStatus = 4
    OrderStatusCancelled OrderStatus = 5
)
```

## 错误处理规范

### WithStack 原则

**底层包一次 WithStack，上层不要重复包**。确保所有错误都有堆栈信息，但避免堆栈重复。

```go
// ✅ 正确：在 repository 层（底层）包 WithStack
// internal/users/repository.go
func GetByID(ctx context.Context, id string) (*User, error) {
    var user User
    err := datastore.MongoDB.Collection("users").FindOne(ctx, bson.M{"_id": id}).Decode(&user)
    if err != nil {
        return nil, errors.WithStack(err)  // 底层包一次
    }
    return &user, nil
}

// ✅ 正确：compound 层直接返回，不要再包
// gin/compound/user.go
func GetUser(ctx context.Context, id string) (*User, error) {
    user, err := users.GetByID(ctx, id)
    if err != nil {
        return nil, err  // 直接返回，不要再 WithStack
    }
    return user, nil
}

// ❌ 错误：重复包 WithStack
func GetUser(ctx context.Context, id string) (*User, error) {
    user, err := users.GetByID(ctx, id)
    if err != nil {
        return nil, errors.WithStack(err)  // 重复了！堆栈会很长
    }
    return user, nil
}
```

### 错误包装层级

| 层级 | 职责 | 是否 WithStack |
|------|------|---------------|
| repository.go / mutation.go | 数据访问 | ✅ 是（底层唯一包装点） |
| compound | 业务编排 | ❌ 否，直接返回或 Wrap 添加上下文 |
| api handler | 响应封装 | ❌ 否，使用 response.Error 处理 |

```go
// 如果需要添加上下文信息，用 Wrap 而不是 WithStack
func ProcessOrder(ctx context.Context, orderID string) error {
    order, err := orders.GetByID(ctx, orderID)
    if err != nil {
        return errors.Wrap(err, "failed to get order")  // 添加上下文，不是 WithStack
    }
    // ...
}
```

## API 路由规范

### 路由分组

```go
// gin/api/router.go
func InitRouter(g *gin.Engine) {
    // 公开接口
    public := g.Group("/api/public/v1")
    {
        public.GET("/ping", Ping)
        public.POST("/login", Login)
    }

    // 需要认证的接口
    v1 := g.Group("/api/v1")
    v1.Use(middlewares.AuthMiddleware())
    {
        v1.POST("/user/info", GetUserInfo)
        v1.POST("/user/update", UpdateUser)
    }

    // 内部接口（服务间调用）
    internal := g.Group("/api/internal/v1")
    internal.Use(middlewares.InternalAuth())
    {
        internal.POST("/notify", InternalNotify)
    }
}
```

### URL 命名规范
- 使用小写 kebab-case：`/api/v1/user-profile`
- 资源用名词，操作用 HTTP 方法
- 版本号放在路径中：`/api/v1/...`

## 中间件使用

### JWT 认证

```go
// gin/middlewares/auth.go
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        claims, err := validate.ParseToken(token)
        if err != nil {
            c.AbortWithStatusJSON(401, ...)
            return
        }
        c.Set("user_id", claims.UserID)
        c.Next()
    }
}
```

### 限流中间件

```go
// 使用 Redis 限流
v1.POST("/sensitive-api",
    datastore.JotmoRedis.WithRatePerUser(handler, rate, getUserId))
```

### 并发锁中间件

```go
// 防止用户并发请求
v1.POST("/order/create",
    datastore.JotmoRedis.WithLockedPerUserRequest(handler, timeout, getUserId))
```

## 响应格式

### 成功响应

```json
{
    "code": 0,
    "msg": "success",
    "data": { ... }
}
```

### 错误响应

```json
{
    "code": 10001,
    "msg": "参数错误",
    "data": null
}
```

## 国内/海外兼容

开发接口时需要考虑：

1. **存储服务**：判断 `config.Cloud.Vendor` 选择 OSS/S3
2. **Redis TLS**：海外版需要 TLS 连接
3. **推送服务**：国内用阿里推送，海外用 FCM

## 必须项
- 日志包含 request_id / user_id / 关键 ids 与耗时
- 错误统一封装：业务错误 vs 系统错误
- 幂等：可能重复请求必须给策略（幂等 key / 去重 / 重放一致）
- L2+：必须 feature flag/version gate

## Checklist
- [ ] 参数校验（缺字段/格式错误/边界）
- [ ] 返回结构符合契约（并给"契约差异摘要"）
- [ ] 错误码稳定且含义清晰
- [ ] 关键日志与耗时（使用 log 包）
- [ ] 单测覆盖 service：happy path + >=2 异常
- [ ] 考虑国内/海外版差异
- [ ] MQ 消息处理幂等
- [ ] **存储选择**：优先 MongoDB，仅严格事务场景用 MySQL
- [ ] **枚举值从 1 开始**：不要用 0 作为有效值
- [ ] **CRUD 隔离**：增删改查限制在 mutation.go / repository.go
- [ ] **错误处理**：底层 WithStack 一次，上层不重复包
