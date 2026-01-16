# Skill：前端 API 调用规范

## 目标

安全、可靠地与后端 API 交互，处理各种边界情况。

---

## API 调用层次

```
UI 组件
    ↓
Provider / Controller
    ↓
API Service（封装请求）
    ↓
HTTP Client（底层请求）
```

---

## API Service 设计

### 1. 基础封装

```dart
class ApiService {
  final HttpClient _client;

  Future<T> get<T>(String path, {
    Map<String, dynamic>? params,
    T Function(Map<String, dynamic>)? fromJson,
  }) async {
    try {
      final response = await _client.get(path, params: params);
      return fromJson != null ? fromJson(response) : response as T;
    } on NetworkException catch (e) {
      throw ApiException.network(e.message);
    } on ServerException catch (e) {
      throw ApiException.server(e.code, e.message);
    }
  }
}
```

### 2. 业务 API

```dart
class MemoApi {
  final ApiService _api;

  Future<List<Memo>> getMemos({int page = 1, int limit = 20}) async {
    final response = await _api.get('/api/v1/memos', params: {
      'page': page,
      'limit': limit,
    });

    final items = response['data']['items'] as List? ?? [];
    return items.map((e) => Memo.fromJson(e)).toList();
  }
}
```

---

## 请求处理

### 1. 请求重试

```dart
Future<T> withRetry<T>(
  Future<T> Function() request, {
  int maxRetries = 3,
  Duration delay = const Duration(seconds: 1),
}) async {
  int attempts = 0;
  while (true) {
    try {
      return await request();
    } catch (e) {
      attempts++;
      if (attempts >= maxRetries || !_isRetryable(e)) {
        rethrow;
      }
      await Future.delayed(delay * attempts);
    }
  }
}
```

### 2. 请求取消

```dart
class MemoProvider extends ChangeNotifier {
  CancelToken? _cancelToken;

  Future<void> fetchMemos() async {
    _cancelToken?.cancel();
    _cancelToken = CancelToken();

    try {
      final memos = await api.getMemos(cancelToken: _cancelToken);
      // ...
    } on CancelledException {
      // 被取消，忽略
    }
  }
}
```

### 3. 请求防抖

```dart
Timer? _debounceTimer;

void search(String query) {
  _debounceTimer?.cancel();
  _debounceTimer = Timer(const Duration(milliseconds: 300), () {
    _doSearch(query);
  });
}
```

---

## 响应处理

### 1. 容忍未知字段

```dart
class Memo {
  final String id;
  final String title;
  final String? content;  // 可选

  factory Memo.fromJson(Map<String, dynamic> json) {
    return Memo(
      id: json['id'] ?? '',
      title: json['title'] ?? '',
      content: json['content'],  // 可能不存在
      // 忽略未知字段
    );
  }
}
```

### 2. 错误处理

```dart
Future<void> fetchData() async {
  try {
    _state = LoadingState();
    notifyListeners();

    final data = await api.getData();
    _state = SuccessState(data);
  } on NetworkException {
    _state = ErrorState('网络连接失败，请检查网络');
  } on UnauthorizedException {
    _state = ErrorState('登录已过期，请重新登录');
    // 跳转登录页
  } on ServerException catch (e) {
    _state = ErrorState(e.userMessage);
  } catch (e) {
    _state = ErrorState('未知错误，请稍后重试');
  } finally {
    notifyListeners();
  }
}
```

### 3. 空数据处理

```dart
Widget build(BuildContext context) {
  final items = data['items'] as List? ?? [];

  if (items.isEmpty) {
    return EmptyStateWidget(
      message: '暂无内容',
      action: TextButton(
        onPressed: onRefresh,
        child: Text('刷新'),
      ),
    );
  }

  return ListView.builder(
    itemCount: items.length,
    itemBuilder: (_, i) => ItemWidget(item: items[i]),
  );
}
```

---

## 缓存策略

### 1. 内存缓存

```dart
class CachedApi {
  final _cache = <String, CacheEntry>{};

  Future<T> getWithCache<T>(
    String key,
    Future<T> Function() fetcher, {
    Duration ttl = const Duration(minutes: 5),
  }) async {
    final cached = _cache[key];
    if (cached != null && !cached.isExpired) {
      return cached.data as T;
    }

    final data = await fetcher();
    _cache[key] = CacheEntry(data: data, expireAt: DateTime.now().add(ttl));
    return data;
  }
}
```

### 2. 刷新策略

```dart
// 先展示缓存，后台刷新
Future<void> fetchWithStaleWhileRevalidate() async {
  // 先展示缓存
  final cached = await cache.get(key);
  if (cached != null) {
    _data = cached;
    notifyListeners();
  }

  // 后台刷新
  try {
    final fresh = await api.fetch();
    _data = fresh;
    await cache.set(key, fresh);
    notifyListeners();
  } catch (e) {
    if (cached == null) rethrow;
    // 有缓存则忽略刷新失败
  }
}
```

---

## Checklist

- [ ] API 调用封装在 Service 层
- [ ] 容忍后端返回的未知字段
- [ ] 处理各种错误（网络/认证/服务器）
- [ ] 处理空数据状态
- [ ] 实现请求取消（页面切换时）
- [ ] 实现防抖（搜索场景）
- [ ] 考虑缓存策略
- [ ] 错误信息对用户友好
