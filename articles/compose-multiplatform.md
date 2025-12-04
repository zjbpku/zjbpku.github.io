# Compose Multiplatform 入门：跨平台 UI 开发实战

> **发布日期**: 2024-05-22  
> **阅读时间**: 约 28 分钟  
> **标签**: KMP, Compose Multiplatform, 跨平台, iOS

Compose Multiplatform 是 JetBrains 基于 Jetpack Compose 打造的跨平台 UI 框架，让你可以用 Kotlin 和 Compose 编写一次代码，运行在 Android、iOS、Desktop 和 Web 上。本文将带你快速入门 Compose Multiplatform 开发。

## 一、什么是 Compose Multiplatform？

Compose Multiplatform 是 Kotlin Multiplatform（KMP）生态的一部分，专注于 UI 层的跨平台共享：

```
┌─────────────────────────────────────────────────┐
│              Compose Multiplatform              │
│         (共享 UI - Composable 函数)              │
├─────────────────────────────────────────────────┤
│              Kotlin Multiplatform               │
│           (共享业务逻辑 - Kotlin)                │
├───────────┬───────────┬───────────┬─────────────┤
│  Android  │    iOS    │  Desktop  │    Web      │
│  (JVM)    │ (Native)  │   (JVM)   │   (Wasm)    │
└───────────┴───────────┴───────────┴─────────────┘
```

### 与 Flutter、React Native 的区别

| 特性 | Compose Multiplatform | Flutter | React Native |
|-----|----------------------|---------|--------------|
| 语言 | Kotlin | Dart | JavaScript |
| UI 渲染 | 原生（Android）/ Skia | Skia | 原生组件 |
| Android 体验 | 最优（原生 Compose）| 良好 | 良好 |
| iOS 体验 | 良好 | 良好 | 良好 |
| 生态成熟度 | 快速发展中 | 成熟 | 成熟 |
| 与原生互操作 | 无缝 | 需要 Channel | 需要 Bridge |

## 二、项目配置

### 使用 KMP Wizard 创建项目

1. 访问 [kmp.jetbrains.com](https://kmp.jetbrains.com/)
2. 选择目标平台（Android、iOS、Desktop、Web）
3. 下载项目模板

### 项目结构

```
my-kmp-app/
├── composeApp/
│   ├── src/
│   │   ├── commonMain/          # 共享代码
│   │   │   └── kotlin/
│   │   │       ├── App.kt       # 共享 UI
│   │   │       └── Platform.kt  # expect 声明
│   │   ├── androidMain/         # Android 特定
│   │   │   └── kotlin/
│   │   │       └── Platform.android.kt
│   │   ├── iosMain/             # iOS 特定
│   │   │   └── kotlin/
│   │   │       └── Platform.ios.kt
│   │   └── desktopMain/         # Desktop 特定
│   └── build.gradle.kts
├── iosApp/                      # iOS 宿主应用
│   └── iosApp/
│       └── ContentView.swift
└── build.gradle.kts
```

### build.gradle.kts 配置

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.androidApplication)
    alias(libs.plugins.jetbrainsCompose)
    alias(libs.plugins.compose.compiler)
}

kotlin {
    androidTarget {
        compilations.all {
            kotlinOptions {
                jvmTarget = "11"
            }
        }
    }
    
    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "ComposeApp"
            isStatic = true
        }
    }
    
    jvm("desktop")
    
    sourceSets {
        commonMain.dependencies {
            implementation(compose.runtime)
            implementation(compose.foundation)
            implementation(compose.material3)
            implementation(compose.ui)
            implementation(compose.components.resources)
        }
        
        androidMain.dependencies {
            implementation(libs.androidx.activity.compose)
        }
        
        desktopMain.dependencies {
            implementation(compose.desktop.currentOs)
        }
    }
}
```

## 三、编写共享 UI

### 基础示例

```kotlin
// commonMain/kotlin/App.kt
@Composable
fun App() {
    MaterialTheme {
        var count by remember { mutableStateOf(0) }
        
        Column(
            modifier = Modifier.fillMaxSize(),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text(
                text = "Count: $count",
                style = MaterialTheme.typography.headlineMedium
            )
            
            Spacer(modifier = Modifier.height(16.dp))
            
            Button(onClick = { count++ }) {
                Text("Increment")
            }
            
            Spacer(modifier = Modifier.height(32.dp))
            
            // 显示平台信息
            Text(
                text = "Running on: ${getPlatform().name}",
                style = MaterialTheme.typography.bodyMedium
            )
        }
    }
}
```

### expect/actual 平台特定实现

```kotlin
// commonMain/kotlin/Platform.kt
interface Platform {
    val name: String
}

expect fun getPlatform(): Platform

// androidMain/kotlin/Platform.android.kt
class AndroidPlatform : Platform {
    override val name: String = "Android ${android.os.Build.VERSION.SDK_INT}"
}

