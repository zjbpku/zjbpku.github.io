# Compose 列表优化进阶：LazyColumn 高级技巧

> **发布日期**: 2024-06-19  
> **阅读时间**: 约 24 分钟  
> **标签**: LazyColumn, LazyGrid, 性能优化, 列表

LazyColumn 是 Compose 中最常用的列表组件，掌握其高级用法对于构建高性能应用至关重要。本文将深入讲解 LazyColumn 的进阶技巧。

## 一、基础优化

### 使用稳定的 Key

```kotlin
@Composable
fun ArticleList(articles: List<Article>) {
    LazyColumn {
        items(
            items = articles,
            key = { article -> article.id }  // 使用唯一稳定的 key
        ) { article ->
            ArticleCard(article)
        }
    }
}
```

### contentType 优化

```kotlin
@Composable
fun MixedContentList(items: List<ListItem>) {
    LazyColumn {
        items(
            items = items,
            key = { it.id },
            contentType = { item ->
                when (item) {
                    is ListItem.Header -> "header"
                    is ListItem.Article -> "article"
                    is ListItem.Ad -> "ad"
                }
            }
        ) { item ->
            when (item) {
                is ListItem.Header -> HeaderItem(item)
                is ListItem.Article -> ArticleItem(item)
                is ListItem.Ad -> AdItem(item)
            }
        }
    }
}
```

