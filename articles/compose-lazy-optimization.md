# LazyColumn/LazyGrid 深度优化：打造丝滑列表体验

> **发布日期**: 2024-04-18  
> **阅读时间**: 约 22 分钟  
> **标签**: LazyColumn, LazyGrid, Paging3, Performance

列表是移动应用中最常见的 UI 模式，也是性能问题的高发区。本文将深入探讨如何优化 Compose 中的 LazyColumn 和 LazyGrid，打造流畅的滚动体验。

## 一、LazyColumn 基础回顾

```kotlin
LazyColumn {
    items(
        items = users,
        key = { user -> user.id }  // 关键：提供稳定的 key
    ) { user ->
        UserRow(user)
    }
}
```

## 二、key 的重要性

`key` 帮助 Compose 识别列表项的身份，是优化的第一步：

```kotlin
// ❌ 没有 key：删除/插入时可能出现错误动画和状态丢失
items(users) { user -> UserRow(user) }

// ✅ 使用唯一 key：正确追踪每个 item
items(
    items = users,
    key = { it.id }
) { user ->
    UserRow(user)
}
```

> 💡 **key 的作用**  
> 1. 正确处理列表项的添加/删除/移动动画  
> 2. 保持列表项内部状态（如展开状态、输入内容）  
> 3. 避免不必要的重组

## 三、contentType 优化

当列表包含多种类型的 item 时，使用 `contentType` 帮助 Compose 复用 Composition：

```kotlin
LazyColumn {
    items.forEach { item ->
        when (item) {
            is Header -> item(
                key = item.id,
                contentType = "header"  // 类型标识
            ) {
                HeaderRow(item)
            }
            is User -> item(
                key = item.id,
                contentType = "user"
            ) {
                UserRow(item)
            }
            is Ad -> item(
                key = item.id,
                contentType = "ad"
            ) {
                AdBanner(item)
            }
        }
    }
}
```

## 四、避免 item 内部重组

### 1. 使用 Immutable 数据

```kotlin
// ✅ 不可变数据类
@Immutable
data class User(
    val id: String,
    val name: String,
    val avatar: String
)
```

### 2. 稳定的 lambda

```kotlin
// ❌ 每次重组创建新 lambda
items(users, key = { it.id }) { user ->
    UserRow(
        user = user,
        onClick = { viewModel.onUserClick(user) }  // 新 lambda
    )
}

// ✅ 使用方法引用或记忆化
items(users, key = { it.id }) { user ->
    UserRow(
        user = user,
        onClick = viewModel::onUserClick  // 方法引用
    )
}
```

### 3. 拆分组件粒度

```kotlin
// ❌ 整个 Row 因 isOnline 变化而重组
@Composable
fun UserRow(user: User, isOnline: Boolean) {
    Row {
        Avatar(user.avatar)
        Text(user.name)
        OnlineIndicator(isOnline)  // 频繁变化
    }
}

// ✅ 将频繁变化的部分独立
@Composable
fun UserRow(user: User, onlineState: State<Boolean>) {
    Row {
        Avatar(user.avatar)
        Text(user.name)
        OnlineIndicator(onlineState)  // 只有这个重组
    }
}
```

## 五、预加载与缓存

```kotlin
val listState = rememberLazyListState()

LazyColumn(
    state = listState,
    // 预加载屏幕外的 item
    beyondBoundsItemCount = 5
) {
    items(items, key = { it.id }) { item ->
        ItemRow(item)
    }
}
```

## 六、Paging3 集成

对于大数据集，使用 Paging3 实现按需加载：

```kotlin
// ViewModel
val usersPager = Pager(
    config = PagingConfig(
        pageSize = 20,
        prefetchDistance = 5,
        enablePlaceholders = false
    )
) {
    UsersPagingSource(repository)
}.flow.cachedIn(viewModelScope)

// Composable
@Composable
fun UserList(viewModel: UserViewModel) {
    val users = viewModel.usersPager.collectAsLazyPagingItems()

    LazyColumn {
        items(
            count = users.itemCount,
            key = users.itemKey { it.id },
            contentType = users.itemContentType { "user" }
        ) { index ->
            val user = users[index]
            if (user != null) {
                UserRow(user)
            } else {
                UserPlaceholder()
            }
        }

        // 加载状态
        when (users.loadState.append) {
            is LoadState.Loading -> item { LoadingItem() }
            is LoadState.Error -> item { ErrorItem(onRetry = { users.retry() }) }
            else -> {}
        }
    }
}
```

## 七、滚动性能监控

```kotlin
@Composable
fun PerformantList(items: List<Item>) {
    val listState = rememberLazyListState()

    // 监控滚动性能
    LaunchedEffect(listState) {
        snapshotFlow { listState.isScrollInProgress }
            .collect { isScrolling ->
                if (isScrolling) {
                    // 滚动时暂停非关键操作
                }
            }
    }

    LazyColumn(state = listState) {
        items(items, key = { it.id }) { item ->
            ItemRow(item)
        }
    }
}
```

## 八、LazyGrid 使用

```kotlin
LazyVerticalGrid(
    columns = GridCells.Adaptive(minSize = 150.dp),
    contentPadding = PaddingValues(16.dp),
    horizontalArrangement = Arrangement.spacedBy(8.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    items(
        items = products,
        key = { it.id },
        contentType = { "product" }
    ) { product ->
        ProductCard(product)
    }
}
```

## 九、Sticky Headers

```kotlin
LazyColumn {
    groupedItems.forEach { (category, items) ->
        stickyHeader(key = category) {
            CategoryHeader(category)
        }

        items(items, key = { it.id }) { item ->
            ItemRow(item)
        }
    }
}
```

## 十、性能优化清单

- ✅ 始终提供稳定的 `key`
- ✅ 使用 `contentType` 区分不同类型的 item
- ✅ 使用 `@Immutable` 或 `@Stable` 标注数据类
- ✅ 避免在 item 内创建新的 lambda
- ✅ 拆分组件，隔离频繁变化的部分
- ✅ 大数据集使用 Paging3
- ✅ 合理设置 `beyondBoundsItemCount`
- ✅ 避免在 item 中进行复杂计算
- ✅ 图片使用 Coil/Glide 等库的 Compose 扩展
- ✅ 使用 Layout Inspector 监控重组

## 总结

优化 Lazy 列表的核心原则：

- **正确的 key**：帮助 Compose 追踪 item 身份
- **减少重组**：使用稳定数据和 lambda
- **按需加载**：大数据集使用 Paging3
- **监控性能**：使用工具发现瓶颈

---

*© 2024 Fidroid. [返回首页](../index.html)*

