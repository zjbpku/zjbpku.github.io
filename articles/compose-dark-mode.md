# Compose 深色模式完全指南：主题切换与动态适配

> **发布日期**: 2024-05-28  
> **阅读时间**: 约 24 分钟  
> **标签**: Dark Mode, Theme, Material You, Dynamic Colors

深色模式已成为现代应用的标配功能，不仅能减少眼睛疲劳、节省电量，还能提供更好的视觉体验。本文将深入讲解如何在 Compose 中实现完善的深色模式支持，包括主题切换、系统适配和 Material You 动态颜色。

## 一、深色模式基础

### 检测系统深色模式

```kotlin
@Composable
fun App() {
    // 检测系统是否处于深色模式
    val isDarkTheme = isSystemInDarkTheme()
    
    MyAppTheme(darkTheme = isDarkTheme) {
        MainContent()
    }
}
```

### 基础主题实现

```kotlin
// Color.kt
val Purple80 = Color(0xFFD0BCFF)
val PurpleGrey80 = Color(0xFFCCC2DC)
val Pink80 = Color(0xFFEFB8C8)

val Purple40 = Color(0xFF6650a4)
val PurpleGrey40 = Color(0xFF625b71)
val Pink40 = Color(0xFF7D5260)

// 深色主题颜色
private val DarkColorScheme = darkColorScheme(
    primary = Purple80,
    secondary = PurpleGrey80,
    tertiary = Pink80,
    background = Color(0xFF1C1B1F),
    surface = Color(0xFF1C1B1F),
    onPrimary = Color.White,
    onSecondary = Color.White,
    onBackground = Color(0xFFE6E1E5),
    onSurface = Color(0xFFE6E1E5)
)

// 浅色主题颜色
private val LightColorScheme = lightColorScheme(
    primary = Purple40,
    secondary = PurpleGrey40,
    tertiary = Pink40,
    background = Color(0xFFFFFBFE),
    surface = Color(0xFFFFFBFE),
    onPrimary = Color.White,
    onSecondary = Color.White,
    onBackground = Color(0xFF1C1B1F),
    onSurface = Color(0xFF1C1B1F)
)

// Theme.kt
@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

## 二、用户可控的主题切换

### 主题状态管理

```kotlin
enum class ThemeMode {
    LIGHT,
    DARK,
    SYSTEM
}

class ThemeViewModel : ViewModel() {
    
    private val _themeMode = MutableStateFlow(ThemeMode.SYSTEM)
    val themeMode: StateFlow<ThemeMode> = _themeMode.asStateFlow()
    
    fun setThemeMode(mode: ThemeMode) {
        _themeMode.value = mode
        // 持久化存储
        saveThemePreference(mode)
    }
    
    private fun saveThemePreference(mode: ThemeMode) {
        viewModelScope.launch {
            // 使用 DataStore 保存
        }
    }
}
```

### 应用主题设置

```kotlin
@Composable
fun App(viewModel: ThemeViewModel = viewModel()) {
    val themeMode by viewModel.themeMode.collectAsState()
    val systemDarkTheme = isSystemInDarkTheme()
    
    val isDarkTheme = when (themeMode) {
        ThemeMode.LIGHT -> false
        ThemeMode.DARK -> true
        ThemeMode.SYSTEM -> systemDarkTheme
    }
    
    MyAppTheme(darkTheme = isDarkTheme) {
        MainContent()
    }
}
```

### 主题选择器 UI

```kotlin
@Composable
fun ThemeSelector(
    currentMode: ThemeMode,
    onModeSelected: (ThemeMode) -> Unit
) {
    Column {
        Text(
            text = "外观设置",
            style = MaterialTheme.typography.titleMedium,
            modifier = Modifier.padding(16.dp)
        )
        
        ThemeOption(
            title = "浅色模式",
            icon = Icons.Outlined.LightMode,
            selected = currentMode == ThemeMode.LIGHT,
            onClick = { onModeSelected(ThemeMode.LIGHT) }
        )
        
        ThemeOption(
            title = "深色模式",
            icon = Icons.Outlined.DarkMode,
            selected = currentMode == ThemeMode.DARK,
            onClick = { onModeSelected(ThemeMode.DARK) }
        )
        
        ThemeOption(
            title = "跟随系统",
            icon = Icons.Outlined.Brightness6,
            selected = currentMode == ThemeMode.SYSTEM,
            onClick = { onModeSelected(ThemeMode.SYSTEM) }
        )
    }
}

@Composable
fun ThemeOption(
    title: String,
    icon: ImageVector,
    selected: Boolean,
    onClick: () -> Unit
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .clickable(onClick = onClick)
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Icon(
            imageVector = icon,
            contentDescription = null,
            tint = if (selected) MaterialTheme.colorScheme.primary 
                   else MaterialTheme.colorScheme.onSurface
        )
        Spacer(modifier = Modifier.width(16.dp))
        Text(
            text = title,
            style = MaterialTheme.typography.bodyLarge,
            modifier = Modifier.weight(1f)
        )
        if (selected) {
            Icon(
                imageVector = Icons.Default.Check,
                contentDescription = "已选择",
                tint = MaterialTheme.colorScheme.primary
            )
        }
    }
}
```

## 三、使用 DataStore 持久化主题设置

```kotlin
// ThemePreferences.kt
class ThemePreferences(private val dataStore: DataStore<Preferences>) {
    
