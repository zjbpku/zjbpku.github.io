# Compose Snapshot 系统：状态观察与事务隔离的秘密

*2024-04-29 · 35 min · 底层原理深度解析*

**Tags:** Snapshot, State Observation, Transaction, Isolation

---

Compose 如何知道状态变化了？为什么 `mutableStateOf` 能触发重组，而普通变量不能？答案藏在 Compose 最核心也最神秘的子系统中——**Snapshot 系统**。本文将揭开它的面纱，深入探讨状态观察、变化追踪和事务隔离的实现原理。

> 📚 **官方参考**
> - [State and Jetpack Compose](https://developer.android.com/jetpack/compose/state)
> - [Snapshot 源码 - AndroidX](https://github.com/androidx/androidx/tree/androidx-main/compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/snapshots)

## 一、Snapshot 系统概述

Snapshot 系统是 Compose Runtime 的核心组件，它提供了：

- **状态观察**：追踪哪些状态被读取，哪些被修改
- **变化通知**：当状态变化时，通知相关的观察者
- **事务隔离**：类似数据库事务，提供一致性的状态视图
- **并发支持**：允许多线程安全地读写状态

```
Snapshot 系统架构

┌─────────────────────────────────────────────────────────────┐
│                     Compose Runtime                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ MutableState │    │ MutableState │    │ MutableState │  │
│  │   count=5    │    │   name="A"   │    │   list=[...]  │  │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘  │
│         │                   │                   │          │
│         └───────────────────┼───────────────────┘          │
│                             │                               │
│                             ▼                               │
│              ┌─────────────────────────────┐               │
│              │      Snapshot System        │               │
│              │  ┌─────────────────────┐    │               │
│              │  │ State Records       │    │               │
│              │  │ Read Observers      │    │               │
│              │  │ Write Observers     │    │               │
│              │  │ Snapshot Stack      │    │               │
│              │  └─────────────────────┘    │               │
│              └─────────────┬───────────────┘               │
│                            │                                │
│                            ▼                                │
│              ┌─────────────────────────────┐               │
│              │    Recomposition Trigger    │               │
│              └─────────────────────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 为什么需要 Snapshot？

考虑这个场景：

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}

// 问题：当 count++ 执行时
// 1. Compose 如何知道 count 变了？
// 2. Compose 如何知道哪些 Composable 需要重组？
// 3. 如果同时有多个状态变化，如何保证一致性？
```

Snapshot 系统正是为了解决这些问题而设计的。

## 二、StateObject 与 StateRecord

理解 Snapshot 系统的第一步是理解它的数据结构。

### StateObject

每个 Compose 状态（如 `MutableState`）都实现了 `StateObject` 接口：

```kotlin
// 简化的 StateObject 接口
interface StateObject {
    // 状态记录链表的头部
    val firstStateRecord: StateRecord
    
    // 添加新的状态记录
    fun prependStateRecord(value: StateRecord)
}

// MutableState 的简化实现
internal class SnapshotMutableStateImpl<T>(
    value: T
) : StateObject, MutableState<T> {
    
    override var firstStateRecord: StateRecord = 
        StateStateRecord(value)
    
    override var value: T
        get() {
            // 读取时通知 Snapshot 系统
            return readable(firstStateRecord).value
        }
        set(value) {
            // 写入时通知 Snapshot 系统
            writable(firstStateRecord).value = value
        }
}
```

### StateRecord：状态的版本链

每个 `StateObject` 维护一个 `StateRecord` 链表，记录不同时间点的状态值：

```
StateRecord 链表结构

StateObject (MutableState)
    │
    ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ StateRecord #3  │───▶│ StateRecord #2  │───▶│ StateRecord #1  │
│ snapshotId: 15  │    │ snapshotId: 10  │    │ snapshotId: 1   │
│ value: "C"      │    │ value: "B"      │    │ value: "A"      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
      ▲                                              ▲
      │                                              │
  最新版本                                        初始版本

当前 Snapshot (id=12) 看到的值是 "B"
当前 Snapshot (id=16) 看到的值是 "C"
```

这种设计允许不同的 Snapshot 看到不同版本的状态，实现了**多版本并发控制（MVCC）**。

> 💡 **MVCC 的优势**
> 
> 多版本并发控制让读操作不会阻塞写操作，写操作也不会阻塞读操作。这对于 UI 框架至关重要——我们不希望状态更新阻塞 UI 渲染。

## 三、Snapshot 的读写追踪

Snapshot 系统的核心能力是追踪状态的读取和写入。

### 读取追踪

当你读取一个 `MutableState` 时，Snapshot 系统会记录这次读取：

```kotlin
// 伪代码：读取追踪机制
internal fun <T> readable(record: StateRecord): T {
    // 1. 获取当前 Snapshot
    val snapshot = Snapshot.current
    
    // 2. 找到对当前 Snapshot 可见的记录
    val visibleRecord = record.findRecord(snapshot.id)
    
    // 3. 通知读取观察者（关键！）
    snapshot.readObserver?.invoke(record.stateObject)
    
    return visibleRecord.value
}
```

在 Composition 期间，Compose 设置了读取观察者来收集依赖：

```kotlin
// Composition 如何设置读取观察
fun compose(content: @Composable () -> Unit) {
    Snapshot.observe(
        readObserver = { state ->
            // 记录：当前 Composable 依赖这个状态
            currentScope.recordRead(state)
        }
    ) {
        content()
    }
}

// 这就是为什么 Compose 知道哪些 Composable 需要重组！
```

### 写入追踪

当你修改一个 `MutableState` 时：

```kotlin
// 伪代码：写入追踪机制
internal fun <T> writable(record: StateRecord): T {
    // 1. 获取当前 Snapshot
    val snapshot = Snapshot.current
    
    // 2. 创建新的 StateRecord（Copy-on-Write）
    val newRecord = record.create()
    newRecord.snapshotId = snapshot.id
    
    // 3. 通知写入观察者
    snapshot.writeObserver?.invoke(record.stateObject)
    
    return newRecord
}
```

全局写入观察者负责触发重组：

```kotlin
// Compose 如何响应状态变化
Snapshot.registerGlobalWriteObserver { changedStates ->
    // 当状态变化时，找到依赖这些状态的 Composable
    changedStates.forEach { state ->
        val affectedScopes = dependencyTracker.getScopesFor(state)
        affectedScopes.forEach { scope ->
            scope.invalidate()  // 标记需要重组
        }
    }
}
```

## 四、Snapshot 事务

Snapshot 系统支持类似数据库的事务操作。

### MutableSnapshot

你可以创建一个可变 Snapshot 来隔离状态变化：

```kotlin
val state = mutableStateOf(0)

// 在全局 Snapshot 中，state = 0

val snapshot = Snapshot.takeMutableSnapshot()
snapshot.enter {
    // 在这个 Snapshot 中修改
    state.value = 42
    
    // 在这个 Snapshot 中，state = 42
    println(state.value)  // 输出: 42
}

// 在全局 Snapshot 中，state 仍然 = 0！
println(state.value)  // 输出: 0

// 提交变化到全局
snapshot.apply()

// 现在全局也是 42 了
println(state.value)  // 输出: 42
```

### 事务隔离的应用

| 场景 | 使用方式 | 目的 |
|------|----------|------|
| Composition | ReadonlySnapshot | 确保组合期间状态一致 |
| 状态恢复 | MutableSnapshot | 批量恢复状态，原子提交 |
| 测试 | MutableSnapshot | 隔离测试状态，不影响其他测试 |
| 动画预览 | MutableSnapshot | 预览状态变化而不实际提交 |

### 冲突检测与解决

当多个 Snapshot 修改同一状态时，可能产生冲突：

```kotlin
val state = mutableStateOf(0)

val snapshot1 = Snapshot.takeMutableSnapshot()
val snapshot2 = Snapshot.takeMutableSnapshot()

snapshot1.enter { state.value = 1 }
snapshot2.enter { state.value = 2 }

snapshot1.apply()  // 成功，state = 1
snapshot2.apply()  // 冲突！抛出 SnapshotApplyConflictException

// 解决冲突：使用 applyTo 并处理冲突
val result = snapshot2.apply()
if (result is SnapshotApplyResult.Failure) {
    // 处理冲突...
}
```

> 📚 **深入阅读**
> 
> [Understanding Jetpack Compose - Leland Richardson](https://www.droidcon.com/2022/06/28/understanding-jetpack-compose-part-2-of-2/)

## 五、snapshotFlow：将 Snapshot 状态转为 Flow

`snapshotFlow` 是连接 Snapshot 系统和 Kotlin Flow 的桥梁：

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    var query by remember { mutableStateOf("") }
    
    // 将 Compose 状态转为 Flow
    LaunchedEffect(Unit) {
        snapshotFlow { query }
            .debounce(300)
            .distinctUntilChanged()
            .collect { searchQuery ->
                viewModel.search(searchQuery)
            }
    }
    
    TextField(
        value = query,
        onValueChange = { query = it }
    )
}
```

### snapshotFlow 的实现原理

```kotlin
// snapshotFlow 简化实现
fun <T> snapshotFlow(block: () -> T): Flow<T> = flow {
    var previousValue: T? = null
    
    while (true) {
        // 在 Snapshot 中执行 block，同时收集读取的状态
        val readStates = mutableSetOf<StateObject>()
        val value = Snapshot.observe(
            readObserver = { readStates.add(it) }
        ) {
            block()
        }
        
        // 如果值变化了，发射新值
        if (value != previousValue) {
            emit(value)
            previousValue = value
        }
        
        // 等待任一读取的状态发生变化
        suspendUntilAnyChange(readStates)
    }
}
```

## 六、derivedStateOf：派生状态的优化

`derivedStateOf` 利用 Snapshot 系统实现高效的派生状态计算：

```kotlin
@Composable
fun FilteredList(items: List<Item>, filter: String) {
    // ❌ 每次重组都重新过滤
    val filtered = items.filter { it.name.contains(filter) }
    
    // ✅ 只有当 items 或 filter 变化时才重新计算
    val filtered by remember(items, filter) {
        derivedStateOf {
            items.filter { it.name.contains(filter) }
        }
    }
}
```

### derivedStateOf 的工作原理

```
derivedStateOf 依赖追踪

┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  items  ─────────┐                                         │
│                  │                                          │
│                  ▼                                          │
│          ┌─────────────────┐                               │
│          │ derivedStateOf  │                               │
│  filter ─▶│                 │──▶ filteredItems            │
│          │ 追踪依赖        │                               │
│          │ 缓存结果        │                               │
│          └─────────────────┘                               │
│                                                              │
│  当 items 或 filter 变化时：                                 │
│  1. derivedStateOf 收到通知                                 │
│  2. 重新执行计算                                            │
│  3. 如果结果变化，通知下游观察者                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```kotlin
// derivedStateOf 简化实现
class DerivedSnapshotState<T>(
    private val calculation: () -> T
) : State<T> {
    
    private var cachedValue: T? = null
    private var dependencies = setOf<StateObject>()
    private var isValid = false
    
    override val value: T
        get() {
            if (!isValid || dependenciesChanged()) {
                // 重新计算，同时追踪新的依赖
                val newDeps = mutableSetOf<StateObject>()
                cachedValue = Snapshot.observe(
                    readObserver = { newDeps.add(it) }
                ) {
                    calculation()
                }
                dependencies = newDeps
                isValid = true
            }
            return cachedValue!!
        }
}
```

> ⚠️ **derivedStateOf 的常见误用**
> 
> 不要在 `derivedStateOf` 中引用 Composable 参数，因为参数变化不会触发重新计算。始终使用 `remember(key)` 来处理参数依赖。

## 七、Snapshot 与线程安全

Snapshot 系统天然支持多线程访问。

### 线程本地 Snapshot

每个线程可以有自己的 Snapshot 视图：

```kotlin
val state = mutableStateOf(0)

// 主线程
launch(Dispatchers.Main) {
    state.value = 1
    println("Main: ${state.value}")  // 1
}

// 后台线程
launch(Dispatchers.IO) {
    // 创建独立的 Snapshot
    Snapshot.takeSnapshot().enter {
        println("IO: ${state.value}")  // 可能是 0 或 1
    }
}
```

### withMutableSnapshot：线程安全的状态修改

```kotlin
// 在任意线程安全地修改状态
launch(Dispatchers.IO) {
    Snapshot.withMutableSnapshot {
        // 这里的修改是原子的
        state1.value = "new value"
        state2.value = 42
    }
    // 退出时自动 apply
}
```

## 八、实战：自定义 StateObject

你可以创建自定义的状态类型，参与 Snapshot 系统：

```kotlin
// 自定义计数器状态
class SnapshotCounter : StateObject {
    
    private class CounterRecord : StateRecord() {
        var count: Int = 0
        
        override fun create() = CounterRecord()
        
        override fun assign(value: StateRecord) {
            count = (value as CounterRecord).count
        }
    }
    
    override var firstStateRecord: StateRecord = CounterRecord()
    
    override fun prependStateRecord(value: StateRecord) {
        value.next = firstStateRecord
        firstStateRecord = value
    }
    
    val count: Int
        get() = readable(firstStateRecord).count
    
    fun increment() {
        writable(firstStateRecord).count++
    }
    
    fun decrement() {
        writable(firstStateRecord).count--
    }
    
    private fun readable(record: StateRecord): CounterRecord {
        return Snapshot.current.readable(record, this) as CounterRecord
    }
    
    private fun writable(record: StateRecord): CounterRecord {
        return Snapshot.current.writable(record, this) as CounterRecord
    }
}
```

## 九、调试 Snapshot 系统

### 使用 Snapshot 监听器

```kotlin
// 监听所有状态变化
val handle = Snapshot.registerGlobalWriteObserver { changedObjects ->
    changedObjects.forEach { obj ->
        println("State changed: $obj")
    }
}

// 不再需要时移除
handle.dispose()
```

### 使用 Composition Tracing

```kotlin
// 在 Debug 构建中启用追踪
if (BuildConfig.DEBUG) {
    Composer.enableTracing = true
}
```

## 十、最佳实践

### DO ✅

- 使用 `mutableStateOf` 而非普通变量来存储 UI 状态
- 使用 `derivedStateOf` 缓存昂贵的计算
- 使用 `snapshotFlow` 将 Compose 状态连接到 Flow
- 在后台线程修改状态时使用 `withMutableSnapshot`

### DON'T ❌

- 不要在 Composable 外部直接修改状态（除非在正确的 Snapshot 上下文中）
- 不要忽略 `snapshotFlow` 的收集——它会持续运行
- 不要在 `derivedStateOf` 中执行副作用
- 不要手动管理 Snapshot 生命周期（除非你知道自己在做什么）

## 总结

Snapshot 系统是 Compose 响应式的基石：

- **StateObject/StateRecord**：多版本状态存储
- **读写追踪**：自动收集依赖和触发更新
- **事务隔离**：确保状态一致性
- **snapshotFlow**：连接 Compose 和 Flow
- **derivedStateOf**：高效的派生状态

理解 Snapshot 系统不仅能帮助你写出更高效的 Compose 代码，还能让你在遇到状态相关问题时更容易定位和解决。

---

## 推荐阅读

- [State and Jetpack Compose - Android Developers](https://developer.android.com/jetpack/compose/state)
- [Compose Internals - Leland Richardson (Video)](https://www.youtube.com/watch?v=Q9MtlmmN4Q0)
- [Jetpack Compose Internals - Jorge Castillo](https://jorgecastillo.dev/book/)


