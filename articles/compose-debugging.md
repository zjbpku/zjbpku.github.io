# Compose 调试技巧：Layout Inspector 与性能分析

> **发布日期**: 2024-05-30  
> **阅读时间**: 约 22 分钟  
> **标签**: Debug, Layout Inspector, Recomposition, Performance

调试是开发过程中不可或缺的一环。Compose 提供了丰富的调试工具，从 Layout Inspector 到重组追踪，帮助你快速定位和解决问题。本文将深入讲解 Compose 的调试技巧和最佳实践。

## 一、Layout Inspector

Layout Inspector 是 Android Studio 内置的布局检查工具，支持 Compose 的实时检查。

### 启动 Layout Inspector

1. 运行应用到设备或模拟器
2. 菜单：View → Tool Windows → Layout Inspector
3. 选择正在运行的进程

### 检查 Compose 层级

```kotlin
// Layout Inspector 会显示 Composable 的完整树结构
@Composable
fun MyScreen() {
    Column {           // 在 Inspector 中显示为 Column
        Header()       // 显示为 Header
        Content()      // 显示为 Content
        Footer()       // 显示为 Footer
    }
}
```

### 查看 Composable 属性

在 Layout Inspector 中选择一个 Composable，可以看到：

- **修饰符链**：应用的所有 Modifier
- **参数值**：传入的参数当前值
- **尺寸和位置**：实际测量和布局结果
- **重组次数**：该 Composable 的重组计数

## 二、重组追踪

### 开启重组高亮

在 Android Studio 中启用重组可视化：

1. Run → Edit Configurations
2. 添加 VM 参数：`-Pandroidx.compose.compiler.config.traceMarkersEnabled=true`

### 使用 Modifier.debugInspectorInfo

```kotlin
fun Modifier.myCustomModifier() = this.then(
    object : ModifierNodeElement<MyModifierNode>() {
        override fun InspectorInfo.inspectableProperties() {
            name = "myCustomModifier"
            properties["customProperty"] = "value"
        }
        // ...
    }
)
```

### 日志追踪重组

```kotlin
@Composable
fun TrackedComposable(data: String) {
    // 开发时使用，发布前移除
    SideEffect {
        Log.d("Recomposition", "TrackedComposable recomposed with: $data")
    }
    
    Text(text = data)
}
```

### 使用 remember 检测重组

```kotlin
@Composable
fun RecompositionCounter(name: String) {
    val recomposeCount = remember { mutableIntStateOf(0) }
    
    SideEffect {
        recomposeCount.intValue++
    }
    
    if (BuildConfig.DEBUG) {
        Text(
            text = "$name: ${recomposeCount.intValue}",
            color = Color.Red,
            fontSize = 10.sp
        )
    }
}
```

## 三、Composition Tracing

### 系统跟踪

```kotlin
// 在代码中添加跟踪点
@Composable
fun MyScreen() {
    trace("MyScreen") {
        // 组件内容
        Column {
            trace("Header") { Header() }
            trace("Content") { Content() }
        }
    }
}
```

### 使用 Perfetto

1. 录制系统跟踪：`adb shell perfetto -o /data/misc/perfetto-traces/trace -t 10s`
2. 拉取文件：`adb pull /data/misc/perfetto-traces/trace`
3. 在 [ui.perfetto.dev](https://ui.perfetto.dev) 中打开分析

### Compose 编译器报告

生成编译器报告来分析稳定性：

```kotlin
// build.gradle.kts
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_reports")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
}
```

运行后会生成：
- `*-classes.txt`：类的稳定性信息
- `*-composables.txt`：Composable 函数信息
- `*-composables.csv`：CSV 格式的详细数据

## 四、性能分析

### CPU Profiler

1. View → Tool Windows → Profiler
2. 选择 CPU
3. 点击 Record 开始录制
4. 执行要分析的操作
5. 点击 Stop 停止录制

### 检测慢重组

```kotlin
@Composable
fun SlowComponent() {
    val startTime = remember { System.nanoTime() }
    
    // 组件内容...
    
    DisposableEffect(Unit) {
        val endTime = System.nanoTime()
        val duration = (endTime - startTime) / 1_000_000.0
        if (duration > 16) { // 超过一帧 (16ms)
            Log.w("SlowCompose", "SlowComponent took ${duration}ms")
        }
        onDispose { }
    }
}
```

### 使用 derivedStateOf 优化

```kotlin
@Composable
fun OptimizedList(items: List<Item>) {
    // 派生状态只在结果变化时触发重组
    val sortedItems by remember(items) {
        derivedStateOf { items.sortedBy { it.name } }
    }
    
    LazyColumn {
        items(sortedItems) { item ->
            ItemRow(item)
        }
    }
}
```

## 五、内存分析

### Memory Profiler

1. View → Tool Windows → Profiler
2. 选择 Memory
3. 观察内存使用情况
4. 点击 Dump Java Heap 分析对象

### 检测内存泄漏

```kotlin
// 错误示例：在 Composable 中持有 Context
@Composable
fun LeakyComponent(context: Context) {
    // ❌ 不要这样做
    val handler = remember { Handler(context.mainLooper) }
}

// 正确示例：使用 LocalContext
@Composable
fun SafeComponent() {
    val context = LocalContext.current
    // ✅ LocalContext 会正确处理生命周期
}
```

### 使用 remember 正确管理资源

```kotlin
@Composable
fun ResourceComponent() {
    val resource = remember {
        ExpensiveResource().also {
            Log.d("Resource", "Created")
        }
    }
    
    DisposableEffect(Unit) {
        onDispose {
            resource.release()
            Log.d("Resource", "Released")
        }
    }
}
```

## 六、常见问题调试

### 1. 无限重组

```kotlin
// ❌ 错误：在组合中修改状态导致无限重组
@Composable
fun InfiniteRecomposition() {
    var count by remember { mutableIntStateOf(0) }
    count++ // 每次重组都增加，触发新的重组
    Text("Count: $count")
}

// ✅ 正确：在事件处理中修改状态
@Composable
fun CorrectRecomposition() {
    var count by remember { mutableIntStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

### 2. 状态丢失

```kotlin
// ❌ 错误：key 变化导致状态重置
@Composable
fun StateLoss(items: List<Item>) {
    items.forEachIndexed { index, item ->
        key(index) { // index 变化时状态会丢失
            EditableItem(item)
        }
    }
}

// ✅ 正确：使用稳定的 key
@Composable
fun StatePreserved(items: List<Item>) {
    items.forEach { item ->
        key(item.id) { // 使用唯一 ID
            EditableItem(item)
        }
    }
}
```

### 3. 闪烁问题

```kotlin
// ❌ 可能闪烁：每次重组创建新对象
@Composable
fun FlickeringImage(url: String) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .build(), // 每次创建新对象
        contentDescription = null
    )
}

