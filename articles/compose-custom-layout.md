# Compose 自定义布局与 Modifier 深度解析

> **发布日期**: 2024-04-12  
> **阅读时间**: 约 24 分钟  
> **标签**: Custom Layout, Modifier, Intrinsic, SubcomposeLayout

当内置的 Row、Column、Box 无法满足需求时，你需要深入 Compose 的布局系统。本文将带你掌握自定义 Layout 和 Modifier 的核心技术。

## 一、布局基础：Measure → Place

Compose 布局分为两个阶段：

1. **Measure（测量）**：确定每个子元素的大小
2. **Place（放置）**：确定每个子元素的位置

```kotlin
@Composable
fun SimpleColumn(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        content = content,
        modifier = modifier
    ) { measurables, constraints ->
        // 1. 测量所有子元素
        val placeables = measurables.map { measurable ->
            measurable.measure(constraints)
        }

        // 2. 计算布局尺寸
        val width = placeables.maxOfOrNull { it.width } ?: 0
        val height = placeables.sumOf { it.height }

        // 3. 放置子元素
        layout(width, height) {
            var yPosition = 0
            placeables.forEach { placeable ->
                placeable.placeRelative(x = 0, y = yPosition)
                yPosition += placeable.height
            }
        }
    }
}
```

## 二、理解 Constraints

`Constraints` 定义了子元素可用的尺寸范围：

```kotlin
data class Constraints(
    val minWidth: Int,
    val maxWidth: Int,
    val minHeight: Int,
    val maxHeight: Int
)

// 常用约束操作
constraints.copy(minWidth = 0)  // 放宽最小宽度
constraints.copy(maxWidth = 200)  // 限制最大宽度
Constraints.fixed(100, 100)  // 固定尺寸
```

## 三、实战：流式布局（FlowRow）

```kotlin
@Composable
fun FlowRow(
    modifier: Modifier = Modifier,
    horizontalSpacing: Dp = 8.dp,
    verticalSpacing: Dp = 8.dp,
    content: @Composable () -> Unit
) {
    Layout(content = content, modifier = modifier) { measurables, constraints ->
        val hSpacing = horizontalSpacing.roundToPx()
        val vSpacing = verticalSpacing.roundToPx()

        val placeables = measurables.map { it.measure(constraints) }

        var x = 0
        var y = 0
        var rowHeight = 0

        val positions = placeables.map { placeable ->
            if (x + placeable.width > constraints.maxWidth) {
                x = 0
                y += rowHeight + vSpacing
                rowHeight = 0
            }
            val pos = IntOffset(x, y)
            x += placeable.width + hSpacing
            rowHeight = maxOf(rowHeight, placeable.height)
            pos
        }

        val totalHeight = y + rowHeight
        layout(constraints.maxWidth, totalHeight) {
            placeables.forEachIndexed { index, placeable ->
                placeable.placeRelative(positions[index])
            }
        }
    }
}
```

## 四、自定义 Modifier

### 基于 composed 的 Modifier

```kotlin
fun Modifier.shimmer(): Modifier = composed {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val translateX by transition.animateFloat(
        initialValue = -1000f,
        targetValue = 1000f,
        animationSpec = infiniteRepeatable(
            animation = tween(1200, easing = LinearEasing)
        ),
        label = "translateX"
    )

    val brush = Brush.linearGradient(
        colors = listOf(
            Color.LightGray.copy(alpha = 0.6f),
            Color.LightGray.copy(alpha = 0.2f),
            Color.LightGray.copy(alpha = 0.6f)
        ),
        start = Offset(translateX, 0f),
        end = Offset(translateX + 500f, 0f)
    )

    background(brush)
}
```

### 基于 layout 的 Modifier

```kotlin
fun Modifier.badgeLayout(): Modifier = layout { measurable, constraints ->
    val placeable = measurable.measure(constraints)

    // Badge 偏移到右上角
    layout(placeable.width, placeable.height) {
        placeable.placeRelative(
            x = placeable.width - 12,
            y = -12
        )
    }
}
```

### 基于 drawWithContent 的 Modifier

