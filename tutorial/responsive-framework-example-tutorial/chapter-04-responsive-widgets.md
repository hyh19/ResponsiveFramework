# 第 4 章：响应式组件详解

## 引言

ResponsiveFramework 提供了丰富的响应式组件，帮助开发者以声明式的方式构建响应式布局。本章将深入解析这些组件的实现原理和使用方法，包括 `ResponsiveValue`、`ResponsiveVisibility`、`ResponsiveRowColumn`、`ResponsiveGridView`、`MaxWidthBox` 和 `ResponsiveScaledBox`。

## ResponsiveValue：条件值提供者

### 核心概念

`ResponsiveValue` 是一个泛型类，根据当前断点返回不同的值。它是其他响应式组件的基础。

### 实现原理

让我们分析 `ResponsiveValue` 的实现：

```18:48:lib/src/responsive_value.dart
class ResponsiveValue<T> {
  late T value;
  final T? defaultValue;
  final List<Condition<T>> conditionalValues;

  final BuildContext context;

  ResponsiveValue(this.context,
      {required this.conditionalValues, this.defaultValue}) {
    // Breakpoint reference check. Verify a parent
    // [ResponsiveBreakpoint] exists if a reference is found.
    if (conditionalValues.firstWhereOrNull((element) => element.name != null) !=
        null) {
      try {
        ResponsiveBreakpoints.of(context);
      } catch (e) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary(
              'A conditional value was caught referencing a nonexistent breakpoint.'),
          ErrorDescription(
              'ResponsiveValue requires a parent ResponsiveBreakpoint '
              'to reference breakpoints. Add a ResponsiveBreakpoint or remove breakpoint references.')
        ]);
      }
    }

    List<Condition> conditions = [];
    conditions.addAll(conditionalValues);
    // Get visible value from active condition.
    value = (getValue(context, conditions) ?? defaultValue) as T;
  }
```

**关键特性**：

1. **泛型设计**：支持任意类型的值
2. **条件列表**：通过 `Condition` 列表定义不同断点的值
3. **默认值**：当没有条件匹配时使用默认值
4. **断点验证**：使用前验证父级 `ResponsiveBreakpoints` 存在

### Condition 类型

`Condition` 支持四种比较类型：

```138:198:lib/src/responsive_value.dart
/// Internal equality comparators.
enum Conditional {
  LARGER_THAN,
  EQUALS,
  SMALLER_THAN,
  BETWEEN,
}

/// A conditional value provider.
///
/// Provides the [value] when the [condition] is active.
/// Compare conditions by setting either [breakpoint] or
/// [name] values.
class Condition<T> {
  final int? breakpointStart;
  final int? breakpointEnd;
  final String? name;
  final Conditional? condition;
  final T? value;
  final T? landscapeValue;

  Condition._(
      {this.breakpointStart,
      this.breakpointEnd,
      this.name,
      this.condition,
      required this.value,
      T? landscapeValue})
      : landscapeValue = (landscapeValue ?? value),
        assert(breakpointStart != null || name != null),
        assert((condition == Conditional.EQUALS) ? name != null : true);

  const Condition.equals({required this.name, this.value, T? landscapeValue})
      : landscapeValue = (landscapeValue ?? value),
        breakpointStart = null,
        breakpointEnd = null,
        condition = Conditional.EQUALS;

  const Condition.largerThan(
      {int? breakpoint, this.name, this.value, T? landscapeValue})
      : landscapeValue = (landscapeValue ?? value),
        breakpointStart = breakpoint,
        breakpointEnd = breakpoint,
        condition = Conditional.LARGER_THAN;

  const Condition.smallerThan(
      {int? breakpoint, this.name, this.value, T? landscapeValue})
      : landscapeValue = (landscapeValue ?? value),
        breakpointStart = breakpoint,
        breakpointEnd = breakpoint,
        condition = Conditional.SMALLER_THAN;

  /// Conditional when screen width is between [start] and [end] inclusive.
  const Condition.between(
      {required int? start, required int? end, this.value, T? landscapeValue})
      : landscapeValue = (landscapeValue ?? value),
        breakpointStart = start,
        breakpointEnd = end,
        name = null,
        condition = Conditional.BETWEEN;
```

