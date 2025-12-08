# Compose Canvas 自定义绘制：图形、路径与动画

> **发布日期**: 2024-05-12  
> **阅读时间**: 约 28 分钟  
> **标签**: Canvas, DrawScope, Path, 自定义绘制

当内置组件无法满足设计需求时，你需要使用 Canvas 进行自定义绘制。Compose 的 Canvas API 提供了强大而简洁的绘制能力，让你可以创建任意复杂的图形和动画效果。

## 一、Canvas 基础

### Canvas Composable

```kotlin
@Composable
fun SimpleCanvas() {
    Canvas(
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
    ) {
        // this: DrawScope
        // size: Size - 画布尺寸
        // center: Offset - 画布中心点
        
        drawCircle(
            color = Color.Blue,
            radius = 100f,
            center = center
        )
    }
}
```

### DrawScope 的核心属性

```kotlin
Canvas(modifier = Modifier.size(300.dp)) {
    // 画布尺寸
    val width = size.width
    val height = size.height
    
    // 画布中心
    val centerX = center.x
    val centerY = center.y
    
    // 像素密度（用于 dp 转换）
    val density = this.density
    
    // 布局方向
    val layoutDirection = this.layoutDirection
}
```

## 二、基础图形绘制

### 绘制矩形

```kotlin
Canvas(modifier = Modifier.size(300.dp)) {
    // 填充矩形
    drawRect(
        color = Color.Blue,
        topLeft = Offset(20f, 20f),
        size = Size(200f, 100f)
    )
    
    // 描边矩形
    drawRect(
        color = Color.Red,
        topLeft = Offset(20f, 140f),
        size = Size(200f, 100f),
        style = Stroke(width = 4f)
    )
    
    // 圆角矩形
    drawRoundRect(
        color = Color.Green,
        topLeft = Offset(240f, 20f),
        size = Size(200f, 100f),
        cornerRadius = CornerRadius(16f, 16f)
    )
}
```

### 绘制圆形和椭圆

```kotlin
Canvas(modifier = Modifier.size(300.dp)) {
    // 圆形
    drawCircle(
        color = Color.Blue,
        radius = 80f,
        center = Offset(100f, 100f)
    )
    
    // 椭圆
    drawOval(
        color = Color.Red,
        topLeft = Offset(200f, 50f),
        size = Size(150f, 100f)
    )
    
    // 圆环
    drawCircle(
        color = Color.Green,
        radius = 60f,
        center = Offset(100f, 250f),
        style = Stroke(width = 8f)
    )
}
```

### 绘制线条

```kotlin
Canvas(modifier = Modifier.size(300.dp)) {
    // 单条直线
    drawLine(
        color = Color.Blue,
        start = Offset(0f, 0f),
        end = Offset(size.width, size.height),
        strokeWidth = 4f
    )
    
    // 带端点样式的线条
    drawLine(
        color = Color.Red,
        start = Offset(0f, size.height),
        end = Offset(size.width, 0f),
        strokeWidth = 8f,
        cap = StrokeCap.Round  // Round, Butt, Square
    )
    
    // 多条连接的线（折线）
    val points = listOf(
        Offset(50f, 200f),
        Offset(100f, 100f),
        Offset(150f, 180f),
        Offset(200f, 80f),
        Offset(250f, 150f)
    )
    drawPoints(
        points = points,
        pointMode = PointMode.Polygon,  // Points, Lines, Polygon
        color = Color.Green,
        strokeWidth = 4f
    )
}
```

### 绘制弧形

```kotlin
Canvas(modifier = Modifier.size(300.dp)) {
    // 扇形（useCenter = true）
    drawArc(
        color = Color.Blue,
        startAngle = 0f,
        sweepAngle = 120f,
        useCenter = true,
        topLeft = Offset(20f, 20f),
        size = Size(150f, 150f)
    )
    
    // 弧线（useCenter = false）
    drawArc(
        color = Color.Red,
        startAngle = 0f,
        sweepAngle = 270f,
        useCenter = false,
        topLeft = Offset(200f, 20f),
        size = Size(150f, 150f),
        style = Stroke(width = 8f, cap = StrokeCap.Round)
    )
}
```

## 三、Path 路径绘制

Path 是自定义绘制的核心，可以创建任意复杂的形状。

### 基本 Path 操作

```kotlin
Canvas(modifier = Modifier.size(300.dp)) {
    val path = Path().apply {
        // 移动到起点
        moveTo(50f, 200f)
        
        // 直线到下一点
        lineTo(150f, 50f)
        lineTo(250f, 200f)
        
        // 闭合路径
        close()
    }
    
    drawPath(
        path = path,
        color = Color.Blue
    )
}
```

### 贝塞尔曲线

