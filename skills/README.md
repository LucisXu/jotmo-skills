# skills/ 使用说明（v0.2）

skills 是"作业套路库"。Claude 写代码前必须：
1) 阅读 CLAUDE.md
2) 选择并阅读相关 skills（在 Plan 中引用要点）
3) 按 skills checklist 输出实现、测试、发布计划
4) 最后输出符合 MR 描述契约 v0.2 的【MR Description】

推荐选择：
- Gatekeeper 过闸与 MR 描述：skill_mr_gatekeeper_ready
- 技术黑盒/选型/不确定性：skill_tech_discovery_and_decision
- 向后兼容：skill_backward_compat
- API 契约：skill_api_contract
- Go/Gin 新接口：skill_be_endpoint_go_gin
- DB 变更：skill_db_change_mongo_mysql
- 测试：skill_testing
- 发布：skill_release_playbook
- 前端规范：skill_fe_guidelines
