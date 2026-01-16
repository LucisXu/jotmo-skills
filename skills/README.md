# skills/ 执行套路库

这里存放**各场景的具体执行方法**，Claude 在写代码前会先阅读相关 skill。

**适用对象**：团队中的任何人。不管你之前是什么角色，都可以使用这些 skills 完成开发任务。

---

## 使用方式

1. Claude 阅读 `CLAUDE.md`（规则）
2. 根据任务选择相关 skill
3. 按 skill 中的 checklist 输出
4. 最后输出符合规范的 MR Description
5. 提交给守门员审核

---

## Skills 索引

### 通用 Skills

| Skill | 用途 |
|-------|------|
| `skill_mr_gatekeeper_ready.md` | 准备符合规范的 MR 描述 |
| `skill_tech_discovery_and_decision.md` | 技术选型和决策 |
| `skill_backward_compat.md` | 向后兼容策略 |
| `skill_api_contract.md` | API 契约规范（前后端共用） |
| `skill_testing.md` | 测试规范 |
| `skill_release_playbook.md` | 发布与灰度 |

### 后端相关 Skills

| Skill | 用途 |
|-------|------|
| `skill_be_endpoint_go_gin.md` | Go/Gin 新接口开发 |
| `skill_mq_consumer.md` | MQ 消费者开发 |
| `skill_db_change_mongo_mysql.md` | 数据库变更 |

### 前端相关 Skills

| Skill | 用途 |
|-------|------|
| `skill_fe_guidelines.md` | 前端开发准则 |
| `skill_fe_component.md` | 组件开发规范 |
| `skill_fe_state.md` | 状态管理规范 |
| `skill_fe_api.md` | 前端 API 调用规范 |

---

## 如何选择 Skill

| 我要做的事 | 用这个 Skill |
|-----------|-------------|
| 写新接口 | `skill_be_endpoint_go_gin.md` |
| 写 MQ 消费者 | `skill_mq_consumer.md` |
| 改数据库 | `skill_db_change_mongo_mysql.md` |
| 写新组件/页面 | `skill_fe_component.md` + `skill_fe_guidelines.md` |
| 处理状态管理 | `skill_fe_state.md` |
| 调用后端 API | `skill_fe_api.md` |
| 技术选型 | `skill_tech_discovery_and_decision.md` |
| 准备 MR | `skill_mr_gatekeeper_ready.md` |
| 处理兼容性 | `skill_backward_compat.md` |

**提示**：如果功能涉及前后端，需要同时阅读相关的 skills。

---

## 守门员关注点

作为守门员审核代码时，重点关注：

| 守门员类型 | 重点检查 |
|-----------|---------|
| **后端守门员** | 架构符合度、API 契约、数据库变更、性能、安全 |
| **前端守门员** | 组件规范、状态管理、用户体验、边界处理 |

---

## 维护指南

### 什么时候新增 Skill

- 发现某类任务反复出现相同问题
- 有新的最佳实践需要固化
- 守门员发现的共性问题需要沉淀

### 如何新增 Skill

1. 复制现有 skill 作为模板
2. 包含：目标、步骤、checklist、示例
3. 提 MR 说明新增原因
4. 更新本文档的索引