```kotlin
Canvas(modifier = Modifier.size(300.dp)) {
    val path = Path().apply {
        moveTo(50f, 200f)
        
        // 二次贝塞尔曲线
        quadraticBezierTo(
            x1 = 150f, y1 = 0f,   // 控制点
            x2 = 250f, y2 = 200f  // 终点
        )
    }
    
    drawPath(path, Color.Blue, style = Stroke(4f))
    
    val cubicPath = Path().apply {
        moveTo(50f, 350f)
        
        // 三次贝塞尔曲线
        cubicTo(
            x1 = 100f, y1 = 200f,  // 控制点1
            x2 = 200f, y2 = 500f,  // 控制点2
            x3 = 250f, y3 = 350f   // 终点
        )
    }
    
    drawPath(cubicPath, Color.Red, style = Stroke(4f))
}
```

### 绘制心形

```kotlin
@Composable
fun HeartShape() {
    Canvas(modifier = Modifier.size(200.dp)) {
        val path = Path().apply {
            val width = size.width
            val height = size.height
            
            // 心形路径
            moveTo(width / 2, height * 0.25f)
            
            // 左半部分
            cubicTo(
                width * 0.1f, height * 0.1f,
                0f, height * 0.5f,
                width / 2, height * 0.9f
            )
            
            // 右半部分
            moveTo(width / 2, height * 0.25f)
            cubicTo(
                width * 0.9f, height * 0.1f,
                width, height * 0.5f,
                width / 2, height * 0.9f
            )
        }
        
        drawPath(
            path = path,
            color = Color.Red
        )
    }
}
```

## 四、渐变与画刷

### 线性渐变

```kotlin
Canvas(modifier = Modifier.size(300.dp)) {
    val gradient = Brush.linearGradient(
        colors = listOf(Color.Red, Color.Yellow, Color.Green),
        start = Offset.Zero,
        end = Offset(size.width, size.height)
    )
    
    drawRect(brush = gradient)
}
```

### 径向渐变

```kotlin
Canvas(modifier = Modifier.size(300.dp)) {
    val gradient = Brush.radialGradient(
        colors = listOf(Color.Yellow, Color.Red, Color.Transparent),
        center = center,
        radius = size.minDimension / 2
    )
    
    drawCircle(brush = gradient)
}
```

### 扫描渐变

```kotlin
Canvas(modifier = Modifier.size(300.dp)) {
    val gradient = Brush.sweepGradient(
        colors = listOf(
            Color.Red, Color.Yellow, Color.Green,
            Color.Cyan, Color.Blue, Color.Magenta, Color.Red
        ),
        center = center
    )
    
    drawCircle(brush = gradient)
}
```

## 五、实战：自定义进度条

### 环形进度条

```kotlin
@Composable
fun CircularProgressIndicator(
    progress: Float,  // 0f - 1f
    modifier: Modifier = Modifier,
    strokeWidth: Dp = 8.dp,
    trackColor: Color = Color.LightGray,
    progressColor: Color = Color.Blue
) {
    Canvas(modifier = modifier.size(100.dp)) {
        val stroke = strokeWidth.toPx()
        val radius = (size.minDimension - stroke) / 2
        
        // 绘制轨道
        drawCircle(
            color = trackColor,
            radius = radius,
            style = Stroke(width = stroke)
        )
        
        // 绘制进度
        drawArc(
            color = progressColor,
            startAngle = -90f,
            sweepAngle = 360f * progress,
            useCenter = false,
            topLeft = Offset(stroke / 2, stroke / 2),
            size = Size(size.width - stroke, size.height - stroke),
            style = Stroke(width = stroke, cap = StrokeCap.Round)
        )
    }
}

// 带动画的版本
@Composable
fun AnimatedCircularProgress(targetProgress: Float) {
    val animatedProgress by animateFloatAsState(
        targetValue = targetProgress,
        animationSpec = tween(durationMillis = 1000, easing = FastOutSlowInEasing),
        label = "progress"
    )
    
    CircularProgressIndicator(progress = animatedProgress)
}
```

### 线性进度条（带渐变）

```kotlin
@Composable
fun GradientProgressBar(
    progress: Float,
    modifier: Modifier = Modifier
) {
    Canvas(
        modifier = modifier
            .fillMaxWidth()
            .height(12.dp)
    ) {
        val cornerRadius = size.height / 2
        
        // 绘制轨道
        drawRoundRect(
            color = Color.LightGray.copy(alpha = 0.3f),
            cornerRadius = CornerRadius(cornerRadius)
        )
        
        // 绘制进度（带渐变）
        val progressWidth = size.width * progress
        if (progressWidth > 0) {
            drawRoundRect(
                brush = Brush.horizontalGradient(
                    colors = listOf(Color(0xFF6366F1), Color(0xFF8B5CF6), Color(0xFFEC4899))
                ),
                size = Size(progressWidth, size.height),
                cornerRadius = CornerRadius(cornerRadius)
            )
        }
    }
}
```

## 六、实战：折线图

