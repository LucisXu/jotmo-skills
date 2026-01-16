# Skill：前端状态管理规范

## 目标

合理管理应用状态，确保数据一致性和可维护性。

---

## 状态分类

| 类型 | 生命周期 | 存储位置 | 示例 |
|------|----------|----------|------|
| **UI 状态** | 组件级 | 组件内部 | 按钮加载中、表单输入 |
| **页面状态** | 页面级 | 页面 Provider | 列表数据、筛选条件 |
| **全局状态** | 应用级 | 全局 Provider | 用户信息、主题设置 |
| **持久状态** | 跨会话 | 本地存储 | 登录态、用户偏好 |

---

## 状态管理选择

### 1. 简单状态：StatefulWidget

```dart
// 仅组件内部使用的状态
class _CounterState extends State<Counter> {
  int _count = 0;

  void _increment() {
    setState(() => _count++);
  }
}
```

### 2. 跨组件状态：Provider

```dart
// 定义 Provider
class UserProvider extends ChangeNotifier {
  User? _user;

  User? get user => _user;

  void setUser(User user) {
    _user = user;
    notifyListeners();
  }
}

// 使用 Provider
final user = context.watch<UserProvider>().user;
```

### 3. 复杂状态：Riverpod（如项目使用）

```dart
// 定义 Provider
final userProvider = StateNotifierProvider<UserNotifier, UserState>((ref) {
  return UserNotifier();
});

// 使用
final user = ref.watch(userProvider);
```

---

## 状态设计原则

### 1. 最小化状态

```dart
// 不好：冗余状态
class BadState {
  List<Item> items;
  int itemCount;  // 冗余，可以从 items 计算
}

// 好：计算属性
class GoodState {
  List<Item> items;
  int get itemCount => items.length;
}
```

### 2. 状态不可变

```dart
// 不好：直接修改
state.items.add(newItem);

// 好：创建新对象
state = state.copyWith(
  items: [...state.items, newItem],
);
```

### 3. 状态归一化

```dart
// 不好：嵌套数据
Map<String, List<Comment>> postComments;

// 好：扁平化
Map<String, Comment> commentsById;
Map<String, List<String>> postCommentIds;
```

---

## 与后端配合

### 1. 开关状态同步

```dart
// 后端开关控制功能可见性
class FeatureFlags extends ChangeNotifier {
  Map<String, bool> _flags = {};

  bool isEnabled(String feature) => _flags[feature] ?? false;

  Future<void> fetchFlags() async {
    _flags = await api.getFeatureFlags();
    notifyListeners();
  }
}
```

### 2. 乐观更新

```dart
// 先更新 UI，失败再回滚
Future<void> toggleLike() async {
  final oldState = _isLiked;
  _isLiked = !_isLiked;
  notifyListeners();

  try {
    await api.toggleLike();
  } catch (e) {
    _isLiked = oldState;  // 回滚
    notifyListeners();
  }
}
```

---

## Checklist

- [ ] 状态分类正确（UI/页面/全局/持久）
- [ ] 使用合适的状态管理方案
- [ ] 状态最小化，无冗余
- [ ] 状态不可变，使用 copyWith
- [ ] 处理后端开关状态
- [ ] 考虑乐观更新和回滚
- [ ] 状态变更有日志（便于调试）
