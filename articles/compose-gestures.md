# Compose 手势处理完全指南：点击、拖拽与自定义手势

> **发布日期**: 2024-05-14  
> **阅读时间**: 约 26 分钟  
> **标签**: Gesture, pointerInput, Drag, Transform

手势交互是移动应用的核心体验。Compose 提供了从简单点击到复杂多点触控的完整手势支持。本文将深入讲解 Compose 的手势系统，帮助你构建流畅自然的交互体验。

## 一、点击手势

### 基础点击

```kotlin
// 简单点击
Box(
    modifier = Modifier
        .size(100.dp)
        .clickable { 
            println("点击了") 
        }
)

// 带涟漪效果控制
Box(
    modifier = Modifier
        .size(100.dp)
        .clickable(
            interactionSource = remember { MutableInteractionSource() },
            indication = rememberRipple(bounded = true),
            onClick = { println("点击了") }
        )
)

// 无涟漪效果
Box(
    modifier = Modifier
        .size(100.dp)
        .clickable(
            interactionSource = remember { MutableInteractionSource() },
            indication = null,
            onClick = { println("点击了") }
        )
)
```

### combinedClickable：组合点击

```kotlin
Box(
    modifier = Modifier
        .size(100.dp)
        .combinedClickable(
            onClick = { println("单击") },
            onLongClick = { println("长按") },
            onDoubleClick = { println("双击") }
        )
)
```

### 监听点击状态

```kotlin
@Composable
fun InteractiveButton() {
    val interactionSource = remember { MutableInteractionSource() }
    val isPressed by interactionSource.collectIsPressedAsState()
    
    val backgroundColor = if (isPressed) Color.DarkGray else Color.Gray
    
    Box(
        modifier = Modifier
            .size(100.dp)
            .background(backgroundColor)
            .clickable(
                interactionSource = interactionSource,
                indication = null,
                onClick = { }
            ),
        contentAlignment = Alignment.Center
    ) {
        Text(if (isPressed) "按下中" else "点击我")
    }
}
```

## 二、拖拽手势

### draggable：单轴拖拽

```kotlin
@Composable
fun HorizontalDraggable() {
    var offsetX by remember { mutableFloatStateOf(0f) }
    
    Box(
        modifier = Modifier
            .offset { IntOffset(offsetX.roundToInt(), 0) }
            .size(50.dp)
            .background(Color.Blue)
            .draggable(
                orientation = Orientation.Horizontal,
                state = rememberDraggableState { delta ->
                    offsetX += delta
                }
            )
    )
}
```

### Modifier.offset + pointerInput：自由拖拽

```kotlin
@Composable
fun FreeDraggable() {
    var offset by remember { mutableStateOf(Offset.Zero) }
    
    Box(
        modifier = Modifier
            .offset { IntOffset(offset.x.roundToInt(), offset.y.roundToInt()) }
            .size(50.dp)
            .background(Color.Blue)
            .pointerInput(Unit) {
                detectDragGestures { change, dragAmount ->
                    change.consume()
                    offset += dragAmount
                }
            }
    )
}
```

### 带边界限制的拖拽

```kotlin
@Composable
fun BoundedDraggable() {
    var offset by remember { mutableStateOf(Offset.Zero) }
    
    BoxWithConstraints(
        modifier = Modifier.fillMaxSize()
    ) {
        val maxX = constraints.maxWidth - 50.dp.toPx()
        val maxY = constraints.maxHeight - 50.dp.toPx()
        
        Box(
            modifier = Modifier
                .offset { IntOffset(offset.x.roundToInt(), offset.y.roundToInt()) }
                .size(50.dp)
                .background(Color.Blue)
                .pointerInput(Unit) {
                    detectDragGestures { change, dragAmount ->
                        change.consume()
                        offset = Offset(
                            x = (offset.x + dragAmount.x).coerceIn(0f, maxX),
                            y = (offset.y + dragAmount.y).coerceIn(0f, maxY)
                        )
                    }
                }
        )
    }
}
```

## 三、滑动手势

### swipeable（Material 2）

