# Compose 动画进阶：Transition 与共享元素

> **发布日期**: 2024-06-27  
> **阅读时间**: 约 28 分钟  
> **标签**: Animation, Transition, SharedElement, AnimatedContent

Compose 提供了丰富的动画 API，从简单的属性动画到复杂的过渡效果。本文将深入讲解 Compose 动画的进阶用法，包括 Transition、AnimatedContent 和共享元素动画。

## 一、Transition API

### updateTransition

```kotlin
@Composable
fun ExpandableCard(expanded: Boolean) {
    val transition = updateTransition(
        targetState = expanded,
        label = "card_transition"
    )
    
    val cardHeight by transition.animateDp(
        label = "height",
        transitionSpec = { spring(stiffness = Spring.StiffnessLow) }
    ) { isExpanded ->
        if (isExpanded) 300.dp else 100.dp
    }
    
    val cardColor by transition.animateColor(
        label = "color"
    ) { isExpanded ->
        if (isExpanded) MaterialTheme.colorScheme.primaryContainer
        else MaterialTheme.colorScheme.surface
    }
    
    val cornerRadius by transition.animateDp(
        label = "corner"
    ) { isExpanded ->
        if (isExpanded) 24.dp else 12.dp
    }
    
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .height(cardHeight),
        shape = RoundedCornerShape(cornerRadius),
        colors = CardDefaults.cardColors(containerColor = cardColor)
    ) {
        // 内容
    }
}
```

### 多状态 Transition

```kotlin
enum class BoxState { Collapsed, Expanded, FullScreen }

@Composable
fun MultiStateBox(state: BoxState) {
    val transition = updateTransition(targetState = state, label = "box")
    
    val size by transition.animateDp(label = "size") { boxState ->
        when (boxState) {
            BoxState.Collapsed -> 50.dp
            BoxState.Expanded -> 150.dp
            BoxState.FullScreen -> 300.dp
        }
    }
    
    val color by transition.animateColor(label = "color") { boxState ->
        when (boxState) {
            BoxState.Collapsed -> Color.Blue
            BoxState.Expanded -> Color.Green
            BoxState.FullScreen -> Color.Red
        }
    }
    
    val corner by transition.animateDp(label = "corner") { boxState ->
        when (boxState) {
            BoxState.Collapsed -> 25.dp
            BoxState.Expanded -> 16.dp
            BoxState.FullScreen -> 0.dp
        }
    }
    
    Box(
        modifier = Modifier
            .size(size)
            .clip(RoundedCornerShape(corner))
            .background(color)
    )
}
```

## 二、AnimatedContent

### 基础用法

```kotlin
@Composable
fun CounterAnimation(count: Int) {
    AnimatedContent(
        targetState = count,
        transitionSpec = {
            if (targetState > initialState) {
                // 增加：从下往上滑入
                slideInVertically { height -> height } + fadeIn() togetherWith
                    slideOutVertically { height -> -height } + fadeOut()
            } else {
                // 减少：从上往下滑入
                slideInVertically { height -> -height } + fadeIn() togetherWith
                    slideOutVertically { height -> height } + fadeOut()
            }.using(SizeTransform(clip = false))
        },
        label = "counter"
    ) { targetCount ->
        Text(
            text = "$targetCount",
            style = MaterialTheme.typography.displayLarge
        )
    }
}
```

### 内容切换动画

```kotlin
@Composable
fun ContentSwitcher(
    showFirst: Boolean,
    firstContent: @Composable () -> Unit,
    secondContent: @Composable () -> Unit
) {
    AnimatedContent(
        targetState = showFirst,
        transitionSpec = {
            fadeIn(animationSpec = tween(300)) togetherWith
                fadeOut(animationSpec = tween(300))
        },
        label = "content_switcher"
    ) { showingFirst ->
        if (showingFirst) {
            firstContent()
        } else {
            secondContent()
        }
    }
}
```

### 带尺寸变化的切换

```kotlin
@Composable
fun ExpandableContent(expanded: Boolean) {
    AnimatedContent(
        targetState = expanded,
        transitionSpec = {
            fadeIn(animationSpec = tween(300)) togetherWith
                fadeOut(animationSpec = tween(300)) using
                SizeTransform { initialSize, targetSize ->
                    if (targetState) {
                        keyframes {
                            // 先横向展开，再纵向展开
                            IntSize(targetSize.width, initialSize.height) at 150
                            durationMillis = 300
                        }
                    } else {
                        keyframes {
                            // 先纵向收缩，再横向收缩
                            IntSize(initialSize.width, targetSize.height) at 150
                            durationMillis = 300
                        }
                    }
                }
        },
        label = "expandable"
    ) { isExpanded ->
        if (isExpanded) {
            ExpandedContent()
        } else {
            CollapsedContent()
        }
    }
}
```

## 三、共享元素动画（Navigation 2.8+）

### 基础共享元素

