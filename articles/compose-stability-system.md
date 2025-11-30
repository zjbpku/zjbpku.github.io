# Compose 稳定性系统：@Stable、@Immutable 与智能重组

*2024-04-28 · 28 min · 原理深度解析*

**Tags:** Stability, @Stable, @Immutable, Smart Recomposition

---

为什么有些 Composable 会被跳过重组，而有些却每次都重新执行？答案在于 Compose 的**稳定性系统（Stability System）**。理解这个系统是优化 Compose 性能的关键。本文将深入探讨稳定性推断的原理、`@Stable` 和 `@Immutable` 注解的区别，以及如何诊断和修复稳定性问题。

> 📚 **官方参考**
> - [Stability in Compose - Android Developers](https://developer.android.com/jetpack/compose/performance/stability)
> - [Compose Compiler Metrics](https://github.com/androidx/androidx/blob/androidx-main/compose/compiler/design/compiler-metrics.md)

## 一、什么是稳定性？

在 Compose 中，**稳定性**指的是编译器能否安全地假设：如果一个值的 `equals()` 返回 true，那么这个值在两次组合之间没有发生变化。

这个假设非常重要，因为它决定了 Compose 能否**跳过重组**：

```kotlin
@Composable
fun UserCard(user: User) {
    // 如果 user 是"稳定的"，且 equals() 返回 true
    // Compose 可以跳过这个函数的执行
    Text(user.name)
    Text(user.email)
}
```

### 稳定性的三个条件

一个类型被认为是**稳定的**，需要满足以下条件：

1. **equals() 一致性**：对于相同的两个实例，`equals()` 的结果永远相同
2. **公开属性变化通知**：如果公开属性发生变化，Composition 会被通知（通过 Snapshot 系统）
3. **公开属性也是稳定的**：所有公开属性的类型也必须是稳定的

> 💡 **核心洞察**
> 
> 稳定性不是关于"值是否会变化"，而是关于"**如果值变化了，Compose 能否知道**"。一个 `MutableState` 是稳定的，因为它的变化会通过 Snapshot 系统通知 Compose。

## 二、编译器的稳定性推断

Compose 编译器会自动分析类型的稳定性。以下类型被自动认为是稳定的：

| 类型 | 稳定性 | 原因 |
|------|--------|------|
| 基本类型 (Int, Float, Boolean...) | ✅ 稳定 | 不可变 |
| String | ✅ 稳定 | 不可变 |
| 函数类型 (Lambda) | ✅ 稳定 | 函数本身不变 |
| MutableState<T> | ✅ 稳定 | 变化会通知 Snapshot |
| 只含不可变属性的 data class | ✅ 稳定 | 编译器推断 |
| List, Set, Map | ❌ 不稳定 | 可能是可变实现 |
| 外部模块的类 | ❌ 不稳定 | 编译器无法分析 |

### 为什么 List 不稳定？

这是一个常见的困惑点。虽然 Kotlin 的 `List` 接口是只读的，但它可能指向一个 `MutableList` 实例：

```kotlin
val mutableList = mutableListOf(1, 2, 3)
val list: List<Int> = mutableList  // 类型是 List，但实际是 MutableList

mutableList.add(4)  // 修改了！但 Compose 不知道

// list.equals(list) 仍然返回 true
// 但内容已经变了，Compose 无法检测到
```

因为编译器无法保证 `List` 的实际实现是不可变的，所以它保守地将其标记为不稳定。

> 📚 **深入阅读**
> 
> [Kotlin List Interface](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-list/) - 注意它只是"只读"而非"不可变"

## 三、@Stable 与 @Immutable 注解

当编译器无法自动推断稳定性时，你可以使用注解来手动声明：

### @Immutable

`@Immutable` 表示一个类型是**完全不可变的**：创建后，所有属性的值永远不会改变。

```kotlin
@Immutable
data class User(
    val id: String,
    val name: String,
    val email: String
)

// 使用 @Immutable 的承诺：
// 1. 所有属性都是 val（不可变）
// 2. 属性的类型也是不可变的
// 3. 创建后永远不会被修改
```

### @Stable

`@Stable` 是一个更宽松的契约：值**可以变化**，但变化会通过 Compose 的 Snapshot 系统被追踪。

```kotlin
@Stable
class UserState(
    initialName: String
) {
    var name by mutableStateOf(initialName)
        private set
    
    fun updateName(newName: String) {
        name = newName  // 变化会被 Snapshot 追踪
    }
}

// 使用 @Stable 的承诺：
// 1. equals() 结果一致（相同实例总是返回 true）
// 2. 属性变化会通知 Composition
// 3. 所有公开属性也是稳定的
```

### 选择哪个注解？

```
决策流程

                    ┌───────────────────┐
                    │ 类的所有属性都是  │
                    │ val 且不可变？    │
                    └─────────┬─────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
        ┌─────────┐                     ┌─────────┐
        │   是    │                     │   否    │
        └────┬────┘                     └────┬────┘
             │                               │
             ▼                               ▼
    ┌────────────────┐            ┌────────────────────┐
    │ @Immutable     │            │ 可变属性是否通过    │
    │                │            │ MutableState 管理？ │
    └────────────────┘            └──────────┬─────────┘
                                             │
                              ┌──────────────┴──────────────┐
                              │                             │
                              ▼                             ▼
                        ┌─────────┐                   ┌─────────┐
                        │   是    │                   │   否    │
                        └────┬────┘                   └────┬────┘
                             │                             │
                             ▼                             ▼
                    ┌────────────────┐           ┌────────────────┐
                    │ @Stable        │           │ 不要使用注解    │
                    │                │           │ 重构代码        │
                    └────────────────┘           └────────────────┘
```

> ⚠️ **注解是契约，不是魔法**
> 
> 这些注解是你对编译器的**承诺**。如果你标记了 `@Immutable` 但实际上类是可变的，Compose 会做出错误的跳过决策，导致 UI 不更新。编译器不会验证你的承诺！

## 四、诊断稳定性问题

### 使用 Compose Compiler Reports

Compose 编译器可以生成详细的稳定性报告：

```kotlin
// build.gradle.kts
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_reports")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
}
```

运行编译后，你会在 `build/compose_reports/` 目录下找到：

- `*-classes.txt`：每个类的稳定性分析
- `*-composables.txt`：每个 Composable 的可跳过性分析
- `*-composables.csv`：CSV 格式的数据，便于分析

### 解读报告

```
// 示例：classes.txt
unstable class UserProfile {
    stable val id: String
    stable val name: String
    unstable val friends: List<User>  // ← 这里导致整个类不稳定！
}

// 示例：composables.txt
restartable fun UserCard(
    unstable user: UserProfile  // 参数不稳定，无法跳过
)

restartable skippable fun SimpleText(
    stable text: String  // 参数稳定，可以跳过
)
```

### 使用 Layout Inspector

Android Studio 的 Layout Inspector 可以实时显示 Recomposition 次数：

1. 以 Debug 模式运行应用
2. 打开 Tools → Layout Inspector
3. 启用 "Show Recomposition Counts"
4. 观察哪些组件频繁重组

## 五、常见稳定性问题与解决方案

### 问题 1：List/Set/Map 参数

```kotlin
// ❌ 不稳定：List 类型
@Composable
fun UserList(users: List<User>) { ... }

// ✅ 解决方案 1：使用 Kotlinx Immutable Collections
@Composable
fun UserList(users: ImmutableList<User>) { ... }

// ✅ 解决方案 2：包装在稳定类中
@Immutable
data class UserListWrapper(val users: List<User>)

@Composable
fun UserList(wrapper: UserListWrapper) { ... }
```

> 📚 **推荐库**
> 
> [Kotlinx Immutable Collections](https://github.com/Kotlin/kotlinx.collections.immutable) - Kotlin 官方的不可变集合库

### 问题 2：外部模块的类

```kotlin
// ❌ 外部库的类，编译器无法分析
@Composable
fun DateDisplay(date: java.time.LocalDate) { ... }

// ✅ 解决方案：配置稳定性配置文件
// stability_config.txt
java.time.LocalDate
java.time.LocalDateTime
kotlinx.datetime.*

// build.gradle.kts
composeCompiler {
    stabilityConfigurationFile = file("stability_config.txt")
}
```

### 问题 3：Lambda 捕获

```kotlin
// ❌ Lambda 捕获了不稳定的值
@Composable
fun ItemList(items: List<Item>, viewModel: ViewModel) {
    items.forEach { item ->
        ItemRow(
            item = item,
            onClick = { viewModel.onItemClick(item) }  // 每次创建新 lambda
        )
    }
}

// ✅ 解决方案：使用 remember 或方法引用
@Composable
fun ItemList(items: List<Item>, viewModel: ViewModel) {
    items.forEach { item ->
        ItemRow(
            item = item,
            onClick = remember(item.id) { { viewModel.onItemClick(item) } }
        )
    }
}
```

### 问题 4：含有 var 的 data class

```kotlin
// ❌ 有 var 属性，不稳定
data class Counter(
    var count: Int  // var！
)

// ✅ 解决方案 1：全部改为 val
data class Counter(
    val count: Int
)

// ✅ 解决方案 2：使用 MutableState
@Stable
class CounterState {
    var count by mutableStateOf(0)
}
```

## 六、稳定性与性能的关系

让我们用一个实际例子来说明稳定性对性能的影响：

```kotlin
// 场景：一个聊天应用的消息列表

// ❌ 不稳定版本
data class Message(
    val id: String,
    val text: String,
    val reactions: List<Reaction>  // List 不稳定！
)

@Composable
fun MessageList(messages: List<Message>) {
    LazyColumn {
        items(messages, key = { it.id }) { message ->
            MessageItem(message)  // 每次都重组！
        }
    }
}

// 当用户发送新消息时：
// - 所有 MessageItem 都会重组
// - 即使它们的内容没有变化
// - 因为 Message 类不稳定
```

```kotlin
// ✅ 稳定版本
@Immutable
data class Message(
    val id: String,
    val text: String,
    val reactions: ImmutableList<Reaction>  // 使用不可变集合
)

@Composable
fun MessageList(messages: ImmutableList<Message>) {
    LazyColumn {
        items(messages, key = { it.id }) { message ->
            MessageItem(message)  // 只有变化的消息才重组
        }
    }
}

// 当用户发送新消息时：
// - 只有新的 MessageItem 会组合
// - 旧消息的 MessageItem 被跳过
// - 性能大幅提升！
```

## 七、Strong Skipping Mode（实验性）

Compose 1.5.4+ 引入了 **Strong Skipping Mode**，它改变了稳定性的默认行为：

```kotlin
// build.gradle.kts
composeCompiler {
    enableStrongSkippingMode = true
}
```

在 Strong Skipping Mode 下：

- 所有 Composable 函数都可以被跳过（不再要求所有参数都稳定）
- 不稳定参数使用 **实例相等性（===）** 而非 `equals()` 来比较
- Lambda 参数会被自动记忆化

> 🔬 **Strong Skipping 的权衡**
> 
> Strong Skipping 减少了对稳定性注解的需求，但也改变了比较语义。如果你依赖 `equals()` 来判断是否重组，需要注意这个变化。

> 📚 **深入阅读**
> 
> [Strong Skipping Mode Explained - Android Developers Blog](https://android-developers.googleblog.com/2024/02/jetpack-compose-strong-skipping-mode.html)

## 八、最佳实践总结

### DO ✅

- 优先使用 `val` 和不可变数据结构
- 为不可变 data class 添加 `@Immutable`
- 为包含 `MutableState` 的类添加 `@Stable`
- 使用 Kotlinx Immutable Collections 替代标准集合
- 定期检查 Compose Compiler Reports
- 使用 Layout Inspector 监控重组

### DON'T ❌

- 不要为可变类添加 `@Immutable`
- 不要忽略编译器报告中的不稳定警告
- 不要在 Composable 内部创建新的 lambda（除非必要）
- 不要假设所有 data class 都是稳定的

## 总结

Compose 的稳定性系统是智能重组的基础：

- **稳定性**决定了 Compose 能否安全跳过重组
- **@Immutable** 用于完全不可变的类型
- **@Stable** 用于变化可被追踪的类型
- **编译器报告**是诊断问题的最佳工具
- **不可变集合**是解决集合稳定性问题的关键

理解并正确使用稳定性系统，可以显著提升 Compose 应用的性能。

---

## 推荐阅读

- [Stability in Compose - Android Developers](https://developer.android.com/jetpack/compose/performance/stability)
- [Composable Metrics - Chris Banes](https://chris.banes.dev/composable-metrics/)
- [Donut-hole Skipping - Jetpack Compose App](https://www.jetpackcompose.app/articles/donut-hole-skipping-in-jetpack-compose)