    companion object {
        private val THEME_MODE_KEY = stringPreferencesKey("theme_mode")
    }
    
    val themeMode: Flow<ThemeMode> = dataStore.data
        .map { preferences ->
            val modeString = preferences[THEME_MODE_KEY] ?: ThemeMode.SYSTEM.name
            ThemeMode.valueOf(modeString)
        }
    
    suspend fun setThemeMode(mode: ThemeMode) {
        dataStore.edit { preferences ->
            preferences[THEME_MODE_KEY] = mode.name
        }
    }
}

// ViewModel 中使用
class ThemeViewModel(
    private val themePreferences: ThemePreferences
) : ViewModel() {
    
    val themeMode: StateFlow<ThemeMode> = themePreferences.themeMode
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = ThemeMode.SYSTEM
        )
    
    fun setThemeMode(mode: ThemeMode) {
        viewModelScope.launch {
            themePreferences.setThemeMode(mode)
        }
    }
}
```

## 四、Material You 动态颜色

Android 12+ 支持从壁纸提取颜色，创建个性化主题。

### 启用动态颜色

```kotlin
@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,  // 是否启用动态颜色
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        // Android 12+ 且启用动态颜色
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) 
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

### 动态颜色设置选项

```kotlin
@Composable
fun AppearanceSettings(
    themeMode: ThemeMode,
    dynamicColorEnabled: Boolean,
    onThemeModeChange: (ThemeMode) -> Unit,
    onDynamicColorChange: (Boolean) -> Unit
) {
    Column {
        // 主题模式选择
        ThemeSelector(
            currentMode = themeMode,
            onModeSelected = onThemeModeChange
        )
        
        HorizontalDivider()
        
        // 动态颜色开关（仅 Android 12+）
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Column(modifier = Modifier.weight(1f)) {
                    Text(
                        text = "动态颜色",
                        style = MaterialTheme.typography.bodyLarge
                    )
                    Text(
                        text = "使用壁纸颜色作为主题色",
                        style = MaterialTheme.typography.bodySmall,
                        color = MaterialTheme.colorScheme.onSurfaceVariant
                    )
                }
                Switch(
                    checked = dynamicColorEnabled,
                    onCheckedChange = onDynamicColorChange
                )
            }
        }
    }
}
```

## 五、颜色适配最佳实践

### 使用语义化颜色

```kotlin
// ✅ 好：使用 MaterialTheme 颜色
@Composable
fun GoodCard() {
    Card(
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surface,
            contentColor = MaterialTheme.colorScheme.onSurface
        )
    ) {
        Text(
            text = "内容",
            color = MaterialTheme.colorScheme.onSurface
        )
    }
}

// ❌ 避免：硬编码颜色
@Composable
fun BadCard() {
    Card(
        colors = CardDefaults.cardColors(
            containerColor = Color.White  // 深色模式下会很刺眼
        )
    ) {
        Text(
            text = "内容",
            color = Color.Black  // 深色模式下不可见
        )
    }
}
```

### 自定义语义化颜色

```kotlin
// 定义自定义颜色
@Immutable
data class CustomColors(
    val success: Color,
    val warning: Color,
    val error: Color,
    val onSuccess: Color,
    val onWarning: Color,
    val onError: Color
)

val LocalCustomColors = staticCompositionLocalOf {
    CustomColors(
        success = Color.Unspecified,
        warning = Color.Unspecified,
        error = Color.Unspecified,
        onSuccess = Color.Unspecified,
        onWarning = Color.Unspecified,
        onError = Color.Unspecified
    )
}

// 浅色和深色版本
val LightCustomColors = CustomColors(
    success = Color(0xFF4CAF50),
    warning = Color(0xFFFFC107),
    error = Color(0xFFF44336),
    onSuccess = Color.White,
    onWarning = Color.Black,
    onError = Color.White
)

val DarkCustomColors = CustomColors(
    success = Color(0xFF81C784),
    warning = Color(0xFFFFD54F),
    error = Color(0xFFE57373),
    onSuccess = Color.Black,
    onWarning = Color.Black,
    onError = Color.Black
)

// 在 Theme 中提供
@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val customColors = if (darkTheme) DarkCustomColors else LightCustomColors
    
    CompositionLocalProvider(LocalCustomColors provides customColors) {
        MaterialTheme(
            colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme,
            content = content
        )
    }
}

// 便捷访问
object CustomTheme {
    val colors: CustomColors
        @Composable
        @ReadOnlyComposable
        get() = LocalCustomColors.current
}

// 使用
@Composable
fun SuccessBadge() {
    Box(
        modifier = Modifier
            .background(CustomTheme.colors.success)
            .padding(8.dp)
    ) {
        Text(
            text = "成功",
            color = CustomTheme.colors.onSuccess
        )
    }
}
```

