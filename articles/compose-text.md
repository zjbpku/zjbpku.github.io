# Compose Text 高级用法：富文本、样式与交互

> **发布日期**: 2024-05-18  
> **阅读时间**: 约 24 分钟  
> **标签**: Text, AnnotatedString, 富文本, Typography

Text 是 UI 中最基础也最重要的组件。Compose 提供了强大的文本处理能力，从简单的文字显示到复杂的富文本、可点击文本、文本选择等。本文将深入讲解 Compose 的高级文本处理技术。

## 一、Text 基础

### 基本用法

```kotlin
Text(
    text = "Hello, Compose!",
    color = Color.Blue,
    fontSize = 20.sp,
    fontWeight = FontWeight.Bold,
    fontStyle = FontStyle.Italic,
    fontFamily = FontFamily.Serif,
    letterSpacing = 2.sp,
    textDecoration = TextDecoration.Underline,
    textAlign = TextAlign.Center,
    lineHeight = 28.sp,
    overflow = TextOverflow.Ellipsis,
    maxLines = 2,
    modifier = Modifier.fillMaxWidth()
)
```

### TextStyle

```kotlin
val customStyle = TextStyle(
    color = Color.DarkGray,
    fontSize = 16.sp,
    fontWeight = FontWeight.Medium,
    fontFamily = FontFamily.SansSerif,
    letterSpacing = 0.5.sp,
    lineHeight = 24.sp
)

Text(
    text = "使用 TextStyle",
    style = customStyle
)

// 使用 Material Theme 预设样式
Text(
    text = "标题",
    style = MaterialTheme.typography.headlineMedium
)
```

## 二、AnnotatedString：富文本

### 基本构建

```kotlin
@Composable
fun RichText() {
    val annotatedString = buildAnnotatedString {
        append("这是")
        
        withStyle(style = SpanStyle(color = Color.Red, fontWeight = FontWeight.Bold)) {
            append("红色粗体")
        }
        
        append("文字，这是")
        
        withStyle(style = SpanStyle(
            fontSize = 20.sp,
            textDecoration = TextDecoration.Underline
        )) {
            append("大号下划线")
        }
        
        append("文字。")
    }
    
    Text(text = annotatedString)
}
```

### 段落样式

```kotlin
@Composable
fun ParagraphStyleExample() {
    val annotatedString = buildAnnotatedString {
        withStyle(style = ParagraphStyle(textAlign = TextAlign.Center)) {
            append("居中标题\n")
        }
        
        withStyle(style = ParagraphStyle(
            textIndent = TextIndent(firstLine = 32.sp),
            lineHeight = 24.sp
        )) {
            append("这是一个有首行缩进的段落。这段文字会有首行缩进效果，模拟传统书籍的排版方式。")
        }
    }
    
    Text(text = annotatedString)
}
```

### 多种样式组合

```kotlin
@Composable
fun CodeSnippetText() {
    val annotatedString = buildAnnotatedString {
        append("使用 ")
        
        withStyle(SpanStyle(
            fontFamily = FontFamily.Monospace,
            background = Color.LightGray.copy(alpha = 0.3f),
            color = Color(0xFFE91E63)
        )) {
            append("remember { mutableStateOf() }")
        }
        
        append(" 来创建状态。")
    }
    
    Text(text = annotatedString)
}
```

## 三、可点击文本

### ClickableText

```kotlin
@Composable
fun ClickableTextExample() {
    val annotatedString = buildAnnotatedString {
        append("请阅读我们的")
        
        pushStringAnnotation(tag = "URL", annotation = "https://example.com/terms")
        withStyle(SpanStyle(color = Color.Blue, textDecoration = TextDecoration.Underline)) {
            append("服务条款")
        }
        pop()
        
        append("和")
        
        pushStringAnnotation(tag = "URL", annotation = "https://example.com/privacy")
        withStyle(SpanStyle(color = Color.Blue, textDecoration = TextDecoration.Underline)) {
            append("隐私政策")
        }
        pop()
        
        append("。")
    }
    
    ClickableText(
        text = annotatedString,
        onClick = { offset ->
            annotatedString.getStringAnnotations(tag = "URL", start = offset, end = offset)
                .firstOrNull()?.let { annotation ->
                    // 处理链接点击
                    println("点击了: ${annotation.item}")
                }
        }
    )
}
```

### 带手势的富文本

