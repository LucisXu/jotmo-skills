# Claude Code 任务起手式（v0.2）
> 目标：先收敛不确定性，再写最小 diff，最后输出 Gatekeeper 过闸的 MR Description。

---

## 1) Spec Interview（规格访谈｜适合 PM/设计/不熟技术的人）
你先不要写代码。
1) 阅读：CLAUDE.md、docs/spec_packet_template.md、skills/README.md。
2) 对我提供的需求进行 Spec Interview：提出 12~20 个关键追问（用户体验/验收、兼容、隐私、性能/成本、上线灰度与观测）。
3) 等我回答后，整理为【Product Spec】（只写用户可见与验收，不写技术黑盒）。
4) 生成【Tech Brief 草案】并标记 Gatekeeper 决策点（含风险等级）。
5) 列出建议引用的 skills 文件清单。

---

## 2) Execution（执行模式｜适合开发同学）
按 CLAUDE.md 执行。
1) 先阅读 CLAUDE.md 与相关 skills（在 Plan 中引用要点）。
2) 输出：Implementation Plan → Diff Plan → Test Plan → Rollout Plan。
3) 等我确认后，输出最小必要代码 diff（禁止顺手重构）。
4) 最后必须输出【MR Description】并严格遵守 CLAUDE.md 的 MR 描述契约 v0.2（必须包含 Change Inventory）。

---

## 3) High Risk（高风险模式｜L2/L3 强制）
你先不要写代码。
1) 阅读：CLAUDE.md、docs/approved_components.md、docs/adr_template.md、skills/skill_tech_discovery_and_decision.md。
2) 先输出【Decision Pack】（按 skill_tech_discovery_and_decision.md 的固定结构）。
3) 标注风险等级与 Gatekeeper 决策点。
4) 只有在我确认 + Gatekeeper 通过后，才允许输出代码 diff。
5) 最后必须输出【MR Description】并严格遵守 MR 描述契约 v0.2。
