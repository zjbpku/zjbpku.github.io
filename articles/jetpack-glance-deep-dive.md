# Jetpack Glance 深度解析：用 Compose 构建桌面小部件

> **发布日期**: 2024-12-23  
> **阅读时间**: 约 55-65 分钟  
> **标签**: Glance, Compose, Widgets, RemoteViews, AppWidget, 状态管理

Jetpack Glance 是 Google 推出的用于构建桌面小部件和穿戴设备界面的新框架，它允许开发者使用类似 Compose 的声明式 API 来创建这些界面。本文将深入探讨 Glance 的核心概念、架构设计、组件体系、状态管理以及实战技巧。

## 📚 官方参考

- [Jetpack Glance Documentation - Android Developers](https://developer.android.com/jetpack/androidx/releases/glance)
- [Glance App Widget Samples - GitHub](https://github.com/android/user-interface-samples/tree/main/AppWidget)
- [Build app widgets with Glance - Android Developers](https://developer.android.com/develop/ui/compose/glance)

## 目录

- [I. Glance 核心架构](#i-glance-核心架构)
- [II. Glance 组件详解](#ii-glance-组件详解)
- [III. 状态管理深入](#iii-状态管理深入)
- [IV. Action 系统](#iv-action-系统)
- [V. 尺寸适配与响应式设计](#v-尺寸适配与响应式设计)
- [VI. 主题与样式](#vi-主题与样式)
- [VII. 高级特性](#vii-高级特性)
- [VIII. 性能优化与调试](#viii-性能优化与调试)

## I. Glance 核心架构

### 1.1 Glance 的工作原理

Glance 并非直接渲染 UI，而是作为一个"翻译器"，将 Compose 风格的代码转换为 RemoteViews。

```text
Glance Composable
       ↓
  Glance Composition
       ↓
  Translation Layer
       ↓
   RemoteViews
       ↓
  System Launcher
```

**关键理解：**
- Glance 代码在 **Widget Provider 进程**中运行
- 最终的 UI 渲染在 **Launcher 进程**中
- RemoteViews 作为跨进程通信的载体

### 1.2 核心类关系

```kotlin
// 核心抽象类
abstract class GlanceAppWidget : GlanceAppWidgetReceiver() {
    // 定义状态存储方式
    open val stateDefinition: GlanceStateDefinition<*>?
    
    // 定义 UI 内容
    abstract suspend fun provideGlance(context: Context, id: GlanceId)
    
    // 尺寸模式
    open val sizeMode: SizeMode = SizeMode.Single
}

// 接收器基类
abstract class GlanceAppWidgetReceiver : BroadcastReceiver() {
    abstract val glanceAppWidget: GlanceAppWidget
}
```

### 1.3 基本项目结构

```kotlin
// 1. 定义 Widget 类
class MyWidget : GlanceAppWidget() {
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        provideContent {
            MyWidgetContent()
        }
    }
}

// 2. 定义 Receiver
class MyWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget: GlanceAppWidget = MyWidget()
}

// 3. 定义 UI
@Composable
fun MyWidgetContent() {
    Text("Hello Glance!")
}
```

## II. Glance 组件详解

### 2.1 布局组件 (Layout Composables)

Glance 提供了与标准 Compose 类似但独立的布局组件。

#### Column - 垂直布局

```kotlin
@Composable
fun VerticalLayoutExample() {
    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .padding(16.dp),
        verticalAlignment = Alignment.Vertical.Top,
        horizontalAlignment = Alignment.Horizontal.Start
    ) {
        Text("Item 1")
        Text("Item 2")
        Text("Item 3")
    }
}
```

#### Row - 水平布局

```kotlin
@Composable
fun HorizontalLayoutExample() {
    Row(
        modifier = GlanceModifier
            .fillMaxWidth()
            .padding(16.dp),
        verticalAlignment = Alignment.Vertical.CenterVertically,
        horizontalAlignment = Alignment.Horizontal.SpaceBetween
    ) {
        Image(
            provider = ImageProvider(R.drawable.icon),
            contentDescription = "Icon"
        )
        Text("Title", style = TextStyle(fontSize = 18.sp))
        Button(text = "Action", onClick = { })
    }
}
```

#### Box - 层叠布局

```kotlin
@Composable
fun StackedLayoutExample() {
    Box(
        modifier = GlanceModifier.size(200.dp),
        contentAlignment = Alignment.Center
    ) {
        // 背景层
        Image(
            provider = ImageProvider(R.drawable.background),
            contentDescription = null,
            modifier = GlanceModifier.fillMaxSize()
        )
        
        // 前景层
        Column(
            modifier = GlanceModifier.padding(16.dp),
            horizontalAlignment = Alignment.Horizontal.CenterHorizontally
        ) {
            Text("Overlay Text", style = TextStyle(color = ColorProvider(Color.White)))
            Button(text = "Action", onClick = { })
        }
    }
}
```

### 2.2 基础组件 (Basic Composables)

#### Text - 文本组件

```kotlin
@Composable
fun TextExamples() {
    Column(verticalAlignment = Alignment.Vertical.Top) {
        // 基础文本
        Text(text = "Simple Text")
        
        // 样式化文本
        Text(
            text = "Styled Text",
            style = TextStyle(
                color = ColorProvider(Color.Blue),
                fontSize = 20.sp,
                fontWeight = FontWeight.Bold,
                fontStyle = FontStyle.Italic,
                textAlign = TextAlign.Center,
                textDecoration = TextDecoration.Underline
            )
        )
        
        // 限制行数
        Text(
            text = "Long text that will be truncated...",
            maxLines = 1,
            style = TextStyle(
                textOverflow = TextOverflow.Ellipsis
            )
        )
    }
}
```

#### Image - 图片组件

```kotlin
@Composable
fun ImageExamples() {
    Column {
        // 1. 资源图片
        Image(
            provider = ImageProvider(R.drawable.my_image),
            contentDescription = "My Image",
            modifier = GlanceModifier.size(64.dp)
        )
        
        // 2. 图标（tinted）
        Image(
            provider = ImageProvider(R.drawable.ic_star),
            contentDescription = "Star",
            colorFilter = ColorFilter.tint(ColorProvider(Color.Yellow)),
            modifier = GlanceModifier.size(24.dp)
        )
        
        // 3. 位图（Bitmap）
        val bitmap: Bitmap = loadBitmap()
        Image(
            provider = ImageProvider(bitmap),
            contentDescription = "Dynamic Image"
        )
    }
}
```

#### Button - 按钮组件

```kotlin
@Composable
fun ButtonExamples() {
    Column(verticalAlignment = Alignment.Vertical.Top) {
        // 标准按钮
        Button(
            text = "Click Me",
            onClick = actionRunCallback<MyCallback>()
        )
        
        // 自定义样式按钮
        Button(
            text = "Styled Button",
            onClick = actionStartActivity<MainActivity>(),
            style = TextStyle(
                color = ColorProvider(Color.White),
                fontSize = 16.sp
            ),
            colors = ButtonDefaults.buttonColors(
                backgroundColor = ColorProvider(Color.Blue),
                contentColor = ColorProvider(Color.White)
            ),
            modifier = GlanceModifier
                .fillMaxWidth()
                .height(48.dp)
        )
        
        // 图标按钮
        Button(
            text = "Refresh",
            icon = ImageProvider(R.drawable.ic_refresh),
            onClick = actionRunCallback<RefreshCallback>()
        )
    }
}
```

#### CircularProgressIndicator - 加载指示器

```kotlin
@Composable
fun LoadingExample() {
    Box(
        modifier = GlanceModifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        CircularProgressIndicator(
            color = ColorProvider(Color.Blue),
            modifier = GlanceModifier.size(48.dp)
        )
    }
}
```

#### LinearProgressIndicator - 进度条

```kotlin
@Composable
fun ProgressExample(progress: Float) {
    Column {
        Text("Download Progress: ${(progress * 100).toInt()}%")
        LinearProgressIndicator(
            progress = progress,
            color = ColorProvider(Color.Green),
            backgroundColor = ColorProvider(Color.LightGray),
            modifier = GlanceModifier
                .fillMaxWidth()
                .height(8.dp)
        )
    }
}
```

### 2.3 复合组件 (Compound Composables)

#### Switch - 开关组件

```kotlin
@Composable
fun SwitchExample() {
    var isEnabled by remember { mutableStateOf(false) }
    
    Row(
        verticalAlignment = Alignment.Vertical.CenterVertically,
        modifier = GlanceModifier.fillMaxWidth()
    ) {
        Text("Enable Notifications", modifier = GlanceModifier.defaultWeight())
        Switch(
            checked = isEnabled,
            onCheckedChange = actionRunCallback<ToggleSwitchCallback>()
        )
    }
}
```

#### CheckBox - 复选框

```kotlin
@Composable
fun CheckBoxExample() {
    Row(verticalAlignment = Alignment.Vertical.CenterVertically) {
        CheckBox(
            checked = true,
            onCheckedChange = actionRunCallback<ToggleCheckBoxCallback>()
        )
        Text("I agree to terms")
    }
}
```

### 2.4 懒加载列表 (LazyColumn)

Glance 1.0+ 支持懒加载列表。

```kotlin
@Composable
fun LazyListExample(items: List<String>) {
    LazyColumn(
        modifier = GlanceModifier.fillMaxSize()
    ) {
        // 单个 item
        item {
            Text("Header", style = TextStyle(fontWeight = FontWeight.Bold))
        }
        
        // 多个 items
        items(items.size) { index ->
            ListItemRow(items[index])
        }
        
        // 带 key 的 items
        items(
            count = items.size,
            key = { index -> items[index] }
        ) { index ->
            ListItemRow(items[index])
        }
    }
}

@Composable
fun ListItemRow(text: String) {
    Row(
        modifier = GlanceModifier
            .fillMaxWidth()
            .padding(vertical = 8.dp, horizontal = 16.dp)
            .clickable(actionRunCallback<ItemClickCallback>()),
        verticalAlignment = Alignment.Vertical.CenterVertically
    ) {
        Image(
            provider = ImageProvider(R.drawable.ic_item),
            contentDescription = null,
            modifier = GlanceModifier.size(24.dp)
        )
        Spacer(modifier = GlanceModifier.width(8.dp))
        Text(text)
    }
}
```

## III. 状态管理深入

### 3.1 使用 DataStore 管理状态

Glance 推荐使用 Preferences DataStore 来持久化小部件状态。

```kotlin
// 1. 定义 Preferences Keys
object WidgetPreferences {
    val TEMPERATURE_KEY = intPreferencesKey("temperature")
    val CITY_KEY = stringPreferencesKey("city")
    val LAST_UPDATE_KEY = longPreferencesKey("last_update")
}

// 2. 定义 StateDefinition
object WeatherStateDefinition : GlanceStateDefinition<Preferences> {
    
    private const val DATA_STORE_FILE = "weather_widget_prefs"
    
    override suspend fun getDataStore(
        context: Context,
        fileKey: String
    ): DataStore<Preferences> {
        return context.dataStore
    }
    
    override fun getLocation(context: Context, fileKey: String): File {
        return context.dataStoreFile(DATA_STORE_FILE)
    }
    
    private val Context.dataStore by preferencesDataStore(name = DATA_STORE_FILE)
}

// 3. 在 GlanceAppWidget 中使用
class WeatherWidget : GlanceAppWidget() {
    
    override val stateDefinition = WeatherStateDefinition
    
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        provideContent {
            // 读取状态
            val prefs = currentState<Preferences>()
            val temperature = prefs[WidgetPreferences.TEMPERATURE_KEY] ?: 0
            val city = prefs[WidgetPreferences.CITY_KEY] ?: "Unknown"
            
            WeatherWidgetContent(
                temperature = temperature,
                city = city
            )
        }
    }
}
```

### 3.2 更新状态

```kotlin
// 在 ActionCallback 中更新状态
class RefreshWeatherCallback : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        // 1. 获取新数据
        val weatherData = fetchWeatherFromApi()
        
        // 2. 更新 DataStore
        updateAppWidgetState(context, glanceId) { prefs ->
            prefs[WidgetPreferences.TEMPERATURE_KEY] = weatherData.temperature
            prefs[WidgetPreferences.CITY_KEY] = weatherData.city
            prefs[WidgetPreferences.LAST_UPDATE_KEY] = System.currentTimeMillis()
        }
        
        // 3. 触发 Widget 更新
        WeatherWidget().update(context, glanceId)
    }
}

// 在 Activity/Service 中更新状态
class WeatherUpdateWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        val weatherData = fetchWeatherFromApi()
        
        // 更新所有 Widget 实例
        GlanceAppWidgetManager(applicationContext)
            .getGlanceIds(WeatherWidget::class.java)
            .forEach { glanceId ->
                updateAppWidgetState(applicationContext, glanceId) { prefs ->
                    prefs[WidgetPreferences.TEMPERATURE_KEY] = weatherData.temperature
                    prefs[WidgetPreferences.CITY_KEY] = weatherData.city
                }
            }
        
        // 刷新所有实例
        WeatherWidget().updateAll(applicationContext)
        
        return Result.success()
    }
}
```

### 3.3 使用自定义状态类

```kotlin
// 1. 定义状态数据类
@Serializable
data class WeatherState(
    val city: String = "Unknown",
    val temperature: Int = 0,
    val condition: WeatherCondition = WeatherCondition.UNKNOWN,
    val lastUpdate: Long = 0
)

enum class WeatherCondition {
    SUNNY, CLOUDY, RAINY, SNOWY, UNKNOWN
}

// 2. 定义自定义 StateDefinition
object CustomWeatherStateDefinition : GlanceStateDefinition<WeatherState> {
    
    private const val DATA_STORE_FILE = "custom_weather_state"
    private val Context.dataStore by dataStore(DATA_STORE_FILE, WeatherStateSerializer)
    
    override suspend fun getDataStore(
        context: Context,
        fileKey: String
    ): DataStore<WeatherState> {
        return context.dataStore
    }
    
    override fun getLocation(context: Context, fileKey: String): File {
        return context.dataStoreFile(DATA_STORE_FILE)
    }
}

// 3. 定义 Serializer
object WeatherStateSerializer : Serializer<WeatherState> {
    override val defaultValue: WeatherState = WeatherState()
    
    override suspend fun readFrom(input: InputStream): WeatherState {
        return Json.decodeFromString(input.readBytes().decodeToString())
    }
    
    override suspend fun writeTo(t: WeatherState, output: OutputStream) {
        output.write(Json.encodeToString(t).encodeToByteArray())
    }
}

// 4. 在 Widget 中使用
class WeatherWidget : GlanceAppWidget() {
    override val stateDefinition = CustomWeatherStateDefinition
    
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        provideContent {
            val state = currentState<WeatherState>()
            WeatherWidgetContent(state)
        }
    }
}
```

## IV. Action 系统

### 4.1 Action 类型详解

Glance 提供了多种 Action 类型来处理用户交互。

#### actionStartActivity - 启动 Activity

```kotlin
@Composable
fun ActivityLaunchExample() {
    Column {
        // 1. 启动不带参数的 Activity
        Button(
            text = "Open App",
            onClick = actionStartActivity<MainActivity>()
        )
        
        // 2. 启动带参数的 Activity
        Button(
            text = "View Details",
            onClick = actionStartActivity(
                activity = DetailActivity::class.java,
                parameters = actionParametersOf(
                    ActionParameters.Key<String>("item_id") to "123",
                    ActionParameters.Key<Int>("page") to 1
                )
            )
        )
        
        // 3. 使用 Intent 启动
        Button(
            text = "Custom Intent",
            onClick = actionStartActivity(
                Intent(Intent.ACTION_VIEW, Uri.parse("https://example.com"))
            )
        )
    }
}
```

#### actionRunCallback - 执行回调

```kotlin
// 1. 定义 Callback
class UpdateDataCallback : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        // 获取参数
        val itemId = parameters[ActionParameters.Key<String>("item_id")]
        
        // 执行业务逻辑
        val result = performUpdate(itemId)
        
        // 更新状态
        updateAppWidgetState(context, glanceId) { prefs ->
            prefs[resultKey] = result
        }
        
        // 刷新 Widget
        MyWidget().update(context, glanceId)
    }
}

// 2. 使用 Callback
@Composable
fun CallbackExample() {
    Button(
        text = "Update",
        onClick = actionRunCallback<UpdateDataCallback>(
            parameters = actionParametersOf(
                ActionParameters.Key<String>("item_id") to "456"
            )
        )
    )
}
```

#### actionSendBroadcast - 发送广播

```kotlin
@Composable
fun BroadcastExample() {
    Button(
        text = "Send Broadcast",
        onClick = actionSendBroadcast(
            Intent("com.example.ACTION_CUSTOM").apply {
                putExtra("key", "value")
            }
        )
    )
}

// 接收广播
class MyBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val value = intent.getStringExtra("key")
        // 处理广播
    }
}
```

#### actionStartService - 启动服务

```kotlin
@Composable
fun ServiceExample() {
    Button(
        text = "Start Service",
        onClick = actionStartService<MyForegroundService>(
            isForegroundService = true
        )
    )
}
```

### 4.2 复合 Action（组合多个操作）

```kotlin
@Composable
fun CompoundActionExample() {
    // 方式1：链式调用（不支持，需要自定义）
    // 方式2：使用自定义 Callback
    Button(
        text = "Complex Action",
        onClick = actionRunCallback<ComplexActionCallback>()
    )
}

class ComplexActionCallback : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        // 步骤1：更新数据
        updateData()
        
        // 步骤2：发送通知
        sendNotification(context)
        
        // 步骤3：启动后台同步
        startSync(context)
        
        // 步骤4：更新 Widget
        updateAppWidgetState(context, glanceId) { prefs ->
            prefs[lastActionKey] = System.currentTimeMillis()
        }
        MyWidget().update(context, glanceId)
    }
}
```

### 4.3 带参数的 Action

```kotlin
@Composable
fun ParameterizedActionExample(items: List<Item>) {
    LazyColumn {
        items(items.size) { index ->
            val item = items[index]
            
            Row(
                modifier = GlanceModifier
                    .fillMaxWidth()
                    .clickable(
                        actionRunCallback<ItemClickCallback>(
                            parameters = actionParametersOf(
                                ActionParameters.Key<String>("item_id") to item.id,
                                ActionParameters.Key<Int>("position") to index
                            )
                        )
                    )
            ) {
                Text(item.title)
            }
        }
    }
}

class ItemClickCallback : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        val itemId = parameters[ActionParameters.Key<String>("item_id")]
        val position = parameters[ActionParameters.Key<Int>("position")]
        
        // 使用参数执行操作
        Log.d("ItemClick", "Clicked item $itemId at position $position")
    }
}
```

## V. 尺寸适配与响应式设计

### 5.1 SizeMode 配置

Glance 提供了三种尺寸模式。

```kotlin
class MyWidget : GlanceAppWidget() {
    
    // 模式1：Single - 单一尺寸（默认）
    override val sizeMode: SizeMode = SizeMode.Single
    
    // 模式2：Exact - 精确尺寸
    override val sizeMode: SizeMode = SizeMode.Exact
    
    // 模式3：Responsive - 响应式（多套布局）
    override val sizeMode: SizeMode = SizeMode.Responsive(
        setOf(
            DpSize(100.dp, 100.dp),  // 小
            DpSize(200.dp, 100.dp),  // 中
            DpSize(300.dp, 200.dp)   // 大
        )
    )
}
```

### 5.2 根据尺寸切换布局

```kotlin
@Composable
fun ResponsiveWidget() {
    val size = LocalSize.current
    
    when {
        size.width < 150.dp -> SmallWidget()
        size.width < 250.dp -> MediumWidget()
        else -> LargeWidget()
    }
}

@Composable
fun SmallWidget() {
    Column(
        modifier = GlanceModifier.fillMaxSize().padding(8.dp),
        horizontalAlignment = Alignment.Horizontal.CenterHorizontally
    ) {
        Image(
            provider = ImageProvider(R.drawable.icon),
            contentDescription = null,
            modifier = GlanceModifier.size(32.dp)
        )
        Text("25°C", style = TextStyle(fontSize = 18.sp))
    }
}

@Composable
fun MediumWidget() {
    Row(
        modifier = GlanceModifier.fillMaxSize().padding(12.dp),
        verticalAlignment = Alignment.Vertical.CenterVertically
    ) {
        Image(
            provider = ImageProvider(R.drawable.icon),
            contentDescription = null,
            modifier = GlanceModifier.size(48.dp)
        )
        Spacer(modifier = GlanceModifier.width(8.dp))
        Column {
            Text("Beijing", style = TextStyle(fontSize = 14.sp))
            Text("25°C", style = TextStyle(fontSize = 24.sp, fontWeight = FontWeight.Bold))
        }
    }
}

@Composable
fun LargeWidget() {
    Column(
        modifier = GlanceModifier.fillMaxSize().padding(16.dp)
    ) {
        Row(
            modifier = GlanceModifier.fillMaxWidth(),
            verticalAlignment = Alignment.Vertical.CenterVertically
        ) {
            Image(
                provider = ImageProvider(R.drawable.icon),
                contentDescription = null,
                modifier = GlanceModifier.size(64.dp)
            )
            Spacer(modifier = GlanceModifier.width(16.dp))
            Column {
                Text("Beijing", style = TextStyle(fontSize = 18.sp))
                Text("25°C", style = TextStyle(fontSize = 32.sp, fontWeight = FontWeight.Bold))
                Text("Sunny", style = TextStyle(fontSize = 14.sp, color = ColorProvider(Color.Gray)))
            }
        }
        Spacer(modifier = GlanceModifier.height(16.dp))
        // 更多详细信息...
    }
}
```

### 5.3 使用 LocalSize 实现细粒度控制

```kotlin
@Composable
fun AdaptiveTextSize() {
    val size = LocalSize.current
    
    val fontSize = when {
        size.width < 120.dp -> 14.sp
        size.width < 200.dp -> 16.sp
        else -> 20.sp
    }
    
    Text(
        text = "Adaptive Text",
        style = TextStyle(fontSize = fontSize)
    )
}
```

## VI. 主题与样式

### 6.1 动态颜色（Android 12+）

```kotlin
@Composable
fun DynamicColorWidget() {
    // 使用 ColorProvider 支持动态颜色
    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .background(
                day = Color.White,
                night = Color.Black
            )
            .padding(16.dp)
    ) {
        Text(
            text = "Dynamic Colors",
            style = TextStyle(
                color = ColorProvider(
                    day = Color.Black,
                    night = Color.White
                )
            )
        )
        
        Button(
            text = "Action",
            onClick = { },
            colors = ButtonDefaults.buttonColors(
                backgroundColor = ColorProvider(
                    day = Color(0xFF6200EE),
                    night = Color(0xFFBB86FC)
                )
            )
        )
    }
}
```

### 6.2 自定义主题系统

```kotlin
// 定义主题对象
object WidgetTheme {
    val colors = WidgetColors(
        primary = Color(0xFF6200EE),
        onPrimary = Color.White,
        surface = Color.White,
        onSurface = Color.Black
    )
    
    val typography = WidgetTypography(
        title = TextStyle(fontSize = 20.sp, fontWeight = FontWeight.Bold),
        body = TextStyle(fontSize = 14.sp),
        caption = TextStyle(fontSize = 12.sp, color = ColorProvider(Color.Gray))
    )
}

data class WidgetColors(
    val primary: Color,
    val onPrimary: Color,
    val surface: Color,
    val onSurface: Color
)

data class WidgetTypography(
    val title: TextStyle,
    val body: TextStyle,
    val caption: TextStyle
)

// 使用主题
@Composable
fun ThemedWidget() {
    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .background(WidgetTheme.colors.surface)
            .padding(16.dp)
    ) {
        Text("Title", style = WidgetTheme.typography.title)
        Text("Body text", style = WidgetTheme.typography.body)
        Text("Caption", style = WidgetTheme.typography.caption)
    }
}
```

## VII. 高级特性

### 7.1 多实例管理

```kotlin
class MultiInstanceWidget : GlanceAppWidget() {
    
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        // 每个实例可以有独立的状态
        val instanceState = getInstanceState(context, id)
        
        provideContent {
            WidgetContent(
                instanceId = id.toString(),
                config = instanceState
            )
        }
    }
    
    private suspend fun getInstanceState(context: Context, id: GlanceId): InstanceConfig {
        val prefs = context.dataStore.data.first()
        return InstanceConfig(
            city = prefs[stringPreferencesKey("city_${id}")] ?: "Beijing",
            showDetails = prefs[booleanPreferencesKey("details_${id}")] ?: false
        )
    }
}

// 更新特定实例
suspend fun updateSpecificInstance(context: Context, glanceId: GlanceId, city: String) {
    updateAppWidgetState(context, glanceId) { prefs ->
        prefs[stringPreferencesKey("city_${glanceId}")] = city
    }
    MultiInstanceWidget().update(context, glanceId)
}

// 更新所有实例
suspend fun updateAllInstances(context: Context) {
    val glanceIds = GlanceAppWidgetManager(context)
        .getGlanceIds(MultiInstanceWidget::class.java)
    
    glanceIds.forEach { glanceId ->
        updateSpecificInstance(context, glanceId, "Shanghai")
    }
}
```

### 7.2 配置 Activity 集成

```kotlin
class WidgetConfigActivity : AppCompatActivity() {
    
    private var glanceId: GlanceId? = null
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 获取 GlanceId
        val extras = intent.extras
        if (extras != null) {
            glanceId = GlanceAppWidgetManager(this)
                .getGlanceIdBy(extras.getInt(
                    AppWidgetManager.EXTRA_APPWIDGET_ID,
                    AppWidgetManager.INVALID_APPWIDGET_ID
                ))
        }
        
        setContent {
            ConfigScreen(
                onSave = { config ->
                    lifecycleScope.launch {
                        saveConfig(config)
                        finish()
                    }
                }
            )
        }
    }
    
    private suspend fun saveConfig(config: WidgetConfig) {
        glanceId?.let { id ->
            updateAppWidgetState(this, id) { prefs ->
                prefs[cityKey] = config.city
                prefs[themeKey] = config.theme
            }
            MyWidget().update(this, id)
        }
    }
}
```

### 7.3 WorkManager 集成实现定期更新

```kotlin
class WidgetUpdateWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            // 1. 获取新数据
            val data = fetchDataFromNetwork()
            
            // 2. 更新所有 Widget 实例
            val glanceIds = GlanceAppWidgetManager(applicationContext)
                .getGlanceIds(MyWidget::class.java)
            
            glanceIds.forEach { glanceId ->
                updateAppWidgetState(applicationContext, glanceId) { prefs ->
                    prefs[dataKey] = data.toString()
                    prefs[lastUpdateKey] = System.currentTimeMillis()
                }
            }
            
            // 3. 刷新 UI
            MyWidget().updateAll(applicationContext)
            
            Result.success()
        } catch (e: Exception) {
            Log.e("WidgetUpdateWorker", "Update failed", e)
            Result.retry()
        }
    }
}

// 调度 Worker
fun scheduleWidgetUpdates(context: Context) {
    val updateRequest = PeriodicWorkRequestBuilder<WidgetUpdateWorker>(
        15, TimeUnit.MINUTES
    )
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .build()
    
    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "WidgetUpdate",
        ExistingPeriodicWorkPolicy.KEEP,
        updateRequest
    )
}
```

## VIII. 性能优化与调试

### 8.1 性能优化建议

#### 减少不必要的更新

```kotlin
// ❌ 不好：频繁更新
class BadWidget : GlanceAppWidget() {
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        // 每次都重新计算
        val heavyData = performHeavyComputation()
        provideContent {
            Content(heavyData)
        }
    }
}

// ✅ 好：缓存和条件更新
class GoodWidget : GlanceAppWidget() {
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        val prefs = currentState<Preferences>()
        val lastUpdate = prefs[lastUpdateKey] ?: 0L
        val now = System.currentTimeMillis()
        
        // 只在需要时更新
        val data = if (now - lastUpdate > 5 * 60 * 1000) {
            val newData = performHeavyComputation()
            updateAppWidgetState(context, id) { p ->
                p[dataKey] = newData
                p[lastUpdateKey] = now
            }
            newData
        } else {
            prefs[dataKey] ?: ""
        }
        
        provideContent {
            Content(data)
        }
    }
}
```

#### 图片优化

```kotlin
// ✅ 使用合适尺寸的图片
@Composable
fun OptimizedImage() {
    Image(
        provider = ImageProvider(R.drawable.icon_small), // 使用小尺寸图片
        contentDescription = null,
        modifier = GlanceModifier.size(24.dp)
    )
}

// ✅ 异步加载并缓存
class ImageCacheManager(private val context: Context) {
    private val cache = LruCache<String, Bitmap>(10 * 1024 * 1024) // 10MB
    
    suspend fun loadImage(url: String): Bitmap? {
        return cache.get(url) ?: downloadAndCache(url)
    }
    
    private suspend fun downloadAndCache(url: String): Bitmap? {
        return withContext(Dispatchers.IO) {
            try {
                val bitmap = downloadBitmap(url)
                cache.put(url, bitmap)
                bitmap
            } catch (e: Exception) {
                null
            }
        }
    }
}
```

### 8.2 调试技巧

#### 日志调试

```kotlin
class DebugWidget : GlanceAppWidget() {
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        Log.d("DebugWidget", "provideGlance called for id: $id")
        
        val prefs = currentState<Preferences>()
        Log.d("DebugWidget", "Current state: $prefs")
        
        provideContent {
            DebugContent()
        }
    }
}

@Composable
fun DebugContent() {
    val size = LocalSize.current
    Log.d("DebugWidget", "Current size: ${size.width} x ${size.height}")
    
    Column {
        Text("Debug Mode")
        Text("Size: ${size.width} x ${size.height}")
    }
}
```

#### 使用 Preview（有限支持）

```kotlin
// Glance Preview 功能有限，主要用于布局预览
@Preview
@Composable
fun WidgetPreview() {
    // 模拟数据
    WeatherWidgetContent(
        temperature = 25,
        city = "Beijing",
        condition = WeatherCondition.SUNNY
    )
}
```

### 8.3 常见问题排查

#### 问题1：Widget 不更新

```kotlin
// 解决方案：确保调用 update
class RefreshCallback : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        updateAppWidgetState(context, glanceId) { prefs ->
            prefs[dataKey] = "new data"
        }
        
        // ⚠️ 必须调用 update
        MyWidget().update(context, glanceId)
    }
}
```

#### 问题2：状态不持久化

```kotlin
// ❌ 错误：使用 remember（不持久化）
@Composable
fun BadState() {
    var count by remember { mutableStateOf(0) }  // 不会保存
    Button(text = "Count: $count", onClick = { count++ })
}

// ✅ 正确：使用 DataStore
@Composable
fun GoodState() {
    val prefs = currentState<Preferences>()
    val count = prefs[countKey] ?: 0
    Button(
        text = "Count: $count",
        onClick = actionRunCallback<IncrementCallback>()
    )
}

class IncrementCallback : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        updateAppWidgetState(context, glanceId) { prefs ->
            val current = prefs[countKey] ?: 0
            prefs[countKey] = current + 1
        }
        MyWidget().update(context, glanceId)
    }
}
```

#### 问题3：点击事件不响应

```kotlin
// ❌ 错误：使用标准 Compose 的 clickable
Row(modifier = Modifier.clickable { }) {  // 不工作
    Text("Click me")
}

// ✅ 正确：使用 Glance 的 clickable
Row(
    modifier = GlanceModifier.clickable(
        actionRunCallback<ClickCallback>()
    )
) {
    Text("Click me")
}
```

## 总结

Jetpack Glance 是构建桌面小部件的现代化方案，核心要点：

### 架构理解
- ✅ Glance 是翻译层，最终转换为 RemoteViews
- ✅ 代码在 Provider 进程运行，UI 在 Launcher 进程渲染
- ✅ 状态通过 DataStore 跨进程传递

### 组件使用
- ✅ 使用 Glance 专有组件（`androidx.glance.*`）
- ✅ 布局：Column, Row, Box, LazyColumn
- ✅ 基础：Text, Image, Button, Progress
- ✅ Modifier：GlanceModifier（而非 Modifier）

### 状态管理
- ✅ 推荐使用 Preferences DataStore
- ✅ 支持自定义状态类和 Serializer
- ✅ 使用 `currentState<T>()` 读取状态
- ✅ 使用 `updateAppWidgetState` 更新状态

### Action 系统
- ✅ 启动 Activity：`actionStartActivity`
- ✅ 执行回调：`actionRunCallback`
- ✅ 发送广播：`actionSendBroadcast`
- ✅ 启动服务：`actionStartService`

### 响应式设计
- ✅ 使用 `SizeMode` 定义适配策略
- ✅ 使用 `LocalSize` 获取当前尺寸
- ✅ 根据尺寸切换布局或调整样式

### 性能优化
- ✅ 缓存计算结果和图片
- ✅ 避免频繁更新
- ✅ 使用 WorkManager 管理后台更新
- ✅ 正确处理生命周期

---

## 推荐阅读

- [Jetpack Glance Documentation - Android Developers](https://developer.android.com/jetpack/androidx/releases/glance)
- [Glance App Widget Samples - GitHub](https://github.com/android/user-interface-samples/tree/main/AppWidget)
- [Build app widgets with Glance - Android Developers](https://developer.android.com/develop/ui/compose/glance)
