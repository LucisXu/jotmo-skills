# Jotmo 后端架构文档

## 系统总览

Jotmo 是一款快记应用，后端由 15 个微服务组成，涵盖用户管理、数据同步、AI 智能、音频处理等功能。

## 服务架构图

```
                                    用户设备（iOS/Android/PC）
                                              │
                              ┌───────────────┼───────────────┐
                              │               │               │
                              ▼               ▼               ▼
                    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
                    │ jotmo-mobile │  │jotmo-backend │  │ jotmo-subject│
                    │  (客户端内核) │  │  (主后端服务) │  │  (主题服务)   │
                    └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                           │                 │                 │
         ┌─────────────────┼─────────────────┼─────────────────┼─────────────────┐
         │                 │                 │                 │                 │
         ▼                 ▼                 ▼                 ▼                 ▼
┌────────────────┐ ┌────────────────┐ ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│  jotmo-data    │ │jotmo-location  │ │ jotmo-keyword  │ │   jotmo-im     │ │  jotmo-push    │
│ (数据池服务)    │ │ (位置天气服务) │ │ (关键词服务)   │ │  (消息服务)    │ │  (推送服务)    │
└───────┬────────┘ └────────────────┘ └────────────────┘ └───────┬────────┘ └───────┬────────┘
        │ MQ                                                     │ MQ              │
        ▼                                                        │                 │
┌────────────────┐                                               │                 ▼
│   jotmo-emb    │                                               │         ┌───────────────┐
│ (向量化服务)    │                                               │         │  FCM/阿里推送  │
└────────────────┘                                               │         └───────────────┘
        │                                                        │
        ▼                                                        │
┌────────────────┐     ┌─────────────────────────────────────────┘
│jotmo-intelligent│ <───┘
│  (AI 智能服务)  │
└────────────────┘

                              音频处理链路
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
┌────────────────┐      ┌────────────────┐      ┌────────────────┐
│  jotmo-audio   │─MQ──>│ jotmo-audio-sd │─MQ──>│jotmo-audio-asr │
│ (音频管理服务)  │      │ (说话人分离)    │      │  (语音识别)    │
└────────────────┘      └────────────────┘      └────────────────┘

┌────────────────┐      ┌────────────────┐
│   jotmo-asr    │      │   jotmo-bury   │
│ (离线语音识别)  │      │  (埋点服务)    │
└────────────────┘      └────────────────┘
```

## 服务职责说明

### 核心业务服务

| 服务名 | 语言 | 端口 | 职责 |
|-------|------|------|------|
| jotmo-backend | Go | 8080 | 主后端服务：用户管理、会员、支付、云存储凭证 |
| jotmo-mobile | Go | 8082 | 客户端内核：多端数据同步、本地存储、文件管理 |
| jotmo-data | Go | 8080 | 数据池服务：接收存储快记数据，分发 MQ 消息 |
| jotmo-subject | Go | 8080 | 主题服务：多用户协作、数据分享、权限控制 |

### AI 智能服务

| 服务名 | 语言 | 端口 | 职责 |
|-------|------|------|------|
| jotmo-intelligent | Go | 8080 | AI 问答服务（问阿森）、年度总结、SSE 流式输出 |
| jotmo-emb | Python | 8001 | 向量化服务：Qwen3-Embedding 模型、Milvus 存储 |
| jotmo-keyword | Go | 8080 | 关键词提取：TF-IDF、TextRank 算法 |

### 音频处理服务

| 服务名 | 语言 | 端口 | 职责 |
|-------|------|------|------|
| jotmo-audio | Go | 8080 | 音频管理：任务调度、进度追踪、结果整合 |
| jotmo-audio-sd | Python | - | 说话人分离：pyannote.audio 模型 |
| jotmo-audio-asr | Go | - | 音频语音识别：SenseVoice 模型 |
| jotmo-asr | Go | - | 离线语音识别：SenseVoice 模型 |

### 辅助服务

