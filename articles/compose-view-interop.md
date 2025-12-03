# Compose 与 View 互操作完全指南：渐进式迁移的正确姿势

> **发布日期**: 2024-05-08  
> **阅读时间**: 约 26 分钟  
> **标签**: AndroidView, ComposeView, Interoperability, Migration

在现实项目中，很少有机会从零开始构建一个纯 Compose 应用。大多数情况下，我们需要在现有的 View 系统中逐步引入 Compose，或者在 Compose 中使用成熟的 View 组件。本文将深入探讨 Compose 与 View 的双向互操作。

## 一、为什么需要互操作？

Compose 虽然是 Android UI 的未来，但现实中有很多场景需要与 View 系统共存：

- **渐进式迁移**：大型项目无法一次性重写
- **复用现有组件**：MapView、WebView、ExoPlayer 等成熟组件
- **第三方 SDK**：很多 SDK 仍基于 View 系统
- **团队过渡期**：部分成员还在学习 Compose

> 💡 **Google 官方建议**  
> 采用渐进式迁移策略，从新功能开始使用 Compose，逐步替换旧代码，而非大规模重写。

## 二、在 Compose 中使用 View（AndroidView）

`AndroidView` 是在 Compose 中嵌入传统 View 的桥梁。

### 基本用法

```kotlin
@Composable
fun LegacyButtonInCompose() {
    AndroidView(
        factory = { context ->
            // 创建 View，只执行一次
            Button(context).apply {
                text = "我是传统 Button"
            }
        },
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp)
    )
}
```

### 更新 View 状态

使用 `update` 参数响应 Compose 状态变化：

```kotlin
@Composable
fun CounterWithLegacyView() {
    var count by remember { mutableStateOf(0) }

    Column {
        AndroidView(
            factory = { context ->
                TextView(context).apply {
                    textSize = 24f
                }
            },
            update = { textView ->
                // count 变化时更新 View
                textView.text = "Count: $count"
            }
        )

        Button(onClick = { count++ }) {
            Text("增加")
        }
    }
}
```

### 实战：嵌入 WebView

```kotlin
@Composable
fun WebViewContainer(url: String) {
    var webView: WebView? = null

    AndroidView(
        factory = { context ->
            WebView(context).apply {
                webViewClient = WebViewClient()
                settings.javaScriptEnabled = true
                webView = this
            }
        },
        update = { view ->
            view.loadUrl(url)
        },
        modifier = Modifier.fillMaxSize()
    )

    // 处理返回键
    BackHandler {
        webView?.let {
            if (it.canGoBack()) {
                it.goBack()
            }
        }
    }
}
```

### 实战：嵌入 MapView

```kotlin
@Composable
fun GoogleMapView(
    modifier: Modifier = Modifier,
    onMapReady: (GoogleMap) -> Unit
) {
    val context = LocalContext.current
    val mapView = remember { MapView(context) }
    val lifecycle = LocalLifecycleOwner.current.lifecycle

    // 管理 MapView 生命周期
    DisposableEffect(lifecycle) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_CREATE -> mapView.onCreate(Bundle())
                Lifecycle.Event.ON_START -> mapView.onStart()
                Lifecycle.Event.ON_RESUME -> mapView.onResume()
                Lifecycle.Event.ON_PAUSE -> mapView.onPause()
                Lifecycle.Event.ON_STOP -> mapView.onStop()
                Lifecycle.Event.ON_DESTROY -> mapView.onDestroy()
                else -> {}
            }
        }
        lifecycle.addObserver(observer)
        onDispose {
            lifecycle.removeObserver(observer)
        }
    }

    AndroidView(
        factory = {
            mapView.apply {
                getMapAsync { googleMap ->
                    onMapReady(googleMap)
                }
            }
        },
        modifier = modifier
    )
}
```

## 三、在 View 中使用 Compose（ComposeView）

