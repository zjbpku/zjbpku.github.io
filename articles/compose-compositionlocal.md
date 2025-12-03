# CompositionLocal 深度解析：Compose 中的隐式依赖传递

> **发布日期**: 2024-05-10  
> **阅读时间**: 约 24 分钟  
> **标签**: CompositionLocal, 依赖注入, Theme, Provider

在 Compose 中，数据通常通过参数显式传递。但有些数据（如主题、语言环境、导航控制器）需要在整个组件树中共享，逐层传递会非常繁琐。`CompositionLocal` 提供了一种隐式传递数据的机制，让深层组件可以直接访问祖先提供的数据。

## 一、什么是 CompositionLocal？

`CompositionLocal` 是 Compose 中的**隐式数据传递机制**，类似于 React 的 Context 或依赖注入框架的作用域。

```kotlin
// 不使用 CompositionLocal：需要层层传递
@Composable
fun App(theme: AppTheme) {
    Screen(theme = theme)
}

@Composable
fun Screen(theme: AppTheme) {
    Card(theme = theme)
}

@Composable
fun Card(theme: AppTheme) {
    Text(color = theme.textColor, ...)  // 终于用到了！
}

// 使用 CompositionLocal：直接获取
@Composable
fun Card() {
    val theme = LocalAppTheme.current
    Text(color = theme.textColor, ...)
}
```

## 二、内置的 CompositionLocal

Compose 提供了许多内置的 CompositionLocal：

| CompositionLocal | 类型 | 用途 |
|-----------------|------|------|
| `LocalContext` | `Context` | Android Context |
| `LocalConfiguration` | `Configuration` | 设备配置（屏幕尺寸、语言等） |
| `LocalDensity` | `Density` | 像素密度，dp/px 转换 |
| `LocalLayoutDirection` | `LayoutDirection` | 布局方向（LTR/RTL） |
| `LocalLifecycleOwner` | `LifecycleOwner` | 生命周期所有者 |
| `LocalView` | `View` | 当前 ComposeView |
| `LocalFocusManager` | `FocusManager` | 焦点管理 |
| `LocalClipboardManager` | `ClipboardManager` | 剪贴板管理 |
| `LocalContentColor` | `Color` | 当前内容颜色 |
| `LocalTextStyle` | `TextStyle` | 当前文本样式 |

### 使用内置 CompositionLocal

```kotlin
@Composable
fun DeviceInfo() {
    val context = LocalContext.current
    val configuration = LocalConfiguration.current
    val density = LocalDensity.current

    Column {
        Text("屏幕宽度: ${configuration.screenWidthDp}dp")
        Text("像素密度: ${density.density}")
        
        // dp 转 px
        val paddingPx = with(density) { 16.dp.toPx() }
        Text("16dp = ${paddingPx}px")
    }
}
```

## 三、创建自定义 CompositionLocal

### staticCompositionLocalOf vs compositionLocalOf

Compose 提供两种创建方式：

```kotlin
// 1. staticCompositionLocalOf：值很少变化时使用
// 值变化时，整个使用该值的子树都会重组
val LocalAppConfig = staticCompositionLocalOf<AppConfig> {
    error("No AppConfig provided")
}

// 2. compositionLocalOf：值可能频繁变化时使用
// 值变化时，只有读取该值的 Composable 重组
val LocalUserPreferences = compositionLocalOf<UserPreferences> {
    UserPreferences()  // 默认值
}
```

> 💡 **如何选择？**  
> - `staticCompositionLocalOf`：主题、配置等很少变化的数据
> - `compositionLocalOf`：用户偏好、动态设置等可能变化的数据

### 完整示例：自定义主题系统

