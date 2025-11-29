# Compose 动画完全指南：从入门到精通

> **发布日期**: 2024-03-28  
> **阅读时间**: 约 20 分钟  
> **标签**: Animation, AnimatedVisibility, Transition, 手势动画

动画是提升用户体验的关键要素。Jetpack Compose 提供了一套声明式、易于组合的动画 API，让开发者可以用更少的代码实现流畅、自然的动画效果。本文将系统介绍 Compose 动画的核心概念与实战技巧。

## 一、Compose 动画 API 概览

Compose 的动画系统分为多个层级，从高层封装到底层控制：

| API 层级 | 典型 API | 适用场景 |
|---------|---------|---------|
| **高层动画** | AnimatedVisibility, AnimatedContent, Crossfade | 元素出现/消失、内容切换 |
| **状态驱动动画** | animateXxxAsState (animateFloatAsState, animateColorAsState...) | 单个属性随状态变化 |
| **Transition** | updateTransition, rememberInfiniteTransition | 多属性协调动画、循环动画 |
| **底层控制** | Animatable, animate() | 手势驱动、精细控制 |

## 二、animateXxxAsState：最简单的动画入口

当你只需要让某个属性（如透明度、大小、颜色）随状态变化时平滑过渡，`animateXxxAsState` 是最简洁的选择。

```kotlin
@Composable
fun ExpandableCard(expanded: Boolean) {
    // 高度随 expanded 状态平滑变化
    val height by animateDpAsState(
        targetValue = if (expanded) 200.dp else 80.dp,
        animationSpec = tween(durationMillis = 300)
    )

    // 背景色随状态变化
    val backgroundColor by animateColorAsState(
        targetValue = if (expanded) Color.Blue else Color.Gray
    )

    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(height)
            .background(backgroundColor)
    )
}
```

> 💡 **animationSpec 参数**  
> 你可以通过 `animationSpec` 自定义动画曲线：`tween()` 指定时长和缓动函数，`spring()` 实现弹簧效果，`keyframes {}` 定义关键帧动画。

## 三、AnimatedVisibility：优雅处理元素出现与消失

传统 View 系统中，让元素带动画地出现/消失需要手动管理 alpha、scale 等属性。Compose 的 `AnimatedVisibility` 把这一切封装好了：

```kotlin
@Composable
fun NotificationBanner(visible: Boolean, message: String) {
    AnimatedVisibility(
        visible = visible,
        enter = fadeIn() + slideInVertically { -it },
        exit = fadeOut() + slideOutVertically { -it }
    ) {
        Surface(
            color = MaterialTheme.colorScheme.primaryContainer,
            modifier = Modifier.fillMaxWidth().padding(16.dp)
        ) {
            Text(message)
        }
    }
}
```

### 组合多种进入/退出效果

Compose 提供了多种 `EnterTransition` 和 `ExitTransition`，可以用 `+` 运算符组合：

- **fadeIn() / fadeOut()** - 透明度渐变
- **slideInHorizontally() / slideOutHorizontally()** - 水平滑动
- **slideInVertically() / slideOutVertically()** - 垂直滑动
- **scaleIn() / scaleOut()** - 缩放
- **expandIn() / shrinkOut()** - 尺寸展开/收缩

## 四、AnimatedContent：内容切换动画

当 UI 内容本身发生变化（而不仅仅是显示/隐藏）时，`AnimatedContent` 可以在新旧内容之间添加过渡动画：

```kotlin
@Composable
fun CounterDisplay(count: Int) {
    AnimatedContent(
        targetState = count,
        transitionSpec = {
            // 新内容从下方滑入，旧内容向上滑出
            if (targetState > initialState) {
                slideInVertically { it } + fadeIn() togetherWith
                    slideOutVertically { -it } + fadeOut()
            } else {
                slideInVertically { -it } + fadeIn() togetherWith
                    slideOutVertically { it } + fadeOut()
            }
        }
    ) { targetCount ->
        Text(
            text = "$targetCount",
            style = MaterialTheme.typography.displayLarge
        )
    }
}
```

## 五、updateTransition：多属性协调动画

当多个属性需要根据同一个状态协调变化时，`updateTransition` 比多个独立的 `animateXxxAsState` 更高效、更易维护：

```kotlin
enum class CardState { Collapsed, Expanded }

@Composable
fun AnimatedCard(state: CardState) {
    val transition = updateTransition(targetState = state, label = "card")

    val height by transition.animateDp(label = "height") { s ->
        when (s) {
            CardState.Collapsed -> 80.dp
            CardState.Expanded -> 200.dp
        }
    }

    val cornerRadius by transition.animateDp(label = "corner") { s ->
        when (s) {
            CardState.Collapsed -> 16.dp
            CardState.Expanded -> 8.dp
        }
    }

    val elevation by transition.animateDp(label = "elevation") { s ->
        when (s) {
            CardState.Collapsed -> 2.dp
            CardState.Expanded -> 8.dp
        }
    }

    Card(
        modifier = Modifier.fillMaxWidth().height(height),
        shape = RoundedCornerShape(cornerRadius),
        elevation = CardDefaults.cardElevation(defaultElevation = elevation)
    ) {
        // Card content...
    }
}
```

## 六、rememberInfiniteTransition：循环动画

需要持续循环的动画（如加载指示器、呼吸灯效果），使用 `rememberInfiniteTransition`：