`ComposeView` 让你可以在传统 View 布局中嵌入 Compose UI。

### 在 Activity 中使用

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 方式一：setContent 扩展函数（推荐）
        setContent {
            MyAppTheme {
                MainScreen()
            }
        }

        // 方式二：使用 ComposeView
        val composeView = ComposeView(this).apply {
            setContent {
                MyAppTheme {
                    MainScreen()
                }
            }
        }
        setContentView(composeView)
    }
}
```

### 在 Fragment 中使用

```kotlin
class ProfileFragment : Fragment() {

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return ComposeView(requireContext()).apply {
            // 设置 ViewCompositionStrategy
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
            setContent {
                MyAppTheme {
                    ProfileScreen()
                }
            }
        }
    }
}
```

### 在 XML 布局中混用

```xml
<!-- fragment_hybrid.xml -->
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="传统 TextView" />

    <androidx.compose.ui.platform.ComposeView
        android:id="@+id/compose_view"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <Button
        android:id="@+id/legacy_button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="传统 Button" />

</LinearLayout>
```

```kotlin
class HybridFragment : Fragment(R.layout.fragment_hybrid) {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        view.findViewById<ComposeView>(R.id.compose_view).apply {
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
            setContent {
                ComposeContent()
            }
        }
    }
}
```

## 四、ViewCompositionStrategy：生命周期策略

`ViewCompositionStrategy` 控制 Composition 何时被释放，选择正确的策略非常重要：

| 策略 | 释放时机 | 适用场景 |
|------|---------|---------|
| `DisposeOnDetachedFromWindow` | View 从窗口分离时 | 默认策略，适用于 Activity |
| `DisposeOnDetachedFromWindowOrReleasedFromPool` | 分离或从 RecyclerView 池释放时 | RecyclerView 中使用 |
| `DisposeOnViewTreeLifecycleDestroyed` | ViewTreeLifecycleOwner 销毁时 | **Fragment 推荐** |
| `DisposeOnLifecycleDestroyed` | 指定 Lifecycle 销毁时 | 自定义生命周期 |

### Fragment 中的陷阱

```kotlin
// ❌ 错误：默认策略在 Fragment 中可能导致问题
class BadFragment : Fragment() {
    override fun onCreateView(...): View {
        return ComposeView(requireContext()).apply {
            // 默认使用 DisposeOnDetachedFromWindow
            // Fragment view 重建时可能出问题
            setContent { ... }
        }
    }
}

