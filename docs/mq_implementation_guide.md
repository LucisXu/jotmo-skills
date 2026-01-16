# RabbitMQ 实现规范

> **重要**：MQ 实现以 `jotmo-intelligent` 的 `xhy_record` 分支为标准参考。

## 概述

本文档定义了 Jotmo 项目中 RabbitMQ 的标准实现方式，包括生产者、消费者、断线重连等关键功能。

## 核心组件

### 1. 连接管理 (rabbitMqHandle)

```go
type rabbitMqHandle struct {
    queueName    string
    exchangeName string
    exchangeKind string
    routingKey   string
    con          *amqp.Connection
    mu           sync.Mutex // 保护连接重连
}
```

### 2. 消费者配置 (StartOptions)

```go
type StartOptions struct {
    Concurrency       int           // 并发 worker 数
    PrefetchPerWorker int           // 每个 worker 的 QoS
    MaxRetryAttempts  int           // handler 失败重试次数
    RetryBaseDelay    time.Duration // 退避基准
    RetryMaxDelay     time.Duration // 退避最大
    NackRequeue       bool          // true=丢回队列；false=交给DLX/丢弃
    HandlerTimeout    time.Duration // 每条消息的处理超时，0=不限制
}
```

## 断线重连机制

### 生产者重连

```go
func (r *rabbitMqHandle) GetChannel() (*amqp.Channel, error) {
    r.mu.Lock()
    defer r.mu.Unlock()

    // 1) conn 丢了：重连
    if r.con == nil || r.con.IsClosed() {
        log.Println("RabbitMQ connection lost, attempting reconnect...")
        if err := r.reconnectLocked(); err != nil {
            return nil, err
        }
    }

    // 2) 开 channel（可能因为 conn 半断开而失败）
    ch, err := r.con.Channel()
    if err == nil {
        return ch, nil
    }

    // 3) 再重连一次再开
    log.Println("RabbitMQ open channel failed, reconnect and retry...", err)
    if err2 := r.reconnectLocked(); err2 != nil {
        return nil, err2
    }
    return r.con.Channel()
}
```

### 消费者重连

消费者使用外层无限循环实现重连：

```go
func (c *Consumer) Start(ctx context.Context, opts StartOptions, handler Handler) error {
    for {
        // 1) 连上 + declare
        if err := c.connectAndDeclare(); err != nil {
            log.Printf("rabbitmq connectAndDeclare failed: %v, retry in %s", err, c.reconnectDelay)
            if !sleepWithCtx(ctx, c.reconnectDelay) {
                return nil
            }
            continue
        }

        // 2) 跑一组 workers，任意异常 => 整组退出 => 重连
        runErr := c.runWorkers(ctx, opts, handler)

        // 3) 清理连接
        c.closeConn()

        if ctx.Err() != nil {
            return nil
        }

        log.Printf("consumer group stopped: %v, restart in %s", runErr, c.reconnectDelay)
        if !sleepWithCtx(ctx, c.reconnectDelay) {
            return nil
        }
    }
}
```

## 消息重试机制

```go
func handleWithRetry(ctx context.Context, handler Handler, msg amqp.Delivery, opts StartOptions) error {
    if err := handler(ctx, msg); err == nil {
        _ = msg.Ack(false)
        return nil
    }

    var lastErr error
    for attempt := 1; attempt <= opts.MaxRetryAttempts; attempt++ {
        // 指数退避
        d := opts.RetryBaseDelay * time.Duration(1<<uint(attempt-1))
        if d > opts.RetryMaxDelay {
            d = opts.RetryMaxDelay
        }
        if !sleepWithCtx(ctx, d) {
            return ctx.Err()
        }

        if err := handler(ctx, msg); err == nil {
            _ = msg.Ack(false)
            return nil
        } else {
            lastErr = err
            log.ErrorToSentry(ctx, "handler retry err", lastErr)
        }
    }

    _ = msg.Nack(false, opts.NackRequeue)
    return fmt.Errorf("handler failed after %d retries: %v", opts.MaxRetryAttempts, lastErr)
}
```

