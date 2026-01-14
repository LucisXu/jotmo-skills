# Skill：Tech Discovery & Decision（通用黑盒收敛 v0.2）

适用：任何涉及"技术黑盒/选型/不确定性"的需求。
例如：模型能力（ASR/Embedding/LLM/TTS/OCR）、向量库/检索方案、MQ/实时链路、存储选择、鉴权与隔离策略。

---

## 0) 触发条件（命中任意一条 -> 至少 L2）
- 引入/更换模型能力（或部署方式）
- 引入/更换存储（含向量库/对象存储/缓存）或改变职责边界
- DB schema/索引/迁移/回填
- 协议字段/枚举/错误码变更
- 实时链路/异步链路改动（SSE/WS/TCP/MQ）
- 权限、鉴权、数据隔离、安全审计

命中触发条件：必须先产出 Decision Pack，并标记 Gatekeeper 决策点。

---

## 1) Tech Discovery Interview（强制先问清约束）
Claude 必须先提出关键追问，覆盖：
- 数据与隐私（能否出网、敏感级别、保留策略）
- 质量定义（什么叫好、不可接受失败）
- 延迟预算（端到端、p95/p99）
- 成本与规模（QPS/日量、GPU/CPU、是否自部署）
- 可靠性与降级（允许降级吗、降级体验）
- 集成与运维（部署形态、可观测、告警）
- 兼容与回滚（旧客户端/旧数据/回滚读写）
- 评测方法（数据集、指标、门槛）

---

## 2) Decision Pack（固定输出结构，禁止散文发挥）
### Decision Pack
- Problem：
- Constraints：
- Options：
  - Option A（Default / Golden Path）：
  - Option B（Alternative）：
  - Option C（Alternative，若有）：
- Trade-offs（质量/延迟/成本/复杂度/风险）：
- Recommendation（推荐哪一个 + 理由）：
- Evaluation（怎么评测：数据集、指标、门槛、如何复现）：
- Rollout（开关、灰度、观测指标、回滚条件与步骤）：
- Open Questions（必须由 Gatekeeper 决策的点）：

---

## 3) 与黄金路径对齐（必须检查）
- 先查 docs/approved_components.md：
  - 若 Recommendation = Default：继续
  - 若偏离 Default：必须写 ADR（docs/adr_template.md）并走 Gatekeeper 审批

---

## 4) 交付要求
- 后续提交 MR 时，必须使用 skill_mr_gatekeeper_ready，确保 MR 描述可过闸。
