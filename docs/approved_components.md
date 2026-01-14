# 批准组件清单（Approved Components / Golden Path v0.2）
> 默认走"黄金路径"；偏离必须写 Decision Pack + ADR + Gatekeeper 审批（L2+）。

## 1) 通用规则
- 默认使用下列 Default 方案
- 只有命中"偏离触发条件"才允许使用 Alternatives
- 清单外组件/模型/基础设施：默认禁止（必须走 ADR）

## 2) 能力与默认选项（请按你们现状填）
### 2.1 语言与框架
- Default：Go（服务端）/ Flutter（客户端）/（你的 FE 框架）
- Alternatives：
- Banned：随意引入新框架/大一统重构

### 2.2 模型能力（按需保留）
- Embedding
  - Default：
  - Alternatives：
  - 偏离触发条件：多语言/特殊域/延迟/成本/数据不能出网

- ASR
  - Default：
  - Alternatives：
  - 偏离触发条件：口音/噪声/实时流式/成本/隐私

- LLM（生成/总结/对话）
  - Default：
  - Alternatives：
  - 偏离触发条件：强隐私/强时延/成本/QPS/合规

### 2.3 检索与存储
- 向量库
  - Default：
  - Alternatives：
  - 偏离触发条件：写入量/查询模式/多租户隔离/运维复杂度

- 结构化存储（交易/元数据）
  - Default：
  - Alternatives：

- 对象存储（音频/图片/附件）
  - Default：
  - Alternatives：

### 2.4 异步与实时
- MQ / 任务队列
  - Default：
  - Alternatives：
  - 偏离触发条件：吞吐/顺序性/延迟/一致性语义诉求

- 实时推送
  - Default：
  - Alternatives：
  - 偏离触发条件：双向/弱网/可靠性等级提升

## 3) 评测门槛（Quality Gates）
- 模型类：离线小评测集（>=20条代表性样本）+ 指标门槛（准确/延迟/成本）
- 存储类：基准压测（写/读/索引命中）+ 成本估算
- 协议类：契约测试 + 兼容测试（旧客户端/旧数据）

## 4) Owner（负责人）
- BE Gatekeeper：
- FE Gatekeeper：
- Mobile Gatekeeper：
- Platform/Infra：
