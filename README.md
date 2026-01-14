# Claude Code 交付流程（v0.2）— 团队快速上手

## 我们在做什么
我们把"需求 -> 实现 -> 测试 -> 上线"变成一条可复制的黄金路径：
- 全员都可以用 Claude Code 做交付
- 但必须在统一工程底线内：向后兼容、可灰度、可回滚、可观测
- 用规则与闸门降低"非技术同学直接写代码"的线上风险

这不是"人人随便改代码"，而是"人人按同一套流程交付"。

---

## 为什么要这么做
Claude 很强，但会过度发挥。
我们用：
- `CLAUDE.md`（宪法）
- `skills/`（套路）
- `docs/`（事实与契约）
把发挥空间关进笼子里，让交付既快又稳。

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

/docs/README.md                         # docs 说明
/docs/spec_packet_template.md           # 需求规格包模板
/docs/rules_precedence.md               # 规则分层与冲突处理
/docs/approved_components.md            # 批准组件清单（黄金路径）
/docs/adr_template.md                   # 架构决策记录模板

/skills/README.md                       # skills 说明
/skills/skill_mr_gatekeeper_ready.md    # Gatekeeper 过闸 MR
/skills/skill_tech_discovery_and_decision.md  # 技术发现与决策
/skills/skill_backward_compat.md        # 向后兼容
/skills/skill_api_contract.md           # API 契约
/skills/skill_be_endpoint_go_gin.md     # Go/Gin 新接口
/skills/skill_db_change_mongo_mysql.md  # DB 变更
/skills/skill_testing.md                # 测试
/skills/skill_release_playbook.md       # 发布与灰度
/skills/skill_fe_guidelines.md          # 前端领域准则
```