```kotlin
@OptIn(ExperimentalMaterialApi::class)
@Composable
fun SwipeToDelete() {
    val swipeableState = rememberSwipeableState(initialValue = 0)
    val sizePx = with(LocalDensity.current) { 100.dp.toPx() }
    val anchors = mapOf(0f to 0, -sizePx to 1)
    
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(60.dp)
            .swipeable(
                state = swipeableState,
                anchors = anchors,
                thresholds = { _, _ -> FractionalThreshold(0.5f) },
                orientation = Orientation.Horizontal
            )
    ) {
        // 背景（删除按钮）
        Box(
            modifier = Modifier
                .fillMaxSize()
                .background(Color.Red),
            contentAlignment = Alignment.CenterEnd
        ) {
            Icon(
                Icons.Default.Delete,
                contentDescription = "删除",
                modifier = Modifier.padding(end = 16.dp),
                tint = Color.White
            )
        }
        
        // 前景内容
        Box(
            modifier = Modifier
                .offset { IntOffset(swipeableState.offset.value.roundToInt(), 0) }
                .fillMaxSize()
                .background(Color.White)
        ) {
            Text("滑动删除", modifier = Modifier.padding(16.dp))
        }
    }
}
```

### AnchoredDraggable（Material 3）

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun AnchoredDraggableExample() {
    val density = LocalDensity.current
    val anchors = with(density) {
        DraggableAnchors {
            DragValue.Start at 0f
            DragValue.End at 100.dp.toPx()
        }
    }
    
    val state = remember {
        AnchoredDraggableState(
            initialValue = DragValue.Start,
            anchors = anchors,
            positionalThreshold = { distance -> distance * 0.5f },
            velocityThreshold = { with(density) { 100.dp.toPx() } },
            animationSpec = tween()
        )
    }
    
    Box(
        modifier = Modifier
            .offset { IntOffset(state.offset.roundToInt(), 0) }
            .anchoredDraggable(state, Orientation.Horizontal)
            .size(50.dp)
            .background(Color.Blue)
    )
}

