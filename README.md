# Jotmo 后端开发指南（v0.3）— 团队快速上手

## 项目简介

Jotmo 是一款快记应用，本仓库包含后端开发的规范、技能包和文档，确保团队（包括非技术人员）能够通过 Claude Code 高效、安全地完成开发任务。

### 后端服务概览

| 服务分类 | 服务名 | 职责 |
|---------|--------|------|
| 核心业务 | jotmo-backend | 用户、会员、支付、云存储凭证 |
| | jotmo-mobile | 客户端内核（gomobile 编译） |
| | jotmo-data | 数据池，接收存储快记数据 |
| | jotmo-subject | 主题协作，多用户数据分享 |
| AI 智能 | jotmo-intelligent | AI 问答（问阿森）、年度总结 |
| | jotmo-emb | 向量化服务（Qwen3-Embedding） |
| | jotmo-keyword | 关键词提取（TF-IDF） |
| 音频处理 | jotmo-audio | 音频任务调度 |
| | jotmo-audio-sd | 说话人分离（pyannote） |
| | jotmo-audio-asr | 语音识别（SenseVoice） |
| 辅助服务 | jotmo-im | 消息处理与分发 |
| | jotmo-push | 推送服务（FCM/阿里） |
| | jotmo-location | 位置与天气服务 |

---

## 我们在做什么

我们把"需求 -> 实现 -> 测试 -> 上线"变成一条可复制的黄金路径：
- 全员都可以用 Claude Code 做交付
- 但必须在统一工程底线内：向后兼容、可灰度、可回滚、可观测
- 用规则与闸门降低"非技术同学直接写代码"的线上风险

这不是"人人随便改代码"，而是"人人按同一套流程交付"。

---

## 关键约定（必读）

### 1. 国内版 vs 海外版
项目同时支持国内（阿里云）和海外（AWS）部署：

| 特性 | 国内版 | 海外版 |
|-----|--------|--------|
| 文件存储 | 阿里云 OSS | AWS S3 |
| Redis | 标准连接 | **需要 TLS** |
| 推送 | 阿里云移动推送 | Firebase FCM |
| 配置标识 | `cloud.vendor: aliyun` | `cloud.vendor: aws` |

详见：`docs/domestic_overseas_guide.md`

### 2. MQ 实现规范
- **标准参考**：`jotmo-intelligent` 的 `xhy_record` 分支
- 必须实现断线重连
- 消息处理必须幂等

详见：`docs/mq_implementation_guide.md`

### 3. 数据安全
- 快记文本存储前需 zstd 压缩 + AES-GCM 加密

---

## 你从哪里开始（3 步）

1) 写 Spec Packet：`docs/spec_packet_template.md`
2) 在 Claude Code 里运行 Spec Interview：`CLAUDE_PROMPTS.md` 的"规格访谈"
3) Claude 先出 Plan，你确认后再写代码（遵守 `CLAUDE.md` 输出顺序）

---

## 标准工作流（从需求到上线）

### 1) Product Spec（任何人都能做）
- 写清：目标、用户路径、验收、约束
- 不写技术黑盒（接口/DB/模型/存储选型）

### 2) Spec Interview（避免脑补）
- 用提示词让 Claude 先追问，再整理 Product Spec + Tech Brief 草案

### 3) 风险分级
- L0：样式文案
- L1：业务逻辑/接口新增不改 schema
- L2：DB/迁移/回填、消息链路、模型/存储选型、兼容策略
- L3：鉴权/安全/一致性/跨服务协议

L2/L3 必须走 Gatekeeper 审批，并要求灰度/回滚/观测齐全。

### 4) 写代码（最小 diff）
- Claude 必须先输出：Implementation Plan → Diff Plan → Test Plan → Rollout Plan
- 你确认后才允许输出代码 diff
- 禁止顺手重构
- **注意国内/海外版兼容性**

### 5) 提 MR（强制过闸）
- 提 MR 前必须使用：`skills/skill_mr_gatekeeper_ready.md`
- MR 描述必须包含：`CLAUDE.md` 的 MR 描述契约 v0.2（含 Change Inventory）
- 缺关键字段：Gatekeeper 不进入代码级 review

### 6) 测试与上线
- 测试规则：`skills/skill_testing.md`
- 灰度发布：`skills/skill_release_playbook.md`

---

## 遇到"技术黑盒/选型"（模型、向量库、MQ、存储……）怎么办

走通用流程，不靠拍脑袋：
1) 使用 `skills/skill_tech_discovery_and_decision.md` 做 Tech Discovery Interview
2) 产出固定格式 Decision Pack（选项对比、权衡、评测、灰度回滚）
3) 对齐 `docs/approved_components.md`（黄金路径）
4) 偏离默认：必须写 ADR（`docs/adr_template.md`）并审批

---

## 维护原则（越用越强）

每次返工/事故/踩坑，问三个问题：
1) 是规则缺失？（加到 CLAUDE.md）
2) 是套路缺失？（加/补 skills）
3) 是事实/契约漂移？（补 docs，并尽量加契约测试）

---

## 目录结构

```text
/CLAUDE.md                              # 总纲（宪法）
/CLAUDE_PROMPTS.md                      # 起手式（提示词模板）
/README.md                              # 本文件

/docs/
├── README.md                           # docs 说明
├── backend_architecture.md             # 【新】后端架构文档
├── mq_implementation_guide.md          # 【新】MQ 实现规范
├── domestic_overseas_guide.md          # 【新】国内/海外版差异
├── approved_components.md              # 批准组件清单（黄金路径）
├── spec_packet_template.md             # 需求规格包模板
├── rules_precedence.md                 # 规则分层与冲突处理
└── adr_template.md                     # 架构决策记录模板

/skills/
├── README.md                           # skills 说明
├── skill_be_endpoint_go_gin.md         # Go/Gin 新接口（已更新）
├── skill_mq_consumer.md                # 【新】MQ 消费者开发
├── skill_mr_gatekeeper_ready.md        # Gatekeeper 过闸 MR
├── skill_tech_discovery_and_decision.md # 技术发现与决策
├── skill_backward_compat.md            # 向后兼容
├── skill_api_contract.md               # API 契约
├── skill_db_change_mongo_mysql.md      # DB 变更
├── skill_testing.md                    # 测试
├── skill_release_playbook.md           # 发布与灰度
└── skill_fe_guidelines.md              # 前端领域准则
```

---

## 快速参考

| 场景 | 参考文档 |
|------|---------|
| 了解系统架构 | `docs/backend_architecture.md` |
| 开发新接口 | `skills/skill_be_endpoint_go_gin.md` |
| 开发 MQ 消费者 | `skills/skill_mq_consumer.md` |
| 涉及国内/海外差异 | `docs/domestic_overseas_guide.md` |
| 技术选型 | `docs/approved_components.md` |
| 数据库变更 | `skills/skill_db_change_mongo_mysql.md` |
| 提交 MR | `skills/skill_mr_gatekeeper_ready.md` |