### 使用示例

```dart
// 根据断点返回不同的字体大小
final fontSize = ResponsiveValue<double>(
  context,
  defaultValue: 14.0,
  conditionalValues: [
    Condition.equals(name: MOBILE, value: 12.0),
    Condition.equals(name: TABLET, value: 14.0),
    Condition.equals(name: DESKTOP, value: 16.0),
  ],
).value;

// 根据断点返回不同的边距
final padding = ResponsiveValue<EdgeInsets>(
  context,
  defaultValue: EdgeInsets.all(16),
  conditionalValues: [
    Condition.smallerThan(name: TABLET, value: EdgeInsets.all(8)),
    Condition.largerThan(name: TABLET, value: EdgeInsets.all(24)),
  ],
).value;
```

## ResponsiveVisibility：条件显示/隐藏

### 实现原理

`ResponsiveVisibility` 基于 `ResponsiveValue` 实现，用于根据断点显示或隐藏组件：

```227:283:lib/src/responsive_value.dart
/// A convenience wrapper for responsive [Visibility].
///
/// ResponsiveVisibility accepts [Condition]s in
/// [visibleConditions] and [hiddenConditions] convenience
/// fields. The [child] widget is [visible] by default.
class ResponsiveVisibility extends StatelessWidget {
  final Widget child;
  final bool visible;
  final List<Condition<bool>> visibleConditions;
  final List<Condition<bool>> hiddenConditions;
  final Widget replacement;
  final bool maintainState;
  final bool maintainAnimation;
  final bool maintainSize;
  final bool maintainSemantics;
  final bool maintainInteractivity;

  const ResponsiveVisibility({
    super.key,
    required this.child,
    this.visible = true,
    this.visibleConditions = const [],
    this.hiddenConditions = const [],
    this.replacement = const SizedBox.shrink(),
    this.maintainState = false,
    this.maintainAnimation = false,
    this.maintainSize = false,
    this.maintainSemantics = false,
    this.maintainInteractivity = false,
  });

  @override
  Widget build(BuildContext context) {
    // Initialize mutable value holders.
    List<Condition<bool>> conditions = [];
    bool visibleValue = visible;

    // Combine Conditions.
    conditions.addAll(visibleConditions.map((e) => e.copyWith(value: true)));
    conditions.addAll(hiddenConditions.map((e) => e.copyWith(value: false)));
    // Get visible value from active condition.
    visibleValue = ResponsiveValue<bool>(context,
            defaultValue: visibleValue, conditionalValues: conditions)
        .value;

    return Visibility(
      replacement: replacement,
      visible: visibleValue,
      maintainState: maintainState,
      maintainAnimation: maintainAnimation,
      maintainSize: maintainSize,
      maintainSemantics: maintainSemantics,
      maintainInteractivity: maintainInteractivity,
      child: child,
    );
  }
}
```

### 使用示例

```dart
// 在移动端隐藏，桌面端显示
ResponsiveVisibility(
  visibleConditions: [
    Condition.largerThan(name: TABLET, value: true),
  ],
  child: Sidebar(),
)

// 在移动端显示，桌面端隐藏
ResponsiveVisibility(
  hiddenConditions: [
    Condition.largerThan(name: MOBILE, value: false),
  ],
  child: MobileMenu(),
)
```

## ResponsiveConstraints：响应式约束

### 实现原理

`ResponsiveConstraints` 允许根据断点设置不同的布局约束：

```285:311:lib/src/responsive_value.dart
class ResponsiveConstraints extends StatelessWidget {
  final Widget child;
  final BoxConstraints? constraint;
  final List<Condition<BoxConstraints?>> conditionalConstraints;

  const ResponsiveConstraints(
      {super.key,
      required this.child,
      this.constraint,
      this.conditionalConstraints = const []});

  @override
  Widget build(BuildContext context) {
    // Initialize mutable value holders.
    BoxConstraints? constraintValue = constraint;
    // Get value from active condition.
    constraintValue = ResponsiveValue<BoxConstraints?>(context,
            defaultValue: constraintValue,
            conditionalValues: conditionalConstraints)
        .value;

    return Container(
      constraints: constraintValue,
      child: child,
    );
  }
}
```