```kotlin
@Composable
fun LineChart(
    data: List<Float>,
    modifier: Modifier = Modifier
) {
    Canvas(modifier = modifier.fillMaxSize()) {
        if (data.isEmpty()) return@Canvas
        
        val maxValue = data.maxOrNull() ?: 0f
        val minValue = data.minOrNull() ?: 0f
        val range = maxValue - minValue
        
        val spacing = size.width / (data.size - 1)
        val points = data.mapIndexed { index, value ->
            val x = index * spacing
            val y = size.height - ((value - minValue) / range * size.height)
            Offset(x, y)
        }
        
        // 绘制填充区域
        val fillPath = Path().apply {
            moveTo(0f, size.height)
            points.forEach { point ->
                lineTo(point.x, point.y)
            }
            lineTo(size.width, size.height)
            close()
        }
        
        drawPath(
            path = fillPath,
            brush = Brush.verticalGradient(
                colors = listOf(
                    Color(0xFF6366F1).copy(alpha = 0.3f),
                    Color.Transparent
                )
            )
        )
        
        // 绘制线条
        val linePath = Path().apply {
            points.forEachIndexed { index, point ->
                if (index == 0) moveTo(point.x, point.y)
                else lineTo(point.x, point.y)
            }
        }
        
        drawPath(
            path = linePath,
            color = Color(0xFF6366F1),
            style = Stroke(width = 3f, cap = StrokeCap.Round)
        )
        
        // 绘制数据点
        points.forEach { point ->
            drawCircle(
                color = Color.White,
                radius = 6f,
                center = point
            )
            drawCircle(
                color = Color(0xFF6366F1),
                radius = 4f,
                center = point
            )
        }
    }
}
```

## 七、Canvas 动画

### 使用 animateFloatAsState

```kotlin
@Composable
fun PulsatingCircle() {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")
    val scale by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 1.2f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )
    
    Canvas(modifier = Modifier.size(100.dp)) {
        drawCircle(
            color = Color.Blue.copy(alpha = 0.5f),
            radius = 40f * scale
        )
    }
}
```

### 旋转动画

```kotlin
@Composable
fun RotatingArc() {
    val infiniteTransition = rememberInfiniteTransition(label = "rotation")
    val rotation by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 360f,
        animationSpec = infiniteRepeatable(
            animation = tween(2000, easing = LinearEasing)
        ),
        label = "rotation"
    )
    
    Canvas(modifier = Modifier.size(100.dp)) {
        rotate(rotation) {
            drawArc(
                color = Color.Blue,
                startAngle = 0f,
                sweepAngle = 270f,
                useCenter = false,
                style = Stroke(width = 8f, cap = StrokeCap.Round)
            )
        }
    }
}
```

## 八、drawWithContent 与 drawBehind

### Modifier.drawBehind

在内容后面绘制（作为背景）：

```kotlin
@Composable
fun GlowingBox() {
    Box(
        modifier = Modifier
            .size(100.dp)
            .drawBehind {
                // 绘制发光效果
                drawCircle(
                    brush = Brush.radialGradient(
                        colors = listOf(
                            Color.Blue.copy(alpha = 0.5f),
                            Color.Transparent
                        )
                    ),
                    radius = size.maxDimension
                )
            }
            .background(Color.Blue, RoundedCornerShape(16.dp))
    )
}
```

### Modifier.drawWithContent

控制绘制顺序：

```kotlin
@Composable
fun WatermarkText(text: String) {
    Text(
        text = text,
        modifier = Modifier.drawWithContent {
            drawContent()  // 先绘制原内容
            
            // 在上面绘制水印
            rotate(degrees = -30f) {
                drawText(
                    textMeasurer = textMeasurer,
                    text = "WATERMARK",
                    style = TextStyle(
                        color = Color.Red.copy(alpha = 0.3f),
                        fontSize = 24.sp
                    )
                )
            }
        }
    )
}
```

## 九、性能优化

### 避免在 Canvas 中创建对象

```kotlin
// ❌ 每次重绘都创建新的 Path
Canvas(modifier = Modifier.size(100.dp)) {
    val path = Path().apply { ... }  // 每帧都创建
    drawPath(path, Color.Blue)
}

// ✅ 使用 remember 缓存 Path
@Composable
fun OptimizedCanvas() {
    val path = remember {
        Path().apply { ... }
    }
    
    Canvas(modifier = Modifier.size(100.dp)) {
        drawPath(path, Color.Blue)
    }
}
```

### 使用 drawWithCache

```kotlin
@Composable
fun CachedDrawing() {
    Box(
        modifier = Modifier
            .size(200.dp)
            .drawWithCache {
                // 这里的计算只在尺寸变化时执行
                val path = Path().apply {
                    // 复杂的路径计算
                }
                val brush = Brush.linearGradient(...)
                
                onDrawBehind {
                    // 这里每帧执行，但使用缓存的 path 和 brush
                    drawPath(path, brush)
                }
            }
    )
}
```

## 总结

Compose Canvas 绘制的核心要点：

- **Canvas Composable**：提供 DrawScope，是自定义绘制的入口
- **基础图形**：drawRect、drawCircle、drawLine、drawArc
- **Path**：创建任意复杂形状，支持贝塞尔曲线
- **Brush**：线性、径向、扫描渐变
- **动画**：结合 animate*AsState 创建动态效果
- **Modifier 绘制**：drawBehind、drawWithContent、drawWithCache
- **性能**：避免在 Canvas 中创建对象，使用缓存

掌握 Canvas API，你可以实现任何设计稿中的自定义图形和动画效果。

---

*© 2024 Fidroid. [返回首页](../index.html)*