enum class DragValue { Start, End }
```

## 四、缩放、旋转、平移

### transformable：组合变换

```kotlin
@Composable
fun TransformableBox() {
    var scale by remember { mutableFloatStateOf(1f) }
    var rotation by remember { mutableFloatStateOf(0f) }
    var offset by remember { mutableStateOf(Offset.Zero) }
    
    val state = rememberTransformableState { zoomChange, offsetChange, rotationChange ->
        scale *= zoomChange
        rotation += rotationChange
        offset += offsetChange
    }
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .transformable(state = state),
        contentAlignment = Alignment.Center
    ) {
        Box(
            modifier = Modifier
                .graphicsLayer {
                    scaleX = scale
                    scaleY = scale
                    rotationZ = rotation
                    translationX = offset.x
                    translationY = offset.y
                }
                .size(100.dp)
                .background(Color.Blue)
        )
    }
}
```

### 图片查看器

```kotlin
@Composable
fun ZoomableImage(painter: Painter) {
    var scale by remember { mutableFloatStateOf(1f) }
    var offset by remember { mutableStateOf(Offset.Zero) }
    
    val state = rememberTransformableState { zoomChange, offsetChange, _ ->
        scale = (scale * zoomChange).coerceIn(1f, 5f)
        
        // 限制平移范围
        val maxX = (scale - 1) * 500 / 2
        val maxY = (scale - 1) * 500 / 2
        offset = Offset(
            x = (offset.x + offsetChange.x).coerceIn(-maxX, maxX),
            y = (offset.y + offsetChange.y).coerceIn(-maxY, maxY)
        )
    }
    
    Image(
        painter = painter,
        contentDescription = null,
        modifier = Modifier
            .fillMaxSize()
            .transformable(state = state)
            .graphicsLayer {
                scaleX = scale
                scaleY = scale
                translationX = offset.x
                translationY = offset.y
            }
    )
}
```

## 五、pointerInput：低级手势 API

### detectTapGestures

```kotlin
@Composable
fun TapGesturesDemo() {
    Box(
        modifier = Modifier
            .size(200.dp)
            .background(Color.LightGray)
            .pointerInput(Unit) {
                detectTapGestures(
                    onPress = { offset ->
                        println("按下: $offset")
                        // 可以 awaitRelease() 等待释放
                    },
                    onTap = { offset ->
                        println("点击: $offset")
                    },
                    onDoubleTap = { offset ->
                        println("双击: $offset")
                    },
                    onLongPress = { offset ->
                        println("长按: $offset")
                    }
                )
            }
    )
}
```

### detectDragGestures

```kotlin
@Composable
fun DragGesturesDemo() {
    var offset by remember { mutableStateOf(Offset.Zero) }
    var isDragging by remember { mutableStateOf(false) }
    
    Box(
        modifier = Modifier
            .offset { IntOffset(offset.x.roundToInt(), offset.y.roundToInt()) }
            .size(50.dp)
            .background(if (isDragging) Color.Red else Color.Blue)
            .pointerInput(Unit) {
                detectDragGestures(
                    onDragStart = { startOffset ->
                        isDragging = true
                        println("开始拖拽: $startOffset")
                    },
                    onDrag = { change, dragAmount ->
                        change.consume()
                        offset += dragAmount
                    },
                    onDragEnd = {
                        isDragging = false
                        println("拖拽结束")
                    },
                    onDragCancel = {
                        isDragging = false
                        println("拖拽取消")
                    }
                )
            }
    )
}
```

### detectTransformGestures

```kotlin
@Composable
fun TransformGesturesDemo() {
    var scale by remember { mutableFloatStateOf(1f) }
    var rotation by remember { mutableFloatStateOf(0f) }
    var offset by remember { mutableStateOf(Offset.Zero) }
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectTransformGestures(
                    panZoomLock = false,
                    onGesture = { centroid, pan, zoom, rotationDelta ->
                        scale *= zoom
                        rotation += rotationDelta
                        offset += pan
                    }
                )
            },
        contentAlignment = Alignment.Center
    ) {
        Box(
            modifier = Modifier
                .graphicsLayer {
                    scaleX = scale
                    scaleY = scale
                    rotationZ = rotation
                    translationX = offset.x
                    translationY = offset.y
                }
                .size(100.dp)
                .background(Color.Blue)
        )
    }
}
```

## 六、自定义手势检测

### 使用 awaitPointerEventScope

```kotlin
@Composable
fun CustomGestureDemo() {
    Box(
        modifier = Modifier
            .size(200.dp)
            .background(Color.LightGray)
            .pointerInput(Unit) {
                awaitPointerEventScope {
                    while (true) {
                        val event = awaitPointerEvent()
                        
                        when (event.type) {
                            PointerEventType.Press -> {
                                println("按下: ${event.changes.first().position}")
                            }
                            PointerEventType.Move -> {
                                println("移动: ${event.changes.first().position}")
                            }
                            PointerEventType.Release -> {
                                println("释放: ${event.changes.first().position}")
                            }
                        }
                    }
                }
            }
    )
}
```

### 检测滑动方向

```kotlin
enum class SwipeDirection { Left, Right, Up, Down }

suspend fun PointerInputScope.detectSwipeDirection(
    onSwipe: (SwipeDirection) -> Unit
) {
    detectDragGestures(
        onDragEnd = { },
        onDrag = { change, dragAmount ->
            change.consume()
            
            val (x, y) = dragAmount
            if (abs(x) > abs(y)) {
                // 水平滑动
                if (x > 0) onSwipe(SwipeDirection.Right)
                else onSwipe(SwipeDirection.Left)
            } else {
                // 垂直滑动
                if (y > 0) onSwipe(SwipeDirection.Down)
                else onSwipe(SwipeDirection.Up)
            }
        }
    )
}

