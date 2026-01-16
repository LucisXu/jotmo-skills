# Skill：前端开发总纲

## 目标

确保前端开发符合团队规范，与后端良好配合，产出高质量代码。

---

## 核心原则

### 1. 与 CLAUDE.md 保持一致

前端开发同样遵守全局规则：
- 安全 / 合规 / 数据不丢
- 向后兼容与线上稳定
- API 契约与跨端一致性
- 可灰度、可回滚、可观测

### 2. 契约优先

- 与后端 API 契约保持一致（参考 `skill_api_contract.md`）
- 容忍未知字段/新增字段
- 若后端使用开关，前端必须配合（开关开/关体验都完整）

---

## 相关 Skills

| Skill | 用途 |
|-------|------|
| `skill_fe_component.md` | 组件开发规范 |
| `skill_fe_state.md` | 状态管理规范 |
| `skill_fe_api.md` | API 调用规范 |
| `skill_api_contract.md` | API 契约规范 |

---

## 必须处理的状态

每个功能页面/组件必须考虑：

| 状态 | 展示 |
|------|------|
| **加载中** | 骨架屏或加载动画 |
| **空数据** | 空状态提示 + 引导操作 |
| **成功** | 正常内容展示 |
| **失败** | 错误提示 + 重试按钮 |
| **无权限** | 权限提示 + 引导操作 |
| **弱网** | 离线提示 + 缓存数据 |

---

## 兼容性要求

### 1. 数据兼容

```dart
// 容忍缺失字段
final name = data['name'] ?? '未知';
final items = (data['items'] as List?) ?? [];

// 容忍未知枚举
final status = StatusEnum.values.firstWhere(
  (e) => e.name == data['status'],
  orElse: () => StatusEnum.unknown,
);
```

### 2. 版本兼容

```dart
// 根据后端开关展示不同 UI
if (featureFlags.isEnabled('new_feature')) {
  return NewFeatureWidget();
} else {
  return OldFeatureWidget();
}
```

### 3. 降级处理

```dart
// 新功能不可用时的降级
Widget build(BuildContext context) {
  if (!featureFlags.isEnabled('ai_summary')) {
    return Text('暂不支持此功能');
  }
  return AISummaryWidget();
}
```

---

## 常见冲突处理

| 冲突 | 处理方式 |
|------|----------|
| 字段重命名 | 禁止。改为新增字段 + 兼容旧字段 + 废弃周期 |
| 大重构 | 拆分多 MR，小步灰度 |
| UI 风格不一致 | 遵循设计系统，有分歧找设计师确认 |

---

## 输出要求

前端开发完成后，MR 描述必须包含：

1. **影响面**：涉及哪些页面/组件
2. **兼容性**：如何处理旧数据/旧版本
3. **状态覆盖**：加载中/空/成功/失败/无权限都处理了吗
4. **开关配合**：后端有开关吗？开关关闭时表现如何？
5. **测试证据**：主流程 + 异常场景的测试截图

---

## Checklist

- [ ] 遵循项目目录结构
- [ ] 组件职责单一（参考 `skill_fe_component.md`）
- [ ] 状态管理合理（参考 `skill_fe_state.md`）
- [ ] API 调用规范（参考 `skill_fe_api.md`）
- [ ] 处理所有状态（加载中/空/成功/失败/无权限）
- [ ] 容忍后端未知字段
- [ ] 配合后端开关
- [ ] 无 console 错误
- [ ] 主流程测试通过