```kotlin
@Composable
fun InteractiveText() {
    val context = LocalContext.current
    val uriHandler = LocalUriHandler.current
    
    val annotatedString = buildAnnotatedString {
        append("访问 ")
        
        pushStringAnnotation(tag = "URL", annotation = "https://developer.android.com")
        withStyle(SpanStyle(
            color = MaterialTheme.colorScheme.primary,
            textDecoration = TextDecoration.Underline
        )) {
            append("Android 开发者网站")
        }
        pop()
        
        append(" 了解更多。")
    }
    
    ClickableText(
        text = annotatedString,
        style = MaterialTheme.typography.bodyLarge,
        onClick = { offset ->
            annotatedString.getStringAnnotations("URL", offset, offset)
                .firstOrNull()?.let { 
                    uriHandler.openUri(it.item)
                }
        }
    )
}
```

## 四、文本选择

### SelectionContainer

```kotlin
@Composable
fun SelectableText() {
    SelectionContainer {
        Column {
            Text("这段文字可以选择和复制。")
            Text("这段也可以。")
            
            DisableSelection {
                Text("这段不能选择。")
            }
            
            Text("这段又可以了。")
        }
    }
}
```

### 自定义选择颜色

```kotlin
@Composable
fun CustomSelectionColors() {
    val customColors = TextSelectionColors(
        handleColor = Color.Magenta,
        backgroundColor = Color.Magenta.copy(alpha = 0.3f)
    )
    
    CompositionLocalProvider(LocalTextSelectionColors provides customColors) {
        SelectionContainer {
            Text("自定义选择颜色的文本")
        }
    }
}
```

## 五、TextField 高级用法

### BasicTextField

```kotlin
@Composable
fun CustomTextField() {
    var text by remember { mutableStateOf("") }
    
    BasicTextField(
        value = text,
        onValueChange = { text = it },
        modifier = Modifier
            .fillMaxWidth()
            .background(Color.LightGray.copy(alpha = 0.2f), RoundedCornerShape(8.dp))
            .padding(16.dp),
        textStyle = TextStyle(
            fontSize = 16.sp,
            color = Color.Black
        ),
        cursorBrush = SolidColor(Color.Blue),
        decorationBox = { innerTextField ->
            Box {
                if (text.isEmpty()) {
                    Text(
                        text = "请输入...",
                        color = Color.Gray
                    )
                }
                innerTextField()
            }
        }
    )
}
```

### 带图标和清除按钮的输入框

```kotlin
@Composable
fun SearchTextField() {
    var text by remember { mutableStateOf("") }
    
    OutlinedTextField(
        value = text,
        onValueChange = { text = it },
        modifier = Modifier.fillMaxWidth(),
        placeholder = { Text("搜索...") },
        leadingIcon = {
            Icon(Icons.Default.Search, contentDescription = "搜索")
        },
        trailingIcon = {
            if (text.isNotEmpty()) {
                IconButton(onClick = { text = "" }) {
                    Icon(Icons.Default.Clear, contentDescription = "清除")
                }
            }
        },
        singleLine = true,
        shape = RoundedCornerShape(24.dp)
    )
}
```

### 密码输入框

```kotlin
@Composable
fun PasswordTextField() {
    var password by remember { mutableStateOf("") }
    var passwordVisible by remember { mutableStateOf(false) }
    
    OutlinedTextField(
        value = password,
        onValueChange = { password = it },
        label = { Text("密码") },
        visualTransformation = if (passwordVisible) 
            VisualTransformation.None 
        else 
            PasswordVisualTransformation(),
        trailingIcon = {
            IconButton(onClick = { passwordVisible = !passwordVisible }) {
                Icon(
                    imageVector = if (passwordVisible) 
                        Icons.Default.Visibility 
                    else 
                        Icons.Default.VisibilityOff,
                    contentDescription = if (passwordVisible) "隐藏密码" else "显示密码"
                )
            }
        },
        keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password)
    )
}
```

## 六、自定义字体

### 加载字体

```kotlin
// 从资源加载
val customFontFamily = FontFamily(
    Font(R.font.my_font_regular, FontWeight.Normal),
    Font(R.font.my_font_bold, FontWeight.Bold),
    Font(R.font.my_font_italic, FontWeight.Normal, FontStyle.Italic)
)

@Composable
fun CustomFontText() {
    Text(
        text = "自定义字体",
        fontFamily = customFontFamily,
        fontWeight = FontWeight.Bold
    )
}
```

### 可下载字体

```kotlin
val provider = GoogleFont.Provider(
    providerAuthority = "com.google.android.gms.fonts",
    providerPackage = "com.google.android.gms",
    certificates = R.array.com_google_android_gms_fonts_certs
)

val fontName = GoogleFont("Noto Sans SC")

val fontFamily = FontFamily(
    Font(googleFont = fontName, fontProvider = provider)
)
```

