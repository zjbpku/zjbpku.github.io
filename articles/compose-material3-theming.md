# Material 3 与 Compose 主题系统：打造品牌一致的 UI

> **发布日期**: 2024-04-22  
> **阅读时间**: 约 20 分钟  
> **标签**: Material 3, Dynamic Color, Typography, Dark Mode

Material 3（Material You）是 Google 最新的设计系统，与 Jetpack Compose 深度集成。本文将带你掌握如何构建一个灵活、可扩展的主题系统。

## 一、Material 3 基础配置

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.compose.material3:material3:1.2.0")
}
```

```kotlin
@Composable
fun MyApp() {
    MaterialTheme(
        colorScheme = lightColorScheme(),
        typography = Typography,
        shapes = Shapes
    ) {
        // App content
    }
}
```

## 二、颜色系统

Material 3 使用基于角色的颜色系统，包含 primary、secondary、tertiary 等色彩角色：

```kotlin
private val LightColorScheme = lightColorScheme(
    primary = Color(0xFF6750A4),
    onPrimary = Color.White,
    primaryContainer = Color(0xFFEADDFF),
    onPrimaryContainer = Color(0xFF21005D),

    secondary = Color(0xFF625B71),
    onSecondary = Color.White,
    secondaryContainer = Color(0xFFE8DEF8),
    onSecondaryContainer = Color(0xFF1D192B),

    tertiary = Color(0xFF7D5260),
    surface = Color(0xFFFFFBFE),
    background = Color(0xFFFFFBFE),
    error = Color(0xFFB3261E)
)

private val DarkColorScheme = darkColorScheme(
    primary = Color(0xFFD0BCFF),
    onPrimary = Color(0xFF381E72),
    primaryContainer = Color(0xFF4F378B),
    onPrimaryContainer = Color(0xFFEADDFF),
    // ... 其他颜色
)
```

## 三、动态颜色（Dynamic Color）

Android 12+ 支持从用户壁纸提取颜色：

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
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
        typography = AppTypography,
        content = content
    )
}
```

> 💡 **Material Theme Builder**  
> 使用 [Material Theme Builder](https://m3.material.io/theme-builder) 工具可以快速生成完整的颜色方案代码。

## 四、Typography 排版系统

```kotlin
val AppTypography = Typography(
    displayLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.W400,
        fontSize = 57.sp,
        lineHeight = 64.sp,
        letterSpacing = (-0.25).sp
    ),
    headlineLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.W400,
        fontSize = 32.sp,
        lineHeight = 40.sp
    ),
    titleLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.W400,
        fontSize = 22.sp,
        lineHeight = 28.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.W400,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.5.sp
    ),
    labelSmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.W500,
        fontSize = 11.sp,
        lineHeight = 16.sp,
        letterSpacing = 0.5.sp
    )
)
```

### 使用自定义字体

```kotlin
val CustomFontFamily = FontFamily(
    Font(R.font.custom_regular, FontWeight.Normal),
    Font(R.font.custom_medium, FontWeight.Medium),
    Font(R.font.custom_bold, FontWeight.Bold)
)

val AppTypography = Typography(
    bodyLarge = TextStyle(
        fontFamily = CustomFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp
    )
)
```

## 五、Shape 形状系统

```kotlin
val AppShapes = Shapes(
    extraSmall = RoundedCornerShape(4.dp),
    small = RoundedCornerShape(8.dp),
    medium = RoundedCornerShape(12.dp),
    large = RoundedCornerShape(16.dp),
    extraLarge = RoundedCornerShape(28.dp)
)
```

## 六、在组件中使用主题

```kotlin
@Composable
fun ThemedCard() {
    Card(
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surfaceVariant
        ),
        shape = MaterialTheme.shapes.medium
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = "标题",
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
            Text(
                text = "正文内容",
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

## 七、扩展主题：自定义属性

```kotlin
// 定义扩展颜色
data class ExtendedColors(
    val success: Color,
    val onSuccess: Color,
    val warning: Color,
    val onWarning: Color
)

val LocalExtendedColors = staticCompositionLocalOf {
    ExtendedColors(
        success = Color.Unspecified,
        onSuccess = Color.Unspecified,
        warning = Color.Unspecified,
        onWarning = Color.Unspecified
    )
}

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val extendedColors = if (darkTheme) {
        ExtendedColors(
            success = Color(0xFF4CAF50),
            onSuccess = Color.White,
            warning = Color(0xFFFF9800),
            onWarning = Color.Black
        )
    } else {
        ExtendedColors(
            success = Color(0xFF2E7D32),
            onSuccess = Color.White,
            warning = Color(0xFFF57C00),
            onWarning = Color.Black
        )
    }

    CompositionLocalProvider(
        LocalExtendedColors provides extendedColors
    ) {
        MaterialTheme(
            colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme,
            content = content
        )
    }
}

// 使用扩展颜色
val extendedColors: ExtendedColors
    @Composable
    get() = LocalExtendedColors.current

@Composable
fun SuccessBadge() {
    Surface(
        color = extendedColors.success,
        contentColor = extendedColors.onSuccess
    ) {
        Text("成功")
    }
}
```

## 八、深色模式最佳实践

- 使用 `surface` 而非纯黑背景
- 降低饱和度，避免刺眼
- 使用 elevation overlay 而非阴影
- 测试不同亮度下的可读性

```kotlin
// 根据 elevation 调整 surface 颜色
Surface(
    tonalElevation = 3.dp  // Material 3 会自动调整颜色
) {
    // content
}
```

## 九、主题预览

```kotlin
@Preview(name = "Light")
@Preview(name = "Dark", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
fun CardPreview() {
    AppTheme {
        ThemedCard()
    }
}
```

## 总结

Material 3 主题系统的核心要点：

- **ColorScheme**：基于角色的颜色系统
- **Dynamic Color**：从壁纸提取个性化颜色
- **Typography**：统一的排版规范
- **Shapes**：一致的圆角系统
- **CompositionLocal**：扩展自定义主题属性

构建好主题系统后，整个应用的 UI 一致性将大大提升，品牌识别度也会更强。

---

*© 2024 Fidroid. [返回首页](../index.html)*