## 六、图片和图标适配

### 为深色模式提供不同资源

```kotlin
@Composable
fun AdaptiveLogo() {
    val isDark = isSystemInDarkTheme()
    
    Image(
        painter = painterResource(
            id = if (isDark) R.drawable.logo_dark else R.drawable.logo_light
        ),
        contentDescription = "Logo"
    )
}
```

### 使用 tint 适配图标

```kotlin
@Composable
fun AdaptiveIcon() {
    Icon(
        imageVector = Icons.Default.Settings,
        contentDescription = "设置",
        // 自动使用当前主题的 onSurface 颜色
        tint = LocalContentColor.current
    )
}
```

### 图片色调适配

```kotlin
@Composable
fun AdaptiveImage(
    painter: Painter,
    contentDescription: String?
) {
    val isDark = isSystemInDarkTheme()
    
    Image(
        painter = painter,
        contentDescription = contentDescription,
        colorFilter = if (isDark) {
            // 深色模式下降低亮度
            ColorFilter.colorMatrix(
                ColorMatrix().apply {
                    setScale(0.9f, 0.9f, 0.9f, 1f)
                }
            )
        } else null
    )
}
```

## 七、状态栏和导航栏适配

```kotlin
@Composable
fun App() {
    val isDarkTheme = isSystemInDarkTheme()
    
    MyAppTheme(darkTheme = isDarkTheme) {
        val systemUiController = rememberSystemUiController()
        val useDarkIcons = !isDarkTheme
        
        DisposableEffect(systemUiController, useDarkIcons) {
            systemUiController.setSystemBarsColor(
                color = Color.Transparent,
                darkIcons = useDarkIcons
            )
            onDispose {}
        }
        
        MainContent()
    }
}

// 使用 accompanist-systemuicontroller
// implementation("com.google.accompanist:accompanist-systemuicontroller:0.32.0")
```

### Edge-to-Edge 支持

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 启用 Edge-to-Edge
        enableEdgeToEdge()
        
        setContent {
            MyAppTheme {
                Scaffold(
                    modifier = Modifier.fillMaxSize()
                ) { innerPadding ->
                    MainContent(
                        modifier = Modifier.padding(innerPadding)
                    )
                }
            }
        }
    }
}
```

## 八、主题切换动画

```kotlin
@Composable
fun AnimatedThemeSwitch() {
    var isDark by remember { mutableStateOf(false) }
    
    val animatedColorScheme by animateColorAsState(
        targetValue = if (isDark) DarkColorScheme.background 
                      else LightColorScheme.background,
        animationSpec = tween(durationMillis = 300),
        label = "theme"
    )
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(animatedColorScheme)
    ) {
        // 内容
        
        IconButton(
            onClick = { isDark = !isDark },
            modifier = Modifier.align(Alignment.TopEnd)
        ) {
            Icon(
                imageVector = if (isDark) Icons.Default.LightMode 
                              else Icons.Default.DarkMode,
                contentDescription = "切换主题"
            )
        }
    }
}
```

## 九、测试深色模式

### UI 测试

```kotlin
@Test
fun darkModeRendersCorrectly() {
    composeTestRule.setContent {
        MyAppTheme(darkTheme = true) {
            MyScreen()
        }
    }
    
    // 验证深色模式下的 UI
    composeTestRule
        .onNodeWithText("标题")
        .assertExists()
}

@Test
fun lightModeRendersCorrectly() {
    composeTestRule.setContent {
        MyAppTheme(darkTheme = false) {
            MyScreen()
        }
    }
    
    // 验证浅色模式下的 UI
    composeTestRule
        .onNodeWithText("标题")
        .assertExists()
}
```

### Preview 预览

```kotlin
@Preview(name = "Light Mode")
@Preview(name = "Dark Mode", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
fun ScreenPreview() {
    MyAppTheme {
        MyScreen()
    }
}
```

## 十、最佳实践清单

- ✅ 使用 `MaterialTheme.colorScheme` 而非硬编码颜色
- ✅ 提供浅色/深色/跟随系统三种选项
- ✅ 使用 DataStore 持久化用户偏好
- ✅ Android 12+ 支持 Material You 动态颜色
- ✅ 为深色模式适配图片和图标
- ✅ 正确处理状态栏和导航栏颜色
- ✅ 使用动画让主题切换更平滑
- ✅ 在 Preview 中同时预览两种主题
- ✅ 为自定义颜色创建语义化扩展

## 总结

Compose 深色模式的核心要点：

- **isSystemInDarkTheme()**：检测系统深色模式
- **ColorScheme**：Material3 的颜色方案
- **dynamicColorScheme**：Android 12+ 动态颜色
- **DataStore**：持久化主题偏好
- **语义化颜色**：避免硬编码，使用 MaterialTheme
- **资源适配**：为不同主题提供对应资源
- **系统栏适配**：状态栏和导航栏颜色

正确实现深色模式可以显著提升用户体验和应用品质。

---

*© 2024 Fidroid. [返回首页](../index.html)*