actual fun getPlatform(): Platform = AndroidPlatform()

// iosMain/kotlin/Platform.ios.kt
import platform.UIKit.UIDevice

class IOSPlatform : Platform {
    override val name: String = UIDevice.currentDevice.systemName() + " " + 
                                 UIDevice.currentDevice.systemVersion
}

actual fun getPlatform(): Platform = IOSPlatform()

// desktopMain/kotlin/Platform.desktop.kt
class DesktopPlatform : Platform {
    override val name: String = "Desktop ${System.getProperty("os.name")}"
}

actual fun getPlatform(): Platform = DesktopPlatform()
```

## 四、平台入口点

### Android

```kotlin
// androidMain/kotlin/MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            App()
        }
    }
}
```

### iOS（Swift）

```swift
// iosApp/ContentView.swift
import SwiftUI
import ComposeApp

struct ContentView: View {
    var body: some View {
        ComposeView()
            .ignoresSafeArea(.all)
    }
}

struct ComposeView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController {
        MainViewControllerKt.MainViewController()
    }
    
    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}
```

```kotlin
// iosMain/kotlin/MainViewController.kt
fun MainViewController() = ComposeUIViewController { App() }
```

### Desktop

```kotlin
// desktopMain/kotlin/main.kt
fun main() = application {
    Window(
        onCloseRequest = ::exitApplication,
        title = "My KMP App"
    ) {
        App()
    }
}
```

## 五、资源管理

### 共享资源

```kotlin
// build.gradle.kts
compose.resources {
    publicResClass = true
    packageOfResClass = "com.myapp.resources"
    generateResClass = always
}
```

```
composeApp/src/commonMain/composeResources/
├── drawable/
│   └── logo.png
├── font/
│   └── roboto_regular.ttf
└── values/
    └── strings.xml
```

### 使用资源

```kotlin
@Composable
fun LogoImage() {
    Image(
        painter = painterResource(Res.drawable.logo),
        contentDescription = "Logo"
    )
}

@Composable
fun AppText() {
    Text(text = stringResource(Res.string.app_name))
}
```

## 六、网络请求（Ktor）

### 配置

```kotlin
// commonMain dependencies
implementation("io.ktor:ktor-client-core:2.3.7")
implementation("io.ktor:ktor-client-content-negotiation:2.3.7")
implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.7")

// androidMain dependencies
implementation("io.ktor:ktor-client-okhttp:2.3.7")

// iosMain dependencies
implementation("io.ktor:ktor-client-darwin:2.3.7")

// desktopMain dependencies
implementation("io.ktor:ktor-client-cio:2.3.7")
```

### 共享网络层

```kotlin
// commonMain/kotlin/data/ApiClient.kt
class ApiClient {
    private val httpClient = HttpClient {
        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true
                prettyPrint = true
            })
        }
    }
    
    suspend fun getUsers(): List<User> {
        return httpClient.get("https://api.example.com/users").body()
    }
}

@Serializable
data class User(
    val id: Int,
    val name: String,
    val email: String
)
```

## 七、本地存储

### 使用 DataStore（Multiplatform）

```kotlin
// 使用 multiplatform-settings
implementation("com.russhwolf:multiplatform-settings:1.1.1")
implementation("com.russhwolf:multiplatform-settings-coroutines:1.1.1")

// commonMain
class SettingsRepository(private val settings: Settings) {
    
    private val themeKey = "theme_mode"
    
    var themeMode: String
        get() = settings.getString(themeKey, "system")
        set(value) = settings.putString(themeKey, value)
    
    fun observeThemeMode(): Flow<String> = settings
        .getStringFlow(themeKey, "system")
}
```

### 平台特定创建

```kotlin
// androidMain
actual fun createSettings(context: Context): Settings {
    return SharedPreferencesSettings(
        context.getSharedPreferences("app_prefs", Context.MODE_PRIVATE)
    )
}

// iosMain
actual fun createSettings(): Settings {
    return NSUserDefaultsSettings(NSUserDefaults.standardUserDefaults)
}
```

## 八、ViewModel 共享

### 使用 KMP-ViewModel

```kotlin
// 依赖
implementation("org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose:2.8.0")

// commonMain/kotlin/viewmodel/HomeViewModel.kt
class HomeViewModel : ViewModel() {
    
    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()
    
    private val apiClient = ApiClient()
    
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                val users = apiClient.getUsers()
                _uiState.update { it.copy(isLoading = false, users = users) }
            } catch (e: Exception) {
                _uiState.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }
}

data class HomeUiState(
    val isLoading: Boolean = false,
    val users: List<User> = emptyList(),
    val error: String? = null
)