```kotlin
@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun SharedElementExample() {
    SharedTransitionLayout {
        val navController = rememberNavController()
        
        NavHost(navController, startDestination = "list") {
            composable("list") {
                SharedElementList(
                    onItemClick = { item ->
                        navController.navigate("detail/${item.id}")
                    },
                    animatedVisibilityScope = this@composable
                )
            }
            
            composable("detail/{id}") { backStackEntry ->
                val itemId = backStackEntry.arguments?.getString("id")
                SharedElementDetail(
                    itemId = itemId,
                    onBack = { navController.popBackStack() },
                    animatedVisibilityScope = this@composable
                )
            }
        }
    }
}

@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun SharedTransitionScope.SharedElementList(
    onItemClick: (Item) -> Unit,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    LazyColumn {
        items(items) { item ->
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .clickable { onItemClick(item) }
                    .padding(16.dp)
            ) {
                AsyncImage(
                    model = item.imageUrl,
                    contentDescription = null,
                    modifier = Modifier
                        .size(80.dp)
                        .sharedElement(
                            state = rememberSharedContentState(key = "image_${item.id}"),
                            animatedVisibilityScope = animatedVisibilityScope
                        )
                )
                
                Spacer(modifier = Modifier.width(16.dp))
                
                Text(
                    text = item.title,
                    modifier = Modifier.sharedElement(
                        state = rememberSharedContentState(key = "title_${item.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
                )
            }
        }
    }
}

@OptIn(ExperimentalSharedTransitionApi::class)
@Composable
fun SharedTransitionScope.SharedElementDetail(
    itemId: String?,
    onBack: () -> Unit,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    val item = getItemById(itemId)
    
    Column(modifier = Modifier.fillMaxSize()) {
        AsyncImage(
            model = item.imageUrl,
            contentDescription = null,
            modifier = Modifier
                .fillMaxWidth()
                .height(300.dp)
                .sharedElement(
                    state = rememberSharedContentState(key = "image_${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                )
        )
        
        Text(
            text = item.title,
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier
                .padding(16.dp)
                .sharedElement(
                    state = rememberSharedContentState(key = "title_${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope
                )
        )
        
        Text(
            text = item.description,
            modifier = Modifier.padding(horizontal = 16.dp)
        )
    }
}
```

## 四、无限循环动画

```kotlin
@Composable
fun PulsingCircle() {
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
    
    val alpha by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 0.5f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )
    
    Box(
        modifier = Modifier
            .size(100.dp)
            .graphicsLayer {
                scaleX = scale
                scaleY = scale
                this.alpha = alpha
            }
            .background(Color.Blue, CircleShape)
    )
}

@Composable
fun RotatingLoader() {
    val infiniteTransition = rememberInfiniteTransition(label = "loader")
    
    val rotation by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 360f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "rotation"
    )
    
    Icon(
        imageVector = Icons.Default.Refresh,
        contentDescription = "Loading",
        modifier = Modifier
            .size(48.dp)
            .graphicsLayer { rotationZ = rotation }
    )
}
```

## 五、手势驱动动画

```kotlin
@Composable
fun SwipeToReveal(
    onDismiss: () -> Unit,
    content: @Composable () -> Unit
) {
    var offsetX by remember { mutableFloatStateOf(0f) }
    val dismissThreshold = 200f
    
    val animatedOffset by animateFloatAsState(
        targetValue = offsetX,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessLow
        ),
        label = "offset"
    )
    
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .pointerInput(Unit) {
                detectHorizontalDragGestures(
                    onDragEnd = {
                        if (kotlin.math.abs(offsetX) > dismissThreshold) {
                            onDismiss()
                        } else {
                            offsetX = 0f
                        }
                    },
                    onHorizontalDrag = { _, dragAmount ->
                        offsetX += dragAmount
                    }
                )
            }
            .offset { IntOffset(animatedOffset.roundToInt(), 0) }
    ) {
        content()
    }
}
```

## 六、组合动画

```kotlin
@Composable
fun StaggeredAnimation() {
    var visible by remember { mutableStateOf(false) }
    
    LaunchedEffect(Unit) {
        visible = true
    }
    
    Column {
        (0..4).forEach { index ->
            AnimatedVisibility(
                visible = visible,
                enter = slideInHorizontally(
                    initialOffsetX = { -it },
                    animationSpec = tween(
                        durationMillis = 300,
                        delayMillis = index * 100  // 错开延迟
                    )
                ) + fadeIn(
                    animationSpec = tween(
                        durationMillis = 300,
                        delayMillis = index * 100
                    )
                )
            ) {
                Card(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(8.dp)
                ) {
                    Text(
                        "Item $index",
                        modifier = Modifier.padding(16.dp)
                    )
                }
            }
        }
    }
}
```

## 七、自定义动画规格

```kotlin
// 自定义弹簧动画
val customSpring = spring<Float>(
    dampingRatio = 0.4f,
    stiffness = 200f
)

// 自定义 Tween
val customTween = tween<Float>(
    durationMillis = 500,
    delayMillis = 100,
    easing = FastOutSlowInEasing
)

// Keyframes 动画
val keyframesSpec = keyframes<Float> {
    durationMillis = 1000
    0f at 0 with LinearEasing
    0.5f at 200
    1f at 500
    0.8f at 700
    1f at 1000
}
```

## 八、最佳实践

- ✅ 使用 label 参数便于调试
- ✅ 选择合适的动画规格（spring vs tween）
- ✅ 使用 graphicsLayer 进行性能优化
- ✅ 避免在动画中重建 Composable
- ✅ 使用 derivedStateOf 减少重组
- ✅ 共享元素使用唯一 key
- ✅ 无限动画注意生命周期
- ✅ 复杂动画使用 Transition API

## 总结

Compose 动画进阶要点：

- **Transition**：协调多个属性动画
- **AnimatedContent**：内容切换动画
- **SharedElement**：跨页面共享元素
- **InfiniteTransition**：无限循环动画
- **手势动画**：响应用户交互
- **Staggered**：错开动画效果

掌握这些动画技巧可以创造出色的用户体验。

---

*© 2024 Fidroid. [返回首页](../index.html)*

