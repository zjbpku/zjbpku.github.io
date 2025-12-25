# Android 桌面小部件开发完全指南：从传统到 Compose

> **发布日期**: 2024-12-23  
> **阅读时间**: 约 50-60 分钟  
> **标签**: Android, Widgets, Glance, RemoteViews, Compose, 桌面小部件

桌面小部件 (Home Screen Widgets) 是 Android 用户体验的重要组成部分，让用户无需打开应用即可查看关键信息。从 Android 1.5 引入至今，桌面小部件经历了多次重大更新。本文将全面介绍桌面小部件的开发，从传统的 XML/RemoteViews 方式到现代的 Jetpack Glance (Compose) 方式，以及各个 Android 版本的特性演进。

## 📚 官方参考

- [App Widgets Overview - Android Developers](https://developer.android.com/develop/ui/views/appwidgets/overview)
- [Jetpack Glance - Android Developers](https://developer.android.com/jetpack/androidx/releases/glance)
- [Build an App Widget - Android Developers](https://developer.android.com/develop/ui/views/appwidgets)

## 目录

- [I. 桌面小部件基础概念](#i-桌面小部件基础概念)
- [II. Android 版本演进历史](#ii-android-版本演进历史)
- [III. 传统方式：RemoteViews + XML](#iii-传统方式remoteviews--xml)
- [IV. 现代方式：Jetpack Glance (Compose)](#iv-现代方式jetpack-glance-compose)
- [V. 状态管理与数据更新](#v-状态管理与数据更新)
- [VI. 交互与响应](#vi-交互与响应)
- [VII. 性能优化与最佳实践](#vii-性能优化与最佳实践)
- [VIII. 高级特性](#viii-高级特性)

## I. 桌面小部件基础概念

### 1.1 什么是桌面小部件？

桌面小部件是显示在设备主屏幕上的迷你应用视图，它们提供：
- **一目了然的信息**：天气、日历、股票等
- **快速操作**：播放控制、便签、倒计时等
- **实时更新**：无需打开应用即可查看最新信息

### 1.2 核心组件

一个完整的桌面小部件包含以下部分：

1. **AppWidgetProvider**：继承自 `BroadcastReceiver`，处理小部件的生命周期事件
2. **AppWidgetProviderInfo**：XML 配置文件，定义小部件的元数据（尺寸、更新频率等）
3. **Layout (RemoteViews 或 Glance)**：小部件的视图层
4. **AppWidgetManager**：系统服务，负责管理所有小部件

### 1.3 生命周期

```text
用户添加小部件
    ↓
onEnabled() - 首次添加时调用一次
    ↓
onUpdate() - 每次更新时调用
    ↓
onAppWidgetOptionsChanged() - 尺寸改变时调用
    ↓
onDeleted() - 用户删除小部件
    ↓
onDisabled() - 最后一个小部件被删除时调用
```

## II. Android 版本演进历史

### 2.1 各版本重要特性

| Android 版本 | API Level | 重要特性 |
|-------------|-----------|---------|
| **Android 1.5 (Cupcake)** | 3 | 首次引入桌面小部件 |
| **Android 3.0 (Honeycomb)** | 11 | 引入可调整大小的小部件 |
| **Android 3.1** | 12 | 引入 ListView/GridView 小部件（集合视图） |
| **Android 4.1 (Jelly Bean)** | 16 | 自动调整大小以适应屏幕 |
| **Android 4.2** | 17 | 锁屏小部件（已在 Android 5.0 移除） |
| **Android 8.0 (Oreo)** | 26 | 可固定的快捷方式和小部件 |
| **Android 12** | 31 | 改进的小部件选择器，圆角和边距支持 |
| **Android 12L** | 32 | 动态颜色主题支持 |
| **Android 13** | 33 | 改进的小部件配置流程 |

### 2.2 Android 12+ 的重大变化

Android 12 引入了全新的小部件系统，主要变化包括：

1. **动态颜色 (Dynamic Colors)**：小部件可以自动适应系统壁纸配色
2. **圆角和边距**：系统自动添加，无需手动处理
3. **改进的选择器**：更美观的小部件预览和添加流程
4. **更灵活的尺寸**：`targetCellWidth`/`targetCellHeight` 替代固定尺寸

```xml
<!-- Android 12+ 推荐的尺寸定义 -->
<appwidget-provider
    android:targetCellWidth="3"
    android:targetCellHeight="2"
    android:maxResizeWidth="180dp"
    android:maxResizeHeight="110dp" />
```

## III. 传统方式：RemoteViews + XML

传统方式是 Android 最初设计的小部件开发模式，直到现在仍然被广泛使用。

### 3.1 创建 AppWidgetProvider

这是小部件的核心逻辑类。

```kotlin
class WeatherWidgetProvider : AppWidgetProvider() {
    
    override fun onUpdate(
        context: Context,
        appWidgetManager: AppWidgetManager,
        appWidgetIds: IntArray
    ) {
        // 遍历所有小部件实例
        for (appWidgetId in appWidgetIds) {
            updateAppWidget(context, appWidgetManager, appWidgetId)
        }
    }
    
    override fun onEnabled(context: Context) {
        // 第一个小部件被添加时调用
        // 启动后台服务、注册监听器等
    }
    
    override fun onDisabled(context: Context) {
        // 最后一个小部件被删除时调用
        // 清理资源、取消注册等
    }
    
    private fun updateAppWidget(
        context: Context,
        appWidgetManager: AppWidgetManager,
        appWidgetId: Int
    ) {
        // 创建 RemoteViews
        val views = RemoteViews(context.packageName, R.layout.widget_weather)
        
        // 更新数据
        views.setTextViewText(R.id.widget_temperature, "25°C")
        views.setTextViewText(R.id.widget_city, "Beijing")
        views.setImageViewResource(R.id.widget_icon, R.drawable.ic_sunny)
        
        // 添加点击事件
        val intent = Intent(context, MainActivity::class.java)
        val pendingIntent = PendingIntent.getActivity(
            context,
            0,
            intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
        views.setOnClickPendingIntent(R.id.widget_container, pendingIntent)
        
        // 更新小部件
        appWidgetManager.updateAppWidget(appWidgetId, views)
    }
}
```

### 3.2 定义布局 (RemoteViews 限制)

RemoteViews 只支持以下布局和视图：

**支持的布局：**
- `FrameLayout`
- `LinearLayout`
- `RelativeLayout`
- `GridLayout`

**支持的视图：**
- `TextView`
- `ImageView`
- `Button`
- `ProgressBar`
- `Chronometer`
- `ListView`
- `GridView`
- `StackView`
- `ViewFlipper`

```xml
<!-- res/layout/widget_weather.xml -->
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/widget_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    android:background="@drawable/widget_background">

    <TextView
        android:id="@+id/widget_city"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="16sp"
        android:textColor="@android:color/white"
        android:text="City" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:gravity="center_vertical">

        <ImageView
            android:id="@+id/widget_icon"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:src="@drawable/ic_sunny" />

        <TextView
            android:id="@+id/widget_temperature"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textSize="32sp"
            android:textColor="@android:color/white"
            android:layout_marginStart="8dp"
            android:text="25°C" />
    </LinearLayout>
</LinearLayout>
```

### 3.3 配置 AppWidgetProviderInfo

在 `res/xml/weather_widget_info.xml` 中定义元数据。

```xml
<?xml version="1.0" encoding="utf-8"?>
<appwidget-provider
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="180dp"
    android:minHeight="110dp"
    android:targetCellWidth="3"
    android:targetCellHeight="2"
    android:updatePeriodMillis="1800000"
    android:initialLayout="@layout/widget_weather"
    android:previewImage="@drawable/widget_preview"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen"
    android:description="@string/widget_description"
    android:previewLayout="@layout/widget_weather" />
```

**关键属性说明：**
- `minWidth`/`minHeight`：最小尺寸（Android 12 前）
- `targetCellWidth`/`targetCellHeight`：目标单元格数量（Android 12+）
- `updatePeriodMillis`：自动更新间隔（最小 30 分钟）
- `resizeMode`：可调整大小的方向
- `widgetCategory`：小部件类型（`home_screen` 或 `keyguard`）

### 3.4 注册到 AndroidManifest.xml

```xml
<receiver
    android:name=".WeatherWidgetProvider"
    android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/weather_widget_info" />
</receiver>
```

## IV. 现代方式：Jetpack Glance (Compose)

Jetpack Glance 允许使用 Compose 风格的代码构建小部件，大大简化了开发流程。

### 4.1 添加依赖

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.glance:glance-appwidget:1.1.0")
    implementation("androidx.glance:glance-material3:1.1.0")
}
```

### 4.2 创建 GlanceAppWidget

Glance 的核心是 `GlanceAppWidget` 类。

```kotlin
import androidx.compose.runtime.*
import androidx.glance.*
import androidx.glance.appwidget.*
import androidx.glance.layout.*
import androidx.glance.text.*
import androidx.glance.action.*
import androidx.glance.unit.ColorProvider

class WeatherGlanceWidget : GlanceAppWidget() {

    override suspend fun provideGlance(context: Context, id: GlanceId) {
        // 加载数据（可以从 Repository/DataStore 获取）
        val weatherData = loadWeatherData(context)
        
        // 提供内容
        provideContent {
            WeatherWidgetContent(weatherData)
        }
    }

    @Composable
    private fun WeatherWidgetContent(data: WeatherData) {
        // 使用 Glance Composable
        Column(
            modifier = GlanceModifier
                .fillMaxSize()
                .background(ImageProvider(R.drawable.widget_background))
                .padding(16.dp),
            verticalAlignment = Alignment.Vertical.Top,
            horizontalAlignment = Alignment.Horizontal.Start
        ) {
            // 城市名
            Text(
                text = data.city,
                style = TextStyle(
                    color = ColorProvider(Color.White),
                    fontSize = 16.sp
                )
            )

            Spacer(modifier = GlanceModifier.height(8.dp))

            // 温度和图标
            Row(
                verticalAlignment = Alignment.Vertical.CenterVertically
            ) {
                Image(
                    provider = ImageProvider(data.iconRes),
                    contentDescription = "Weather Icon",
                    modifier = GlanceModifier.size(48.dp)
                )

                Spacer(modifier = GlanceModifier.width(8.dp))

                Text(
                    text = "${data.temperature}°C",
                    style = TextStyle(
                        color = ColorProvider(Color.White),
                        fontSize = 32.sp,
                        fontWeight = FontWeight.Bold
                    )
                )
            }

            Spacer(modifier = GlanceModifier.height(8.dp))

            // 刷新按钮
            Button(
                text = "Refresh",
                onClick = actionRunCallback<RefreshWeatherCallback>(),
                colors = ButtonDefaults.buttonColors(
                    backgroundColor = ColorProvider(Color.White.copy(alpha = 0.2f))
                )
            )
        }
    }

    private suspend fun loadWeatherData(context: Context): WeatherData {
        // 从 DataStore/Repository 加载数据
        return WeatherData("Beijing", 25, R.drawable.ic_sunny)
    }
}

data class WeatherData(
    val city: String,
    val temperature: Int,
    val iconRes: Int
)
```

### 4.3 创建 GlanceAppWidgetReceiver

这是 Glance 版本的 Receiver。

```kotlin
class WeatherGlanceWidgetReceiver : GlanceAppWidgetReceiver() {
    override val glanceAppWidget: GlanceAppWidget = WeatherGlanceWidget()
}
```

### 4.4 配置和注册

配置文件和注册方式与传统方式相同。

```xml
<!-- res/xml/weather_glance_widget_info.xml -->
<appwidget-provider
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:targetCellWidth="3"
    android:targetCellHeight="2"
    android:updatePeriodMillis="1800000"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen"
    android:description="@string/widget_description"
    android:previewLayout="@layout/widget_preview" />
```

```xml
<!-- AndroidManifest.xml -->
<receiver
    android:name=".WeatherGlanceWidgetReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/weather_glance_widget_info" />
</receiver>
```

### 4.5 Glance vs 标准 Compose 对比

| 特性 | 标准 Compose | Glance Compose |
|------|-------------|----------------|
| **导入包** | `androidx.compose.ui.*` | `androidx.glance.*` |
| **Modifier** | `Modifier` | `GlanceModifier` |
| **状态管理** | `mutableStateOf`, `remember` | 不支持实时状态，需用 DataStore |
| **布局组件** | `Box`, `Row`, `Column` 等 | `Box`, `Row`, `Column`（Glance 版） |
| **点击事件** | `onClick = { }` | `onClick = actionRunCallback<>()` |
| **Canvas 绘图** | 支持 | 不支持 |
| **动画** | 完整支持 | 非常有限 |

## V. 状态管理与数据更新

### 5.1 使用 DataStore 持久化状态

Glance 推荐使用 DataStore 来管理小部件状态。

```kotlin
// 定义状态
data class WeatherWidgetState(
    val city: String = "Unknown",
    val temperature: Int = 0,
    val lastUpdate: Long = 0
)

// 定义 StateDefinition
object WeatherStateDefinition : GlanceStateDefinition<WeatherWidgetState> {
    
    private const val DATA_STORE_FILE = "weather_widget_state"
    
    override suspend fun getDataStore(
        context: Context,
        fileKey: String
    ): DataStore<WeatherWidgetState> {
        return context.dataStore
    }
    
    override fun getLocation(context: Context, fileKey: String): File {
        return context.dataStoreFile(DATA_STORE_FILE)
    }
    
    private val Context.dataStore by preferencesDataStore(name = DATA_STORE_FILE)
}

// 在 GlanceAppWidget 中使用
class WeatherGlanceWidget : GlanceAppWidget() {
    
    override val stateDefinition = WeatherStateDefinition
    
    override suspend fun provideGlance(context: Context, id: GlanceId) {
        provideContent {
            val state = currentState<WeatherWidgetState>()
            WeatherWidgetContent(state)
        }
    }
}
```

### 5.2 手动触发更新

```kotlin
// 在 Activity 或 Service 中更新小部件
class UpdateWeatherService : Service() {
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        lifecycleScope.launch {
            val weatherData = fetchWeatherFromApi()
            
            // 更新 Glance 小部件
            WeatherGlanceWidget().updateAll(this@UpdateWeatherService)
        }
        return START_NOT_STICKY
    }
}
```

### 5.3 定期更新策略

由于 `updatePeriodMillis` 最小为 30 分钟，如果需要更频繁的更新：

**方式 1：使用 WorkManager**
```kotlin
val updateRequest = PeriodicWorkRequestBuilder<WeatherUpdateWorker>(
    15, TimeUnit.MINUTES
).build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "WeatherWidgetUpdate",
    ExistingPeriodicWorkPolicy.KEEP,
    updateRequest
)
```

**方式 2：使用 AlarmManager（精确时间）**
```kotlin
val alarmManager = context.getSystemService<AlarmManager>()
val intent = Intent(context, WeatherWidgetReceiver::class.java).apply {
    action = ACTION_UPDATE_WIDGET
}
val pendingIntent = PendingIntent.getBroadcast(
    context, 0, intent, PendingIntent.FLAG_IMMUTABLE
)

alarmManager?.setRepeating(
    AlarmManager.RTC,
    System.currentTimeMillis(),
    AlarmManager.INTERVAL_FIFTEEN_MINUTES,
    pendingIntent
)
```

## VI. 交互与响应

### 6.1 Glance 中的点击事件

Glance 提供了多种 Action 类型：

```kotlin
@Composable
fun InteractiveWidget() {
    Column {
        // 1. 启动 Activity
        Button(
            text = "Open App",
            onClick = actionStartActivity<MainActivity>()
        )
        
        // 2. 运行 Callback（后台任务）
        Button(
            text = "Refresh",
            onClick = actionRunCallback<RefreshCallback>()
        )
        
        // 3. 发送广播
        Button(
            text = "Broadcast",
            onClick = actionSendBroadcast(Intent("com.example.ACTION"))
        )
        
        // 4. 启动 Service
        Button(
            text = "Start Service",
            onClick = actionStartService<UpdateService>()
        )
    }
}
```

### 6.2 实现 ActionCallback

```kotlin
class RefreshWeatherCallback : ActionCallback {
    override suspend fun onAction(
        context: Context,
        glanceId: GlanceId,
        parameters: ActionParameters
    ) {
        // 执行后台任务
        val newData = fetchWeatherFromNetwork()
        
        // 更新状态
        updateAppWidgetState(context, glanceId) { prefs ->
            prefs[temperatureKey] = newData.temperature
            prefs[cityKey] = newData.city
        }
        
        // 刷新小部件
        WeatherGlanceWidget().update(context, glanceId)
    }
}
```

### 6.3 传统 RemoteViews 中的交互

```kotlin
// 点击跳转 Activity
val intent = Intent(context, MainActivity::class.java)
val pendingIntent = PendingIntent.getActivity(
    context, 0, intent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)
views.setOnClickPendingIntent(R.id.widget_button, pendingIntent)

// 点击触发广播
val broadcastIntent = Intent(context, WeatherWidgetProvider::class.java).apply {
    action = ACTION_REFRESH
}
val broadcastPendingIntent = PendingIntent.getBroadcast(
    context, 0, broadcastIntent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)
views.setOnClickPendingIntent(R.id.refresh_button, broadcastPendingIntent)
```

## VII. 性能优化与最佳实践

### 7.1 性能优化建议

**1. 减少更新频率**
```kotlin
// ❌ 不推荐：频繁更新
updatePeriodMillis="60000"  // 1分钟

// ✅ 推荐：合理间隔
updatePeriodMillis="1800000"  // 30分钟
```

**2. 懒加载数据**
```kotlin
override suspend fun provideGlance(context: Context, id: GlanceId) {
    provideContent {
        // ✅ 只在需要时加载
        val data = remember { loadData(context) }
        WidgetContent(data)
    }
}
```

**3. 使用缓存**
```kotlin
class WeatherRepository {
    private var cache: WeatherData? = null
    private var lastFetch: Long = 0
    
    suspend fun getWeather(): WeatherData {
        val now = System.currentTimeMillis()
        if (cache != null && now - lastFetch < 10 * 60 * 1000) {
            return cache!!
        }
        
        cache = fetchFromNetwork()
        lastFetch = now
        return cache!!
    }
}
```

### 7.2 内存和电量优化

**避免内存泄漏：**
```kotlin
override fun onDisabled(context: Context) {
    super.onDisabled(context)
    // 取消所有后台任务
    WorkManager.getInstance(context).cancelUniqueWork("WidgetUpdate")
    // 清理资源
    repository.clear()
}
```

**电量优化：**
- 使用 `WorkManager` 替代 `AlarmManager`
- 合并网络请求
- 避免持续运行的后台服务

### 7.3 用户体验最佳实践

1. **提供加载状态**
```kotlin
@Composable
fun WidgetWithLoading(isLoading: Boolean, data: WeatherData?) {
    if (isLoading) {
        Box(modifier = GlanceModifier.fillMaxSize()) {
            CircularProgressIndicator()
        }
    } else {
        WeatherContent(data ?: WeatherData.empty())
    }
}
```

2. **错误处理**
```kotlin
sealed class WidgetState {
    object Loading : WidgetState()
    data class Success(val data: WeatherData) : WidgetState()
    data class Error(val message: String) : WidgetState()
}
```

3. **响应式设计**
```kotlin
@Composable
fun ResponsiveWidget(size: DpSize) {
    if (size.width > 200.dp) {
        LargeWidgetLayout()
    } else {
        SmallWidgetLayout()
    }
}
```

## VIII. 高级特性

### 8.1 集合视图 (ListView/GridView)

传统方式中支持显示列表数据：

```kotlin
// RemoteViewsService
class WeatherListService : RemoteViewsService() {
    override fun onGetViewFactory(intent: Intent): RemoteViewsFactory {
        return WeatherRemoteViewsFactory(this.applicationContext)
    }
}

class WeatherRemoteViewsFactory(
    private val context: Context
) : RemoteViewsService.RemoteViewsFactory {
    
    private var dataList: List<WeatherItem> = emptyList()
    
    override fun onCreate() {
        // 初始化
    }
    
    override fun onDataSetChanged() {
        // 数据变化时重新加载
        dataList = loadWeatherList()
    }
    
    override fun getViewAt(position: Int): RemoteViews {
        val item = dataList[position]
        return RemoteViews(context.packageName, R.layout.widget_list_item).apply {
            setTextViewText(R.id.city_name, item.city)
            setTextViewText(R.id.temperature, "${item.temp}°C")
        }
    }
    
    override fun getCount(): Int = dataList.size
    
    // 其他必需方法...
}
```

### 8.2 配置 Activity

允许用户在添加小部件时进行配置：

```kotlin
class WeatherWidgetConfigActivity : AppCompatActivity() {
    
    private var appWidgetId = AppWidgetManager.INVALID_APPWIDGET_ID
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setResult(RESULT_CANCELED)
        
        appWidgetId = intent.extras?.getInt(
            AppWidgetManager.EXTRA_APPWIDGET_ID,
            AppWidgetManager.INVALID_APPWIDGET_ID
        ) ?: AppWidgetManager.INVALID_APPWIDGET_ID
        
        if (appWidgetId == AppWidgetManager.INVALID_APPWIDGET_ID) {
            finish()
            return
        }
        
        setContent {
            ConfigScreen(
                onConfirm = { city ->
                    saveConfiguration(city)
                    updateWidget()
                    
                    val resultValue = Intent().putExtra(
                        AppWidgetManager.EXTRA_APPWIDGET_ID,
                        appWidgetId
                    )
                    setResult(RESULT_OK, resultValue)
                    finish()
                }
            )
        }
    }
    
    private fun saveConfiguration(city: String) {
        val prefs = getSharedPreferences("widget_prefs", MODE_PRIVATE)
        prefs.edit().putString("city_$appWidgetId", city).apply()
    }
}
```

在 AppWidgetProviderInfo 中声明：
```xml
<appwidget-provider
    android:configure="com.example.WeatherWidgetConfigActivity"
    ... />
```

### 8.3 自适应图标和动态颜色 (Android 12+)

```kotlin
@Composable
fun DynamicColorWidget() {
    // 使用系统动态颜色
    Column(
        modifier = GlanceModifier
            .fillMaxSize()
            .background(ColorProvider(day = Color.White, night = Color.Black))
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
    }
}
```

### 8.4 多尺寸布局 (Responsive Widgets)

```kotlin
@Composable
fun ResponsiveWeatherWidget() {
    val size = LocalSize.current
    
    when {
        size.width < 150.dp -> SmallWidget()
        size.width < 250.dp -> MediumWidget()
        else -> LargeWidget()
    }
}
```

## 总结

Android 桌面小部件开发的核心要点：

### 传统方式 (RemoteViews)
- ✅ 兼容性好：支持所有 Android 版本
- ✅ 性能稳定：经过长期验证
- ❌ 开发复杂：XML + RemoteViews API
- ❌ 功能受限：仅支持有限的视图类型

### 现代方式 (Glance)
- ✅ 开发简单：Compose 风格代码
- ✅ 类型安全：编译时检查
- ✅ 易于维护：声明式 UI
- ❌ 需要学习曲线：Glance 特有 API
- ❌ 功能限制：底层仍是 RemoteViews

### 最佳实践
- ✅ 使用 WorkManager 管理定期更新
- ✅ 使用 DataStore 持久化状态
- ✅ 提供加载和错误状态
- ✅ 优化更新频率和网络请求
- ✅ 适配 Android 12+ 新特性

---

## 推荐阅读

- [App Widgets Overview - Android Developers](https://developer.android.com/develop/ui/views/appwidgets/overview)
- [Jetpack Glance - Android Developers](https://developer.android.com/jetpack/androidx/releases/glance)
- [Build an App Widget - Android Developers](https://developer.android.com/develop/ui/views/appwidgets)
