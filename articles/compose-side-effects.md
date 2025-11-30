# Compose Side Effects 完全指南：副作用处理的正确姿势

> **发布日期**: 2024-04-15  
> **阅读时间**: 约 20 分钟  
> **标签**: LaunchedEffect, DisposableEffect, SideEffect, rememberCoroutineScope

在 Compose 的声明式世界中，副作用（Side Effects）是指那些需要在 Composable 函数外部执行的操作，如网络请求、数据库访问、日志记录等。正确处理副作用是写好 Compose 代码的关键。

## 一、什么是副作用？

Composable 函数应该是**纯函数**：给定相同的输入，总是产生相同的 UI 输出，且没有可观察的副作用。但现实中我们需要：

- 在组件显示时加载数据
- 监听生命周期事件
- 与外部系统交互（传感器、定位等）
- 记录分析事件

这些操作需要通过 Compose 提供的 Effect API 来安全执行。

## 二、Effect API 速查表

| API | 执行时机 | 典型用途 |
|-----|---------|---------|
| `LaunchedEffect` | 进入组合时，key 变化时重启 | 协程中执行一次性操作 |
| `DisposableEffect` | 进入组合时，离开时清理 | 需要清理的资源（监听器） |
| `SideEffect` | 每次成功重组后 | 同步非 Compose 状态 |
| `rememberCoroutineScope` | 获取与组合绑定的协程作用域 | 响应事件时启动协程 |
| `produceState` | 将非 Compose 状态转为 State | 订阅外部数据源 |
| `derivedStateOf` | 派生状态变化时 | 减少不必要的重组 |
| `snapshotFlow` | 将 Compose State 转为 Flow | 在 Flow 中观察状态变化 |

## 三、LaunchedEffect：协程副作用

`LaunchedEffect` 在进入组合时启动一个协程，当 key 变化时取消并重启：

```kotlin
@Composable
fun UserProfile(userId: String) {
    var user by remember { mutableStateOf<User?>(null) }

    // userId 变化时重新加载
    LaunchedEffect(userId) {
        user = repository.getUser(userId)
    }

    user?.let { ProfileContent(it) }
        ?: LoadingIndicator()
}
```

### 多个 key

```kotlin
// 任一 key 变化都会重启
LaunchedEffect(userId, refreshTrigger) {
    user = repository.getUser(userId)
}
```

### 只执行一次

```kotlin
// 使用 Unit 或 true 作为 key，只在首次进入时执行
LaunchedEffect(Unit) {
    analytics.logScreenView("profile")
}
```

> ⚠️ **常见错误**  
> 不要在 LaunchedEffect 中使用会频繁变化的 key（如每次重组都变化的对象），这会导致协程不断重启。

## 四、DisposableEffect：需要清理的副作用

当副作用需要在离开组合时清理（如取消监听器），使用 `DisposableEffect`：

```kotlin
@Composable
fun LocationTracker(onLocationUpdate: (Location) -> Unit) {
    val context = LocalContext.current
    val locationManager = remember {
        context.getSystemService(LocationManager::class.java)
    }

    DisposableEffect(locationManager) {
        val listener = LocationListener { location ->
            onLocationUpdate(location)
        }

        locationManager.requestLocationUpdates(
            LocationManager.GPS_PROVIDER,
            1000L,
            10f,
            listener
        )

        // 清理：离开组合时调用
        onDispose {
            locationManager.removeUpdates(listener)
        }
    }
}
```

### 生命周期观察