```kotlin
// 1. 定义数据类
@Immutable
data class CustomColors(
    val primary: Color,
    val secondary: Color,
    val background: Color,
    val surface: Color,
    val onPrimary: Color,
    val onBackground: Color
)

@Immutable
data class CustomTypography(
    val heading: TextStyle,
    val body: TextStyle,
    val caption: TextStyle
)

@Immutable
data class CustomSpacing(
    val small: Dp = 4.dp,
    val medium: Dp = 8.dp,
    val large: Dp = 16.dp,
    val extraLarge: Dp = 24.dp
)

// 2. 创建 CompositionLocal
val LocalCustomColors = staticCompositionLocalOf<CustomColors> {
    error("No CustomColors provided")
}

val LocalCustomTypography = staticCompositionLocalOf<CustomTypography> {
    error("No CustomTypography provided")
}

val LocalCustomSpacing = staticCompositionLocalOf {
    CustomSpacing()
}

// 3. 创建便捷访问对象
object CustomTheme {
    val colors: CustomColors
        @Composable
        @ReadOnlyComposable
        get() = LocalCustomColors.current

    val typography: CustomTypography
        @Composable
        @ReadOnlyComposable
        get() = LocalCustomTypography.current

    val spacing: CustomSpacing
        @Composable
        @ReadOnlyComposable
        get() = LocalCustomSpacing.current
}

// 4. 创建主题 Provider
@Composable
fun CustomTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colors = if (darkTheme) {
        CustomColors(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC6),
            background = Color(0xFF121212),
            surface = Color(0xFF1E1E1E),
            onPrimary = Color.White,
            onBackground = Color.White
        )
    } else {
        CustomColors(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC6),
            background = Color.White,
            surface = Color(0xFFF5F5F5),
            onPrimary = Color.White,
            onBackground = Color.Black
        )
    }

    val typography = CustomTypography(
        heading = TextStyle(
            fontSize = 24.sp,
            fontWeight = FontWeight.Bold
        ),
        body = TextStyle(
            fontSize = 16.sp,
            fontWeight = FontWeight.Normal
        ),
        caption = TextStyle(
            fontSize = 12.sp,
            fontWeight = FontWeight.Light
        )
    )

    CompositionLocalProvider(
        LocalCustomColors provides colors,
        LocalCustomTypography provides typography,
        LocalCustomSpacing provides CustomSpacing()
    ) {
        content()
    }
}

// 5. 使用自定义主题
@Composable
fun ThemedCard(title: String, description: String) {
    Card(
        backgroundColor = CustomTheme.colors.surface,
        modifier = Modifier.padding(CustomTheme.spacing.medium)
    ) {
        Column(modifier = Modifier.padding(CustomTheme.spacing.large)) {
            Text(
                text = title,
                style = CustomTheme.typography.heading,
                color = CustomTheme.colors.onBackground
            )
            Spacer(modifier = Modifier.height(CustomTheme.spacing.small))
            Text(
                text = description,
                style = CustomTheme.typography.body,
                color = CustomTheme.colors.onBackground.copy(alpha = 0.7f)
            )
        }
    }
}
```

## 四、CompositionLocalProvider 的使用

### 基本用法

```kotlin
@Composable
fun App() {
    // 提供单个值
    CompositionLocalProvider(LocalCustomColors provides darkColors) {
        MainContent()
    }

    // 提供多个值
    CompositionLocalProvider(
        LocalCustomColors provides darkColors,
        LocalCustomTypography provides typography,
        LocalContentColor provides Color.White
    ) {
        MainContent()
    }
}
```

### 嵌套覆盖

内层 Provider 可以覆盖外层的值：

```kotlin
@Composable
fun NestedProviders() {
    CompositionLocalProvider(LocalContentColor provides Color.Black) {
        Text("黑色文字")  // 黑色
        
        CompositionLocalProvider(LocalContentColor provides Color.Red) {
            Text("红色文字")  // 红色（覆盖外层）
        }
        
        Text("还是黑色")  // 黑色
    }
}
```

### 实战：局部深色模式

```kotlin
@Composable
fun DarkSection(content: @Composable () -> Unit) {
    val darkColors = CustomColors(
        primary = Color(0xFFBB86FC),
        background = Color(0xFF121212),
        surface = Color(0xFF1E1E1E),
        onBackground = Color.White,
        // ...
    )

    Surface(color = darkColors.background) {
        CompositionLocalProvider(LocalCustomColors provides darkColors) {
            content()
        }
    }
}

@Composable
fun MixedThemeScreen() {
    Column {
        // 正常主题区域
        Text("浅色模式内容")
        
        // 局部深色区域
        DarkSection {
            Text("深色模式内容")
        }
        
        // 回到正常主题
        Text("又是浅色模式")
    }
}
```