// ✅ 正确：使用正确的策略
class GoodFragment : Fragment() {
    override fun onCreateView(...): View {
        return ComposeView(requireContext()).apply {
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
            setContent { ... }
        }
    }
}
```

## 五、状态同步与数据传递

### Compose 状态 → View

```kotlin
@Composable
fun ComposeToView(items: List<String>) {
    AndroidView(
        factory = { context ->
            RecyclerView(context).apply {
                adapter = SimpleAdapter()
                layoutManager = LinearLayoutManager(context)
            }
        },
        update = { recyclerView ->
            // Compose 状态变化时更新 RecyclerView
            (recyclerView.adapter as SimpleAdapter).submitList(items)
        }
    )
}
```

### View 事件 → Compose

```kotlin
@Composable
fun ViewToCompose() {
    var selectedDate by remember { mutableStateOf<Long?>(null) }

    Column {
        Text("选中日期: ${selectedDate?.let { formatDate(it) } ?: "未选择"}")

        AndroidView(
            factory = { context ->
                CalendarView(context).apply {
                    setOnDateChangeListener { _, year, month, day ->
                        // View 事件更新 Compose 状态
                        selectedDate = Calendar.getInstance().apply {
                            set(year, month, day)
                        }.timeInMillis
                    }
                }
            }
        )
    }
}
```

### 双向数据绑定模式

```kotlin
@Composable
fun TwoWayBinding() {
    var text by remember { mutableStateOf("") }

    Column {
        // Compose TextField
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            label = { Text("Compose 输入") }
        )

        Spacer(modifier = Modifier.height(16.dp))

        // Legacy EditText
        AndroidView(
            factory = { context ->
                EditText(context).apply {
                    hint = "View 输入"
                    addTextChangedListener(object : TextWatcher {
                        override fun afterTextChanged(s: Editable?) {
                            val newText = s?.toString() ?: ""
                            if (newText != text) {
                                text = newText
                            }
                        }
                        override fun beforeTextChanged(...) {}
                        override fun onTextChanged(...) {}
                    })
                }
            },
            update = { editText ->
                if (editText.text.toString() != text) {
                    editText.setText(text)
                }
            }
        )
    }
}
```

## 六、Fragment 与 Navigation 集成

### 在 Navigation Fragment 中使用 Compose

```kotlin
// 使用 navigation-compose 与 fragment 混合
class NavHostFragment : Fragment() {

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return ComposeView(requireContext()).apply {
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
            setContent {
                val navController = rememberNavController()
                
                NavHost(navController, startDestination = "home") {
                    composable("home") { HomeScreen(navController) }
                    composable("detail/{id}") { DetailScreen(it) }
                    
                    // 混合使用：导航到 Fragment
                    dialog("legacy_dialog") {
                        // 使用 DialogFragment
                    }
                }
            }
        }
    }
}
```

### Compose 导航到 Fragment

```kotlin
// 在 Compose Navigation 中嵌入 Fragment
@Composable
fun HybridNavigation() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = "compose_home") {
        composable("compose_home") {
            ComposeHomeScreen(
                onNavigateToLegacy = {
                    // 导航到包含 Fragment 的路由
                    navController.navigate("legacy_screen")
                }
            )
        }

        composable("legacy_screen") {
            // 使用 AndroidViewBinding 嵌入 Fragment
            AndroidViewBinding(FragmentContainerBinding::inflate) {
                val fragment = LegacyFragment()
                childFragmentManager.commit {
                    replace(R.id.fragment_container, fragment)
                }
            }
        }
    }
}
```

## 七、迁移策略

### 策略一：自底向上（推荐）

从最小的 UI 组件开始，逐步向上迁移：

```
Phase 1: 基础组件
Button, TextField, Card → Compose

Phase 2: 复合组件  
ListItem, FormField → Compose

Phase 3: 屏幕组件
整个 Screen → Compose

Phase 4: 导航
Navigation → Compose Navigation
```

```kotlin
// Phase 1: 替换单个组件
@Composable
fun ComposeButton(
    text: String,
    onClick: () -> Unit
) {
    Button(onClick = onClick) {
        Text(text)
    }
}

// 在 View 中使用
class LegacyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_legacy)

        findViewById<ComposeView>(R.id.compose_button).setContent {
            ComposeButton("点击我") {
                // 处理点击
            }
        }
    }
}
```

### 策略二：自顶向下

适合新功能或独立模块：

```kotlin
// 新功能直接用 Compose 实现整个屏幕
class NewFeatureActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            NewFeatureScreen()
        }
    }
}

@Composable
fun NewFeatureScreen() {
    // 完整的 Compose 屏幕
    // 需要时使用 AndroidView 嵌入必要的 View
    Column {
        ComposeHeader()
        
        // 嵌入必要的 View 组件
        AndroidView(
            factory = { LegacyChartView(it) }
        )
        
        ComposeContent()
    }
}
```

### 迁移清单

- ✅ 新功能优先使用 Compose
- ✅ 从叶子节点组件开始迁移
- ✅ 建立 Compose 设计系统（Theme、Components）
- ✅ 保持 View 和 Compose 版本组件的 API 一致
- ✅ 编写互操作测试
- ✅ 监控性能指标

## 八、性能注意事项

### 避免频繁创建 View

```kotlin
// ❌ 错误：factory 中创建复杂 View 但未缓存
@Composable
fun BadPerformance(items: List<Item>) {
    LazyColumn {
        items(items) { item ->
            AndroidView(
                factory = { context ->
                    // 每个 item 都创建新的复杂 View！
                    ComplexLegacyView(context)
                }
            )
        }
    }
}

