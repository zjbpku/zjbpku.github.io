# Compose Runtime 深入原理：从 Composition 到 Applier

> 深入理解 Compose Runtime 的核心机制，包括 Composition、Applier、Recomposer 的工作原理

## 📅 2024-07-05 | ⏱ 28 min | 🏷️ Runtime, Internals, Composition

---

## I. 引言：Runtime 是 Compose 的心脏

Jetpack Compose 的强大之处不仅在于其声明式 API，更在于其精心设计的 Runtime 系统。理解 Runtime 原理，能帮助你：

- 写出更高性能的 Compose 代码
- 深入理解重组（Recomposition）机制
- 排查疑难问题时有据可依

```
┌─────────────────────────────────────────────────────────┐
│                    Compose 架构层次                      │
├─────────────────────────────────────────────────────────┤
│  compose-ui        → UI 组件、Layout、绘制               │
│  compose-runtime   → Composition、Snapshot、Recomposer  │
│  compose-compiler  → @Composable 代码转换               │
└─────────────────────────────────────────────────────────┘
```

本文聚焦 **compose-runtime** 层，这是 Compose 的核心引擎。

---

## II. Composition：组合的容器

### 2.1 什么是 Composition

`Composition` 是 Composable 函数执行后产生的"组合实例"。它持有：

- **Slot Table**：存储所有组合数据的核心数据结构
- **Applier**：将组合变化应用到目标树（如 UI 树）
- **Parent Context**：父级上下文，用于 CompositionLocal 查找

```kotlin
// 简化的 Composition 接口
interface Composition {
    val hasInvalidations: Boolean
    val isDisposed: Boolean
    
    fun setContent(content: @Composable () -> Unit)
    fun dispose()
}

// 创建 Composition
val composition = Composition(
    applier = UiApplier(rootNode),
    parent = recomposer
)
```

### 2.2 Composition 的生命周期

```
创建 → 首次组合 → [重组循环] → 销毁
  │        │           │          │
  │        ▼           ▼          ▼
  │   setContent()  invalidate  dispose()
  │        │           │
  │        ▼           ▼
  └─→ SlotTable 初始化/更新
```

```kotlin
// ComposeView 内部简化逻辑
class ComposeView : ViewGroup {
    private var composition: Composition? = null
    
    override fun onAttachedToWindow() {
        super.onAttachedToWindow()
        composition = setContent { 
            // Your composables
        }
    }
    
    override fun onDetachedFromWindow() {
        composition?.dispose()
        composition = null
        super.onDetachedFromWindow()
    }
}
```

---

## III. Slot Table：组合数据的存储核心

### 3.1 Slot Table 结构

Slot Table 是一个扁平化的数组结构，存储所有组合过程中产生的数据：

```
┌────────────────────────────────────────────────────────┐
│                      Slot Table                        │
├────────────────────────────────────────────────────────┤
│ Group  │ Group  │ Data │ Data │ Group  │ Data │ ...   │
│ Start  │ Fields │ Slot │ Slot │ Start  │ Slot │       │
├────────────────────────────────────────────────────────┤
│ [0]    │ [1-4]  │ [5]  │ [6]  │ [7]    │ [8]  │ ...   │
└────────────────────────────────────────────────────────┘
```

**Group（组）**：代表一个 Composable 调用
- Group Key：用于识别 Composable
- Node Count：子节点数量
- Data Count：数据槽数量
- Parent Index：父组索引

**Data Slot（数据槽）**：存储 `remember` 的值、状态等

### 3.2 Gap Buffer 机制

Slot Table 使用 **Gap Buffer** 优化插入/删除操作：

```
初始状态（Gap 在末尾）：
[A][B][C][D][_][_][_][_]
              ↑ gap start

在 B 后插入 X：
1. 移动 Gap 到插入位置
[A][B][_][_][_][_][C][D]
       ↑ gap start

2. 在 Gap 位置写入
[A][B][X][_][_][_][C][D]
          ↑ gap start
```

```kotlin
// 简化的 Gap Buffer 操作
class SlotTable {
    private var slots: Array<Any?> = arrayOfNulls(32)
    private var gapStart: Int = 0
    private var gapLen: Int = 32
    
    fun moveGapTo(index: Int) {
        if (index < gapStart) {
            // 向左移动：将 [index, gapStart) 复制到 gap 末尾
            val moveCount = gapStart - index
            slots.copyInto(slots, gapStart + gapLen - moveCount, index, gapStart)
            gapStart = index
        } else if (index > gapStart) {
            // 向右移动：将 [gapStart + gapLen, index + gapLen) 复制到 gap 起始
            val moveCount = index - gapStart
            slots.copyInto(slots, gapStart, gapStart + gapLen, gapStart + gapLen + moveCount)
            gapStart = index
        }
    }
    
    fun insert(value: Any?) {
        if (gapLen == 0) grow()
        slots[gapStart++] = value
        gapLen--
    }
}
```