## 五、Material 3 中的 CompositionLocal

Material 3 大量使用 CompositionLocal 来传递主题数据：

```kotlin
// Material 3 主题源码简化版
@Composable
fun MaterialTheme(
    colorScheme: ColorScheme = MaterialTheme.colorScheme,
    typography: Typography = MaterialTheme.typography,
    shapes: Shapes = MaterialTheme.shapes,
    content: @Composable () -> Unit
) {
    CompositionLocalProvider(
        LocalColorScheme provides colorScheme,
        LocalTypography provides typography,
        LocalShapes provides shapes
    ) {
        content()
    }
}

// MaterialTheme 对象提供便捷访问
object MaterialTheme {
    val colorScheme: ColorScheme
        @Composable
        @ReadOnlyComposable
        get() = LocalColorScheme.current

    val typography: Typography
        @Composable
        @ReadOnlyComposable
        get() = LocalTypography.current

    val shapes: Shapes
        @Composable
        @ReadOnlyComposable
        get() = LocalShapes.current
}
```

### LocalContentColor 的妙用

`LocalContentColor` 是 Material 组件中广泛使用的模式：

```kotlin
@Composable
fun Surface(
    color: Color,
    contentColor: Color = contentColorFor(color),
    content: @Composable () -> Unit
) {
    CompositionLocalProvider(LocalContentColor provides contentColor) {
        Box(modifier = Modifier.background(color)) {
            content()
        }
    }
}

// Icon 和 Text 会自动使用 LocalContentColor
@Composable
fun MyButton() {
    Surface(
        color = MaterialTheme.colorScheme.primary,
        contentColor = MaterialTheme.colorScheme.onPrimary
    ) {
        Row {
            Icon(Icons.Default.Add, null)  // 自动使用 onPrimary 颜色
            Text("添加")                    // 自动使用 onPrimary 颜色
        }
    }
}
```

## 六、性能考量

### staticCompositionLocalOf 的重组范围

```kotlin
val LocalCounter = staticCompositionLocalOf { 0 }

@Composable
fun Parent() {
    var counter by remember { mutableStateOf(0) }

    CompositionLocalProvider(LocalCounter provides counter) {
        // counter 变化时，整个 Child 子树都会重组！
        Child()
    }

    Button(onClick = { counter++ }) {
        Text("增加")
    }
}

@Composable
fun Child() {
    // 即使不读取 LocalCounter，也会重组
    ExpensiveComponent()
    
    // 读取 LocalCounter 的组件
    Text("Counter: ${LocalCounter.current}")
}
```

### compositionLocalOf 的精确重组

```kotlin
val LocalCounter = compositionLocalOf { 0 }

@Composable
fun Parent() {
    var counter by remember { mutableStateOf(0) }

    CompositionLocalProvider(LocalCounter provides counter) {
        Child()
    }

    Button(onClick = { counter++ }) {
        Text("增加")
    }
}

@Composable
fun Child() {
    // 不读取 LocalCounter，不会重组
    ExpensiveComponent()
    
    // 只有这里重组
    CounterDisplay()
}

@Composable
fun CounterDisplay() {
    Text("Counter: ${LocalCounter.current}")  // 只有这个重组
}
```

### 最佳实践

```kotlin
// ✅ 好：用 staticCompositionLocalOf 传递不变的配置
val LocalAppConfig = staticCompositionLocalOf<AppConfig> {
    error("No config")
}

// ✅ 好：用 compositionLocalOf 传递可能变化的数据
val LocalUserState = compositionLocalOf<UserState> {
    UserState.Guest
}

// ❌ 避免：在 CompositionLocal 中传递频繁变化的数据
// 应该使用 State 或 Flow
val LocalScrollPosition = compositionLocalOf { 0f }  // 不推荐
```

## 七、常见使用场景

### 1. 导航控制器

