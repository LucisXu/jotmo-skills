# docs/ 技术文档说明

这里存放**技术事实与契约文档**，供 Claude 和技术人员参考。

---

## 文档清单

| 文档 | 内容 | 谁需要看 |
|------|------|----------|
| `backend_architecture.md` | 系统架构、服务关系、技术栈 | 所有开发者 |
| `approved_components.md` | 批准的技术组件清单（黄金路径） | 需要技术选型时 |
| `domestic_overseas_guide.md` | 国内版/海外版差异与兼容指南 | 涉及云服务时 |
| `mq_implementation_guide.md` | RabbitMQ 实现规范 | 开发 MQ 消费者时 |
| `spec_packet_template.md` | 需求规格包模板 | 写需求文档时 |
| `adr_template.md` | 架构决策记录模板 | 偏离黄金路径时 |
| `rules_precedence.md` | 规则分层与冲突处理 | 遇到规则冲突时 |

---

## 文档原则

1. **只写事实**：可验证的技术信息，不写主观判断
2. **契约优先**：实现必须服从契约
3. **可审计**：契约变更必须有记录
4. **可回滚**：变更必须可逆

---

## 维护指南

### 什么时候更新

- 架构发生变化 → 更新 `backend_architecture.md`
- 引入新组件 → 更新 `approved_components.md`
- 国内/海外差异变化 → 更新 `domestic_overseas_guide.md`
- MQ 规范变化 → 更新 `mq_implementation_guide.md`

### 如何更新

1. 提 MR 说明变更原因
2. 通知相关团队成员
3. 如有必要，同步更新相关 skills
