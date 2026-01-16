# Skill：MQ 消费者开发规范

> **重要**：MQ 实现以 `jotmo-intelligent` 的 `xhy_record` 分支为标准参考。

## 目标

实现可靠的消息消费者，具备断线重连、重试机制、优雅退出能力。

## 消费者创建

### 1. 基础消费者

```go
import "your-service/pkg/mq"

// 创建消费者
consumer := mq.NewConsumer(
    config.Config.RabbitMqConfig.DSN(), // DSN
    "jm.your-queue",                     // 队列名
    "jm.your-exchange",                  // Exchange 名
    "direct",                            // Exchange 类型
    "jm.your-routing-key",               // Routing Key
)
```

### 2. 临时队列消费者（如 fanout 广播）

```go
consumer := mq.NewConsumerWithQueueOpt(
    dsn,
    "",            // 空字符串 = 服务端自动生成队列名
    "jm.broadcast",
    "fanout",
    "",
    mq.QueueDeclareOptions{
        Durable:    false,
        AutoDelete: true,
        Exclusive:  true,
    },
)
```

## 启动消费者

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 启动消费者
    go consumer.Start(ctx, mq.StartOptions{
        Concurrency:       10,             // 并发 worker 数
        PrefetchPerWorker: 5,              // 每个 worker 的 QoS
        MaxRetryAttempts:  3,              // 重试次数
        RetryBaseDelay:    time.Second,    // 重试基准延迟
        RetryMaxDelay:     30 * time.Second, // 重试最大延迟
        NackRequeue:       true,           // 失败后重新入队
        HandlerTimeout:    5 * time.Minute, // 处理超时
    }, handleMessage)

    // 等待退出信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    cancel()
}
```

## 消息处理函数

```go
func handleMessage(ctx context.Context, msg amqp.Delivery) error {
    // 1. 解析消息
    var data YourMessageType
    if err := json.Unmarshal(msg.Body, &data); err != nil {
        log.Printf("unmarshal failed: %v", err)
        return nil // 返回 nil 表示 Ack（无法解析的消息不重试）
    }

    // 2. 根据消息类型处理
    switch msg.Type {
    case "type_a":
        return handleTypeA(ctx, data)
    case "type_b":
        return handleTypeB(ctx, data)
    default:
        log.Printf("unknown message type: %s", msg.Type)
        return nil
    }
}
```

## 消息发送

### 1. 发送消息

```go
import "your-service/pkg/mq"

// 使用已初始化的 MQ handle
err := mq.LlmMq.SendMsg(ctx, msgBody, mq.LlmMsgType("your_msg_type"))
if err != nil {
    log.ErrorToSentry(ctx, "send mq failed", err)
    return err
}
```

### 2. 消息结构

```go
type YourMessage struct {
    ID        string `json:"id"`
    UserID    int64  `json:"user_id"`
    Timestamp int64  `json:"timestamp"`
    Data      any    `json:"data"`
}
```

## 幂等性设计

消息可能重复投递，必须保证幂等性：

### 1. 使用唯一标识检查

```go
func handleMessage(ctx context.Context, msg amqp.Delivery) error {
    var data Message
    json.Unmarshal(msg.Body, &data)

    // 检查是否已处理
    processed, _ := redis.Get(ctx, "msg:"+data.ID)
    if processed != "" {
        return nil // 已处理，直接 Ack
    }

    // 处理业务逻辑
    if err := process(ctx, data); err != nil {
        return err
    }

    // 标记已处理
    redis.Set(ctx, "msg:"+data.ID, "1", 24*time.Hour)
    return nil
}
```

### 2. 数据库唯一约束

```go
func handleMessage(ctx context.Context, msg amqp.Delivery) error {
    var data Message
    json.Unmarshal(msg.Body, &data)

    // Upsert 操作
    result := db.Clauses(clause.OnConflict{
        Columns:   []clause.Column{{Name: "unique_id"}},
        DoUpdates: clause.AssignmentColumns([]string{"updated_at"}),
    }).Create(&record)

    return result.Error
}
```

## 错误处理

### 1. 可重试错误

```go
// 网络错误、临时故障 - 返回 error 触发重试
if errors.Is(err, io.EOF) || errors.Is(err, context.DeadlineExceeded) {
    return err
}
```

### 2. 不可重试错误

```go
// 数据格式错误、业务逻辑错误 - 返回 nil 直接 Ack
if errors.Is(err, ErrInvalidData) {
    log.Printf("invalid data, skip: %v", err)
    return nil
}
```

### 3. 记录到 Sentry

```go
if err != nil {
    log.ErrorToSentry(ctx, "handle message failed", err)
    return err
}
```

## 配置参数说明

| 参数 | 说明 | 建议值 |
|------|------|--------|
| Concurrency | 并发 worker 数 | 根据 CPU 核数，一般 10-20 |
| PrefetchPerWorker | 每个 worker 预取数 | 1-10，防止消息积压在内存 |
| MaxRetryAttempts | 最大重试次数 | 3-5 |
| RetryBaseDelay | 重试基准延迟 | 1s |
| RetryMaxDelay | 重试最大延迟 | 30s |
| NackRequeue | 失败后重新入队 | true（除非有死信队列） |
| HandlerTimeout | 处理超时时间 | 根据业务，1-10 分钟 |

## Checklist

- [ ] 消息处理幂等
- [ ] 错误区分可重试/不可重试
- [ ] 关键日志包含 message_id、user_id
- [ ] 异常上报到 Sentry
- [ ] 优雅退出（响应 context 取消）
- [ ] 合理的重试策略
- [ ] 监控消息积压告警

## 常见问题

### Q: 消费者突然停止工作？
检查 RabbitMQ 连接是否断开，消费者会自动重连，查看日志中的重连记录。

### Q: 消息处理很慢？
1. 增加 Concurrency
2. 检查是否有慢查询
3. 考虑异步处理

### Q: 消息丢失？
1. 确保 queue 声明为 durable
2. 确保消息处理完成后才 Ack
3. 检查 NackRequeue 是否开启
