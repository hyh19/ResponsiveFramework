# 第 7 章：路由与导航系统

## 引言

路由系统是 Flutter Web 应用的核心部分，它决定了用户如何在页面间导航。本章将深入分析示例项目中的路由实现，包括路由配置、导航处理、URL 策略，以及如何实现条件路由。

## Routes 类：路由封装

### 实现分析

`Routes` 类封装了路由的创建逻辑：

```1:30:example/lib/routes.dart
import 'package:animations/animations.dart';
import 'package:flutter/widgets.dart';

class Routes {
  static Route<T> fadeThrough<T>(
      {required RouteSettings settings,
      required WidgetBuilder builder,
      int duration = 300}) {
    return PageRouteBuilder<T>(
      settings: settings,
      transitionDuration: Duration(milliseconds: duration),
      pageBuilder: (context, animation, secondaryAnimation) => builder(context),
      transitionsBuilder: (context, animation, secondaryAnimation, child) {
        return FadeScaleTransition(animation: animation, child: child);
      },
    );
  }

  static Route<T> noAnimation<T>(
      {required RouteSettings settings, required WidgetBuilder builder}) {
    return PageRouteBuilder<T>(
      settings: settings,
      transitionDuration: Duration.zero,
      pageBuilder: (context, animation, secondaryAnimation) => builder(context),
      transitionsBuilder: (context, animation, secondaryAnimation, child) {
        return child;
      },
    );
  }
}
```

### 路由类型

1. **fadeThrough**：淡入淡出转场动画
   - 使用 `FadeScaleTransition` 实现
   - 可配置动画时长

2. **noAnimation**：无动画路由
   - 适用于 Web 应用
   - 提供即时页面切换

### 设计优势

- **统一管理**：所有路由创建逻辑集中管理
- **易于扩展**：可以轻松添加新的路由类型
- **类型安全**：使用泛型确保类型安全

## 路由配置

### MaterialApp 配置

在 `main.dart` 中配置路由：

```24:48:example/lib/main.dart
    return MaterialApp(
      // Wrapping the app with a builder method makes breakpoints
      // accessible throughout the widget tree.
      builder: (context, child) => ResponsiveBreakpoints.builder(
        breakpoints: [
          const Breakpoint(start: 0, end: 450, name: MOBILE),
          const Breakpoint(start: 451, end: 800, name: TABLET),
          const Breakpoint(start: 801, end: 1920, name: DESKTOP),
          const Breakpoint(start: 1921, end: double.infinity, name: '4K'),
        ],
        child: child!,
      ),
      initialRoute: '/',
      onGenerateInitialRoutes: (initialRoute) {
        final Uri uri = Uri.parse(initialRoute);
        return [
          buildPage(path: uri.path, queryParams: uri.queryParameters),
        ];
      },
      onGenerateRoute: (RouteSettings settings) {
        final Uri uri = Uri.parse(settings.name ?? '/');
        return buildPage(path: uri.path, queryParams: uri.queryParameters);
      },
      debugShowCheckedModeBanner: false,
    );
```

### 关键配置

1. **initialRoute**：设置初始路由为 `/`
2. **onGenerateInitialRoutes**：生成初始路由
3. **onGenerateRoute**：生成其他路由
4. **buildPage**：统一的页面构建方法

## buildPage 方法：路由构建

### 实现分析

```51:75:example/lib/main.dart
  // onGenerateRoute route switcher.
  // Navigate using the page name, `Navigator.pushNamed(context, ListPage.name)`.
  Route<dynamic> buildPage(
      {required String path, Map<String, String> queryParams = const {}}) {
    return Routes.noAnimation(
        settings: RouteSettings(
            name: (path.startsWith('/') == false) ? '/$path' : path),
        builder: (context) {
          String pathName =
              path != '/' && path.startsWith('/') ? path.substring(1) : path;
          return switch (pathName) {
            '/' || ListPage.name => const ListPage(),
            PostPage.name =>
              // Breakpoints can be nested.
              // Here's an example of custom "per-page" breakpoints.
              const ResponsiveBreakpoints(breakpoints: [
                Breakpoint(start: 0, end: 480, name: MOBILE),
                Breakpoint(start: 481, end: 1200, name: TABLET),
                Breakpoint(start: 1201, end: double.infinity, name: DESKTOP),
              ], child: PostPage()),
            TypographyPage.name => const TypographyPage(),
            _ => const SizedBox.shrink(),
          };
        });
  }
```

### 关键特性

1. **路径规范化**：确保路径以 `/` 开头
2. **Switch 表达式**：使用 Dart 3.0 的 switch 表达式
3. **嵌套断点**：`PostPage` 使用自定义断点
4. **默认处理**：未匹配的路由返回空 Widget

### 路由匹配逻辑

