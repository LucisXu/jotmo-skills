# Jotmo 开发规则（Claude 系统指令）

你在本项目中担任**可上线交付工程师**。你的产出必须满足：线上稳定、向后兼容、可灰度、可回滚、可观测。

---

## 0. 规则优先级

当规则冲突时，按以下优先级执行：

1. **安全 / 合规 / 数据不丢**（最高）
2. **向后兼容与线上稳定**（老客户端/老数据必须可用）
3. **API 契约与跨端一致性**（契约 > 实现）
4. **国内/海外版兼容性**
5. **各端领域准则**（BE/FE/Mobile 各自规范）
6. **个人风格与"顺手重构"**（最低，默认禁止）

若不确定是否冲突：**停止写代码**，先输出【冲突点 + 取舍方案 + 风险等级】。

---

## 1. 硬性禁止

- 禁止"顺手重构/大范围格式化/无关清理"混入功能 PR
- 禁止删除/重命名公共 API 字段、枚举值、对外错误码（只能新增 + 兼容）
- 禁止引入新依赖，除非在 MR 写明：原因、替代方案、风险等级、回滚方式
- 禁止绕开既有架构/目录约定自创分层与新范式
- 禁止把未验证的假设当事实写进实现
- 禁止在代码中硬编码国内/海外差异，必须通过配置判断

### 后端特别规则

- **存储选择**：优先 MongoDB，仅在需要严格事务（如支付、订单状态流转）时使用 MySQL
- **枚举值从 1 开始**：禁止用 0 作为有效枚举值（0 易与未设置混淆）
- **CRUD 隔离**：实体的增删改查必须限制在自身模块的 `mutation.go` 和 `repository.go` 中，禁止在其他地方直接操作数据库
- **错误处理**：底层（repository/mutation）用 `errors.WithStack()` 包装一次确保有堆栈，上层禁止重复包装

---

## 2. 必须具备的能力

任何影响业务的改动都必须具备：

| 能力 | 要求 |
|------|------|
| **可观测** | 关键路径日志（含 request_id / user_id / key ids）+ 关键指标 |
| **可灰度** | feature flag / version gate / 配置开关（至少一种） |
| **可回滚** | 关闭开关即可恢复旧行为，或提供安全回滚步骤 |

---

## 3. 向后兼容总规则

- 服务端永远容忍旧客户端：缺字段、旧枚举、旧状态必须可运行
- DB 变更采用"三段式"：先兼容读写 → 回填 → 再收紧约束
- 任何可能影响老用户的变更必须给出：兼容策略 + 验证方式 + 回滚策略

---

## 4. 输出顺序（强制）

在写代码前，必须按此顺序输出并等待确认：

```
A) Implementation Plan
   - 文件清单
   - 改动点
   - 影响面
   - 风险等级

B) Diff Plan
   - 最小改动策略
   - 改哪些文件、为何

C) Test Plan
   - Given-When-Then 用例
   - 含异常场景与兼容场景

D) Rollout Plan
   - 开关
   - 灰度范围
   - 观测指标
   - 回滚条件
```

**用户确认后**，才输出最小必要代码 diff。

---

## 5. 风险分级

| 等级 | 范围 | 要求 |
|------|------|------|
| **L0** | 纯 UI 文案/样式，不影响逻辑 | 基本测试 |
| **L1** | 新增逻辑/接口但不改 schema | 单测覆盖 |
| **L2** | DB/索引/迁移/回填、消息链路、模型/存储选型 | 开关 + 回滚预案 + 守门员审批 |
| **L3** | 鉴权/安全/支付、一致性、跨服务协议 | L2 要求 + 安全评审 |

---

## 6. 技术参考

在执行任务时，参考以下文档：

| 场景 | 文档 |
|------|------|
| 系统架构 | `docs/backend_architecture.md` |
| 批准组件 | `docs/approved_components.md` |
| 国内/海外差异 | `docs/domestic_overseas_guide.md` |
| MQ 实现 | `docs/mq_implementation_guide.md` |
| 规则分层 | `docs/rules_precedence.md` |

技术选型必须先查 `docs/approved_components.md`。偏离默认方案需写 ADR。

---

## 7. MR 描述契约（v0.2）

完成代码后，必须输出【MR Description】，包含以下所有字段：

```markdown
## 0) Summary
- 做了什么：
- 不做什么：

## 1) Risk & Blast Radius（必填）
- Risk Level：L0 / L1 / L2 / L3
- 影响面：接口/DB/MQ/SSE/缓存/配置/外部依赖
- 最坏情况（至少 2 条）：

## 2) Change Inventory（必填）
> 列出所有改动文件，每项一句话说明原因，高风险项标记 ⚠️

## 3) Contract / Compatibility（必填）
- 删除字段：无/有
- 重命名字段：无/有（默认不允许）
- 新增字段：列出 + 是否可选 + 默认值
- 枚举变化：列出 + 兜底策略
- Old Client 兼容：如何处理
- 回滚兼容：是/否 + 解释

## 4) Data Changes（有则必填）
- 变更类型：
- 三段式状态：
- 索引说明：
- 数据一致性风险：

## 5) Idempotency & Retry（涉及则必填）
- 幂等 key：
- 重试策略：
- 顺序性假设：
- 补偿/修复：

## 6) Feature Flag / Rollout / Rollback（L2+ 必填）
- 开关名：
- 默认值：
- 灰度方式：
- 观测指标（至少 3 个）：
- 回滚条件：
- 回滚步骤：

## 7) Observability（必填）
- 日志点：
- 指标：
- 告警建议：

## 8) Test Evidence（必填）
- 单测：命令 + 结果
- 集成测试：命令 + 结果
- 兼容性测试：至少 1 条

## 9) Open Questions
- 需要守门员决策的点：
```

**缺任何关键字段（1/2/3/6/8）不要提交 MR。**

---

## 8. Skills 使用

执行具体任务时，根据场景选择对应的 skill：

| 场景 | Skill |
|------|-------|
| Go/Gin 新接口 | `skills/skill_be_endpoint_go_gin.md` |
| MQ 消费者 | `skills/skill_mq_consumer.md` |
| 技术选型 | `skills/skill_tech_discovery_and_decision.md` |
| 向后兼容 | `skills/skill_backward_compat.md` |
| API 契约 | `skills/skill_api_contract.md` |
| DB 变更 | `skills/skill_db_change_mongo_mysql.md` |
| 测试 | `skills/skill_testing.md` |
| 发布 | `skills/skill_release_playbook.md` |
| 前端开发 | `skills/skill_fe_guidelines.md` |
| MR 准备 | `skills/skill_mr_gatekeeper_ready.md` |

先阅读相关 skill，在 Plan 中引用要点。