### 3.3 Positional Memoization

Compose 使用**位置记忆化**来匹配 Composable 调用与 Slot Table 中的数据：

```kotlin
@Composable
fun Parent() {
    Child("A")  // 位置 0
    Child("B")  // 位置 1
    Child("C")  // 位置 2
}
```

编译器为每个 Composable 调用生成唯一的 **Group Key**：

```kotlin
// 编译后的伪代码
fun Parent($composer: Composer) {
    $composer.startGroup(123)  // Parent 的 key
    
    $composer.startGroup(456)  // Child("A") 的 key
    Child("A", $composer)
    $composer.endGroup()
    
    $composer.startGroup(789)  // Child("B") 的 key
    Child("B", $composer)
    $composer.endGroup()
    
    // ...
    $composer.endGroup()
}
```

---

## IV. Composer：组合过程的协调者

### 4.1 Composer 的角色

`Composer` 是执行 Composable 函数时的上下文对象，负责：

- 管理 Slot Table 的读写
- 跟踪当前组合位置
- 处理 `remember`、`key` 等操作
- 判断是否需要跳过重组

```kotlin
interface Composer {
    // 组管理
    fun startGroup(key: Int)
    fun endGroup()
    
    // 数据管理
    fun rememberedValue(): Any?
    fun updateRememberedValue(value: Any?)
    
    // 跳过检测
    val skipping: Boolean
    fun skipToGroupEnd()
    
    // 变更记录
    fun recordInsert(anchor: Anchor)
    fun recordRemove(anchor: Anchor, count: Int)
}
```

### 4.2 组合执行流程

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

执行流程：

```
1. Composer.startGroup(Counter_key)
   │
2. ├─ Composer.rememberedValue() → 获取 mutableStateOf
   │  └─ 如果不存在，创建并存储
   │
3. ├─ Composer.startGroup(Button_key)
   │  ├─ 检查参数是否变化
   │  └─ 如果没变化且 skipping=true，跳过子树
   │
4. │  ├─ Composer.startGroup(Text_key)
   │  │  └─ 更新 Text 内容
   │  └─ Composer.endGroup()
   │
5. └─ Composer.endGroup()
   │
6. Composer.endGroup()
```

### 4.3 $changed 参数机制

编译器为每个 Composable 添加 `$changed` 参数，用于跳过优化：

```kotlin
// 原始代码
@Composable
fun Greeting(name: String) {
    Text("Hello, $name")
}

// 编译后（简化）
fun Greeting(
    name: String,
    $composer: Composer,
    $changed: Int  // 位标记：参数是否变化
) {
    $composer.startGroup(Greeting_key)
    
    // 检查是否可以跳过
    if ($changed and 0b0001 == 0 && $composer.skipping) {
        $composer.skipToGroupEnd()
    } else {
        Text("Hello, $name", $composer, ...)
    }
    
    $composer.endGroup()
}
```

**$changed 位编码**：

```
bit 0-1: 参数 1 的状态
  00 = Unknown (需要检查)
  01 = Same (未变化)
  10 = Different (已变化)
  11 = Static (编译期常量)

bit 2-3: 参数 2 的状态
...
```

---

## V. Recomposer：重组的调度器

### 5.1 Recomposer 职责

`Recomposer` 是整个重组系统的调度中心：

```kotlin
class Recomposer : CompositionContext() {
    // 待重组的 Composition 队列
    private val compositionsAwaitingRecomposition = mutableSetOf<Composition>()
    
    // 运行重组循环
    suspend fun runRecomposeAndApplyChanges() {
        while (true) {
            // 等待失效通知
            awaitWorkAvailable()
            
            // 执行所有待处理的重组
            performRecompose()
        }
    }
    
    // 标记需要重组
    fun invalidate(composition: Composition) {
        compositionsAwaitingRecomposition.add(composition)
        // 通知有工作要做
    }
}
```

### 5.2 重组调度流程