```dart
switch (pathName) {
  case '/' || ListPage.name:
    return const ListPage();
  case PostPage.name:
    return const ResponsiveBreakpoints(...);
  case TypographyPage.name:
    return const TypographyPage();
  default:
    return const SizedBox.shrink();
}
```

## URL 策略配置

### usePathUrlStrategy

在 `main.dart` 中配置 URL 策略：

```9:17:example/lib/main.dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();

  if (kIsWeb) {
    usePathUrlStrategy();
  }

  runApp(const MyApp());
}
```

### URL 策略类型

1. **Hash 策略**（默认）
   - URL 格式：`https://example.com/#/list`
   - 兼容性好，但 URL 不友好

2. **Path 策略**（推荐）
   - URL 格式：`https://example.com/list`
   - URL 更友好，但需要服务器配置

### 服务器配置

使用 Path 策略时，需要配置服务器将所有路由指向 `index.html`：

**Nginx 配置**：

```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

**Apache 配置**：

```apache
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>
```

## 导航使用

### 命名路由导航

```dart
// 导航到列表页
Navigator.pushNamed(context, ListPage.name);

// 导航到文章页
Navigator.pushNamed(context, PostPage.name);

// 导航到首页并清除历史
Navigator.pushNamedAndRemoveUntil(
  context,
  Navigator.defaultRouteName,
  ModalRoute.withName(Navigator.defaultRouteName),
);
```

### 在组件中使用

```318:318:example/lib/components/blog.dart
              onPressed: () => Navigator.pushNamed(context, PostPage.name),
```

## 条件路由组件

### ConditionalRouteWidget

示例项目提供了条件路由组件（虽然未在代码中直接使用，但提供了实现思路）：

条件路由可以根据某些条件（如用户权限、设备类型等）决定是否允许导航。

### 实现思路

```dart
class ConditionalRouteWidget extends StatelessWidget {
  final Widget child;
  final bool condition;
  final Widget? fallback;

  const ConditionalRouteWidget({
    super.key,
    required this.child,
    required this.condition,
    this.fallback,
  });

  @override
  Widget build(BuildContext context) {
    if (condition) {
      return child;
    }
    return fallback ?? const SizedBox.shrink();
  }
}
```

## 路由参数处理

### 查询参数

`buildPage` 方法接收查询参数：

```dart
Route<dynamic> buildPage({
  required String path,
  Map<String, String> queryParams = const {},
}) {
  // 可以使用 queryParams 传递参数
  // 例如：/post?id=123
  final id = queryParams['id'];
  // ...
}
```

### 路径参数

虽然示例项目未使用路径参数，但可以实现：

```dart
// 路由定义：/post/:id
// URL：/post/123

// 解析路径参数
final segments = path.split('/');
if (segments.length > 2 && segments[1] == 'post') {
  final id = segments[2];
  return PostPage(id: id);
}
```

## 路由最佳实践

### 1. 使用命名路由

```dart
class ListPage extends StatelessWidget {
  static const String name = 'list';
  // ...
}

// 使用
Navigator.pushNamed(context, ListPage.name);
```

**优势**：

- 类型安全：编译时检查
- 易于重构：修改路由名称只需改一处
- 代码清晰：路由名称一目了然

### 2. 集中管理路由

将所有路由逻辑集中在 `buildPage` 方法中：

```dart
Route<dynamic> buildPage({required String path, ...}) {
  return Routes.noAnimation(
    builder: (context) {
      return switch (pathName) {
        // 所有路由都在这里
      };
    },
  );
}
```

### 3. 处理未匹配路由

```dart
_ => const NotFoundPage(),  // 显示 404 页面
// 或
_ => const SizedBox.shrink(),  // 显示空页面
```

### 4. 路由动画选择

- **Web 应用**：使用 `noAnimation` 提供即时切换
- **移动应用**：使用 `fadeThrough` 提供流畅动画

## 实践练习

### 练习 1：添加新路由

添加一个新的页面路由，并在导航栏中添加链接。

### 练习 2：实现路由参数

实现带参数的路由，如 `/post/:id`。

### 练习 3：添加 404 页面

创建一个 404 页面，处理未匹配的路由。

## 总结与检查清单

### 本章要点

- `Routes` 类封装了路由创建逻辑
- `buildPage` 方法统一处理路由匹配
- URL 策略影响 URL 格式
- 命名路由提供类型安全
- 条件路由可以实现权限控制

### 检查清单

在进入下一章之前，请确保你：

- [ ] 理解了路由配置的方式
- [ ] 掌握了命名路由的使用
- [ ] 理解了 URL 策略的选择
- [ ] 能够添加新的路由
- [ ] 完成了实践练习

### 下一步

准备好后，让我们进入第 8 章，学习高级技巧与性能优化。

