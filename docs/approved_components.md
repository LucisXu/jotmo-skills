# 批准组件清单（Approved Components / Golden Path v0.2）
> 默认走"黄金路径"；偏离必须写 Decision Pack + ADR + Gatekeeper 审批（L2+）。

## 1) 通用规则
- 默认使用下列 Default 方案
- 只有命中"偏离触发条件"才允许使用 Alternatives
- 清单外组件/模型/基础设施：默认禁止（必须走 ADR）

## 2) 能力与默认选项

### 2.1 语言与框架
- Default：
  - **Go 1.22+**（后端服务）
  - **Python 3.10+**（AI/ML 服务）
  - **Flutter**（客户端）
  - **Gin**（Go HTTP 框架）
  - **FastAPI**（Python HTTP 框架）
- Alternatives：无
- Banned：随意引入新框架/大一统重构

### 2.2 模型能力

- **Embedding（文本向量化）**
  - Default：**Qwen3-Embedding-0.6B**（1024 维，本地部署）
  - Alternatives：OpenAI text-embedding-3-small
  - 偏离触发条件：多语言/特殊域/延迟/成本/数据不能出网

- **ASR（语音识别）**
  - Default：**SenseVoice**（sherpa-onnx，支持中/英/日/韩/粤）
  - Alternatives：阿里云 ASR、Whisper
  - 偏离触发条件：口音/噪声/实时流式/成本/隐私

- **说话人分离（Speaker Diarization）**
  - Default：**pyannote.audio**
  - Alternatives：无
  - 偏离触发条件：精度要求/多人会议/实时性

- **LLM（生成/总结/对话）**
  - Default：根据业务选择（通过配置指定）
  - Alternatives：OpenAI GPT-4、Claude、通义千问
  - 偏离触发条件：强隐私/强时延/成本/QPS/合规

### 2.3 检索与存储

- **向量库**
  - Default：**Milvus 2.4+**（阿里云版/自建）
  - Alternatives：Pinecone、Qdrant
  - 偏离触发条件：写入量/查询模式/多租户隔离/运维复杂度

- **结构化存储（交易/元数据）**
  - Default：**MySQL 8.0+**
  - Alternatives：PostgreSQL
  - 用途：用户、订单、会员、配置等

- **文档存储**
  - Default：**MongoDB**
  - Alternatives：无
  - 用途：快记数据、AI 会话、音频处理记录

- **缓存**
  - Default：**Redis**
  - Alternatives：无
  - 注意：海外版 AWS ElastiCache 需要 TLS

- **对象存储（音频/图片/附件）**
  - Default：
    - 国内：**阿里云 OSS**
    - 海外：**AWS S3**
  - Alternatives：无

### 2.4 异步与实时

- **MQ / 任务队列**
  - Default：**RabbitMQ**（direct exchange）
  - Alternatives：Kafka（高吞吐场景）
  - 偏离触发条件：吞吐/顺序性/延迟/一致性语义诉求
  - 实现参考：`jotmo-intelligent` 服务的 `pkg/mq/` 目录

- **实时推送（客户端）**
  - Default：
    - 国内：**阿里云移动推送**
    - 海外：**Firebase Cloud Messaging (FCM)**
  - Alternatives：APNs（iOS 单独推送）
  - 偏离触发条件：双向/弱网/可靠性等级提升

- **实时流式输出（AI 响应）**
  - Default：**SSE (Server-Sent Events)**
  - Alternatives：WebSocket
  - 注意：SSE 接口需排除 gzip 压缩

### 2.5 监控与追踪

- **错误监控**
  - Default：**Sentry**
  - Alternatives：无

- **链路追踪**
  - Default：**OpenTelemetry**
  - Alternatives：Jaeger

- **日志**
  - Default：标准日志 + Sentry
  - 必须包含：request_id、user_id、关键 ids

### 2.6 外部服务

- **地图/逆地理编码**
  - Default：**天地图 API**
  - Alternatives：高德地图、Google Maps

- **天气服务**
  - Default：**Apple Weather Kit**
  - Alternatives：和风天气

- **支付**
  - Default：
    - 国内：**支付宝、微信支付**
    - 海外：**Apple IAP、Google Play Billing**
  - Alternatives：Stripe

- **短信**
  - Default：**阿里云短信**
  - Alternatives：Twilio（海外）

- **内容审核**
  - Default：**阿里云内容安全**
  - Alternatives：无

## 3) 评测门槛（Quality Gates）
- 模型类：离线小评测集（>=20条代表性样本）+ 指标门槛（准确/延迟/成本）
- 存储类：基准压测（写/读/索引命中）+ 成本估算
- 协议类：契约测试 + 兼容测试（旧客户端/旧数据）

## 4) 配置管理
- Default：**Viper**（Go 服务）
- 配置文件位置：`assets/config/config.yaml`
- 敏感配置：通过环境变量或密钥管理服务

## 5) Owner（负责人）
- BE Gatekeeper：（待填）
- FE Gatekeeper：（待填）
- Mobile Gatekeeper：（待填）
- Platform/Infra：（待填）