// ✅ 使用 remember 缓存
@Composable
fun StableImage(url: String) {
    val model = remember(url) {
        ImageRequest.Builder(LocalContext.current)
            .data(url)
            .build()
    }
    AsyncImage(model = model, contentDescription = null)
}
```

## 七、调试工具函数

### 自定义调试 Modifier

```kotlin
fun Modifier.debugBorder(color: Color = Color.Red) = 
    if (BuildConfig.DEBUG) {
        this.border(1.dp, color)
    } else {
        this
    }

fun Modifier.debugBackground() = 
    if (BuildConfig.DEBUG) {
        this.background(Color.Red.copy(alpha = 0.1f))
    } else {
        this
    }

// 使用
@Composable
fun MyComponent() {
    Box(
        modifier = Modifier
            .fillMaxSize()
            .debugBorder()
    ) {
        // 内容
    }
}
```

### 重组计数器组件

```kotlin
@Composable
fun RecompositionTracker(
    name: String,
    modifier: Modifier = Modifier
) {
    if (!BuildConfig.DEBUG) return
    
    val count = remember { mutableIntStateOf(0) }
    SideEffect { count.intValue++ }
    
    Box(modifier = modifier) {
        Text(
            text = "$name: ${count.intValue}",
            color = Color.White,
            fontSize = 8.sp,
            modifier = Modifier
                .background(Color.Red.copy(alpha = 0.7f))
                .padding(2.dp)
        )
    }
}
```

## 八、日志最佳实践

```kotlin
object ComposeLogger {
    private const val TAG = "ComposeDebug"
    
    fun logRecomposition(name: String, params: Map<String, Any?> = emptyMap()) {
        if (BuildConfig.DEBUG) {
            val paramsStr = params.entries.joinToString { "${it.key}=${it.value}" }
            Log.d(TAG, "Recomposition: $name($paramsStr)")
        }
    }
    
    fun logEffect(name: String, type: String) {
        if (BuildConfig.DEBUG) {
            Log.d(TAG, "Effect: $name - $type")
        }
    }
}

// 使用
@Composable
fun TrackedScreen(userId: String) {
    ComposeLogger.logRecomposition("TrackedScreen", mapOf("userId" to userId))
    
    LaunchedEffect(userId) {
        ComposeLogger.logEffect("TrackedScreen", "LaunchedEffect started")
    }
    
    // ...
}
```

## 九、最佳实践清单

- ✅ 使用 Layout Inspector 检查 Compose 层级
- ✅ 开启重组高亮定位不必要的重组
- ✅ 使用编译器报告分析稳定性问题
- ✅ 使用 CPU Profiler 分析性能瓶颈
- ✅ 使用 Memory Profiler 检测内存泄漏
- ✅ 添加调试用的边框和日志（仅 DEBUG）
- ✅ 使用 remember 避免重复创建对象
- ✅ 使用稳定的 key 防止状态丢失

## 总结

Compose 调试的核心工具：

- **Layout Inspector**：检查 Compose 层级和属性
- **重组追踪**：定位不必要的重组
- **Composition Tracing**：系统级性能分析
- **CPU Profiler**：分析性能瓶颈
- **Memory Profiler**：检测内存问题
- **编译器报告**：分析稳定性

掌握这些调试技巧，可以快速定位和解决 Compose 应用中的问题。

---

*© 2024 Fidroid. [返回首页](../index.html)*