```kotlin
@Composable
fun LifecycleObserver(
    onStart: () -> Unit,
    onStop: () -> Unit
) {
    val lifecycleOwner = LocalLifecycleOwner.current

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_START -> onStart()
                Lifecycle.Event.ON_STOP -> onStop()
                else -> {}
            }
        }

        lifecycleOwner.lifecycle.addObserver(observer)

        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

## 五、SideEffect：每次重组后执行

`SideEffect` 在每次**成功**重组后执行，用于将 Compose 状态同步到非 Compose 代码：

```kotlin
@Composable
fun AnalyticsScreen(screenName: String) {
    // 每次重组后同步到分析系统
    SideEffect {
        analytics.setCurrentScreen(screenName)
    }

    // UI content...
}
```

> 💡 **SideEffect vs LaunchedEffect**  
> `SideEffect` 是同步的、每次重组都执行；`LaunchedEffect` 是异步的、只在 key 变化时执行。

## 六、rememberCoroutineScope：事件驱动的协程

当需要在事件回调（如点击）中启动协程时，使用 `rememberCoroutineScope`：

```kotlin
@Composable
fun SubmitButton(onSubmit: suspend () -> Unit) {
    val scope = rememberCoroutineScope()
    var isLoading by remember { mutableStateOf(false) }

    Button(
        onClick = {
            scope.launch {
                isLoading = true
                try {
                    onSubmit()
                } finally {
                    isLoading = false
                }
            }
        },
        enabled = !isLoading
    ) {
        if (isLoading) {
            CircularProgressIndicator(modifier = Modifier.size(16.dp))
        } else {
            Text("提交")
        }
    }
}
```

## 七、produceState：将外部数据转为 State

`produceState` 将非 Compose 的异步数据源转换为 Compose State：

```kotlin
@Composable
fun NetworkImage(url: String): State<ImageBitmap?> {
    return produceState<ImageBitmap?>(initialValue = null, url) {
        value = loadNetworkImage(url)
    }
}

// 使用
@Composable
fun Avatar(imageUrl: String) {
    val image by NetworkImage(imageUrl)

    image?.let {
        Image(bitmap = it, contentDescription = null)
    } ?: Placeholder()
}
```

### 订阅 Flow

```kotlin
@Composable
fun connectivityState(): State<Boolean> {
    val context = LocalContext.current

    return produceState(initialValue = true) {
        context.observeConnectivity().collect { isConnected ->
            value = isConnected
        }
    }
}
```

## 八、snapshotFlow：State 转 Flow

`snapshotFlow` 将 Compose State 转换为 Flow，便于在协程中处理：

```kotlin
@Composable
fun ScrollTracker(listState: LazyListState) {
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                analytics.logScroll(index)
            }
    }
}
```

## 九、rememberUpdatedState：捕获最新值

在长时间运行的副作用中，确保使用最新的回调或值：

```kotlin
@Composable
fun SplashScreen(onTimeout: () -> Unit) {
    // 捕获最新的 onTimeout，即使它在 LaunchedEffect 运行期间变化
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(Unit) {
        delay(3000)
        currentOnTimeout()  // 使用最新的回调
    }

    // Splash UI...
}
```

## 十、选择正确的 Effect

### 决策流程图

1. **需要在协程中执行？**
   - 是，且由状态变化触发 → `LaunchedEffect`
   - 是，且由事件触发 → `rememberCoroutineScope`

2. **需要清理资源？** → `DisposableEffect`

3. **需要同步到非 Compose 代码？** → `SideEffect`

4. **需要将外部数据转为 State？** → `produceState`

5. **需要将 State 转为 Flow？** → `snapshotFlow`

## 十一、常见陷阱

### 1. 在 LaunchedEffect 中直接使用变化的 lambda

```kotlin
// ❌ 错误：onComplete 变化不会触发重启
LaunchedEffect(Unit) {
    doSomething()
    onComplete()  // 可能是旧的 lambda
}

// ✅ 正确：使用 rememberUpdatedState
val currentOnComplete by rememberUpdatedState(onComplete)
LaunchedEffect(Unit) {
    doSomething()
    currentOnComplete()
}
```

### 2. 忘记 onDispose

```kotlin
// ❌ 错误：资源泄漏
DisposableEffect(key) {
    listener.register()
    // 忘记 onDispose！
}

// ✅ 正确
DisposableEffect(key) {
    listener.register()
    onDispose { listener.unregister() }
}
```

### 3. 在 Composition 中直接执行副作用

```kotlin
// ❌ 错误：每次重组都会执行
@Composable
fun BadExample() {
    analytics.logEvent("screen_view")  // 直接调用！
}

// ✅ 正确：使用 Effect
@Composable
fun GoodExample() {
    LaunchedEffect(Unit) {
        analytics.logEvent("screen_view")
    }
}
```

## 总结

正确使用 Effect API 是写好 Compose 代码的关键：

- **LaunchedEffect**：状态驱动的协程副作用
- **DisposableEffect**：需要清理的副作用
- **SideEffect**：同步 Compose 状态到外部
- **rememberCoroutineScope**：事件驱动的协程
- **produceState**：外部数据 → State
- **snapshotFlow**：State → Flow
- **rememberUpdatedState**：在副作用中捕获最新值

---

*© 2024 Fidroid. [返回首页](../index.html)*