### 使用示例

```dart
ResponsiveConstraints(
  conditionalConstraints: [
    Condition.equals(
      name: MOBILE,
      value: BoxConstraints(maxWidth: 400),
    ),
    Condition.equals(
      name: DESKTOP,
      value: BoxConstraints(maxWidth: 1200),
    ),
  ],
  child: ContentWidget(),
)
```

## MaxWidthBox：最大宽度容器

### 实现原理

`MaxWidthBox` 是一个常用的响应式组件，用于限制内容的最大宽度并居中显示：

```1:49:lib/src/max_width_box.dart
import 'package:flutter/material.dart';

class MaxWidthBox extends StatelessWidget {
  final double? maxWidth;
  final Widget child;

  /// Control child alignment.
  /// Defaults to [Alignment.topCenter] because app
  /// content is usually top aligned.
  final AlignmentGeometry alignment;
  final EdgeInsets? padding;
  final Color? backgroundColor;

  const MaxWidthBox(
      {super.key,
      required this.maxWidth,
      required this.child,
      this.alignment = Alignment.topCenter,
      this.padding,
      this.backgroundColor});

  @override
  Widget build(BuildContext context) {
    MediaQueryData mediaQuery = MediaQuery.of(context);

    if (maxWidth != null) {
      if (mediaQuery.size.width > maxWidth!) {
        mediaQuery = mediaQuery.copyWith(
            size: Size(maxWidth! - (padding?.horizontal ?? 0),
                mediaQuery.size.height - (padding?.vertical ?? 0)));
      }
    }

    return Align(
      alignment: alignment,
      child: ConstrainedBox(
        constraints: BoxConstraints(maxWidth: maxWidth ?? double.infinity),
        child: Container(
          color: backgroundColor,
          padding: padding,
          child: MediaQuery(
            data: mediaQuery,
            child: child,
          ),
        ),
      ),
    );
  }
}
```

**关键特性**：

1. **宽度限制**：当屏幕宽度大于 `maxWidth` 时，限制内容宽度
2. **居中显示**：使用 `Align` 实现居中
3. **MediaQuery 覆盖**：更新子 Widget 的 `MediaQuery`，使其认为屏幕宽度为 `maxWidth`
4. **背景和边距**：支持设置背景色和内边距

### 在示例项目中的使用

示例项目通过扩展方法简化了 `MaxWidthBox` 的使用：

```1:28:example/lib/utils/max_width_extension.dart
import 'package:flutter/material.dart';
import 'package:responsive_framework/responsive_framework.dart';

extension MaxWidthExtension on List<Widget> {
  List<Widget> toMaxWidth() {
    return map(
      (item) => MaxWidthBox(
        maxWidth: 1200,
        padding: const EdgeInsets.symmetric(horizontal: 32),
        backgroundColor: Colors.white,
        child: item,
      ),
    ).toList();
  }

  List<Widget> toMaxWidthSliver() {
    return map(
      (item) => SliverToBoxAdapter(
        child: MaxWidthBox(
          maxWidth: 1200,
          padding: const EdgeInsets.symmetric(horizontal: 32),
          backgroundColor: Colors.white,
          child: item,
        ),
      ),
    ).toList();
  }
}
```

**使用示例**：

```dart
// 在 ListPage 中的使用
...[widget1, widget2, widget3].toMaxWidthSliver()
```

## ResponsiveRowColumn：响应式行列布局

### 实现原理

`ResponsiveRowColumn` 允许在行和列布局之间切换，常用于响应式布局：