```
┌─────────────────────────────────────────────────────────┐
│ State 变化                                               │
│     │                                                   │
│     ▼                                                   │
│ Snapshot.notifyObjectsInitialized()                     │
│     │                                                   │
│     ▼                                                   │
│ SnapshotStateObserver 收到通知                           │
│     │                                                   │
│     ▼                                                   │
│ 找到读取该 State 的 Scope                                │
│     │                                                   │
│     ▼                                                   │
│ Recomposer.invalidate(composition)                      │
│     │                                                   │
│     ▼                                                   │
│ 在下一帧执行 recompose()                                 │
└─────────────────────────────────────────────────────────┘
```

### 5.3 与 Android 帧同步

```kotlin
// AndroidUiDispatcher 将重组与 Choreographer 同步
internal class AndroidUiDispatcher : CoroutineDispatcher() {
    private val choreographer = Choreographer.getInstance()
    
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        choreographer.postFrameCallback {
            block.run()
        }
    }
}

// 使用示例
val recomposer = Recomposer(AndroidUiDispatcher.Main)
scope.launch(AndroidUiDispatcher.Main) {
    recomposer.runRecomposeAndApplyChanges()
}
```

---

## VI. Applier：将变化应用到目标树

### 6.1 Applier 接口

`Applier` 是 Compose Runtime 与目标平台的桥梁：

```kotlin
interface Applier<N> {
    val current: N  // 当前节点
    
    fun onBeginChanges() {}
    fun onEndChanges() {}
    
    // 导航
    fun down(node: N)
    fun up()
    
    // 修改
    fun insertTopDown(index: Int, instance: N)
    fun insertBottomUp(index: Int, instance: N)
    fun remove(index: Int, count: Int)
    fun move(from: Int, to: Int, count: Int)
    
    fun clear()
}
```

### 6.2 UiApplier 实现

Compose UI 使用 `UiApplier` 管理 `LayoutNode` 树：

```kotlin
internal class UiApplier(root: LayoutNode) : AbstractApplier<LayoutNode>(root) {
    
    override fun insertTopDown(index: Int, instance: LayoutNode) {
        // 不使用 top-down 插入
    }
    
    override fun insertBottomUp(index: Int, instance: LayoutNode) {
        // Bottom-up: 先构建子树，再插入父节点
        // 这样可以避免多次 measure/layout
        current.insertAt(index, instance)
    }
    
    override fun remove(index: Int, count: Int) {
        current.removeAt(index, count)
    }
    
    override fun move(from: Int, to: Int, count: Int) {
        current.move(from, to, count)
    }
    
    override fun onClear() {
        root.removeAll()
    }
}
```

### 6.3 自定义 Applier

你可以为非 UI 目标创建自定义 Applier：

```kotlin
// 为 DOM 树创建 Applier（Compose for Web 示例）
class DomApplier(root: Element) : AbstractApplier<Element>(root) {
    
    override fun insertBottomUp(index: Int, instance: Element) {
        val children = current.children
        if (index < children.length) {
            current.insertBefore(instance, children[index])
        } else {
            current.appendChild(instance)
        }
    }
    
    override fun remove(index: Int, count: Int) {
        repeat(count) {
            current.children[index]?.remove()
        }
    }
    
    override fun move(from: Int, to: Int, count: Int) {
        // 实现元素移动
    }
}
```

---

## VII. 变更记录与批量应用

### 7.1 Changes 记录

重组过程中，Composer 不直接修改节点树，而是记录变更：

```kotlin
// 变更类型
sealed class Change {
    data class Insert(val index: Int, val node: Any) : Change()
    data class Remove(val index: Int, val count: Int) : Change()
    data class Move(val from: Int, val to: Int, val count: Int) : Change()
    data class SetValue(val node: Any, val value: Any) : Change()
}

// Composer 记录变更
class ComposerImpl : Composer {
    private val changes = mutableListOf<Change>()
    
    fun recordInsert(index: Int, node: Any) {
        changes.add(Change.Insert(index, node))
    }
    
    // 应用所有变更
    fun applyChanges() {
        applier.onBeginChanges()
        changes.forEach { change ->
            when (change) {
                is Change.Insert -> applier.insertBottomUp(change.index, change.node)
                is Change.Remove -> applier.remove(change.index, change.count)
                // ...
            }
        }
        applier.onEndChanges()
        changes.clear()
    }
}
```

### 7.2 批量应用的优势

