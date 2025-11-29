# Compose 性能优化深度指南：Recomposition 与渲染优化

> **发布日期**: 2024-04-01  
> **阅读时间**: 约 25 分钟  
> **标签**: Recomposition, Stable, derivedStateOf, Profiling

Compose 的声明式 UI 模型带来了开发效率的提升，但也引入了新的性能考量。理解 Recomposition 机制并掌握优化技巧，是打造流畅 Compose 应用的关键。

## 一、理解 Compose 的三个阶段

Compose 渲染一帧分为三个阶段：

- **Composition（组合）**：执行 @Composable 函数，构建 UI 树
- **Layout（布局）**：测量和放置 UI 元素
- **Drawing（绘制）**：将 UI 绘制到屏幕

性能优化的核心是：**尽量减少不必要的 Composition，并将变化推迟到 Layout 或 Drawing 阶段**。

## 二、Recomposition 触发条件

Recomposition 发生在状态变化时。Compose 会智能地只重组受影响的部分，但前提是你正确使用了状态 API：

```kotlin
// ✅ 正确：状态变化只触发读取它的 Composable 重组
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Column {
        Text("Count: $count")  // 只有这行会重组
        Button(onClick = { count++ }) {
            Text("Increment")  // 这里不会重组
        }
    }
}
```

## 三、使用 remember 避免重复计算

```kotlin
// ❌ 每次重组都会重新计算
@Composable
fun ExpensiveList(items: List<Item>) {
    val sortedItems = items.sortedBy { it.name }  // 每次都排序！
    LazyColumn { ... }
}

// ✅ 只在 items 变化时重新计算
@Composable
fun ExpensiveList(items: List<Item>) {
    val sortedItems = remember(items) {
        items.sortedBy { it.name }
    }
    LazyColumn { ... }
}
```

## 四、derivedStateOf：派生状态优化

当你需要从其他状态派生出新状态，且派生结果不是每次都变化时，使用 `derivedStateOf`：

```kotlin
@Composable
fun SearchResults(query: String, items: List<Item>) {
    // 只在过滤结果实际变化时才触发重组
    val filteredItems by remember(query, items) {
        derivedStateOf {
            items.filter { it.name.contains(query, ignoreCase = true) }
        }
    }

    LazyColumn {
        items(filteredItems) { item ->
            ItemRow(item)
        }
    }
}
```

> ⚠️ **何时使用 derivedStateOf**  
> 当输入状态变化频繁，但派生结果变化较少时使用。例如：滚动位置 → 是否显示"返回顶部"按钮。

## 五、Stable 与 Immutable 注解

Compose 编译器会分析参数的稳定性来决定是否跳过重组。对于自定义类，可以使用注解帮助编译器：

```kotlin
// 不可变数据类，Compose 可以安全跳过重组
@Immutable
data class User(
    val id: String,
    val name: String,
    val avatar: String
)

// 稳定类：内部状态可变，但 Compose 可以通过 equals 判断是否变化
@Stable
class PagingState(
    val items: List<Item>,
    val isLoading: Boolean,
    val error: String?
)
```

## 六、Lambda 稳定性优化

Lambda 表达式默认是不稳定的，可能导致不必要的重组：

```kotlin
// ❌ 每次重组都创建新的 lambda
@Composable
fun ItemList(viewModel: ItemViewModel) {
    LazyColumn {
        items(viewModel.items) { item ->
            ItemRow(
                item = item,
                onClick = { viewModel.onItemClick(item) }  // 新 lambda！
            )
        }
    }
}

// ✅ 使用 remember 缓存 lambda
@Composable
fun ItemList(viewModel: ItemViewModel) {
    val onItemClick = remember(viewModel) {
        { item: Item -> viewModel.onItemClick(item) }
    }

    LazyColumn {
        items(viewModel.items) { item ->
            ItemRow(item = item, onClick = { onItemClick(item) })
        }
    }
}
```

## 七、使用 key 优化列表

```kotlin
LazyColumn {
    items(
        items = users,
        key = { user -> user.id }  // 帮助 Compose 识别 item 身份
    ) { user ->
        UserRow(user)
    }
}
```

## 八、graphicsLayer 避免重组

对于纯视觉变换（alpha、scale、rotation），使用 `graphicsLayer` 将变化限制在绘制阶段：

```kotlin
// ✅ 只影响 Drawing 阶段，不触发重组
Box(
    modifier = Modifier.graphicsLayer {
        alpha = animatedAlpha
        scaleX = animatedScale
        scaleY = animatedScale
        rotationZ = animatedRotation
    }
)

// ❌ 会触发重组
Box(
    modifier = Modifier
        .alpha(animatedAlpha)
        .scale(animatedScale)
)
```

## 九、使用 Layout Inspector 调试

Android Studio 的 Layout Inspector 可以显示 Recomposition 计数：

1. 运行应用（Debug 模式）
2. 打开 Tools → Layout Inspector
3. 在 Component Tree 中查看 Recomposition 次数
4. 高频重组的组件需要优化

## 十、性能优化清单

- ✅ 使用 `remember` 缓存计算结果
- ✅ 使用 `derivedStateOf` 处理派生状态
- ✅ 为数据类添加 `@Immutable` 或 `@Stable`
- ✅ 使用 `key` 参数优化列表
- ✅ 用 `graphicsLayer` 处理视觉变换
- ✅ 避免在 Composable 中创建新对象
- ✅ 将状态下沉到最小作用域
- ✅ 使用 Layout Inspector 监控重组

## 总结

Compose 性能优化的核心原则：

- **减少 Composition 范围**：让状态变化只影响必要的组件
- **推迟到后续阶段**：能在 Layout/Drawing 阶段处理的，不要在 Composition 阶段处理
- **帮助编译器**：通过注解和正确的代码结构，让编译器能够跳过不必要的重组

掌握这些技巧，你的 Compose 应用将获得丝滑的 60fps 体验。

---

*© 2024 Fidroid. [返回首页](../index.html)*