```kotlin
fun Modifier.fadingEdge(
    startFade: Dp = 0.dp,
    endFade: Dp = 16.dp
): Modifier = drawWithContent {
    drawContent()

    val colors = listOf(Color.Transparent, Color.Black)
    drawRect(
        brush = Brush.horizontalGradient(
            colors = colors,
            startX = size.width - endFade.toPx(),
            endX = size.width
        ),
        blendMode = BlendMode.DstIn
    )
}
```

## 五、Intrinsic Measurements

当需要在测量前知道子元素的"固有尺寸"时，使用 Intrinsic：

```kotlin
@Composable
fun TwoTextsWithDivider(text1: String, text2: String) {
    Row(modifier = Modifier.height(IntrinsicSize.Min)) {
        Text(text1, modifier = Modifier.weight(1f))
        Divider(
            modifier = Modifier
                .fillMaxHeight()  // 高度等于 Row 的固有高度
                .width(1.dp),
            color = Color.Gray
        )
        Text(text2, modifier = Modifier.weight(1f))
    }
}
```

> ⚠️ **性能注意**  
> Intrinsic 会导致额外的测量 pass，应谨慎使用。优先考虑能否用其他方式实现。

## 六、SubcomposeLayout

当需要根据一个子元素的测量结果来决定另一个子元素的组合时，使用 SubcomposeLayout：

```kotlin
@Composable
fun AdaptiveText(
    text: String,
    modifier: Modifier = Modifier
) {
    SubcomposeLayout(modifier) { constraints ->
        // 先测量完整文本
        val fullTextPlaceable = subcompose("full") {
            Text(text, maxLines = 1)
        }.first().measure(constraints)

        // 如果放不下，显示省略版本
        val placeable = if (fullTextPlaceable.width > constraints.maxWidth) {
            subcompose("ellipsis") {
                Text(text, maxLines = 1, overflow = TextOverflow.Ellipsis)
            }.first().measure(constraints)
        } else {
            fullTextPlaceable
        }

        layout(placeable.width, placeable.height) {
            placeable.placeRelative(0, 0)
        }
    }
}
```

## 七、ParentDataModifier

让子元素向父布局传递数据：

```kotlin
interface WeightScope {
    fun Modifier.weight(weight: Float): Modifier
}

private class WeightData(val weight: Float) : ParentDataModifier {
    override fun Density.modifyParentData(parentData: Any?) = this@WeightData
}

@Composable
fun WeightedRow(
    modifier: Modifier = Modifier,
    content: @Composable WeightScope.() -> Unit
) {
    val scope = object : WeightScope {
        override fun Modifier.weight(weight: Float) =
            this.then(WeightData(weight))
    }

    Layout(content = { scope.content() }, modifier = modifier) { measurables, constraints ->
        val totalWeight = measurables.sumOf {
            (it.parentData as? WeightData)?.weight?.toDouble() ?: 1.0
        }.toFloat()

        val placeables = measurables.map { measurable ->
            val weight = (measurable.parentData as? WeightData)?.weight ?: 1f
            val width = (constraints.maxWidth * weight / totalWeight).toInt()
            measurable.measure(constraints.copy(minWidth = width, maxWidth = width))
        }

        layout(constraints.maxWidth, placeables.maxOf { it.height }) {
            var x = 0
            placeables.forEach { placeable ->
                placeable.placeRelative(x, 0)
                x += placeable.width
            }
        }
    }
}
```

## 八、布局性能优化

- ✅ 避免在 measure/place 中创建对象
- ✅ 使用 `Modifier.layout` 而非完整的 Layout 组件
- ✅ 避免不必要的 Intrinsic 测量
- ✅ 使用 `LookaheadScope` 优化动画布局

## 总结

自定义布局的核心知识点：

- **Layout**：完全自定义的布局逻辑
- **Modifier.layout**：修改单个元素的测量和放置
- **Intrinsic**：在测量前获取子元素的固有尺寸
- **SubcomposeLayout**：根据测量结果决定组合内容
- **ParentDataModifier**：子元素向父布局传递数据

掌握这些技术，你可以实现任何复杂的 UI 布局需求。

---

*© 2024 Fidroid. [返回首页](../index.html)*