// 使用
@Composable
fun HomeScreen(viewModel: HomeViewModel = viewModel { HomeViewModel() }) {
    val uiState by viewModel.uiState.collectAsState()
    
    LaunchedEffect(Unit) {
        viewModel.loadUsers()
    }
    
    // UI...
}
```

## 九、导航

### 使用 Voyager

```kotlin
// 依赖
implementation("cafe.adriel.voyager:voyager-navigator:1.0.0")

// commonMain
class HomeScreen : Screen {
    @Composable
    override fun Content() {
        val navigator = LocalNavigator.currentOrThrow
        
        Column {
            Text("Home Screen")
            Button(onClick = { navigator.push(DetailScreen("123")) }) {
                Text("Go to Detail")
            }
        }
    }
}

data class DetailScreen(val id: String) : Screen {
    @Composable
    override fun Content() {
        val navigator = LocalNavigator.currentOrThrow
        
        Column {
            Text("Detail: $id")
            Button(onClick = { navigator.pop() }) {
                Text("Go Back")
            }
        }
    }
}

// App.kt
@Composable
fun App() {
    Navigator(HomeScreen())
}
```

## 十、iOS 特定适配

### 安全区域

```kotlin
@Composable
fun SafeAreaScreen() {
    val safeAreaInsets = WindowInsets.safeDrawing
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .windowInsetsPadding(safeAreaInsets)
    ) {
        // 内容
    }
}
```

### 与 SwiftUI 互操作

```swift
// 在 SwiftUI 中使用 Compose
struct ContentView: View {
    @State private var count = 0
    
    var body: some View {
        VStack {
            // SwiftUI 原生组件
            Text("SwiftUI: \(count)")
            
            // 嵌入 Compose
            ComposeView(count: $count)
                .frame(height: 200)
        }
    }
}
```

```kotlin
// 暴露给 Swift 的 Composable
fun ComposeCounterView(count: Int, onIncrement: () -> Unit) = ComposeUIViewController {
    Button(onClick = onIncrement) {
        Text("Compose Count: $count")
    }
}
```

## 十一、性能优化

### 1. 条件编译

```kotlin
@Composable
fun PlatformOptimizedList(items: List<Item>) {
    if (getPlatform().name.startsWith("iOS")) {
        // iOS 优化：使用 LazyColumn
        LazyColumn {
            items(items) { item -> ItemRow(item) }
        }
    } else {
        // Android/Desktop 可以使用更多特性
        LazyColumn(
            modifier = Modifier.fillMaxSize(),
            contentPadding = PaddingValues(16.dp)
        ) {
            items(items, key = { it.id }) { item -> 
                ItemRow(item) 
            }
        }
    }
}
```

### 2. 图片加载优化

```kotlin
// 使用 Coil（支持 KMP）
implementation("io.coil-kt.coil3:coil-compose:3.0.0-alpha06")
implementation("io.coil-kt.coil3:coil-network-ktor:3.0.0-alpha06")

@Composable
fun NetworkImage(url: String) {
    AsyncImage(
        model = url,
        contentDescription = null,
        modifier = Modifier.size(100.dp)
    )
}
```

## 十二、测试

### 共享测试

```kotlin
// commonTest
class HomeViewModelTest {
    
    @Test
    fun `loadUsers updates state correctly`() = runTest {
        val viewModel = HomeViewModel(FakeApiClient())
        
        viewModel.loadUsers()
        advanceUntilIdle()
        
        assertEquals(2, viewModel.uiState.value.users.size)
    }
}
```

### UI 测试

```kotlin
// 使用 compose-ui-test
@OptIn(ExperimentalTestApi::class)
class AppTest {
    
    @Test
    fun counterIncrements() = runComposeUiTest {
        setContent { App() }
        
        onNodeWithText("Count: 0").assertExists()
        onNodeWithText("Increment").performClick()
        onNodeWithText("Count: 1").assertExists()
    }
}
```

## 十三、最佳实践

- ✅ **最大化共享代码**：业务逻辑、ViewModel、Repository 放在 commonMain
- ✅ **使用 expect/actual**：平台特定功能用 expect/actual 抽象
- ✅ **资源共享**：图片、字符串、字体放在 composeResources
- ✅ **依赖注入**：使用 Koin 或手动 DI 管理依赖
- ✅ **渐进式采用**：可以先在新模块中使用 KMP
- ✅ **CI/CD**：为每个平台配置独立的构建流程

## 总结

Compose Multiplatform 的核心要点：

- **KMP + Compose**：共享 Kotlin 业务逻辑 + 共享 UI
- **expect/actual**：处理平台差异的标准方式
- **资源管理**：统一管理图片、字符串、字体
- **网络层**：Ktor 提供跨平台 HTTP 客户端
- **状态管理**：ViewModel 可以完全共享
- **导航**：Voyager 等库提供跨平台导航支持

Compose Multiplatform 正在快速成熟，是 Kotlin 开发者进行跨平台开发的绝佳选择。

---

*© 2024 Fidroid. [返回首页](../index.html)*