```kotlin
@Composable
fun PulsingDot() {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")

    val scale by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 1.3f,
        animationSpec = infiniteRepeatable(
            animation = tween(600, easing = FastOutSlowInEasing),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )

    val alpha by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 0.5f,
        animationSpec = infiniteRepeatable(
            animation = tween(600),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )

    Box(
        modifier = Modifier
            .size(24.dp)
            .scale(scale)
            .alpha(alpha)
            .background(Color.Red, CircleShape)
    )
}
```

## 七、Animatable：手势驱动与精细控制

当你需要在协程中控制动画、响应手势拖拽或实现复杂的动画序列时，`Animatable` 提供了最大的灵活性：

```kotlin
@Composable
fun SwipeToRefresh(onRefresh: () -> Unit) {
    val offsetY = remember { Animatable(0f) }
    val coroutineScope = rememberCoroutineScope()

    Box(
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectVerticalDragGestures(
                    onDragEnd = {
                        coroutineScope.launch {
                            if (offsetY.value > 100f) {
                                onRefresh()
                            }
                            // 弹回原位
                            offsetY.animateTo(
                                targetValue = 0f,
                                animationSpec = spring(
                                    dampingRatio = Spring.DampingRatioMediumBouncy,
                                    stiffness = Spring.StiffnessLow
                                )
                            )
                        }
                    },
                    onVerticalDrag = { change, dragAmount ->
                        coroutineScope.launch {
                            val newValue = (offsetY.value + dragAmount).coerceAtLeast(0f)
                            offsetY.snapTo(newValue)
                        }
                    }
                )
            }
            .offset { IntOffset(0, offsetY.value.roundToInt()) }
    ) {
        // Content...
    }
}
```

## 八、动画性能优化建议

### 1. 优先使用 graphicsLayer

对于 alpha、scale、rotation、translation 等变换，使用 `Modifier.graphicsLayer` 可以避免触发 recomposition：

```kotlin
// ✅ 推荐：使用 graphicsLayer，只影响绘制阶段
Box(
    modifier = Modifier.graphicsLayer {
        alpha = animatedAlpha
        scaleX = animatedScale
        scaleY = animatedScale
    }
)

// ❌ 避免：直接使用 Modifier，会触发 recomposition
Box(
    modifier = Modifier
        .alpha(animatedAlpha)
        .scale(animatedScale)
)
```

### 2. 使用 label 参数

为动画添加 `label` 参数，方便在 Layout Inspector 和 Compose Tracing 中调试：

```kotlin
val alpha by animateFloatAsState(
    targetValue = if (visible) 1f else 0f,
    label = "cardAlpha"  // 便于调试追踪
)
```

### 3. 避免在动画中创建新对象

动画回调中避免创建新的 Offset、Size 等对象，使用 `remember` 缓存：

```kotlin
// ✅ 推荐
val offset = remember(animatedX, animatedY) {
    Offset(animatedX, animatedY)
}

// ❌ 避免：每帧创建新对象
val offset = Offset(animatedX, animatedY)
```

## 九、实战案例：卡片展开动画

综合运用上述技术，实现一个点击展开的卡片动画：

```kotlin
@Composable
fun ExpandableNewsCard(
    title: String,
    summary: String,
    content: String
) {
    var expanded by remember { mutableStateOf(false) }
    val transition = updateTransition(expanded, label = "expand")

    val cardPadding by transition.animateDp(label = "padding") {
        if (it) 24.dp else 16.dp
    }
    val cardElevation by transition.animateDp(label = "elevation") {
        if (it) 12.dp else 4.dp
    }
    val arrowRotation by transition.animateFloat(label = "arrow") {
        if (it) 180f else 0f
    }

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .clickable { expanded = !expanded },
        elevation = CardDefaults.cardElevation(cardElevation)
    ) {
        Column(modifier = Modifier.padding(cardPadding)) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                Text(title, style = MaterialTheme.typography.titleMedium)
                Spacer(Modifier.weight(1f))
                Icon(
                    Icons.Default.KeyboardArrowDown,
                    contentDescription = null,
                    modifier = Modifier.graphicsLayer {
                        rotationZ = arrowRotation
                    }
                )
            }

            Spacer(Modifier.height(8.dp))
            Text(summary, color = MaterialTheme.colorScheme.onSurfaceVariant)

            AnimatedVisibility(visible = expanded) {
                Column {
                    Spacer(Modifier.height(16.dp))
                    HorizontalDivider()
                    Spacer(Modifier.height(16.dp))
                    Text(content)
                }
            }
        }
    }
}
```

## 总结

Jetpack Compose 的动画系统设计得非常灵活：

- **简单场景**：用 `animateXxxAsState` 一行代码搞定
- **元素显隐**：用 `AnimatedVisibility` 自动处理进入/退出
- **内容切换**：用 `AnimatedContent` 实现平滑过渡
- **多属性协调**：用 `updateTransition` 统一管理
- **循环动画**：用 `rememberInfiniteTransition`
- **手势/复杂控制**：用 `Animatable` 在协程中精细操控

掌握这些 API 后，你可以为应用打造流畅、自然、令人愉悦的交互体验。下一步建议：结合 [Navigation Compose](compose-navigation-best-practices.md) 实现页面切换动画，让整个应用的动效体验更加统一。

---

*© 2024 Fidroid. [返回首页](../index.html)*