```kotlin
val LocalNavController = staticCompositionLocalOf<NavHostController> {
    error("No NavController provided")
}

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    CompositionLocalProvider(LocalNavController provides navController) {
        NavHost(navController, startDestination = "home") {
            composable("home") { HomeScreen() }
            composable("detail/{id}") { DetailScreen() }
        }
    }
}

@Composable
fun DeepNestedButton() {
    val navController = LocalNavController.current
    
    Button(onClick = { navController.navigate("detail/123") }) {
        Text("查看详情")
    }
}
```

### 2. 用户会话

```kotlin
@Immutable
data class UserSession(
    val userId: String?,
    val isLoggedIn: Boolean,
    val permissions: Set<Permission>
)

val LocalUserSession = compositionLocalOf {
    UserSession(null, false, emptySet())
}

@Composable
fun AuthenticatedApp(userSession: UserSession) {
    CompositionLocalProvider(LocalUserSession provides userSession) {
        AppContent()
    }
}

@Composable
fun AdminPanel() {
    val session = LocalUserSession.current
    
    if (Permission.ADMIN in session.permissions) {
        // 显示管理面板
    } else {
        Text("无权限访问")
    }
}
```

### 3. 功能开关

```kotlin
@Immutable
data class FeatureFlags(
    val newHomeEnabled: Boolean = false,
    val darkModeEnabled: Boolean = true,
    val analyticsEnabled: Boolean = true
)

val LocalFeatureFlags = staticCompositionLocalOf { FeatureFlags() }

@Composable
fun HomeScreen() {
    val flags = LocalFeatureFlags.current

    if (flags.newHomeEnabled) {
        NewHomeScreen()
    } else {
        LegacyHomeScreen()
    }
}
```

## 八、CompositionLocal vs 其他方案

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **参数传递** | 少量层级、明确依赖 | 显式、易追踪 | 层级多时繁琐 |
| **CompositionLocal** | 跨多层级的共享数据 | 简洁、自动传递 | 隐式依赖、难追踪 |
| **ViewModel** | 屏幕级状态、业务逻辑 | 生命周期感知 | 需要 Hilt 等 DI |
| **全局单例** | 真正的全局状态 | 简单直接 | 难测试、难管理 |

## 九、常见陷阱

### 1. 忘记提供默认值或 Provider

```kotlin
// ❌ 没有默认值，也没有 Provider，运行时崩溃
val LocalData = staticCompositionLocalOf<Data> {
    error("No Data provided")
}

@Composable
fun Child() {
    val data = LocalData.current  // 💥 崩溃！
}

// ✅ 确保有 Provider
@Composable
fun App() {
    CompositionLocalProvider(LocalData provides Data()) {
        Child()
    }
}
```

### 2. 在 CompositionLocal 中存储可变对象

```kotlin
// ❌ 可变对象，变化不会触发重组
val LocalMutableList = compositionLocalOf { mutableListOf<String>() }

// ✅ 使用不可变对象
val LocalImmutableList = compositionLocalOf { listOf<String>() }
```

### 3. 过度使用 CompositionLocal

```kotlin
// ❌ 所有数据都用 CompositionLocal
val LocalUserName = compositionLocalOf { "" }
val LocalUserAge = compositionLocalOf { 0 }
val LocalUserEmail = compositionLocalOf { "" }

// ✅ 组合成一个数据类
@Immutable
data class User(val name: String, val age: Int, val email: String)
val LocalUser = compositionLocalOf<User?> { null }
```

## 总结

CompositionLocal 的核心要点：

- **用途**：跨组件树传递共享数据，避免层层传参
- **staticCompositionLocalOf**：值很少变化，如主题、配置
- **compositionLocalOf**：值可能变化，需要精确重组控制
- **CompositionLocalProvider**：提供值，支持嵌套覆盖
- **性能**：注意选择合适的类型，避免不必要的重组
- **适度使用**：只用于真正需要跨层级共享的数据

正确使用 CompositionLocal，可以让你的代码更简洁，同时保持良好的可维护性。

---

*© 2024 Fidroid. [返回首页](../index.html)*

