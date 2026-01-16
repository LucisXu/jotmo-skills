# Skill：前端组件开发规范

## 目标

创建可复用、可测试、符合团队规范的 UI 组件。

---

## 组件分类

| 类型 | 说明 | 示例 |
|------|------|------|
| **基础组件** | 最小粒度，无业务逻辑 | Button, Input, Icon |
| **业务组件** | 包含业务逻辑，可复用 | MemoCard, UserAvatar |
| **页面组件** | 特定页面使用 | HomePage, SettingsPage |

---

## 组件结构

```
components/
├── common/           # 基础组件
│   ├── Button/
│   │   ├── index.dart
│   │   └── button_test.dart
│   └── Input/
├── business/         # 业务组件
│   ├── MemoCard/
│   └── UserAvatar/
└── pages/            # 页面组件
    ├── HomePage/
    └── SettingsPage/
```

---

## 组件设计原则

### 1. 单一职责

- 一个组件只做一件事
- 复杂组件拆分为多个子组件

### 2. Props 设计

```dart
// 好的设计：明确的 Props
class MemoCard extends StatelessWidget {
  final String title;
  final String? content;  // 可选
  final DateTime createdAt;
  final VoidCallback? onTap;  // 可选回调

  const MemoCard({
    required this.title,
    this.content,
    required this.createdAt,
    this.onTap,
  });
}
```

### 3. 状态处理

- 优先使用无状态组件
- 状态尽量提升到父组件
- 复杂状态使用状态管理方案

### 4. 错误边界

```dart
// 处理各种状态
Widget build(BuildContext context) {
  if (isLoading) return LoadingWidget();
  if (hasError) return ErrorWidget(error: error);
  if (isEmpty) return EmptyWidget();
  return ContentWidget(data: data);
}
```

---

## 兼容性要求

### 1. 容忍未知数据

```dart
// 好的做法：兜底处理
String displayName = data['name'] ?? '未知';
List items = data['items'] ?? [];
```

### 2. 响应式设计

```dart
// 适配不同屏幕
Widget build(BuildContext context) {
  final screenWidth = MediaQuery.of(context).size.width;
  if (screenWidth < 600) {
    return MobileLayout();
  } else {
    return DesktopLayout();
  }
}
```

### 3. 无障碍

```dart
// 提供语义化标签
Semantics(
  label: '删除按钮',
  child: IconButton(
    icon: Icon(Icons.delete),
    onPressed: onDelete,
  ),
)
```

---

## Checklist

- [ ] 组件职责单一
- [ ] Props 类型明确，可选项有默认值
- [ ] 处理各种状态（加载中/空/成功/失败）
- [ ] 容忍后端返回的未知字段
- [ ] 支持响应式布局（如需要）
- [ ] 提供无障碍标签（如需要）
- [ ] 编写组件测试