## 二、Sticky Headers

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun GroupedList(groups: Map<String, List<Contact>>) {
    LazyColumn {
        groups.forEach { (letter, contacts) ->
            stickyHeader {
                Surface(
                    modifier = Modifier.fillMaxWidth(),
                    color = MaterialTheme.colorScheme.surfaceVariant
                ) {
                    Text(
                        text = letter,
                        modifier = Modifier.padding(16.dp),
                        style = MaterialTheme.typography.titleMedium
                    )
                }
            }
            
            items(contacts, key = { it.id }) { contact ->
                ContactItem(contact)
            }
        }
    }
}
```

## 三、滚动控制

### 滚动到指定位置

```kotlin
@Composable
fun ScrollableList(items: List<String>) {
    val listState = rememberLazyListState()
    val coroutineScope = rememberCoroutineScope()
    
    Column {
        Row {
            Button(
                onClick = {
                    coroutineScope.launch {
                        listState.animateScrollToItem(0)
                    }
                }
            ) {
                Text("滚动到顶部")
            }
            
            Button(
                onClick = {
                    coroutineScope.launch {
                        listState.animateScrollToItem(items.lastIndex)
                    }
                }
            ) {
                Text("滚动到底部")
            }
        }
        
        LazyColumn(state = listState) {
            items(items) { item -> Text(item) }
        }
    }
}
```

### 监听滚动状态

```kotlin
@Composable
fun ScrollAwareList() {
    val listState = rememberLazyListState()
    
    // 是否在滚动
    val isScrolling = listState.isScrollInProgress
    
    // 第一个可见项
    val firstVisibleIndex = remember {
        derivedStateOf { listState.firstVisibleItemIndex }
    }
    
    // 显示 FAB 返回顶部
    val showScrollToTop = remember {
        derivedStateOf { listState.firstVisibleItemIndex > 5 }
    }
    
    Box {
        LazyColumn(state = listState) {
            items(100) { index ->
                Text("Item $index", modifier = Modifier.padding(16.dp))
            }
        }
        
        AnimatedVisibility(
            visible = showScrollToTop.value,
            modifier = Modifier
                .align(Alignment.BottomEnd)
                .padding(16.dp)
        ) {
            FloatingActionButton(
                onClick = {
                    coroutineScope.launch {
                        listState.animateScrollToItem(0)
                    }
                }
            ) {
                Icon(Icons.Default.KeyboardArrowUp, "返回顶部")
            }
        }
    }
}
```

## 四、LazyGrid

### 固定列数

```kotlin
@Composable
fun PhotoGrid(photos: List<Photo>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(3),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(photos, key = { it.id }) { photo ->
            AsyncImage(
                model = photo.url,
                contentDescription = null,
                modifier = Modifier
                    .aspectRatio(1f)
                    .clip(RoundedCornerShape(8.dp)),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

### 自适应列数

```kotlin
@Composable
fun AdaptiveGrid(items: List<Item>) {
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 160.dp),
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        items(items, key = { it.id }) { item ->
            ItemCard(item)
        }
    }
}
```

### Span Size

```kotlin
@Composable
fun SpanGrid(items: List<GridItem>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(4)
    ) {
        items(
            items = items,
            key = { it.id },
            span = { item ->
                GridItemSpan(
                    when (item.type) {
                        ItemType.FULL -> maxLineSpan  // 占满一行
                        ItemType.HALF -> 2            // 占半行
                        ItemType.NORMAL -> 1          // 占 1/4
                    }
                )
            }
        ) { item ->
            GridItemContent(item)
        }
    }
}
```

## 五、LazyStaggeredGrid

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun PinterestGrid(items: List<PinItem>) {
    LazyVerticalStaggeredGrid(
        columns = StaggeredGridCells.Fixed(2),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalItemSpacing = 8.dp
    ) {
        items(items, key = { it.id }) { item ->
            Card(
                modifier = Modifier.fillMaxWidth()
            ) {
                Column {
                    AsyncImage(
                        model = item.imageUrl,
                        contentDescription = null,
                        modifier = Modifier
                            .fillMaxWidth()
                            .height(item.height.dp),  // 不同高度
                        contentScale = ContentScale.Crop
                    )
                    Text(
                        text = item.title,
                        modifier = Modifier.padding(8.dp)
                    )
                }
            }
        }
    }
}
```

## 六、Item 动画

### 添加/删除动画

```kotlin
@Composable
fun AnimatedList(items: List<Item>) {
    LazyColumn {
        items(
            items = items,
            key = { it.id }
        ) { item ->
            Box(
                modifier = Modifier.animateItem(
                    fadeInSpec = tween(300),
                    fadeOutSpec = tween(300),
                    placementSpec = spring()
                )
            ) {
                ItemCard(item)
            }
        }
    }
}
```

### 拖拽排序

```kotlin
@Composable
fun ReorderableList(
    items: List<Item>,
    onMove: (from: Int, to: Int) -> Unit
) {
    val listState = rememberLazyListState()
    val dragDropState = rememberDragDropState(listState) { from, to ->
        onMove(from, to)
    }
    
    LazyColumn(
        state = listState,
        modifier = Modifier.dragContainer(dragDropState)
    ) {
        itemsIndexed(items, key = { _, item -> item.id }) { index, item ->
            DraggableItem(dragDropState, index) { isDragging ->
                Card(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(8.dp),
                    elevation = CardDefaults.cardElevation(
                        defaultElevation = if (isDragging) 8.dp else 1.dp
                    )
                ) {
                    Text(item.title, modifier = Modifier.padding(16.dp))
                }
            }
        }
    }
}
```

## 七、性能优化

### 避免不必要的重组

```kotlin
// ❌ 错误：每次重组都创建新 lambda
LazyColumn {
    items(items) { item ->
        ItemCard(
            item = item,
            onClick = { viewModel.onItemClick(item.id) }  // 每次创建新 lambda
        )
    }
}

// ✅ 正确：使用 remember 缓存
LazyColumn {
    items(items, key = { it.id }) { item ->
        val onClick = remember(item.id) {
            { viewModel.onItemClick(item.id) }
        }
        ItemCard(item = item, onClick = onClick)
    }
}

// ✅ 更好：在 ItemCard 内部处理
@Composable
fun ItemCard(
    item: Item,
    onItemClick: (String) -> Unit  // 接收 id 而不是完整 lambda
) {
    Card(
        onClick = { onItemClick(item.id) }
    ) {
        // content
    }
}
```

### 使用 derivedStateOf

```kotlin
@Composable
fun OptimizedScrollDetection() {
    val listState = rememberLazyListState()
    
    // ✅ 使用 derivedStateOf 减少重组
    val showButton = remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }
    
    // 只有 showButton 变化时才重组
    if (showButton.value) {
        ScrollToTopButton()
    }
}
```

## 八、嵌套滚动

```kotlin
@Composable
fun NestedScrollExample() {
    val lazyListState = rememberLazyListState()
    
    LazyColumn(state = lazyListState) {
        item {
            // 水平列表
            LazyRow {
                items(horizontalItems) { item ->
                    HorizontalCard(item)
                }
            }
        }
        
        items(verticalItems) { item ->
            VerticalCard(item)
        }
    }
}
```

## 九、最佳实践

- ✅ 始终提供稳定的 key
- ✅ 使用 contentType 优化复用
- ✅ 使用 derivedStateOf 减少重组
- ✅ 避免在 items 中创建新 lambda
- ✅ 使用 remember 缓存计算结果
- ✅ 合理使用 LazyGrid 替代嵌套列表
- ✅ 使用 animateItem 添加动画
- ✅ 大列表考虑使用 Paging 3

## 总结

LazyColumn 进阶要点：

- **Key**：提供稳定唯一的标识
- **contentType**：优化 item 复用
- **Sticky Header**：分组固定头部
- **滚动控制**：编程控制滚动位置
- **LazyGrid**：网格布局
- **动画**：添加/删除/排序动画
- **性能**：避免不必要的重组

掌握这些技巧可以构建高性能的列表界面。

---

*© 2024 Fidroid. [返回首页](../index.html)*

