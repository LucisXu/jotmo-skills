# 国内版与海外版差异指南

## 概述

Jotmo 同时支持国内版（阿里云）和海外版（AWS），在开发时需要考虑两者的差异，确保代码兼容两种部署环境。

## 云服务商配置

通过 `cloud.vendor` 配置项区分：

```yaml
cloud:
  vendor: "aliyun"  # 国内版
  # 或
  vendor: "aws"     # 海外版
```

## 主要差异点

### 1. 文件存储

| 特性 | 国内版 (OSS) | 海外版 (S3) |
|------|-------------|-------------|
| 服务商 | 阿里云 OSS | AWS S3 |
| 认证方式 | AccessKey + STS | AccessKey + AssumeRole |
| 端点格式 | oss-cn-hangzhou.aliyuncs.com | s3.us-west-2.amazonaws.com |
| 凭证有效期 | 1小时 | 1小时 |

#### 配置示例

**国内版 (OSS)**
```yaml
aliyun:
  access-key-id: "your-access-key-id"
  access-secret: "your-access-secret"
  arn: "acs:ram::xxx:role/xxx"
  endpoint: "oss-cn-hangzhou.aliyuncs.com"
  session-name: "jotmo-sts"
  data-bucket: "jotmo-userdata"
  file-bucket: "jotmo-files"
```

**海外版 (S3)**
```yaml
aws:
  acc-id: "your-access-key-id"
  secret: "your-secret-access-key"
  region: "us-west-2"
  role-arn: "arn:aws:iam::xxx:role/xxx"
  role-session-name: "jotmo-sts"
  data-bucket: "jotmo-userdata"
  file-bucket: "jotmo-files"
```

### 2. Redis 配置

海外版 AWS ElastiCache 需要 TLS 加密连接。

| 特性 | 国内版 | 海外版 (AWS) |
|------|--------|-------------|
| TLS | 不需要 | **必须开启** |
| 端口 | 6379 | 6379 |
| 证书 | - | 使用系统 CA |

#### 配置示例

**国内版**
```yaml
data-redis:
  host: "redis.internal"
  port: "6379"
  password: "password"
  db: "0"
  tls-status: "0"  # 不开启 TLS
```

**海外版**
```yaml
data-redis:
  host: "redis.xxx.cache.amazonaws.com"
  port: "6379"
  password: "password"
  db: "0"
  tls-status: "1"  # 必须开启 TLS
```

#### 代码实现

```go
func initRedis() *redis.Client {
    option := &redis.Options{
        Addr:     config.Addr(),
        Password: config.Password,
        DB:       config.DB,
    }

    // 海外版需要 TLS
    if config.UseTls() {
        caPool, _ := x509.SystemCertPool()
        option.TLSConfig = &tls.Config{
            RootCAs:    caPool,           // 系统默认 CA
            MinVersion: tls.VersionTLS12, // AWS 要求 TLS 1.2+
            ServerName: config.Host,      // 验证服务器域名
        }
    }

    return redis.NewClient(option)
}
```

### 3. 推送服务

| 特性 | 国内版 | 海外版 |
|------|--------|--------|
| Android 推送 | 阿里云移动推送 | Firebase Cloud Messaging |
| iOS 推送 | 阿里云移动推送 | Firebase Cloud Messaging |

#### 配置示例

**国内版**
```yaml
alipush:
  access-key-id: "your-access-key-id"
  access-key-secret: "your-access-key-secret"
  android-app-key: "android-key"
  ios-app-key: "ios-key"
```

**海外版**
```yaml
firebase:
  credentials-file: "/path/to/firebase-credentials.json"
```

### 4. 向量数据库

| 特性 | 国内版 | 海外版 |
|------|--------|--------|
| 服务 | 阿里云 Milvus | 自建 Milvus / AWS |
| 认证 | Token | Token |

配置通过环境变量：
```bash
CLOUD_VENDOR=aliyun  # 或 aws
MILVUS_URI=http://milvus.host:19530
MILVUS_TOKEN=root:password
```

### 5. 短信服务

| 特性 | 国内版 | 海外版 |
|------|--------|--------|
| 服务商 | 阿里云短信 | AWS SNS / Twilio |
| 签名 | 需要备案 | 不需要 |

## 代码兼容性设计

### 1. 使用配置判断

```go
func GetStorageClient() StorageClient {
    switch config.Config.Cloud.Vendor {
    case "aliyun":
        return NewOSSClient(config.Config.AliyunOss)
    case "aws":
        return NewS3Client(config.Config.AwsS3)
    default:
        panic("unsupported cloud vendor")
    }
}
```

### 2. 抽象接口

定义统一接口，不同实现：

```go
type StorageClient interface {
    GetPresignedURL(bucket, key string, expiry time.Duration) (string, error)
    PutObject(bucket, key string, data io.Reader) error
    GetObject(bucket, key string) (io.ReadCloser, error)
    DeleteObject(bucket, key string) error
}

// 阿里云实现
type OSSClient struct { ... }

// AWS 实现
type S3Client struct { ... }
```

### 3. 工厂模式

```go
// vector_store/factory.go
func NewVectorStore() VectorStore {
    switch os.Getenv("CLOUD_VENDOR") {
    case "aliyun":
        return NewAliyunMilvus()
    case "aws":
        return NewAWSMilvus()
    default:
        return NewAliyunMilvus()
    }
}
```

## 开发检查清单

在开发涉及以下功能时，需要考虑国内/海外差异：

- [ ] **文件上传/下载**：OSS vs S3 API 差异
- [ ] **Redis 连接**：TLS 配置
- [ ] **推送通知**：阿里推送 vs FCM
- [ ] **短信发送**：阿里云短信 vs SNS
- [ ] **地图服务**：天地图 vs Google Maps
- [ ] **支付**：支付宝/微信 vs Stripe
- [ ] **合规**：数据存储位置、隐私政策

## 测试建议

1. **单元测试**：Mock 存储接口
2. **集成测试**：分别在国内/海外环境测试
3. **配置测试**：验证两种配置都能正确加载

## 常见问题

### Q: 如何判断当前是国内版还是海外版？
```go
func IsOverseas() bool {
    return config.Config.Cloud.Vendor == "aws"
}
```

### Q: Redis 连接超时怎么办？
海外版 Redis 必须开启 TLS，检查 `tls-status` 配置是否为 "1"。

### Q: 文件上传失败？
检查对应云服务商的凭证配置是否正确，Bucket 是否存在。