```
传统方式（逐个修改）：        批量方式（Compose）：
─────────────────────       ─────────────────────
Insert A → Layout          记录 Insert A
Insert B → Layout          记录 Insert B  
Insert C → Layout          记录 Insert C
                           ────────────────
3 次 Layout                 批量应用 → 1 次 Layout
```

---

## VIII. 实战：追踪一次完整的重组

### 8.1 示例代码

```kotlin
@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }
    
    Column {
        Text("Current: $count")
        Button(onClick = { count++ }) {
            Text("Increment")
        }
    }
}
```

### 8.2 完整重组流程

```
用户点击 Button
       │
       ▼
onClick { count++ } 执行
       │
       ▼
mutableStateOf.value = 1
       │
       ▼
Snapshot.sendApplyNotifications()
       │
       ▼
SnapshotStateObserver 检测到 count 被修改
       │
       ▼
找到读取 count 的 Scope: Text("Current: $count")
       │
       ▼
Recomposer.invalidate() 标记该 Composition
       │
       ▼
Choreographer 下一帧回调
       │
       ▼
Recomposer.performRecompose()
       │
       ├─→ 进入 CounterScreen
       │   └─ skipping = false（有失效 scope）
       │
       ├─→ remember { mutableStateOf(0) }
       │   └─ 从 Slot Table 读取已存在的 state
       │
       ├─→ Column { ... }
       │   └─ 参数无变化，但内部有失效 scope，继续
       │
       ├─→ Text("Current: $count")
       │   ├─ 参数变化：$count = 1
       │   └─ 记录变更：SetValue(textNode, "Current: 1")
       │
       └─→ Button { ... }
           └─ 参数无变化，skipping = true，跳过
       │
       ▼
Composer.applyChanges()
       │
       ▼
UiApplier 更新 LayoutNode 树
       │
       ▼
触发 Layout & Draw
       │
       ▼
屏幕更新显示 "Current: 1"
```

---

## IX. 性能优化要点

### 9.1 减少 Slot Table 操作

```kotlin
// ❌ 不稳定的 key 导致大量 Slot Table 变动
items.forEachIndexed { index, item ->
    key(Random.nextInt()) {  // 错误！每次都是新 key
        ItemCard(item)
    }
}

// ✅ 使用稳定的 key
items.forEach { item ->
    key(item.id) {  // 正确：稳定的业务 ID
        ItemCard(item)
    }
}
```

### 9.2 控制重组范围

```kotlin
// ❌ 整个函数重组
@Composable
fun BadExample(items: List<Item>, selectedId: String) {
    Column {
        items.forEach { item ->
            ItemCard(
                item = item,
                isSelected = item.id == selectedId  // selectedId 变化导致全部重组
            )
        }
    }
}

// ✅ 使用 derivedStateOf 缩小重组范围
@Composable
fun GoodExample(items: List<Item>, selectedId: String) {
    Column {
        items.forEach { item ->
            val isSelected by remember(item.id) {
                derivedStateOf { item.id == selectedId }
            }
            ItemCard(item = item, isSelected = isSelected)
        }
    }
}
```

### 9.3 利用 skipping 机制

```kotlin
// 确保参数稳定，让 Composer 可以 skip
@Stable
data class UserUiState(
    val name: String,
    val avatarUrl: String
)

@Composable
fun UserCard(state: UserUiState) {  // @Stable 参数，便于跳过
    // ...
}
```

---

## X. 总结

### 核心组件回顾

| 组件 | 职责 |
|------|------|
| **Composition** | 组合的容器，持有 Slot Table 和 Applier |
| **Slot Table** | 存储组合数据，使用 Gap Buffer 优化 |
| **Composer** | 执行 Composable 函数，管理读写和跳过 |
| **Recomposer** | 调度重组，与帧同步 |
| **Applier** | 将变更应用到目标树（如 LayoutNode） |

### 关键机制

1. **Positional Memoization**：基于调用位置匹配数据
2. **$changed 参数**：编译器生成的跳过优化
3. **批量变更应用**：减少布局计算次数
4. **与 Choreographer 同步**：确保 16ms 帧预算

理解这些原理，你就掌握了 Compose 的"发动机"工作方式！

---

## 📚 参考资料

- [Compose Runtime Source Code](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/)
- [Under the hood of Jetpack Compose — Part 1](https://medium.com/androiddevelopers/under-the-hood-of-jetpack-compose-part-1-of-2-7466b12e3d29)
- [Compose Compiler Reports](https://developer.android.com/jetpack/compose/performance#compose-compiler)