// ✅ 正确：使用 key 和合理的复用策略
@Composable
fun GoodPerformance(items: List<Item>) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            // 使用 Compose 重写的组件
            ComposeItemView(item)
        }
    }
}
```

### AndroidView 的 onReset 和 onRelease

```kotlin
@Composable
fun OptimizedAndroidView() {
    AndroidView(
        factory = { context ->
            ExpensiveView(context)
        },
        onReset = { view ->
            // View 即将被复用时调用
            view.reset()
        },
        onRelease = { view ->
            // View 被释放时调用，清理资源
            view.cleanup()
        }
    )
}
```

### 减少桥接开销

```kotlin
// ❌ 频繁跨越 View-Compose 边界
@Composable
fun TooManyBridges() {
    Column {
        AndroidView { TextView(it) }  // 桥接
        Text("Compose")
        AndroidView { ImageView(it) } // 桥接
        Text("Compose")
        AndroidView { Button(it) }    // 桥接
    }
}

// ✅ 减少桥接，批量处理
@Composable
fun FewerBridges() {
    Column {
        // 一个 AndroidView 包含多个 View
        AndroidView(
            factory = { context ->
                LinearLayout(context).apply {
                    addView(TextView(context))
                    addView(ImageView(context))
                    addView(Button(context))
                }
            }
        )
        // 或者更好：直接用 Compose 重写
        Text("Compose Text")
        Image(...)
        Button(...)
    }
}
```

## 九、常见问题与解决方案

### 1. Theme 不一致

```kotlin
// 确保 Compose 使用与 View 一致的主题
@Composable
fun ThemedContent() {
    // 从 View 系统读取颜色
    val context = LocalContext.current
    val primaryColor = remember {
        context.getColor(R.color.primary)
    }

    MaterialTheme(
        colorScheme = MaterialTheme.colorScheme.copy(
            primary = Color(primaryColor)
        )
    ) {
        // 内容
    }
}
```

### 2. 软键盘处理

```kotlin
@Composable
fun KeyboardAwareAndroidView() {
    val imeInsets = WindowInsets.ime

    AndroidView(
        factory = { EditText(it) },
        modifier = Modifier
            .fillMaxWidth()
            .padding(bottom = with(LocalDensity.current) {
                imeInsets.getBottom(this).toDp()
            })
    )
}
```

### 3. 焦点管理

```kotlin
@Composable
fun FocusInterop() {
    val focusRequester = remember { FocusRequester() }

    Column {
        TextField(
            value = "",
            onValueChange = {},
            modifier = Modifier.focusRequester(focusRequester)
        )

        AndroidView(
            factory = { context ->
                EditText(context).apply {
                    setOnFocusChangeListener { _, hasFocus ->
                        if (!hasFocus) {
                            // View 失去焦点时，转移到 Compose
                            focusRequester.requestFocus()
                        }
                    }
                }
            }
        )
    }
}
```

## 总结

Compose 与 View 互操作的核心要点：

- **AndroidView**：在 Compose 中嵌入 View，用于复用现有组件
- **ComposeView**：在 View 中嵌入 Compose，用于渐进式迁移
- **ViewCompositionStrategy**：Fragment 中务必设置正确的策略
- **状态同步**：通过 `update` 参数和回调实现双向数据流
- **迁移策略**：推荐自底向上，从小组件开始
- **性能优化**：减少桥接开销，避免频繁创建 View

掌握这些技术，你就可以在任何项目中平滑地引入 Compose，无需担心与现有代码的兼容性问题。

---

*© 2024 Fidroid. [返回首页](../index.html)*