| 服务名 | 语言 | 端口 | 职责 |
|-------|------|------|------|
| jotmo-im | Go | 8080 | 消息服务：MQ 消费、通知分发 |
| jotmo-push | Go | 8080 | 推送服务：FCM（海外）、阿里云推送（国内） |
| jotmo-location | Go | 8080 | 位置天气：天地图逆地理编码、Apple Weather Kit |
| jotmo-bury | Go | 8080 | 埋点服务：用户行为数据收集 |

## 技术栈总览

### 编程语言
- **Go 1.22+**: 主要后端服务（12 个服务）
- **Python 3.10+**: AI/ML 服务（jotmo-emb、jotmo-audio-sd）

### 框架
- **Gin**: Go HTTP 框架（所有 Go 服务）
- **FastAPI**: Python HTTP 框架（jotmo-emb）

### 数据存储
- **MySQL 8.0+**: 关系数据（用户、订单、会员等）
- **MongoDB**: 文档数据（快记、AI 会话等）
- **Redis**: 缓存、限流、分布式锁
- **Milvus 2.4+**: 向量数据库（语义搜索）

### 消息队列
- **RabbitMQ**: 异步消息（所有服务间通信）

### 文件存储
- **阿里云 OSS**: 国内版文件存储
- **AWS S3**: 海外版文件存储

### AI 模型
- **Qwen3-Embedding-0.6B**: 文本向量化（1024 维）
- **SenseVoice**: 多语言语音识别（中/英/日/韩/粤）
- **pyannote.audio**: 说话人分离

### 监控与追踪
- **Sentry**: 错误监控
- **OpenTelemetry**: 链路追踪

## 数据流向

### 快记数据流
```
客户端 → jotmo-mobile → OSS/S3（文件）
                     → jotmo-data（元数据）→ MongoDB
                                         → MQ → jotmo-emb → Milvus
```

### AI 问答流
```
客户端 → jotmo-intelligent → jotmo-data（获取上下文）
                          → jotmo-emb（语义搜索）
                          → LLM API → SSE → 客户端
```

### 音频处理流
```
客户端 → OSS（上传音频）
      → jotmo-audio（接收通知）
      → MQ → jotmo-audio-sd（说话人分离）
      → MQ → jotmo-audio-asr（语音识别）
      → MongoDB（存储结果）
```

### 消息推送流
```
业务服务 → MQ → jotmo-im（消息处理）
              → jotmo-push（推送分发）
              → FCM/阿里云推送 → 用户设备
```

## 服务依赖关系

### 被依赖最多的服务
1. **jotmo-backend**: 用户认证、会员验证
2. **jotmo-data**: 数据来源
3. **jotmo-emb**: 向量搜索

### 依赖链路
```
jotmo-mobile ──→ jotmo-backend（认证）
             ──→ jotmo-data（数据上传）
             ──→ jotmo-subject（主题同步）
             ──→ jotmo-location（位置天气）

jotmo-data ────→ jotmo-emb（MQ）

jotmo-intelligent ──→ jotmo-data（查询）
                  ──→ jotmo-emb（搜索）

jotmo-subject ──→ jotmo-im（MQ）
              ──→ jotmo-backend（认证）

jotmo-im ──────→ jotmo-push（推送）
```

## 配置管理

所有服务使用 Viper 进行配置管理，配置文件位于 `assets/config/config.yaml`。

### 配置文件结构
```yaml
debug-mode: false

# 数据库配置
data-mysql:
  host: "mysql.host"
  port: "3306"
  ...

data-mongo:
  url: "mongodb://..."
  db: "database-name"

data-redis:
  host: "redis.host"
  port: "6379"
  tls-status: "1"  # AWS Redis 需要开启
  ...

# 消息队列
rabbitmq:
  host: "rabbitmq.host"
  ...

# 云服务商
cloud:
  vendor: "aliyun"  # aliyun 或 aws

# JWT
access-token-secret-key: "..."
```

## 部署架构

### 国内版
- 云服务商：阿里云
- 区域：杭州
- 存储：OSS
- 推送：阿里云移动推送
- Redis：标准版（无需 TLS）

### 海外版
- 云服务商：AWS
- 区域：us-west-2
- 存储：S3
- 推送：FCM
- Redis：ElastiCache（需要 TLS）
