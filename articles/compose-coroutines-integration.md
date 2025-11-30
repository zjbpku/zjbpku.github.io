# Compose 与 Coroutines 深度集成：异步 UI 的正确姿势

*2024-05-05 · 28 min · 实战深度解析*

**Tags:** Coroutines, LaunchedEffect, rememberCoroutineScope, Flow

---

Compose 和 Kotlin Coroutines 是天作之合。但要正确使用它们的组合，你需要理解：什么时候用 `LaunchedEffect`？什么时候用 `rememberCoroutineScope`？Flow 如何安全地收集？本文将深入探讨 Compose 与 Coroutines 的集成机制，帮助你写出正确且高效的异步 UI 代码。

> 📚 **官方参考**
> - [Side Effects in Compose](https://developer.android.com/jetpack/compose/side-effects)
> - [Kotlin Coroutines on Android](https://developer.android.com/kotlin/coroutines)

## 一、Compose 的协程作用域

Compose 提供了多种方式来启动协程，每种都有其特定的生命周期和使用场景。

### 核心 API 对比

| API | 生命周期 | 使用场景 |
|-----|----------|----------|
| LaunchedEffect | 跟随 Composable | 自动启动的副作用 |
| rememberCoroutineScope | 跟随 Composable | 事件触发的协程 |
| DisposableEffect | 跟随 Composable | 需要清理的资源 |
| produceState | 跟随 Composable | 将异步数据转为 State |
| ViewModel.viewModelScope | 跟随 ViewModel | 业务逻辑协程 |

## 二、LaunchedEffect 深度解析

`LaunchedEffect` 是最常用的协程 API，它在 Composable 进入组合时启动协程，离开时自动取消。

### 基本用法

```kotlin
@Composable
fun UserProfile(userId: String) {
    var user by remember { mutableStateOf<User?>(null) }
    
    // 当 userId 变化时，取消旧协程，启动新协程
    LaunchedEffect(userId) {
        user = fetchUser(userId)  // 挂起函数
    }
    
    user?.let { UserCard(it) }
}
```

### Key 参数的作用

`LaunchedEffect` 的 key 参数决定了协程何时重启：

```kotlin
// 1. 固定 key：只在首次组合时执行一次
LaunchedEffect(Unit) {
    // 只执行一次
    analytics.trackScreenView("UserProfile")
}

// 2. 动态 key：key 变化时重启
LaunchedEffect(userId) {
    // userId 变化时重新执行
    user = fetchUser(userId)
}

// 3. 多个 key：任一变化时重启
LaunchedEffect(userId, refreshTrigger) {
    // userId 或 refreshTrigger 变化时重新执行
    user = fetchUser(userId)
}

// 4. 无 key（不推荐）：每次重组都重启
// LaunchedEffect { } // 编译错误：必须提供 key
```

```
LaunchedEffect 生命周期

┌─────────────────────────────────────────────────────────────┐
│                     Composable 生命周期                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  进入组合 ────────────────────────────────────▶ 离开组合   │
│      │                                              │        │
│      ▼                                              ▼        │
│  启动协程                                      取消协程      │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     Key 变化时                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  key = "A"          key = "B"                               │
│      │                  │                                   │
│      ▼                  ▼                                   │
│  启动协程 A  ───▶  取消协程 A  ───▶  启动协程 B           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### LaunchedEffect 的内部实现

```kotlin
// 简化的 LaunchedEffect 实现
@Composable
fun LaunchedEffect(
    key1: Any?,
    block: suspend CoroutineScope.() -> Unit
) {
    val currentBlock by rememberUpdatedState(block)
    
    DisposableEffect(key1) {
        val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
        scope.launch {
            currentBlock()
        }
        
        onDispose {
            scope.cancel()  // 离开组合时取消
        }
    }
}
```

> 💡 **为什么使用 rememberUpdatedState？**
>
> `rememberUpdatedState` 确保协程中始终使用最新的 lambda，而不会因为捕获了旧值导致问题。这在长时间运行的协程中特别重要。

## 三、rememberCoroutineScope 详解

`rememberCoroutineScope` 提供一个与 Composable 生命周期绑定的 `CoroutineScope`，用于在事件回调中启动协程。

```kotlin
@Composable
fun RefreshableList() {
    val scope = rememberCoroutineScope()
    var items by remember { mutableStateOf(emptyList<Item>()) }
    var isRefreshing by remember { mutableStateOf(false) }
    
    PullRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = {
            // 在事件回调中启动协程
            scope.launch {
                isRefreshing = true
                items = fetchItems()
                isRefreshing = false
            }
        }
    ) {
        LazyColumn {
            items(items) { ItemRow(it) }
        }
    }
}
```

### LaunchedEffect vs rememberCoroutineScope

| 特性 | LaunchedEffect | rememberCoroutineScope |
|------|----------------|------------------------|
| 启动时机 | 自动（进入组合时） | 手动（事件触发） |
| 取消时机 | 自动（离开组合或 key 变化） | 自动（离开组合） |
| 典型场景 | 初始化、数据加载 | 点击、滑动等事件 |
| 重启控制 | 通过 key 参数 | 手动控制 |

```kotlin
// ❌ 错误：在 LaunchedEffect 中响应事件
@Composable
fun BadExample(onClick: () -> Unit) {
    var clicked by remember { mutableStateOf(false) }
    
    LaunchedEffect(clicked) {
        if (clicked) {
            doSomething()  // 不好：用状态触发副作用
        }
    }
    
    Button(onClick = { clicked = true }) { ... }
}

// ✅ 正确：使用 rememberCoroutineScope
@Composable
fun GoodExample() {
    val scope = rememberCoroutineScope()
    
    Button(
        onClick = {
            scope.launch { doSomething() }
        }
    ) { ... }
}
```

## 四、安全收集 Flow

在 Compose 中收集 Flow 需要特别注意生命周期安全。

### collectAsState

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    // 最简单的方式：自动处理生命周期
    val uiState by viewModel.uiState.collectAsState()
    
    when (uiState) {
        is UiState.Loading -> LoadingIndicator()
        is UiState.Success -> UserContent(uiState.data)
        is UiState.Error -> ErrorMessage(uiState.message)
    }
}
```

### collectAsStateWithLifecycle（推荐）

```kotlin
// 需要添加依赖：androidx.lifecycle:lifecycle-runtime-compose

@Composable
fun UserScreen(viewModel: UserViewModel) {
    // 更安全：在 Activity/Fragment 不可见时停止收集
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    // ...
}
```

```
collectAsState vs collectAsStateWithLifecycle

┌─────────────────────────────────────────────────────────────┐
│                   collectAsState                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Composable 可见                    Composable 不可见       │
│        │                                    │               │
│        ▼                                    ▼               │
│    收集 Flow                          停止收集             │
│                                                              │
│  问题：Activity 在后台时仍然收集                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│               collectAsStateWithLifecycle                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Activity STARTED                    Activity STOPPED       │
│        │                                    │               │
│        ▼                                    ▼               │
│    收集 Flow                          停止收集             │
│                                                              │
│  优势：遵循 Activity 生命周期，更省资源                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

> 📚 **深入阅读**
>
> [Consuming Flows Safely in Jetpack Compose](https://medium.com/androiddevelopers/consuming-flows-safely-in-jetpack-compose-cde014d0d5a3)

### 在 LaunchedEffect 中收集 Flow

```kotlin
@Composable
fun EventHandler(viewModel: MyViewModel) {
    val context = LocalContext.current
    
    // 收集一次性事件（如 Toast、导航）
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is Event.ShowToast -> {
                    Toast.makeText(context, event.message, Toast.LENGTH_SHORT).show()
                }
                is Event.Navigate -> {
                    // 导航...
                }
            }
        }
    }
}
```

## 五、produceState：将异步数据转为 State

`produceState` 是一个便捷函数，用于将异步数据源转换为 Compose State。

```kotlin
@Composable
fun UserProfile(userId: String) {
    // 将挂起函数结果转为 State
    val user by produceState<User?>(initialValue = null, userId) {
        value = fetchUser(userId)
    }
    
    user?.let { UserCard(it) } ?: LoadingIndicator()
}