## 七、文本测量

### 使用 TextMeasurer

```kotlin
@Composable
fun MeasuredText() {
    val textMeasurer = rememberTextMeasurer()
    
    Canvas(modifier = Modifier.size(200.dp)) {
        val textLayoutResult = textMeasurer.measure(
            text = "Hello",
            style = TextStyle(fontSize = 20.sp)
        )
        
        // 获取文本尺寸
        val textWidth = textLayoutResult.size.width
        val textHeight = textLayoutResult.size.height
        
        // 居中绘制
        drawText(
            textLayoutResult = textLayoutResult,
            topLeft = Offset(
                (size.width - textWidth) / 2,
                (size.height - textHeight) / 2
            )
        )
    }
}
```

### 自动调整字体大小

```kotlin
@Composable
fun AutoSizeText(
    text: String,
    modifier: Modifier = Modifier,
    maxFontSize: TextUnit = 24.sp,
    minFontSize: TextUnit = 12.sp
) {
    var fontSize by remember { mutableStateOf(maxFontSize) }
    var readyToDraw by remember { mutableStateOf(false) }
    
    Text(
        text = text,
        fontSize = fontSize,
        maxLines = 1,
        overflow = TextOverflow.Ellipsis,
        modifier = modifier.drawWithContent {
            if (readyToDraw) drawContent()
        },
        onTextLayout = { textLayoutResult ->
            if (textLayoutResult.didOverflowWidth && fontSize > minFontSize) {
                fontSize = (fontSize.value - 1).sp
            } else {
                readyToDraw = true
            }
        }
    )
}
```

## 八、文本动画

### 逐字显示

```kotlin
@Composable
fun TypewriterText(text: String) {
    var visibleCharCount by remember { mutableIntStateOf(0) }
    
    LaunchedEffect(text) {
        visibleCharCount = 0
        for (i in text.indices) {
            delay(50)
            visibleCharCount = i + 1
        }
    }
    
    Text(text = text.take(visibleCharCount))
}
```

### 数字滚动

```kotlin
@Composable
fun AnimatedCounter(count: Int) {
    val animatedCount by animateIntAsState(
        targetValue = count,
        animationSpec = tween(durationMillis = 1000),
        label = "counter"
    )
    
    Text(
        text = animatedCount.toString(),
        style = MaterialTheme.typography.displayLarge
    )
}
```

## 九、HTML 文本

```kotlin
@Composable
fun HtmlText(html: String) {
    val spanned = remember(html) {
        HtmlCompat.fromHtml(html, HtmlCompat.FROM_HTML_MODE_LEGACY)
    }
    
    val annotatedString = remember(spanned) {
        buildAnnotatedString {
            append(spanned.toString())
            // 可以根据 spanned 的 spans 添加样式
        }
    }
    
    Text(text = annotatedString)
}
```

## 十、最佳实践

### 1. 使用 Material Typography

```kotlin
// 定义应用的 Typography
val AppTypography = Typography(
    displayLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 57.sp
    ),
    headlineMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.SemiBold,
        fontSize = 28.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp
    )
)

// 使用
Text(
    text = "标题",
    style = MaterialTheme.typography.headlineMedium
)
```

### 2. 复用 AnnotatedString

```kotlin
// 缓存复杂的 AnnotatedString
@Composable
fun CachedAnnotatedString(text: String) {
    val annotatedString = remember(text) {
        buildAnnotatedString {
            // 复杂的样式构建...
        }
    }
    
    Text(text = annotatedString)
}
```

### 3. 性能考虑

```kotlin
// ✅ 好：简单文本直接使用 Text
Text("简单文本")

// ✅ 好：需要样式时使用 AnnotatedString
Text(buildAnnotatedString { ... })

// ❌ 避免：不必要的 AnnotatedString
Text(buildAnnotatedString { append("简单文本") })
```

## 总结

Compose Text 的核心要点：

- **Text**：基础文本组件，支持丰富的样式参数
- **AnnotatedString**：构建富文本，支持多种样式
- **ClickableText**：可点击的文本区域
- **SelectionContainer**：启用文本选择
- **BasicTextField**：完全自定义的输入框
- **TextMeasurer**：测量和绘制文本
- **动画**：逐字显示、数字滚动等效果

掌握这些技术，你可以实现任何复杂的文本需求。

---

*© 2024 Fidroid. [返回首页](../index.html)*

