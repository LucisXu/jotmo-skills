# Skill：Go/Gin 新接口实现模板（v0.2）

## 目标
参数校验清晰、错误码一致、日志可追踪、可灰度可回滚。

## 分层建议
- handler：参数解析/校验/调用 service/返回 response
- service：业务逻辑（可单测）
- repo：数据访问（可 mock）

## 必须项
- 日志包含 request_id / user_id / 关键 ids 与耗时
- 错误统一封装：业务错误 vs 系统错误
- 幂等：可能重复请求必须给策略（幂等 key / 去重 / 重放一致）
- L2+：必须 feature flag/version gate

## checklist
- [ ] 参数校验（缺字段/格式错误/边界）
- [ ] 返回结构符合契约（并给"契约差异摘要"）
- [ ] 错误码稳定且含义清晰
- [ ] 关键日志与耗时
- [ ] 单测覆盖 service：happy path + >=2 异常
