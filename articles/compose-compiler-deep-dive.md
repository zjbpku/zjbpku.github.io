# Compose Compiler 深度剖析：从源码到优化的完整之旅

> **发布日期**: 2024-12-16  
> **阅读时间**: 约 50-60 分钟  
> **标签**: Compose Compiler, IR Transform, Stability, Restartable/Skippable, Strong Skipping, Lambda Memoization

当你写下第一个 `@Composable` 函数时，可能不会意识到 Compose 编译器在背后做了大量工作。从简单的函数调用到复杂的重组优化，这一切都依赖于 Compose Compiler Plugin 的强大能力。本文将带你深入探索这个编译器的内部机制，理解它如何将你的 Kotlin 代码转换为高效的运行时实现。

## 📚 官方参考

- [Thinking in Compose - Android Developers](https://developer.android.com/jetpack/compose/mental-model)
- [Compose Compiler Source Code](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/compiler/compiler-hosted/)

## 目录

- [I. 引言：为什么需要 Compose Compiler？](#i-引言为什么需要-compose-compiler)
- [II. IR 转换完整流程](#ii-ir-转换完整流程)
- [III. 稳定性推断系统](#iii-稳定性推断系统-stability-inference)
- [IV. Restartable 与 Skippable 机制](#iv-restartable-与-skippable-机制)
- [V. Lambda 记忆化原理](#v-lambda-记忆化原理)
- [VI. Strong Skipping Mode 深度解析](#vi-strong-skipping-mode-深度解析)
- [VII. 编译器报告与调试](#vii-编译器报告与调试)
- [VIII. 总结与最佳实践](#viii-总结与最佳实践)

## I. 引言：为什么需要 Compose Compiler？

Compose 声明式 UI 与传统命令式的本质区别在于：你描述 UI 应该是什么样子，而不是如何修改它。这种转变带来了革命性的优势，但也需要一个强大的编译器来支持。

### Compose 声明式 UI 与传统命令式的本质区别

传统的 Android View 系统采用**命令式**编程模型：你创建 View 对象，持有它们的引用，然后通过方法调用来修改它们的状态。这种方式直观，但存在明显的性能和维护问题：

- **状态同步困难**：UI 状态分散在各个 View 对象中，难以保证一致性
- **性能开销大**：每次更新都需要遍历 View 树，找到目标 View 并修改
- **内存占用高**：每个 View 都是独立的对象，持有大量状态

Compose 采用了**声明式**模型：你描述 UI 应该是什么样子，而不是如何修改它。这种转变带来了革命性的优势，但也需要一个强大的编译器来支持：

> 💡 **核心洞察**  
> Compose 的声明式模型要求编译器能够：
> - 追踪函数调用和参数变化
> - 自动推断哪些部分可以跳过重组
> - 优化 Lambda 表达式的记忆化
> - 生成高效的运行时代码

### 编译器插件在 Compose 架构中的核心地位

Compose Compiler Plugin 是 Compose 架构的核心组件，它工作在 Kotlin 编译器的 IR（Intermediate Representation）层面，在源码和字节码之间进行转换。这个插件负责：

1. **函数转换**：将 `@Composable` 函数转换为带有 `$composer` 参数的版本
2. **稳定性推断**：分析类型系统，推断哪些类型是稳定的
3. **跳过优化**：生成代码来检测参数变化，实现智能跳过
4. **Lambda 处理**：自动包装需要记忆化的 Lambda 表达式

```
Compose 架构层次

┌─────────────────────────────────────┐
│   Kotlin Source Code                │
│   @Composable fun MyComponent()     │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│   Compose Compiler Plugin           │
│   • IR Transformation                │
│   • Stability Inference              │
│   • Skip Logic Generation            │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│   Transformed IR                     │
│   fun MyComponent($composer, ...)   │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│   Bytecode                           │
│   (JVM/Android Runtime)             │
└─────────────────────────────────────┘
```

### 本文将回答的关键问题

通过本文，你将理解：

- 编译器如何将你的 `@Composable` 函数转换为运行时代码？
- 稳定性推断系统如何工作，为什么某些类型被标记为不稳定？
- Restartable 和 Skippable 机制如何实现智能重组优化？
- Lambda 表达式为什么需要特殊处理，编译器如何自动优化？
- Strong Skipping Mode 带来了哪些改进，如何正确使用？
- 如何利用编译器报告来诊断和优化性能问题？

## II. IR 转换完整流程

Compose Compiler Plugin 工作在 Kotlin 编译器的 IR 层面，这是理解整个转换流程的关键。让我们从 Kotlin 源码到最终字节码的完整路径开始。

```
IR 转换完整流程

Kotlin Source
    │
    ▼
Frontend (Parser, Type Checker)
    │
    ▼
IR (Intermediate Representation)
    │
    ▼
Compose Plugin (IR Transformation)
    │  • ComposableFunctionBodyTransformer
    │  • ComposerParamTransformer
    │  • LiveLiteralTransformer
    │
    ▼
Transformed IR
    │
    ▼
Backend (Code Generator)
    │
    ▼
Bytecode
```

### 2.1 Compose Compiler Plugin 架构

Compose Compiler Plugin 通过 Kotlin 编译器的插件机制集成，主要入口是 `ComposeIrGenerationExtension`。这个扩展在 IR 生成阶段被调用，负责注册各种 IR 转换器。

```kotlin
// Compose Compiler Plugin 入口（简化版）
class ComposeIrGenerationExtension : IrGenerationExtension {
    override fun generate(
        moduleFragment: IrModuleFragment,
        pluginContext: IrPluginContext
    ) {
        // 注册各种 IR 转换器
        val transformer = ComposableFunctionBodyTransformer(pluginContext)
        val paramTransformer = ComposerParamTransformer(pluginContext)
        val liveLiteralTransformer = LiveLiteralTransformer(pluginContext)
        
        // 按顺序执行转换
        moduleFragment.transformChildren(transformer, null)
        moduleFragment.transformChildren(paramTransformer, null)
        moduleFragment.transformChildren(liveLiteralTransformer, null)
    }
}
```

#### Lowering Phases 设计

Kotlin 编译器使用 Lowering Phases 来组织 IR 转换。每个 Phase 负责特定的转换任务，Compose Plugin 注册了多个自定义 Phase：

- **Phase 1: Composer Parameter Injection**：为 `@Composable` 函数注入 `$composer` 参数
- **Phase 2: Function Body Transformation**：转换函数体，添加 Group 管理和跳过逻辑
- **Phase 3: Stability Inference**：推断类型稳定性，生成 `$stable` 信息
- **Phase 4: Skip Logic Generation**：生成参数变化检测和跳过代码
- **Phase 5: Lambda Memoization**：处理 Lambda 表达式的记忆化

> 🔬 **为什么使用 IR？**  
> IR（Intermediate Representation）是编译器在源码和字节码之间的中间表示。使用 IR 而不是直接操作 AST（抽象语法树）或字节码的优势在于：
> - **类型信息完整**：IR 保留了完整的类型信息，便于进行类型推断和优化
> - **结构清晰**：IR 的结构更适合进行程序分析和转换
> - **平台无关**：同样的 IR 可以生成 JVM、Native 或 JS 代码

### 2.2 关键转换步骤

#### ComposableFunctionBodyTransformer：函数体转换

这是最核心的转换器，负责将普通的函数体转换为 Compose 运行时需要的结构。主要工作包括：

1. 在函数开始处插入 `startRestartGroup` 调用
2. 在函数结束处插入 `endRestartGroup` 调用
3. 为每个 Composable 调用添加 Group Key
4. 生成跳过逻辑（如果参数稳定）

```kotlin
// 转换前：你写的代码
@Composable
fun Greeting(name: String) {
    Text("Hello, $name")
    Button(onClick = {}) {
        Text("Click me")
    }
}

// 转换后：编译器生成的代码（简化版）
fun Greeting(
    name: String,
    $composer: Composer,
    $changed: Int
) {
    // 开始 Group，使用基于源码位置的唯一 Key
    $composer.startRestartGroup(0x12345678)
    
    // 跳过逻辑（如果所有参数都稳定且未变化）
    if ($changed == 0 && $composer.skipping) {
        $composer.skipToGroupEnd()
        return
    }
    
    // 转换后的 Composable 调用
    Text(
        "Hello, $name",
        $composer,
        0b00000001  // $changed 位掩码
    )
    Button(
        onClick = {},
        $composer,
        0b00000010,
        content = { Text("Click me", $composer, 0) }
    )
    
    // 结束 Group，返回更新 Scope
    $composer.endRestartGroup()?.updateScope { 
        Greeting(name, $composer, $changed or 1) 
    }
}
```

#### ComposerParamTransformer：参数注入

这个转换器负责为所有 `@Composable` 函数注入必要的参数：

- `$composer: Composer`：用于访问 Slot Table 和执行组合操作
- `$changed: Int`：位掩码，表示哪些参数发生了变化
- `$default: Int`（可选）：用于默认参数的处理

> ⚠️ **参数顺序的重要性**  
> 编译器注入的参数必须按照特定顺序排列，且不能与用户定义的参数冲突。这是为什么 `@Composable` 函数不能有某些特定名称的参数。

#### LiveLiteralTransformer：Live Edit 支持

这个转换器支持 Android Studio 的 Live Edit 功能，允许在运行时修改某些字面量值（如字符串、数字）而无需完全重新编译。

```kotlin
// 转换前
@Composable
fun Example() {
    Text("Hello")  // 字面量
}

// 转换后（启用 Live Edit）
@Composable
fun Example($composer: Composer) {
    val liveString = $composer.liveLiteral("Hello")
    Text(liveString, $composer, 0)
}
```

### 2.3 代码示例：转换前后对比

让我们看一个更复杂的例子，展示编译器如何处理嵌套的 Composable 调用和条件逻辑：

```kotlin
// 转换前：原始代码
@Composable
fun UserProfile(user: User, showDetails: Boolean) {
    Column {
        Text(text = user.name)
        if (showDetails) {
            Text(text = user.email)
            Text(text = user.bio)
        }
    }
}

// 转换后：编译器生成的代码（简化版，展示关键部分）
fun UserProfile(
    user: User,
    showDetails: Boolean,
    $composer: Composer,
    $changed: Int
) {
    $composer.startRestartGroup(0xABCD1234)
    
    // 参数变化检测（简化）
    val changedUser = $changed and 0b00000001 != 0
    val changedShowDetails = $changed and 0b00000010 != 0
    
    if (!changedUser && !changedShowDetails && $composer.skipping) {
        $composer.skipToGroupEnd()
        return
    }
    
    // 转换后的 Column 调用
    Column(
        $composer = $composer,
        $changed = 0,
        content = {
            Text(
                text = user.name,
                $composer = $composer,
                $changed = if (changedUser) 0b00000001 else 0
            )
            
            // 条件逻辑：编译器会为每个分支生成不同的 Group Key
            if (showDetails) {
                $composer.startRestartGroup(0xEF567890)  // if 分支的 Key
                Text(text = user.email, $composer, 0)
                Text(text = user.bio, $composer, 0)
                $composer.endRestartGroup()
            } else {
                $composer.startRestartGroup(0xEF567891)  // else 分支的 Key（如果存在）
                $composer.endRestartGroup()
            }
        }
    )
    
    $composer.endRestartGroup()?.updateScope {
        UserProfile(user, showDetails, $composer, $changed or 0b00000011)
    }
}
```

> 💡 **关键观察**  
> 从这个例子可以看出：
> - 每个 Composable 调用都有自己的 Group Key，基于源码位置生成
> - 条件分支会生成不同的 Group Key，确保位置记忆化的正确性
> - `$changed` 位掩码会逐层传递，子组件可以基于父组件的参数变化信息进行优化

> 📚 **源码参考**  
> [Compose Compiler Plugin Source](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/compiler/compiler-hosted/src/main/java/androidx/compose/compiler/plugins/kotlin/)

## III. 稳定性推断系统 (Stability Inference)

稳定性推断是 Compose Compiler 的核心功能之一。它决定了编译器能否生成跳过优化代码，直接影响重组性能。理解稳定性系统的工作原理，是编写高效 Compose 代码的关键。

### 3.1 稳定性的定义

在 Compose 中，一个类型被认为是**稳定（Stable）**的，当且仅当满足以下两个条件：

1. **值相等性（Value Equality）**：如果两个实例的值相等（通过 `equals()` 比较），那么它们被认为是相同的，不会触发重组
2. **不可变性（Immutability）**：类型的所有公共属性在创建后不会改变，或者如果改变，会通知 Compose 系统

> 💡 **为什么稳定性重要？**  
> 只有当参数类型稳定时，编译器才能生成跳过逻辑。如果参数不稳定，Compose 无法判断参数是否真的发生了变化，只能每次都执行重组，这会严重影响性能。

#### 稳定类型的示例

```kotlin
// ✅ 稳定类型：基本类型
@Composable
fun Example(
    count: Int,           // 稳定
    name: String,        // 稳定（String 是不可变的）
    enabled: Boolean      // 稳定
) { }

// ✅ 稳定类型：标记为 @Stable 的类
@Stable
class User(
    val name: String,
    val age: Int
) {
    // 所有属性都是 val，且类型稳定
}

// ✅ 稳定类型：标记为 @Immutable 的类
@Immutable
data class Point(
    val x: Int,
    val y: Int
)
```

#### 不稳定类型的示例

```kotlin
// ❌ 不稳定类型：可变类
class MutableUser {
    var name: String = ""  // var 属性导致不稳定
    var age: Int = 0
}

// ❌ 不稳定类型：包含不稳定属性的类
class Container {
    val items: MutableList<String> = mutableListOf()  // MutableList 不稳定
}

// ❌ 不稳定类型：接口或抽象类（除非标记为 @Stable）
interface Shape  // 默认不稳定
abstract class BaseComponent  // 默认不稳定
```

### 3.2 推断算法

Compose Compiler 使用一套复杂的算法来推断类型的稳定性。这个算法在编译时运行，分析类型的结构来确定其稳定性。

#### 基本类型的稳定性

所有 Kotlin 基本类型（`Int`、`String`、`Boolean`、`Float`、`Double` 等）都被认为是稳定的：

- 它们是不可变的
- 值相等性比较是可靠的
- 编译器可以生成高效的比较代码

#### 类的稳定性推断规则

对于自定义类，编译器使用以下规则：

1. **检查注解**：如果类被标记为 `@Stable` 或 `@Immutable`，直接认为稳定
2. **检查属性**：所有公共属性必须是 `val`，且类型稳定
3. **检查构造函数**：所有构造函数参数的类型必须稳定
4. **检查继承**：如果继承自其他类，父类也必须稳定

```kotlin
// 编译器推断过程（伪代码）
fun isStable(type: KotlinType): Boolean {
    // 1. 检查注解
    if (type.hasAnnotation("Stable") || type.hasAnnotation("Immutable")) {
        return true
    }
    
    // 2. 基本类型总是稳定
    if (type is PrimitiveType) {
        return true
    }
    
    // 3. 检查所有属性
    for (property in type.getProperties()) {
        if (property.isVar()) {  // var 属性导致不稳定
            return false
        }
        if (!isStable(property.type)) {  // 递归检查属性类型
            return false
        }
    }
    
    return true
}
```

#### 泛型参数的处理

对于泛型类型，编译器需要检查类型参数：

```kotlin
// 示例：List 的稳定性
List<String>      // ✅ 稳定：String 稳定，List 本身不可变
List<User>        // ✅ 稳定：如果 User 稳定
List<MutableUser> // ❌ 不稳定：MutableUser 不稳定

MutableList<String> // ❌ 不稳定：MutableList 本身可变
```

> ⚠️ **常见陷阱**  
> 即使泛型参数类型稳定，如果容器类型本身可变（如 `MutableList`、`HashMap`），整个类型仍然不稳定。这是因为容器内容可能在外部被修改，而 Compose 无法追踪这些变化。

### 3.3 $stable 位掩码生成

编译器为每个 `@Composable` 函数生成一个 `$stable` 位掩码，用于快速判断哪些参数是稳定的。这个位掩码与 `$changed` 位掩码配合使用，实现跳过优化。

#### 位掩码结构

每个参数占用 1 位，表示该参数类型是否稳定：

```
参数 0 (name: String)        [1] 稳定
参数 1 (user: User)          [1] 稳定
参数 2 (items: List<Item>)   [1] 稳定
参数 3 (callback: () -> Unit) [0] 不稳定（Lambda）
```

```kotlin
// 示例：$stable 位掩码生成
@Composable
fun Example(
    name: String,              // bit 0: 1 (稳定)
    user: User,                // bit 1: 1 (稳定)
    items: List<Item>,        // bit 2: 1 (稳定)
    callback: () -> Unit        // bit 3: 0 (不稳定)
) { }

// 编译器生成的 $stable 位掩码
val $stable = 0b0111  // 前三个参数稳定，最后一个不稳定

// 在跳过逻辑中使用
val stableChanged = $changed and $stable
if (stableChanged == 0 && $composer.skipping) {
    // 所有稳定参数都没变化，可以跳过
    $composer.skipToGroupEnd()
}
```

#### 编译器如何生成稳定性信息

编译器在 IR 转换阶段分析每个参数类型，生成稳定性信息：

1. **类型分析**：对每个参数类型调用稳定性推断算法
2. **位掩码构建**：根据推断结果构建 `$stable` 位掩码
3. **代码生成**：在函数开始处生成稳定性检查代码

```
稳定性推断流程

@Composable 函数签名
    │
    ▼
遍历每个参数
    │
    ├─→ 参数类型分析
    │   ├─ 检查 @Stable/@Immutable 注解
    │   ├─ 检查属性可变性
    │   └─ 递归检查嵌套类型
    │
    ▼
生成 $stable 位掩码
    │
    ▼
在跳过逻辑中使用
```

### 3.4 实战：查看和优化稳定性报告

Compose Compiler 可以生成详细的稳定性报告，帮助你诊断和优化性能问题。

#### 启用报告生成

```kotlin
// build.gradle.kts
android {
    composeOptions {
        compilerExtensionVersion = "1.5.4"
    }
}

composeCompiler {
    // 启用报告生成
    reportsDestination = layout.buildDirectory.dir("compose_reports")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
}
```

#### 解读 composables.txt

编译后，在 `build/compose_metrics/composables.txt` 文件中可以查看每个 Composable 函数的稳定性信息：

```
// composables.txt 示例
restartable skippable scheme("[androidx.compose.ui.UiComposable]") fun UserProfile(
  stable user: User
  unstable onEdit: Function0<kotlin.Unit>
  stable count: Int
)
  Nested classes = []
  Nested parameters = []
  Unstable parameters = [onEdit]
  Stable parameters = [user, count]
  Skippable: true
  Restartable: true
  Strong Skipping: enabled
  Source location: UserProfile.kt:15:5
```

关键字段解读：

- **restartable**：函数可以被重新调用（重组）
- **skippable**：函数可以被跳过（如果参数没变化）
- **stable/unstable parameters**：列出稳定和不稳定的参数
- **Strong Skipping**：是否启用了 Strong Skipping Mode
- **Source location**：源码位置（如果启用了 `includeSourceInformation`）

#### 优化策略

根据稳定性报告，可以采取以下优化策略：

1. **添加 @Stable 注解**：对于不可变的数据类，添加 `@Stable` 注解
2. **使用不可变集合**：将 `MutableList` 改为 `List`
3. **提取稳定参数**：将不稳定的复杂对象拆分为稳定的基本类型参数
4. **使用 @Immutable**：对于完全不可变的数据类，使用 `@Immutable`

```kotlin
// ❌ 优化前：不稳定
@Composable
fun UserCard(user: MutableUser) {  // MutableUser 不稳定
    Text(user.name)
}

// ✅ 优化后：稳定
@Stable
data class User(
    val name: String,
    val age: Int
)

@Composable
fun UserCard(user: User) {  // 现在稳定了
    Text(user.name)
}
```

> ✅ **最佳实践**  
> 定期检查稳定性报告，确保关键路径上的 Composable 函数参数尽可能稳定。这可以显著提升重组性能，特别是在列表渲染等高频场景中。

> 📚 **深入阅读**  
> - [Compose Stability - Android Developers](https://developer.android.com/jetpack/compose/performance/stability)
> - [StabilityInferencer.kt - Compose Compiler](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/compiler/compiler-hosted/src/main/java/androidx/compose/compiler/plugins/kotlin/analysis/StabilityInferencer.kt)

## IV. Restartable 与 Skippable 机制

Restartable 和 Skippable 是 Compose 重组优化的核心机制。理解这两个概念的区别和实现原理，对于编写高性能的 Compose 代码至关重要。

### 4.1 概念辨析

虽然 Restartable 和 Skippable 经常一起出现，但它们代表不同的概念：

#### Restartable：可以被重新调用

一个 `@Composable` 函数是 **Restartable** 的，意味着当它读取的状态发生变化时，它可以被重新调用（重组）。这是 Compose 的基本能力，几乎所有 Composable 函数都是 Restartable 的。

```kotlin
// Restartable 函数示例
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")  // 读取 count，当 count 变化时会重组
    }
}
```

#### Skippable：可以被跳过

一个 `@Composable` 函数是 **Skippable** 的，意味着当它的参数没有变化时，可以跳过执行，直接复用之前的结果。这需要满足特定条件：

- 所有参数类型都是稳定的
- 参数值没有变化（通过 `$changed` 位掩码判断）
- 函数体中没有读取可变状态

```kotlin
// Skippable 函数示例
@Composable
fun Greeting(name: String) {  // String 稳定
    Text("Hello, $name")  // 如果 name 没变，可以跳过
}

// 调用
Greeting("Alice")  // 首次执行
Greeting("Alice")  // 跳过！参数没变
Greeting("Bob")    // 执行，参数变了
```

#### 两者的关系和区别

| 特性 | Restartable | Skippable |
|------|-------------|-----------|
| **定义** | 可以被重新调用 | 可以被跳过执行 |
| **触发条件** | 读取的状态变化 | 参数没有变化 |
| **前提条件** | 无特殊要求 | 参数必须稳定 |
| **性能影响** | 必要的重组 | 避免不必要的重组 |
| **默认状态** | 所有 Composable 都是 | 需要满足条件 |

> 🔬 **关键理解**  
> 一个函数可以同时是 Restartable 和 Skippable：当参数变化时，它是 Restartable 的（会被重新调用）；当参数没变化时，它是 Skippable 的（会被跳过）。这两个特性是互补的，共同实现了 Compose 的智能重组优化。

### 4.2 $changed 参数深度解析

`$changed` 参数是 Compose Compiler 注入的一个位掩码，用于追踪哪些参数发生了变化。这是实现 Skippable 机制的核心。

#### 位掩码结构（每参数 3 位）

每个参数在 `$changed` 位掩码中占用 3 位，编码了参数的状态信息：

```
参数 0 (name: String)
    bit 0: 0 (SAME)
    bit 1-2: 11 (STATIC)

参数 1 (count: Int)
    bit 0: 1 (CHANGED)
    bit 1-2: 01 (SAME)

参数 2 (user: User)
    bit 0: 0 (SAME)
    bit 1-2: 10 (UNCERTAIN)
```

#### 状态编码

每 3 位编码了以下信息：

- **bit 0 (最低位)**：是否"脏"（需要检查）
  - `0`：参数未变化（SAME）
  - `1`：参数可能变化（CHANGED）
- **bit 1-2**：稳定性信息
  - `00`：UNCERTAIN（不确定，需要运行时比较）
  - `01`：SAME（相同，编译时确定）
  - `10`：STATIC（静态，永远不会变化）
  - `11`：CHANGED（已变化，编译时确定）

```kotlin
// $changed 位掩码示例
@Composable
fun Example(
    name: String,      // bits 0-2
    count: Int,        // bits 3-5
    user: User         // bits 6-8
) { }

// 调用示例
val name = "Alice"
val count = 42
val user = User("Bob", 30)

// 首次调用：所有参数都是 CHANGED
Example(name, count, user, $composer, $changed = 0b111_111_111)

// 第二次调用，name 和 count 没变，user 变了
val newUser = User("Charlie", 25)
Example(name, count, newUser, $composer, $changed = 0b111_000_000)
//                                    ↑      ↑      ↑
//                                  user   count  name
//                                 变化    未变   未变
```

#### 参数比较逻辑

编译器生成的代码会检查每个参数是否变化。对于稳定类型，使用值比较；对于不稳定类型，使用引用比较（在传统模式下）或实例相等性（在 Strong Skipping 模式下）：

```kotlin
// 编译器生成的参数比较逻辑（简化版）
fun calculateChanged(
    oldName: String,
    newName: String,
    oldCount: Int,
    newCount: Int,
    oldUser: User,
    newUser: User
): Int {
    var changed = 0
    
    // 参数 0: name (String - 稳定)
    if (oldName != newName) {
        changed = changed or 0b000000001  // bit 0
    }
    
    // 参数 1: count (Int - 稳定)
    if (oldCount != newCount) {
        changed = changed or 0b000001000  // bit 3
    }
    
    // 参数 2: user (User - 如果稳定，使用值比较；否则使用引用比较)
    if (oldUser != newUser) {  // 或 oldUser !== newUser（不稳定时）
        changed = changed or 0b001000000  // bit 6
    }
    
    return changed
}
```

### 4.3 跳过逻辑生成

编译器根据函数的参数稳定性，决定是否生成跳过逻辑。只有当所有参数都稳定时，才会生成完整的跳过代码。

#### 编译器如何决定是否生成跳过代码

编译器使用以下规则：

1. **检查参数稳定性**：如果所有参数都稳定，生成跳过逻辑
2. **检查函数体**：如果函数体内读取了可变状态，仍然可以跳过（但会在状态变化时重组）
3. **生成位掩码检查**：生成代码来检查 `$changed` 位掩码

```kotlin
// 完全 Skippable 的函数
@Composable
fun FullySkippable(
    name: String,      // 稳定
    count: Int          // 稳定
) {
    Text("$name: $count")
}

// 编译器生成的代码（简化版）
fun FullySkippable(
    name: String,
    count: Int,
    $composer: Composer,
    $changed: Int
) {
    $composer.startRestartGroup(0x12345678)
    
    // 跳过逻辑：所有参数稳定，检查是否都没变化
    if ($changed == 0 && $composer.skipping) {
        // 所有稳定参数都没变化，可以跳过
        $composer.skipToGroupEnd()
        return
    }
    
    Text("$name: $count", $composer, 0)
    
    $composer.endRestartGroup()?.updateScope {
        FullySkippable(name, count, $composer, $changed or 0b00000011)
    }
}
```

#### startRestartGroup / endRestartGroup 机制

每个 Restartable 函数都会被包装在 `startRestartGroup` 和 `endRestartGroup` 调用之间：

- **startRestartGroup**：开始一个新的 Group，记录函数的位置和参数信息
- **endRestartGroup**：结束 Group，返回一个 `RecomposeScope`，用于后续的重组触发

```kotlin
// startRestartGroup 的作用
fun Composer.startRestartGroup(key: Int) {
    // 1. 在 Slot Table 中创建新的 Group
    // 2. 记录 Group Key（用于位置匹配）
    // 3. 设置当前 Group 的上下文
}

// endRestartGroup 的作用
fun Composer.endRestartGroup(): RecomposeScope? {
    // 1. 关闭当前 Group
    // 2. 返回 RecomposeScope（用于触发重组）
    // 3. 如果函数不可重启，返回 null
}
```

#### updateScope 与 Invalidation 关联

`updateScope` 返回的 Lambda 会在状态变化时被调用，触发重组：

```kotlin
// updateScope 的使用
$composer.endRestartGroup()?.updateScope { // 返回 RecomposeScope
    // 这个 Lambda 会被存储，当相关状态变化时调用
    MyComposable(param1, param2, $composer, $changed or 0b11)
}

// 当函数内读取的状态变化时
val state = remember { mutableStateOf(0) }
Text(state.value.toString())  // 读取 state

// state 变化时，会调用 updateScope 中的 Lambda，触发重组
state.value = 1  // → 触发重组 → 调用 updateScope 的 Lambda
```

### 4.4 代码示例：完整的转换流程图解

让我们看一个完整的例子，展示从源码到运行时的完整转换过程：

```kotlin
// 转换前：原始代码
@Composable
fun ProductCard(
    product: Product,
    onAddToCart: () -> Unit
) {
    Column {
        Text(product.name)
        Text(product.price.toString())
        Button(onClick = onAddToCart) {
            Text("Add to Cart")
        }
    }
}

// 转换后：编译器生成的代码（完整版）
fun ProductCard(
    product: Product,
    onAddToCart: () -> Unit,
    $composer: Composer,
    $changed: Int,
    $default: Int = 0
) {
    // 1. 开始 Restart Group
    $composer.startRestartGroup(0xABCD1234)
    
    // 2. 参数变化检测（假设 Product 稳定，Lambda 不稳定）
    val changedProduct = $changed and 0b00000001 != 0
    val changedCallback = $changed and 0b00000010 != 0
    
    // 3. 跳过逻辑（部分跳过：只检查稳定参数）
    val $stable = 0b00000001  // 只有 product 稳定
    val stableChanged = $changed and $stable
    
    if (stableChanged == 0 && !changedCallback && $composer.skipping) {
        // 稳定参数没变，且回调没变（通过引用比较），可以跳过
        $composer.skipToGroupEnd()
    } else {
        // 4. 执行函数体
        Column(
            $composer = $composer,
            $changed = 0,
            content = {
                Text(
                    text = product.name,
                    $composer = $composer,
                    $changed = if (changedProduct) 0b00000001 else 0
                )
                Text(
                    text = product.price.toString(),
                    $composer = $composer,
                    $changed = if (changedProduct) 0b00000001 else 0
                )
                Button(
                    onClick = onAddToCart,
                    $composer = $composer,
                    $changed = if (changedCallback) 0b00000010 else 0,
                    content = {
                        Text("Add to Cart", $composer, 0)
                    }
                )
            }
        )
    }
    
    // 5. 结束 Restart Group，返回 RecomposeScope
    $composer.endRestartGroup()?.updateScope {
        // 当函数内读取的状态变化时，这个 Lambda 会被调用
        ProductCard(product, onAddToCart, $composer, $changed or 0b00000011)
    }
}
```

```
Restartable/Skippable 决策流程

┌─────────────────────┐
│  @Composable 函数   │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 检查参数稳定性       │
│ • 所有参数稳定？     │
└──────────┬──────────┘
           ├─→ 是 → 生成完整跳过逻辑
           │
           └─→ 否 → 生成部分跳过逻辑
                      │
                      ▼
           ┌─────────────────────┐
           │ 生成 $changed 检查   │
           │ • 稳定参数变化？     │
           └──────────┬──────────┘
                      ├─→ 是 → 执行函数体
                      │
                      └─→ 否 → 跳过执行
```

> ✅ **性能优化要点**  
> 要最大化 Skippable 的效果：
> - 确保参数类型稳定（使用 `@Stable` 或 `@Immutable`）
> - 避免在参数中使用不稳定的类型（如 `MutableList`）
> - 对于 Lambda 参数，考虑使用 `rememberUpdatedState` 或启用 Strong Skipping Mode

> 📚 **深入阅读**  
> - [Compose Skippable - Android Developers](https://developer.android.com/jetpack/compose/performance/skippable)
> - [RecomposeScope.kt - Compose Runtime](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/RecomposeScope.kt)

## V. Lambda 记忆化原理

Lambda 表达式在 Compose 中需要特殊处理。理解 Lambda 记忆化的原理，对于避免不必要的重组和编写高性能代码至关重要。

### 5.1 问题背景

在 Compose 中，Lambda 表达式作为参数传递时，会遇到一个根本性问题：每次重组时，Lambda 实例都是新的，即使逻辑完全相同。

#### 为什么普通 Lambda 会破坏跳过优化

考虑以下代码：

```kotlin
// 问题示例
@Composable
fun MyScreen() {
    var count by remember { mutableStateOf(0) }
    
    // 每次重组时，这个 Lambda 都是新实例！
    Button(onClick = { count++ }) {  // ❌ 新 Lambda 实例
        Text("Count: $count")
    }
}

// 编译器生成的代码（简化）
fun MyScreen($composer: Composer) {
    val count = remember { mutableStateOf(0) }
    
    // 每次调用都创建新的 Lambda
    Button(
        onClick = { count.value++ },  // 新实例 → 引用不同 → 无法跳过
        $composer = $composer,
        $changed = 0b00000010  // Lambda 总是"变化"
    )
}
```

问题在于：即使 Lambda 的逻辑完全相同，每次重组时都会创建新的实例。由于 Lambda 默认使用引用相等性比较（`===`），编译器会认为参数"变化"了，导致无法跳过。

> ⚠️ **Lambda 的性能陷阱**  
> 如果 Composable 函数接受 Lambda 参数，且该函数是 Skippable 的，那么每次重组时 Lambda 参数都会"变化"（因为是新实例），导致跳过优化失效。这就是为什么需要 Lambda 记忆化。

#### Capture 变量带来的挑战

当 Lambda 捕获外部变量时，问题更加复杂：

```kotlin
// Lambda 捕获外部变量
@Composable
fun ProductList(products: List<Product>) {
    var selectedId by remember { mutableStateOf(-1) }
    
    products.forEach { product ->
        // Lambda 捕获了 product 和 selectedId
        ProductCard(
            product = product,
            onClick = { selectedId = product.id }  // 捕获了 product 和 selectedId
        )
    }
}
```

这个 Lambda 不仅每次重组时是新实例，还捕获了外部变量。如果 `selectedId` 变化，所有捕获了它的 Lambda 都需要更新，导致大量不必要的重组。

### 5.2 remember 包装机制

Compose Compiler 会自动为需要记忆化的 Lambda 生成 `remember` 包装代码，将其转换为 `ComposableLambda` 类型。

#### ComposableLambda 类型

`ComposableLambda` 是 Compose 运行时提供的特殊类型，用于表示需要记忆化的 Lambda：

```kotlin
// ComposableLambda 的定义（简化）
class ComposableLambda<P, R> {
    val block: (P) -> R
    val changed: Int  // 捕获变量的变化标记
}
```

#### 自动生成的 remember 调用

编译器会检测哪些 Lambda 需要记忆化，并自动生成 `remember` 包装：

```kotlin
// 转换前：你写的代码
@Composable
fun MyButton(onClick: () -> Unit) {
    Button(onClick = onClick) {
        Text("Click me")
    }
}

// 转换后：编译器生成的代码
@Composable
fun MyButton(
    onClick: () -> Unit,
    $composer: Composer,
    $changed: Int
) {
    // 编译器自动生成 remember 包装
    val rememberedOnClick = $composer.remember(onClick) { onClick }
    
    Button(
        onClick = rememberedOnClick,
        $composer = $composer,
        $changed = 0  // 现在可以跳过了！
    ) {
        Text("Click me", $composer, 0)
    }
}
```

#### Capture 变量的追踪

当 Lambda 捕获外部变量时，编译器会追踪这些变量，并在它们变化时更新 Lambda：

```kotlin
// 转换前
@Composable
fun Counter(initialValue: Int) {
    var count by remember { mutableStateOf(initialValue) }
    
    // Lambda 捕获了 count
    Button(onClick = { count++ }) {
        Text("$count")
    }
}

// 转换后（简化）
@Composable
fun Counter(
    initialValue: Int,
    $composer: Composer,
    $changed: Int
) {
    var count by remember { mutableStateOf(initialValue) }
    
    // 编译器追踪捕获的变量（count）
    val rememberedOnClick = $composer.remember(
        count  // 依赖：当 count 变化时，更新 Lambda
    ) { { count++ } }
    
    Button(
        onClick = rememberedOnClick,
        $composer = $composer,
        $changed = 0
    ) {
        Text("$count", $composer, 0)
    }
}
```

> 💡 **编译器如何决定是否记忆化**  
> 编译器使用以下规则决定是否自动记忆化 Lambda：
> - Lambda 作为参数传递给 Composable 函数
> - 接收 Lambda 的 Composable 函数是 Skippable 的
> - Lambda 不是内联的（inline Composable 不需要记忆化）

### 5.3 优化策略

虽然编译器会自动处理大部分情况，但了解优化策略可以帮助你编写更高效的代码。

#### Non-capturing Lambda 的优化

不捕获任何外部变量的 Lambda（Non-capturing Lambda）可以被优化为单例：

```kotlin
// Non-capturing Lambda：可以优化为单例
@Composable
fun SimpleButton() {
    // 这个 Lambda 不捕获任何变量，可以复用
    Button(onClick = { println("Clicked") }) {
        Text("Click")
    }
}

// 编译器优化后（概念上）
val SINGLETON_LAMBDA = { println("Clicked") }

@Composable
fun SimpleButton($composer: Composer) {
    // 直接使用单例，无需 remember
    Button(onClick = SINGLETON_LAMBDA, $composer, 0) {
        Text("Click", $composer, 0)
    }
}
```

#### Inline Composable 的处理

对于内联的 Composable Lambda（如 `Column`、`Row` 的 content 参数），编译器会内联展开，不需要记忆化：

```kotlin
// Inline Composable Lambda
@Composable
inline fun Column(
    content: @Composable () -> Unit  // inline，直接展开
) {
    // content() 直接内联，无需记忆化
}

// 使用
Column {
    Text("Hello")  // 直接内联到 Column 中
}
```

#### rememberUpdatedState 的使用场景

当 Lambda 需要捕获可能变化的值，但你不希望 Lambda 本身变化时，使用 `rememberUpdatedState`：

```kotlin
// 问题场景：Lambda 捕获了会变化的值
@Composable
fun MyScreen(userId: String) {
    // 每次 userId 变化，这个 Lambda 都会是新实例
    LaunchedEffect(Unit) {
        loadUser(userId)  // 捕获了 userId
    }
}

// 解决方案：使用 rememberUpdatedState
@Composable
fun MyScreen(userId: String) {
    val currentUserId by rememberUpdatedState(userId)
    
    // Lambda 实例不变，但内部使用的值会更新
    LaunchedEffect(Unit) {
        loadUser(currentUserId())  // 使用最新的 userId
    }
}
```

> ✅ **最佳实践**  
> 对于需要长期存在的 Lambda（如 `LaunchedEffect`、`DisposableEffect` 的 key），如果它们捕获了可能变化的值，使用 `rememberUpdatedState` 可以避免不必要的重新创建。

### 5.4 代码示例：Lambda 转换前后对比

让我们看一个完整的例子，展示 Lambda 记忆化的完整过程：

```kotlin
// 转换前：原始代码
@Composable
fun ProductList(products: List<Product>) {
    var selectedId by remember { mutableStateOf(-1) }
    
    LazyColumn {
        products.forEach { product ->
            ProductCard(
                product = product,
                isSelected = product.id == selectedId,
                onClick = { selectedId = product.id }  // Lambda 捕获了 product 和 selectedId
            )
        }
    }
}

// 转换后：编译器生成的代码（简化版）
@Composable
fun ProductList(
    products: List<Product>,
    $composer: Composer,
    $changed: Int
) {
    var selectedId by remember { mutableStateOf(-1) }
    
    LazyColumn(
        $composer = $composer,
        $changed = 0,
        content = {
            products.forEach { product ->
                // 编译器为每个 Lambda 生成 remember 包装
                val rememberedOnClick = $composer.remember(
                    product,      // 依赖：product 变化时更新
                    selectedId    // 依赖：selectedId 变化时更新
                ) {
                    { selectedId = product.id }  // Lambda 体
                }
                
                ProductCard(
                    product = product,
                    isSelected = product.id == selectedId,
                    onClick = rememberedOnClick,  // 使用记忆化的 Lambda
                    $composer = $composer,
                    $changed = 0  // 现在可以跳过了！
                )
            }
        }
    )
}
```

```
Lambda 记忆化流程

原始 Lambda
    │
    ├─→ 检查是否捕获变量
    │
    ├─→ 是 → 生成 remember 包装
    │       │
    │       ├─ 追踪捕获的变量
    │       ├─ 变量变化时更新 Lambda
    │       └─ 返回 ComposableLambda
    │
    └─→ 否 → 优化为单例（如果可能）
            │
            └─ 直接复用，无需记忆化
```

> ⚠️ **注意事项**  
> 虽然编译器会自动记忆化 Lambda，但要注意：
> - 记忆化会增加内存开销（每个 Lambda 都需要存储）
> - 捕获的变量变化时，Lambda 会被重新创建
> - 在列表等高频场景中，考虑使用 `key` 来优化

> 📚 **深入阅读**  
> - [Lambda Memoization - Android Developers](https://developer.android.com/jetpack/compose/performance/lambda-memoization)
> - [ComposableLambda.kt - Compose Runtime](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/ComposableLambda.kt)

## VI. Strong Skipping Mode 深度解析

Strong Skipping Mode 是 Compose 1.5.4+ 引入的重大优化，它放宽了跳过优化的条件，允许对不稳定类型也进行跳过优化。理解这个模式的工作原理，对于充分利用 Compose 的性能优化至关重要。

### 6.1 传统模式的局限性

在传统模式下，只有当所有参数类型都稳定时，Composable 函数才能被跳过。这导致了许多实际场景中无法享受跳过优化的好处。

#### 不稳定参数导致无法跳过

在传统模式下，只要有一个参数不稳定，整个函数就无法跳过：

```kotlin
// 传统模式下的问题
@Composable
fun UserProfile(
    user: User,              // 稳定
    onEdit: () -> Unit         // 不稳定（Lambda）
) {
    Column {
        Text(user.name)
        Button(onClick = onEdit) {
            Text("Edit")
        }
    }
}

// 传统模式：即使 user 没变，onEdit 是新 Lambda 实例，无法跳过
UserProfile(
    user = sameUser,        // 相同实例
    onEdit = { edit() }  // 新 Lambda 实例 → 无法跳过
)
```

#### Lambda 参数的性能问题

Lambda 参数是最常见的不稳定类型。在传统模式下，即使 Lambda 的逻辑完全相同，由于是新实例，也会导致无法跳过：

```kotlin
// 每次重组都创建新的 Lambda
@Composable
fun MyScreen() {
    var count by remember { mutableStateOf(0) }
    
    // 每次重组，这个 Lambda 都是新实例
    Counter(
        count = count,
        onIncrement = { count++ }  // 新实例 → Counter 无法跳过
    )
}
```

> ⚠️ **传统模式的限制**  
> 在传统模式下，即使你使用了 `remember` 来记忆化 Lambda，如果接收 Lambda 的 Composable 函数本身无法跳过（因为其他不稳定参数），仍然会导致性能问题。

### 6.2 Strong Skipping 的核心改进

Strong Skipping Mode 通过放宽跳过条件，解决了传统模式的局限性。

#### 对不稳定类型使用实例相等性

在 Strong Skipping 模式下，对于不稳定类型，编译器使用实例相等性（`===`）而不是值相等性来判断参数是否变化：

```kotlin
// Strong Skipping 模式下的比较逻辑
fun calculateChanged(
    oldUser: User,
    newUser: User,
    oldCallback: () -> Unit,
    newCallback: () -> Unit
): Int {
    var changed = 0
    
    // 稳定类型：使用值比较
    if (oldUser != newUser) {  // equals()
        changed = changed or 0b00000001
    }
    
    // 不稳定类型：使用实例比较（Strong Skipping）
    if (oldCallback !== newCallback) {  // ===
        changed = changed or 0b00000010
    }
    
    return changed
}
```

这意味着，如果 Lambda 实例相同（即使类型不稳定），函数仍然可以被跳过。

#### Lambda 的自动记忆化

在 Strong Skipping 模式下，编译器会自动为 Lambda 参数生成记忆化代码，即使接收 Lambda 的函数本身不是完全 Skippable 的：

```kotlin
// Strong Skipping 模式：自动记忆化 Lambda
@Composable
fun MyScreen() {
    var count by remember { mutableStateOf(0) }
    
    // 编译器自动记忆化这个 Lambda
    val rememberedOnIncrement = remember { { count++ } }
    
    Counter(
        count = count,
        onIncrement = rememberedOnIncrement  // 实例不变 → 可以跳过
    )
}
```

#### 默认启用与向后兼容

从 Compose 1.5.4 开始，Strong Skipping Mode 默认启用。它完全向后兼容，不会破坏现有代码：

- 稳定类型的行为保持不变（仍然使用值比较）
- 不稳定类型的行为改变（使用实例比较），但这是性能优化，不会影响正确性
- 如果确实需要传统行为，可以通过编译器选项禁用

> 💡 **为什么默认启用？**  
> Strong Skipping Mode 是一个纯性能优化，不会改变程序的语义。它只是让更多的函数可以享受跳过优化，从而提升性能。因此，默认启用是安全的，且能带来显著的性能提升。

### 6.3 实现细节

Strong Skipping Mode 的实现涉及编译器代码生成逻辑的修改。

#### 比较逻辑的变化

在 Strong Skipping 模式下，编译器生成的比较代码会区分稳定和不稳定类型：

```kotlin
// 传统模式：所有参数都使用值比较（如果稳定）
if (oldParam != newParam) {  // 只对稳定类型有效
    changed = changed or mask
}

// Strong Skipping 模式：不稳定类型使用实例比较
if (paramType.isStable()) {
    if (oldParam != newParam) {  // 值比较
        changed = changed or mask
    }
} else {
    if (oldParam !== newParam) {  // 实例比较
        changed = changed or mask
    }
}
```

#### 对 $changed 参数的影响

在 Strong Skipping 模式下，`$changed` 位掩码的生成逻辑保持不变，但解释方式不同：

- 对于稳定类型：`$changed` 表示值是否变化
- 对于不稳定类型：`$changed` 表示实例是否变化（引用是否相同）

```kotlin
// $changed 位掩码的使用（Strong Skipping）
val stableChanged = $changed and $stable  // 稳定参数的变化
val unstableChanged = $changed and inv($stable)  // 不稳定参数的变化

// 跳过逻辑：所有参数（稳定+不稳定）都没变化
if ($changed == 0 && $composer.skipping) {
    $composer.skipToGroupEnd()
}
```

#### 与传统模式的差异

| 特性 | 传统模式 | Strong Skipping Mode |
|------|----------|---------------------|
| **跳过条件** | 所有参数必须稳定且未变化 | 所有参数（稳定或不稳定）实例未变化 |
| **不稳定类型比较** | 不比较，直接认为变化 | 使用实例相等性（===） |
| **Lambda 处理** | 需要手动 remember | 自动记忆化 |
| **性能** | 较低（很多函数无法跳过） | 较高（更多函数可以跳过） |
| **向后兼容** | N/A | 完全兼容 |

### 6.4 启用方式与注意事项

Strong Skipping Mode 在 Compose 1.5.4+ 中默认启用，但你可以通过编译器选项控制。

#### 启用方式

在 Compose 1.5.4+ 中，Strong Skipping Mode 默认启用，无需额外配置：

```kotlin
// build.gradle.kts
android {
    composeOptions {
        compilerExtensionVersion = "1.5.4"  // 或更高版本
    }
}

// Strong Skipping Mode 默认启用，无需配置
```

如果需要显式控制，可以使用编译器选项：

```kotlin
// build.gradle.kts（显式启用）
composeCompiler {
    enableStrongSkippingMode = true  // 默认 true
}

// 如果需要禁用（不推荐）
composeCompiler {
    enableStrongSkippingMode = false  // 回退到传统模式
}
```

#### 注意事项

虽然 Strong Skipping Mode 是向后兼容的，但需要注意以下几点：

1. **实例相等性假设**：Strong Skipping 假设相同实例表示相同值。如果你的代码依赖值相等性但实例不同，可能会有问题（但这种情况很少见）
2. **Lambda 记忆化**：编译器会自动记忆化 Lambda，但如果你手动记忆化，行为是一致的
3. **性能监控**：使用编译器报告来验证 Strong Skipping 是否生效

> ✅ **最佳实践**  
> 在大多数情况下，你应该使用 Strong Skipping Mode（默认启用）。它提供了更好的性能，且不会影响代码的正确性。只有在特殊情况下（如需要精确控制跳过行为），才考虑禁用。

#### 验证 Strong Skipping 是否生效

可以通过编译器报告来验证 Strong Skipping Mode 是否生效：

```
// composables.txt 中会显示是否启用了 Strong Skipping
restartable skippable scheme("[androidx.compose.ui.UiComposable]") fun Example(
  stable name: String
  unstable callback: Function0<kotlin.Unit>
)
  Strong Skipping: enabled  // 显示是否启用
  Skippable: true           // 即使有不稳定参数，也可以跳过
```

```
Strong Skipping Mode 工作流程

┌─────────────────────┐
│  @Composable 函数   │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ 检查参数类型         │
│ • 稳定？             │
│ • 不稳定？           │
└──────────┬──────────┘
           ├─→ 稳定 → 使用值比较 (==)
           │
           └─→ 不稳定 → 使用实例比较 (===)
                      │
                      ▼
           ┌─────────────────────┐
           │ 生成比较代码         │
           │ • 稳定：old != new   │
           │ • 不稳定：old !== new │
           └──────────┬──────────┘
                      ▼
           ┌─────────────────────┐
           │ 所有参数实例相同？   │
           └──────────┬──────────┘
                      ├─→ 是 → 跳过执行
                      │
                      └─→ 否 → 执行函数体
```

> 📚 **深入阅读**  
> - [Strong Skipping Mode - Android Developers](https://developer.android.com/jetpack/compose/performance/strong-skipping)
> - [What's New in Jetpack Compose Performance - Android Developers Blog](https://android-developers.googleblog.com/2024/01/whats-new-in-jetpack-compose-performance.html)

## VII. 编译器报告与调试

Compose Compiler 提供了丰富的报告和调试工具，帮助你诊断性能问题、理解编译器行为，并优化代码。掌握这些工具的使用，是成为 Compose 性能优化专家的关键。

### 7.1 Compose Compiler Metrics

Compose Compiler 可以生成详细的指标报告，包括每个 Composable 函数的稳定性、可跳过性等信息。

#### 启用报告生成

在 `build.gradle.kts` 中配置编译器选项：

```kotlin
// build.gradle.kts
android {
    composeOptions {
        compilerExtensionVersion = "1.5.4"
    }
}

composeCompiler {
    // 启用报告生成
    reportsDestination = layout.buildDirectory.dir("compose_reports")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
    
    // 可选：启用详细报告
    includeSourceInformation = true
}
```

编译后，报告文件会生成在 `build/compose_reports/` 和 `build/compose_metrics/` 目录下。

#### composables.txt 解读

`composables.txt` 文件包含了每个 `@Composable` 函数的详细信息：

```
// composables.txt 示例
restartable skippable scheme("[androidx.compose.ui.UiComposable]") fun UserProfile(
  stable user: User
  unstable onEdit: Function0<kotlin.Unit>
  stable count: Int
)
  Nested classes = []
  Nested parameters = []
  Unstable parameters = [onEdit]
  Stable parameters = [user, count]
  Skippable: true
  Restartable: true
  Strong Skipping: enabled
  Source location: UserProfile.kt:15:5
```

关键字段解读：

- **restartable**：函数可以被重新调用（重组）
- **skippable**：函数可以被跳过（如果参数没变化）
- **stable/unstable parameters**：列出稳定和不稳定的参数
- **Strong Skipping**：是否启用了 Strong Skipping Mode
- **Source location**：源码位置（如果启用了 `includeSourceInformation`）

#### classes.txt 解读

`classes.txt` 文件包含了类型稳定性信息：

```
// classes.txt 示例
stable class User {
  stable name: String
  stable age: Int
  stable constructor(name: String, age: Int)
}

unstable class MutableUser {
  unstable name: String
  unstable age: Int
  unstable constructor(name: String, age: Int)
}

unstable interface ClickListener {
  unstable onClick: Function0<kotlin.Unit>
}
```

这个文件帮助你理解哪些类型是稳定的，哪些不是，以及为什么。

#### 稳定性报告分析

通过分析报告，你可以：

1. **识别不稳定类型**：找出哪些类型导致函数无法跳过
2. **优化稳定性**：为需要稳定的类型添加 `@Stable` 或 `@Immutable` 注解
3. **验证优化效果**：在优化后重新生成报告，确认改进

> 💡 **报告分析技巧**  
> 重点关注：
> - 高频调用的 Composable 函数（如列表项）
> - 包含不稳定参数的函数
> - 标记为不可跳过的函数（`Skippable: false`）

### 7.2 常见问题诊断

在实际开发中，经常会遇到性能问题。以下是常见问题的诊断和解决方法。

#### 为什么我的 Composable 不能跳过？

如果 Composable 函数无法跳过，可能的原因包括：

1. **参数不稳定**：检查 `composables.txt`，查看哪些参数被标记为不稳定
2. **未启用 Strong Skipping**：确认 Compose Compiler 版本 >= 1.5.4，且未禁用 Strong Skipping
3. **函数体内读取状态**：即使参数没变化，如果函数体内读取的状态变化，仍然会重组

```kotlin
// 问题示例：参数不稳定
@Composable
fun Problematic(data: MutableList<String>) {  // MutableList 不稳定
    Text(data.joinToString())
}

// 解决方案：使用不可变类型
@Composable
fun Fixed(data: List<String>) {  // List 稳定
    Text(data.joinToString())
}
```

#### 如何定位不稳定的类型？

使用以下步骤定位不稳定类型：

1. **查看 composables.txt**：找出哪些参数被标记为不稳定
2. **查看 classes.txt**：检查这些参数的类型定义
3. **分析类型结构**：找出导致不稳定的原因（var 属性、可变集合等）
4. **应用修复**：添加注解或修改类型定义

```kotlin
// 步骤 1: 查看 composables.txt
// 发现 onAction 参数不稳定
unstable onAction: Function0<kotlin.Unit>

// 步骤 2: 查看 classes.txt
// 发现 Function0 接口默认不稳定
unstable interface Function0<R>

// 步骤 3: 解决方案（在 Strong Skipping 模式下自动处理）
// 或使用 remember 手动记忆化
val rememberedOnAction = remember { onAction }
```

#### Lambda 导致重组的排查

Lambda 参数是最常见的导致无法跳过的原因。排查步骤：

1. **确认 Lambda 是否被记忆化**：检查编译器是否自动记忆化，或手动使用 `remember`
2. **检查 Lambda 的依赖**：如果 Lambda 捕获了外部变量，确保这些变量被正确追踪
3. **使用 rememberUpdatedState**：对于需要长期存在的 Lambda，考虑使用 `rememberUpdatedState`

```kotlin
// 问题：Lambda 每次都是新实例
@Composable
fun MyScreen() {
    var count by remember { mutableStateOf(0) }
    
    // 每次重组都创建新 Lambda
    Counter(onIncrement = { count++ })
}

// 解决方案 1: 手动记忆化（传统模式）
@Composable
fun MyScreen() {
    var count by remember { mutableStateOf(0) }
    
    val onIncrement = remember { { count++ } }
    Counter(onIncrement = onIncrement)
}

// 解决方案 2: Strong Skipping 模式自动处理（推荐）
// 在 Compose 1.5.4+ 中，编译器会自动记忆化
```

#### 使用 Layout Inspector 调试

Android Studio 的 Layout Inspector 可以显示 Recomposition 计数，帮助你找到频繁重组的组件：

1. 运行应用并连接到设备/模拟器
2. 打开 Layout Inspector（Tools → Layout Inspector）
3. 选择 Compose 视图
4. 查看 Recomposition 计数列

> ✅ **调试工作流**  
> 推荐的调试流程：
> 1. 使用 Layout Inspector 找出频繁重组的组件
> 2. 查看编译器报告，分析为什么无法跳过
> 3. 应用优化（添加注解、修改类型等）
> 4. 重新生成报告，验证优化效果
> 5. 使用 Layout Inspector 确认重组次数减少

#### 性能分析工具

除了编译器报告，还可以使用以下工具：

- **Android Studio Profiler**：分析 CPU 使用和重组性能
- **Compose Recomposition 计数器**：在代码中添加计数器，追踪重组次数
- **Compose 性能监控库**：使用第三方库来监控和报告性能指标

```kotlin
// 添加重组计数器（调试用）
@Composable
fun TrackedComponent(content: @Composable () -> Unit) {
    val recompositionCount = remember { mutableStateOf(0) }
    recompositionCount.value++
    
    // 在 Logcat 中输出重组次数
    SideEffect {
        Log.d("Recomposition", "Count: ${recompositionCount.value}")
    }
    
    content()
}
```

> 📚 **深入阅读**  
> - [Debug Compose Performance - Android Developers](https://developer.android.com/jetpack/compose/performance/debugging)
> - [Investigate Recomposition - Android Developers](https://developer.android.com/jetpack/compose/performance/investigate-recomposition)

## VIII. 总结与最佳实践

通过本文，我们深入探索了 Compose Compiler 的内部机制。现在让我们总结核心原理，并提供实用的最佳实践指南。

### 核心原理回顾

Compose Compiler 的核心工作流程可以总结为以下几个关键步骤：

```
Compose Compiler 核心流程

┌─────────────────────────────────────────┐
│  1. IR 转换                              │
│  • 注入 $composer 和 $changed 参数      │
│  • 添加 startRestartGroup/endRestartGroup │
│  • 生成 Group Key                        │
└──────────────┬──────────────────────────┘
               ▼
┌─────────────────────────────────────────┐
│  2. 稳定性推断                            │
│  • 分析类型结构                          │
│  • 生成 $stable 位掩码                  │
│  • 标记稳定/不稳定参数                   │
└──────────────┬──────────────────────────┘
               ▼
┌─────────────────────────────────────────┐
│  3. 跳过逻辑生成                          │
│  • 检查参数变化（$changed 位掩码）       │
│  • 生成跳过代码                         │
│  • 支持 Strong Skipping Mode            │
└──────────────┬──────────────────────────┘
               ▼
┌─────────────────────────────────────────┐
│  4. Lambda 处理                          │
│  • 自动记忆化 Lambda                    │
│  • 追踪捕获变量                         │
│  • 优化 Non-capturing Lambda            │
└──────────────┬──────────────────────────┘
               ▼
┌─────────────────────────────────────────┐
│  5. 代码生成                              │
│  • 生成字节码                           │
│  • 运行时优化                           │
└─────────────────────────────────────────┘
```

### 性能优化 Checklist

使用以下清单来确保你的 Compose 代码达到最佳性能：

#### 类型稳定性

- [ ] 为不可变数据类添加 `@Stable` 或 `@Immutable` 注解
- [ ] 使用不可变集合（`List` 而非 `MutableList`）
- [ ] 避免在稳定类型中使用 `var` 属性

#### 参数设计

- [ ] 尽可能使用稳定类型作为参数
- [ ] 将复杂对象拆分为基本类型参数（如果可能）
- [ ] 避免传递不稳定的容器类型

#### Lambda 优化

- [ ] 启用 Strong Skipping Mode（Compose 1.5.4+ 默认启用）
- [ ] 对于长期存在的 Lambda，使用 `rememberUpdatedState`
- [ ] 避免在 Lambda 中捕获频繁变化的值

#### 状态管理

- [ ] 使用 `remember` 缓存计算结果
- [ ] 将状态提升到合适的层级
- [ ] 使用 `derivedStateOf` 派生状态

#### 列表优化

- [ ] 使用 `key` 为列表项提供稳定标识
- [ ] 使用 `LazyColumn`/`LazyRow` 进行虚拟化
- [ ] 避免在列表项中使用不稳定的参数

#### 调试和监控

- [ ] 定期查看编译器报告（composables.txt, classes.txt）
- [ ] 使用 Layout Inspector 监控重组次数
- [ ] 使用 Profiler 分析性能瓶颈

### 常见陷阱与解决方案

| 常见陷阱 | 问题 | 解决方案 |
|---------|------|---------|
| **可变集合作为参数** | MutableList 等类型不稳定，导致无法跳过 | 使用不可变集合（List）或添加 @Stable 注解 |
| **Lambda 每次都是新实例** | 导致接收 Lambda 的函数无法跳过 | 启用 Strong Skipping Mode 或手动 remember |
| **条件语句中的 Composable** | 位置变化导致状态错乱 | 使用 key 明确标识，或提取到独立函数 |
| **忘记使用 remember** | 每次重组都重新计算，性能差 | 对计算结果使用 remember 缓存 |
| **不稳定的数据类** | 包含 var 属性或可变集合 | 改为 val 属性，使用不可变集合，添加 @Stable |

### 性能优化示例

让我们看一个完整的优化示例：

```kotlin
// ❌ 优化前：性能问题
@Composable
fun ProductList(products: MutableList<Product>) {  // 不稳定
    LazyColumn {
        products.forEach { product ->
            // 没有 key，位置记忆化可能出错
            ProductCard(
                product = product,
                onClick = { select(product) }  // 新 Lambda 实例
            )
        }
    }
}

class Product {
    var name: String = ""  // var 导致不稳定
    var price: Double = 0.0
}

// ✅ 优化后：高性能
@Stable  // 添加注解
data class Product(
    val name: String,      // val 属性
    val price: Double,
    val id: Long
)

@Composable
fun ProductList(products: List<Product>) {  // 不可变集合
    LazyColumn {
        products.forEach { product ->
            key(product.id) {  // 使用 key
                // Strong Skipping Mode 自动记忆化 Lambda
                ProductCard(
                    product = product,
                    onClick = { select(product) }
                )
            }
        }
    }
}
```

### 推荐阅读资源

#### 官方文档

- [Compose Performance - Android Developers](https://developer.android.com/jetpack/compose/performance)
- [Thinking in Compose - Android Developers](https://developer.android.com/jetpack/compose/mental-model)
- [Compose Stability - Android Developers](https://developer.android.com/jetpack/compose/performance/stability)

#### 源码参考

- [Compose Compiler Source Code](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/compiler/)
- [Compose Runtime Source Code](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/)

#### 深度文章

- [Jetpack Compose Internals - Jorge Castillo](https://jorgecastillo.dev/book/)
- [Under the hood of Jetpack Compose - Android Developers Blog](https://medium.com/androiddevelopers/under-the-hood-of-jetpack-compose-part-2-of-2-37b2c20c852)

### 总结

通过本文，我们深入理解了 Compose Compiler 的工作原理：

- **IR 转换**：编译器在 IR 层面转换代码，注入必要的参数和逻辑
- **稳定性推断**：编译器分析类型结构，推断稳定性，生成位掩码
- **跳过优化**：基于参数变化检测，智能跳过不必要的重组
- **Lambda 记忆化**：自动处理 Lambda 参数，避免不必要的重组
- **Strong Skipping Mode**：放宽跳过条件，提升性能

理解这些底层机制，不仅能帮助你写出更高效的 Compose 代码，还能在遇到性能问题时快速定位和解决。记住：

> ✅ **关键要点**  
> - 稳定性是性能优化的基础：确保关键路径上的类型稳定
> - 利用编译器报告：定期检查，持续优化
> - 启用 Strong Skipping Mode：享受更好的性能（默认启用）
> - 遵循最佳实践：使用不可变类型、remember、key 等

希望本文能帮助你深入理解 Compose Compiler，写出更高效、更优雅的 Compose 代码！