```26:104:lib/src/responsive_row_column.dart
class ResponsiveRowColumn extends StatelessWidget {
  final List<ResponsiveRowColumnItem> children;
  final ResponsiveRowColumnType layout;
  // ... 其他属性

  @override
  Widget build(BuildContext context) {
    if (layout == ResponsiveRowColumnType.ROW) {
      return Padding(
        padding: rowPadding,
        child: Flex(
          direction: Axis.horizontal,
          mainAxisAlignment: rowMainAxisAlignment,
          mainAxisSize: rowMainAxisSize,
          crossAxisAlignment: rowCrossAxisAlignment,
          textDirection: rowTextDirection,
          verticalDirection: rowVerticalDirection,
          textBaseline: rowTextBaseline,
          children: [
            ...buildChildren(children, true, rowSpacing),
          ],
        ),
      );
    }

    return Padding(
      padding: columnPadding,
      child: Flex(
        direction: Axis.vertical,
        mainAxisAlignment: columnMainAxisAlignment,
        mainAxisSize: columnMainAxisSize,
        crossAxisAlignment: columnCrossAxisAlignment,
        textDirection: columnTextDirection,
        verticalDirection: columnVerticalDirection,
        textBaseline: columnTextBaseline,
        children: [
          ...buildChildren(children, false, columnSpacing),
        ],
      ),
    );
  }
```

### ResponsiveRowColumnItem

每个子元素必须是 `ResponsiveRowColumnItem`，支持排序和 Flex 属性：

```144:198:lib/src/responsive_row_column.dart
class ResponsiveRowColumnItem extends StatelessWidget {
  final Widget child;
  final int rowOrder;
  final int columnOrder;
  final bool rowColumn;
  final int? rowFlex;
  final int? columnFlex;
  final FlexFit? rowFit;
  final FlexFit? columnFit;

  const ResponsiveRowColumnItem(
      {super.key,
      required this.child,
      this.rowOrder = 1073741823,
      this.columnOrder = 1073741823,
      this.rowColumn = true,
      this.rowFlex,
      this.columnFlex,
      this.rowFit,
      this.columnFit});

  @override
  Widget build(BuildContext context) {
    if (rowColumn && (rowFlex != null || rowFit != null)) {
      return Flexible(
          flex: rowFlex ?? 1, fit: rowFit ?? FlexFit.loose, child: child);
    } else if (!rowColumn && (columnFlex != null || columnFit != null)) {
      return Flexible(
          flex: columnFlex ?? 1, fit: columnFit ?? FlexFit.loose, child: child);
    }

    return child;
  }
```

### 使用示例

```dart
ResponsiveRowColumn(
  layout: ResponsiveBreakpoints.of(context).isMobile
      ? ResponsiveRowColumnType.COLUMN
      : ResponsiveRowColumnType.ROW,
  rowSpacing: 16,
  columnSpacing: 8,
  children: [
    ResponsiveRowColumnItem(
      rowOrder: 1,
      columnOrder: 2,
      child: Widget1(),
    ),
    ResponsiveRowColumnItem(
      rowOrder: 2,
      columnOrder: 1,
      child: Widget2(),
    ),
  ],
)
```

## ResponsiveGridView：响应式网格布局

### 实现原理

`ResponsiveGridView` 提供了灵活的网格布局，支持固定、最大和最小尺寸：

```15:234:lib/src/responsive_grid.dart
class ResponsiveGridView extends StatelessWidget {
  // ... 属性定义

  @override
  Widget build(BuildContext context) {
    // LayoutBuilder provides constraints required for item sizing calculation.
    return LayoutBuilder(builder: (context, constraints) {
      // 计算交叉轴数量
      int crossAxisCount;
      // 根据不同的尺寸策略计算
      if (gridDelegate.crossAxisExtent != null) {
        // 固定尺寸
        crossAxisCount = (crossAxisExtent /
                (gridDelegate.crossAxisExtent! + gridDelegate.crossAxisSpacing))
            .floor();
      } else if (gridDelegate.maxCrossAxisExtent != null) {
        // 最大尺寸
        crossAxisCount = (crossAxisExtent /
                (gridDelegate.maxCrossAxisExtent! +
                    gridDelegate.crossAxisSpacing))
            .ceil();
      } else {
        // 最小尺寸
        crossAxisCount = (crossAxisExtent /
                (gridDelegate.minCrossAxisExtent! +
                    gridDelegate.crossAxisSpacing))
            .floor();
      }
      // ... 对齐和布局逻辑
    });
  }
}
```

### ResponsiveGridDelegate

`ResponsiveGridDelegate` 定义了网格的布局规则：