@Composable
fun SwipeDirectionDemo() {
    var direction by remember { mutableStateOf<SwipeDirection?>(null) }
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectSwipeDirection { swipeDirection ->
                    direction = swipeDirection
                }
            },
        contentAlignment = Alignment.Center
    ) {
        Text("滑动方向: ${direction?.name ?: "未检测到"}")
    }
}
```

## 七、嵌套滚动

### nestedScroll

```kotlin
@Composable
fun NestedScrollDemo() {
    val nestedScrollConnection = remember {
        object : NestedScrollConnection {
            override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
                // 在子组件滚动前拦截
                println("onPreScroll: $available")
                return Offset.Zero  // 不消费，交给子组件
            }
            
            override fun onPostScroll(
                consumed: Offset,
                available: Offset,
                source: NestedScrollSource
            ): Offset {
                // 子组件滚动后处理剩余
                println("onPostScroll: consumed=$consumed, available=$available")
                return Offset.Zero
            }
        }
    }
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .nestedScroll(nestedScrollConnection)
    ) {
        LazyColumn {
            items(50) { index ->
                Text(
                    "Item $index",
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(16.dp)
                )
            }
        }
    }
}
```

### 可折叠 Toolbar

```kotlin
@Composable
fun CollapsibleToolbar() {
    var toolbarHeight by remember { mutableFloatStateOf(200f) }
    val minHeight = 56f
    val maxHeight = 200f
    
    val nestedScrollConnection = remember {
        object : NestedScrollConnection {
            override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
                val delta = available.y
                val newHeight = toolbarHeight + delta
                toolbarHeight = newHeight.coerceIn(minHeight, maxHeight)
                
                // 如果 toolbar 还能收缩/展开，消费掉滚动
                return if (newHeight in minHeight..maxHeight) {
                    Offset(0f, delta)
                } else {
                    Offset.Zero
                }
            }
        }
    }
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .nestedScroll(nestedScrollConnection)
    ) {
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .height(toolbarHeight.dp)
                .background(Color.Blue),
            contentAlignment = Alignment.Center
        ) {
            Text("Toolbar", color = Color.White)
        }
        
        LazyColumn {
            items(50) { index ->
                Text("Item $index", modifier = Modifier.padding(16.dp))
            }
        }
    }
}
```

## 八、手势冲突处理

### 手势优先级

```kotlin
@Composable
fun GesturePriorityDemo() {
    Box(
        modifier = Modifier
            .size(200.dp)
            .background(Color.LightGray)
            .pointerInput(Unit) {
                // 外层：检测点击
                detectTapGestures {
                    println("外层点击")
                }
            }
    ) {
        Box(
            modifier = Modifier
                .size(100.dp)
                .align(Alignment.Center)
                .background(Color.Blue)
                .pointerInput(Unit) {
                    // 内层：检测拖拽
                    detectDragGestures { change, _ ->
                        change.consume()
                        println("内层拖拽")
                    }
                }
        )
    }
}
```

### 使用 consume() 阻止传播

```kotlin
@Composable
fun ConsumeGestureDemo() {
    Box(
        modifier = Modifier
            .size(200.dp)
            .clickable { println("外层点击") }
    ) {
        Box(
            modifier = Modifier
                .size(100.dp)
                .align(Alignment.Center)
                .pointerInput(Unit) {
                    detectTapGestures {
                        println("内层点击")
                        // 手势已被消费，不会传递到外层
                    }
                }
        )
    }
}
```

## 九、最佳实践

### 1. 使用高级 API 优先

```kotlin
// ✅ 优先使用高级 API
Modifier.clickable { }
Modifier.draggable(...)
Modifier.transformable(...)

// 只在需要自定义行为时使用低级 API
Modifier.pointerInput(Unit) {
    detectDragGestures { ... }
}
```

### 2. 正确使用 key

```kotlin
// ✅ key 变化时重新设置手势
Modifier.pointerInput(itemId) {
    detectTapGestures {
        onItemClick(itemId)
    }
}

// ❌ 使用 Unit 可能导致闭包捕获旧值
Modifier.pointerInput(Unit) {
    detectTapGestures {
        onItemClick(itemId)  // itemId 可能是旧值
    }
}
```

### 3. 手势性能优化

```kotlin
// ✅ 使用 graphicsLayer 进行变换
Modifier.graphicsLayer {
    scaleX = scale
    translationX = offset.x
}

// ❌ 避免在手势中触发重组
Modifier.scale(scale)  // 会触发重组
```

## 总结

Compose 手势系统的核心要点：

- **clickable**：简单点击，支持涟漪效果
- **combinedClickable**：单击、双击、长按组合
- **draggable**：单轴拖拽
- **transformable**：缩放、旋转、平移
- **pointerInput**：底层 API，自定义手势逻辑
- **nestedScroll**：嵌套滚动协调
- **consume()**：控制手势事件传播

掌握这些 API，你可以实现任何复杂的手势交互。

---

*© 2024 Fidroid. [返回首页](../index.html)*