// 更复杂的例子：带加载状态
@Composable
fun UserProfile(userId: String) {
    val userState by produceState<Result<User>>(
        initialValue = Result.Loading,
        userId
    ) {
        value = try {
            Result.Success(fetchUser(userId))
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
    
    when (val state = userState) {
        is Result.Loading -> LoadingIndicator()
        is Result.Success -> UserCard(state.data)
        is Result.Error -> ErrorMessage(state.exception.message)
    }
}
```

### produceState 与 Flow

```kotlin
// 将 Flow 转为 State（等同于 collectAsState）
@Composable
fun <T> Flow<T>.collectAsStateCustom(initial: T): State<T> {
    return produceState(initial, this) {
        collect { value = it }
    }
}
```

## 六、snapshotFlow：将 Compose State 转为 Flow

`snapshotFlow` 是 `produceState` 的反向操作，将 Compose State 转换为 Flow。

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    var query by remember { mutableStateOf("") }
    
    // 将 Compose State 转为 Flow，应用 Flow 操作符
    LaunchedEffect(Unit) {
        snapshotFlow { query }
            .debounce(300)           // 防抖
            .filter { it.isNotEmpty() }  // 过滤空查询
            .distinctUntilChanged()   // 去重
            .collect { searchQuery ->
                viewModel.search(searchQuery)
            }
    }
    
    TextField(
        value = query,
        onValueChange = { query = it },
        placeholder = { Text("搜索...") }
    )
}
```

### snapshotFlow 的原理

```kotlin
// snapshotFlow 简化实现
fun <T> snapshotFlow(block: () -> T): Flow<T> = flow {
    var previousValue: T? = null
    
    while (true) {
        // 在 Snapshot 中执行 block，追踪读取的状态
        val readStates = mutableSetOf<StateObject>()
        val value = Snapshot.observe(
            readObserver = { readStates.add(it) }
        ) {
            block()
        }
        
        // 如果值变化，发射
        if (value != previousValue) {
            emit(value)
            previousValue = value
        }
        
        // 挂起直到任一状态变化
        suspendUntilChanged(readStates)
    }
}
```

## 七、处理协程取消

正确处理协程取消是避免内存泄漏和 bug 的关键。

### 使用 NonCancellable

```kotlin
@Composable
fun DataEditor(viewModel: EditorViewModel) {
    val scope = rememberCoroutineScope()
    
    Button(
        onClick = {
            scope.launch {
                try {
                    viewModel.saveData()
                } finally {
                    // 即使协程被取消，也要执行清理
                    withContext(NonCancellable) {
                        viewModel.cleanup()
                    }
                }
            }
        }
    ) {
        Text("保存")
    }
}
```

### 检查取消状态

```kotlin
LaunchedEffect(items) {
    items.forEach { item ->
        // 在长循环中检查取消
        ensureActive()  // 如果已取消，抛出 CancellationException
        
        processItem(item)
    }
}
```

## 八、协程异常处理

### 使用 CoroutineExceptionHandler

```kotlin
@Composable
fun SafeCoroutineExample() {
    val errorHandler = remember {
        CoroutineExceptionHandler { _, throwable ->
            Log.e("Coroutine", "Error: ${throwable.message}")
        }
    }
    
    val scope = rememberCoroutineScope()
    
    Button(
        onClick = {
            scope.launch(errorHandler) {
                // 异常会被 errorHandler 捕获
                riskyOperation()
            }
        }
    ) {
        Text("执行")
    }
}
```

### 使用 try-catch

```kotlin
LaunchedEffect(userId) {
    try {
        user = fetchUser(userId)
    } catch (e: CancellationException) {
        // 重新抛出取消异常，不要吞掉它！
        throw e
    } catch (e: Exception) {
        error = e.message
    }
}
```

> ⚠️ **不要吞掉 CancellationException**
>
> `CancellationException` 是协程取消的信号。如果你捕获了它但不重新抛出，协程将无法正确取消，可能导致资源泄漏。

## 九、实战模式

### 模式 1：加载-显示-错误

```kotlin
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}

@Composable
fun <T> AsyncContent(
    state: UiState<T>,
    onRetry: () -> Unit,
    content: @Composable (T) -> Unit
) {
    when (state) {
        is UiState.Loading -> {
            Box(Modifier.fillMaxSize(), Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is UiState.Success -> content(state.data)
        is UiState.Error -> {
            Column(
                Modifier.fillMaxSize(),
                Arrangement.Center,
                Alignment.CenterHorizontally
            ) {
                Text(state.message)
                Button(onClick = onRetry) {
                    Text("重试")
                }
            }
        }
    }
}
```

### 模式 2：防抖输入

```kotlin
@Composable
fun DebouncedTextField(
    onValueChange: (String) -> Unit,
    debounceMs: Long = 300
) {
    var text by remember { mutableStateOf("") }
    
    LaunchedEffect(text) {
        delay(debounceMs)
        onValueChange(text)
    }
    
    TextField(
        value = text,
        onValueChange = { text = it }
    )
}
```

### 模式 3：超时处理

```kotlin
LaunchedEffect(userId) {
    try {
        withTimeout(5000) {
            user = fetchUser(userId)
        }
    } catch (e: TimeoutCancellationException) {
        error = "请求超时，请重试"
    }
}
```

## 十、最佳实践总结

### DO ✅

- 使用 `LaunchedEffect` 处理自动启动的副作用
- 使用 `rememberCoroutineScope` 处理事件触发的协程
- 使用 `collectAsStateWithLifecycle` 收集 Flow
- 正确处理 `CancellationException`
- 为 `LaunchedEffect` 提供正确的 key

### DON'T ❌

- 不要在 Composable 中直接调用 `GlobalScope.launch`
- 不要吞掉 `CancellationException`
- 不要用状态变化来触发事件（用 `rememberCoroutineScope`）
- 不要忘记处理协程异常

## 总结

Compose 与 Coroutines 的集成是现代 Android 开发的基石：

- **LaunchedEffect**：自动管理生命周期的副作用
- **rememberCoroutineScope**：事件驱动的协程
- **collectAsStateWithLifecycle**：安全的 Flow 收集
- **produceState**：异步数据转 State
- **snapshotFlow**：State 转 Flow

掌握这些 API，你就能写出安全、高效的异步 UI 代码。

---

## 推荐阅读

- [Side Effects in Compose](https://developer.android.com/jetpack/compose/side-effects)
- [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [Kotlin Coroutines 101 (Video)](https://www.youtube.com/watch?v=BOHK_w09pVA)

