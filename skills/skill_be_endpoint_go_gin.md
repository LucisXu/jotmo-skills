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

```go
// internal/users/mutation.go
func Create(ctx context.Context, user *User) error {
    return datastore.MySQL.Create(user).Error
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