## 队列与 Exchange 定义

### 现有队列清单

| 队列名 | Exchange | 类型 | 用途 |
|--------|----------|------|------|
| jm.llm | jm.llm | direct | LLM 相关任务 |
| jm.coreRecordEvent | jm.coreRecordEvent | direct | 快记事件 |
| jm.embed | jm.embed | direct | 向量嵌入任务 |
| jm.sd | jm.sd | direct | 说话人分离任务 |
| jm.asr | jm.asr | direct | 语音识别任务 |

### 资源声明

```go
func declareResources(ch *amqp.Channel, queueName, exchangeName, exchangeKind, routingKey string) error {
    // 先声明 exchange（幂等）
    if err := ch.ExchangeDeclare(exchangeName, exchangeKind, true, false, false, false, nil); err != nil {
        return err
    }

    // queueName 为空：publish-only 模式，不创建队列/不绑定
    if queueName == "" {
        return nil
    }

    // 再声明 queue（幂等）
    if _, err := ch.QueueDeclare(queueName, true, false, false, false, nil); err != nil {
        return err
    }

    // 绑定 queue -> exchange
    if err := ch.QueueBind(queueName, routingKey, exchangeName, false, nil); err != nil {
        return err
    }

    return nil
}
```

## 最佳实践

### 1. 生产者

- 每次发送消息都创建新的 channel（channel 非线程安全）
- 发送完成后立即关闭 channel
- 使用 `PublishWithContext` 支持超时控制
- 消息类型通过 `amqp.Publishing.Type` 字段传递

```go
func (r *rabbitMqHandle) sendMsg(ctx context.Context, msgBody []byte, msgType string) error {
    channel, err := r.GetChannel()
    if err != nil {
        return err
    }
    defer channel.Close()

    return channel.PublishWithContext(
        ctx,
        r.exchangeName,
        r.routingKey,
        false, false,
        amqp.Publishing{
            ContentType: "text/plain",
            Type:        msgType,
            Body:        msgBody,
        })
}
```

### 2. 消费者

- 使用 QoS 控制预取数量，避免消息积压
- 每个 worker 独立 channel
- 使用 `NotifyClose` 监听连接/channel 关闭事件
- 优雅退出：响应 context 取消

```go
// 启动消费者示例
consumer := mq.NewConsumer(dsn, "jm.llm", "jm.llm", "direct", "jm.llm")
go consumer.Start(ctx, mq.StartOptions{
    Concurrency:       10,
    PrefetchPerWorker: 5,
    MaxRetryAttempts:  3,
    RetryBaseDelay:    time.Second,
    RetryMaxDelay:     30 * time.Second,
    NackRequeue:       true,
    HandlerTimeout:    5 * time.Minute,
}, handleMessage)
```

### 3. 消息处理

- 处理函数必须幂等
- 失败时记录到 Sentry
- 合理设置重试次数和超时时间
- 使用 Ack/Nack 显式确认

## 配置示例

```yaml
rabbitmq:
  host: "rabbitmq.example.com"
  port: "5672"
  name: "username"
  password: "password"
  vhost: "/"
```

## 监控指标

建议监控以下指标：

1. **队列积压数**：消息堆积告警
2. **消费速率**：处理能力
3. **重试次数**：异常率
4. **连接数**：连接池状态
5. **Nack 率**：处理失败率

## 注意事项

1. **幂等性**：消息可能重复投递，handler 必须幂等
2. **顺序性**：RabbitMQ 不保证全局顺序，同一 consumer 内顺序处理
3. **持久化**：queue 和 exchange 都声明为 durable
4. **资源清理**：优雅退出时等待消息处理完成
5. **错误处理**：区分可重试错误和不可重试错误