```272:377:lib/src/responsive_grid.dart
class ResponsiveGridDelegate extends SliverGridDelegate {
  const ResponsiveGridDelegate({
    this.crossAxisExtent,
    this.maxCrossAxisExtent,
    this.minCrossAxisExtent,
    this.mainAxisSpacing = 0,
    this.crossAxisSpacing = 0,
    this.childAspectRatio = 1,
  })  : assert(
            (crossAxisExtent != null && crossAxisExtent >= 0) ||
                (maxCrossAxisExtent != null && maxCrossAxisExtent >= 0) ||
                (minCrossAxisExtent != null && minCrossAxisExtent >= 0),
            'Must provide a valid cross axis extent.'),
        assert(mainAxisSpacing >= 0),
        assert(crossAxisSpacing >= 0),
        assert(childAspectRatio > 0);

  /// Fixed item size.
  final double? crossAxisExtent;

  /// Maximum item size.
  final double? maxCrossAxisExtent;

  /// Minimum item size.
  final double? minCrossAxisExtent;
  final double mainAxisSpacing;
  final double crossAxisSpacing;
  final double childAspectRatio;
```

### 使用示例

```dart
ResponsiveGridView(
  gridDelegate: ResponsiveGridDelegate(
    maxCrossAxisExtent: 300,
    crossAxisSpacing: 16,
    mainAxisSpacing: 16,
    childAspectRatio: 1.5,
  ),
  children: [
    Item1(),
    Item2(),
    Item3(),
  ],
)
```

## ResponsiveScaledBox：响应式缩放容器

### 实现原理

`ResponsiveScaledBox` 用于按比例缩放内容，常用于保持设计稿的宽高比：

```5:71:lib/src/responsive_scaled_box.dart
class ResponsiveScaledBox extends StatelessWidget {
  final double? width;
  final Widget child;
  final bool autoCalculateMediaQueryData;

  const ResponsiveScaledBox(
      {super.key,
      required this.width,
      required this.child,
      this.autoCalculateMediaQueryData = true});

  @override
  Widget build(BuildContext context) {
    if (width != null) {
      return LayoutBuilder(
        builder: (context, constraints) {
          double aspectRatio = constraints.maxWidth / constraints.maxHeight;
          double scaledWidth = width!;
          double scaledHeight = width! / aspectRatio;

          Widget childHolder = FittedBox(
            fit: BoxFit.fitWidth,
            alignment: Alignment.topCenter,
            child: Container(
              width: width,
              height: scaledHeight,
              alignment: Alignment.center,
              child: child,
            ),
          );

          if (autoCalculateMediaQueryData) {
            // 更新 MediaQuery 数据
            // ...
          }

          return childHolder;
        },
      );
    }

    return child;
  }
}
```

### 使用场景

`ResponsiveScaledBox` 适用于需要保持固定宽高比的场景，如设计稿还原。

## 实践练习

### 练习 1：使用 ResponsiveValue

创建一个响应式的文本样式组件，根据断点返回不同的字体大小和颜色。

### 练习 2：实现响应式导航栏

使用 `ResponsiveVisibility` 实现一个导航栏，在移动端显示汉堡菜单，桌面端显示完整菜单。

### 练习 3：创建响应式卡片网格

使用 `ResponsiveGridView` 创建一个卡片网格，在不同断点下显示不同数量的列。

## 总结与检查清单

### 本章要点

- `ResponsiveValue` 是条件值提供者，支持四种比较类型
- `ResponsiveVisibility` 用于根据断点显示/隐藏组件
- `ResponsiveConstraints` 允许设置响应式布局约束
- `MaxWidthBox` 限制内容最大宽度并居中显示
- `ResponsiveRowColumn` 在行列布局间切换
- `ResponsiveGridView` 提供灵活的网格布局
- `ResponsiveScaledBox` 按比例缩放内容

### 检查清单

在进入下一章之前，请确保你：

- [ ] 理解了 `ResponsiveValue` 的工作原理
- [ ] 掌握了各种响应式组件的使用方法
- [ ] 能够根据需求选择合适的组件
- [ ] 完成了实践练习

### 下一步

准备好后，让我们进入第 5 章，学习如何在实际页面中使用这些组件构建响应式布局。

