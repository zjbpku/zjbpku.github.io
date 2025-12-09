# Compose + Paging 3：分页加载与无限列表

> **发布日期**: 2024-06-03  
> **阅读时间**: 约 26 分钟  
> **标签**: Paging 3, LazyColumn, 分页, 无限滚动

Paging 3 是 Jetpack 提供的分页加载库，可以高效地加载和显示大量数据。结合 Compose 的 LazyColumn，可以轻松实现无限滚动列表。本文将深入讲解 Paging 3 在 Compose 中的最佳实践。

## 一、Paging 3 简介

Paging 3 的核心组件：

```
┌─────────────────────────────────────────────────────┐
│                      UI Layer                        │
│   LazyColumn + collectAsLazyPagingItems()           │
├─────────────────────────────────────────────────────┤
│                   ViewModel                          │
│              Pager + PagingData                      │
├─────────────────────────────────────────────────────┤
│                  Repository                          │
│     PagingSource / RemoteMediator                   │
├─────────────────────────────────────────────────────┤
│                   Data Source                        │
│              Network API / Database                  │
└─────────────────────────────────────────────────────┘
```

## 二、基础配置

### 添加依赖

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.paging:paging-runtime-ktx:3.2.1")
    implementation("androidx.paging:paging-compose:3.2.1")
    
    // 可选：Room 集成
    implementation("androidx.room:room-paging:2.6.1")
}
```

## 三、实现 PagingSource

### 网络数据源

```kotlin
class ArticlePagingSource(
    private val apiService: ApiService,
    private val query: String
) : PagingSource<Int, Article>() {
    
    override fun getRefreshKey(state: PagingState<Int, Article>): Int? {
        // 刷新时返回最接近当前位置的页码
        return state.anchorPosition?.let { anchorPosition ->
            val anchorPage = state.closestPageToPosition(anchorPosition)
            anchorPage?.prevKey?.plus(1) ?: anchorPage?.nextKey?.minus(1)
        }
    }
    
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        val page = params.key ?: 1
        
        return try {
            val response = apiService.searchArticles(
                query = query,
                page = page,
                pageSize = params.loadSize
            )
            
            LoadResult.Page(
                data = response.articles,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.articles.isEmpty()) null else page + 1
            )
        } catch (e: IOException) {
            LoadResult.Error(e)
        } catch (e: HttpException) {
            LoadResult.Error(e)
        }
    }
}
```

### 使用 Key 的分页

```kotlin
class CursorPagingSource(
    private val apiService: ApiService
) : PagingSource<String, Item>() {
    
    override fun getRefreshKey(state: PagingState<String, Item>): String? {
        return null // 基于 cursor 的分页通常从头开始刷新
    }
    
    override suspend fun load(params: LoadParams<String>): LoadResult<String, Item> {
        val cursor = params.key
        
        return try {
            val response = apiService.getItems(
                cursor = cursor,
                limit = params.loadSize
            )
            
            LoadResult.Page(
                data = response.items,
                prevKey = null, // 单向分页
                nextKey = response.nextCursor
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}
```

## 四、创建 Pager

### Repository 层

```kotlin
class ArticleRepository(
    private val apiService: ApiService
) {
    fun getArticleStream(query: String): Flow<PagingData<Article>> {
        return Pager(
            config = PagingConfig(
                pageSize = 20,
                prefetchDistance = 5,
                enablePlaceholders = false,
                initialLoadSize = 40
            ),
            pagingSourceFactory = { ArticlePagingSource(apiService, query) }
        ).flow
    }
}
```

### ViewModel 层

```kotlin
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val repository: ArticleRepository
) : ViewModel() {
    
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    val articles: Flow<PagingData<Article>> = _searchQuery
        .debounce(300)
        .flatMapLatest { query ->
            repository.getArticleStream(query)
        }
        .cachedIn(viewModelScope)
    
    fun search(query: String) {
        _searchQuery.value = query
    }
}
```

## 五、在 Compose 中使用

### collectAsLazyPagingItems

```kotlin
@Composable
fun ArticleListScreen(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()
    
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(
            count = articles.itemCount,
            key = articles.itemKey { it.id }
        ) { index ->
            val article = articles[index]
            if (article != null) {
                ArticleCard(article = article)
            } else {
                ArticlePlaceholder()
            }
        }
    }
}
```

### 处理加载状态

```kotlin
@Composable
fun ArticleListWithStates(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()
    
    Box(modifier = Modifier.fillMaxSize()) {
        LazyColumn(
            modifier = Modifier.fillMaxSize(),
            contentPadding = PaddingValues(16.dp)
        ) {
            items(
                count = articles.itemCount,
                key = articles.itemKey { it.id }
            ) { index ->
                articles[index]?.let { ArticleCard(it) }
            }
            
            // 底部加载状态
            when (val state = articles.loadState.append) {
                is LoadState.Loading -> {
                    item {
                        Box(
                            modifier = Modifier.fillMaxWidth().padding(16.dp),
                            contentAlignment = Alignment.Center
                        ) {
                            CircularProgressIndicator()
                        }
                    }
                }
                is LoadState.Error -> {
                    item {
                        ErrorItem(
                            message = state.error.localizedMessage ?: "加载失败",
                            onRetry = { articles.retry() }
                        )
                    }
                }
                else -> {}
            }
        }
        
        // 初始加载状态
        when (val state = articles.loadState.refresh) {
            is LoadState.Loading -> {
                CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
            }
            is LoadState.Error -> {
                ErrorContent(
                    message = state.error.localizedMessage ?: "加载失败",
                    onRetry = { articles.refresh() },
                    modifier = Modifier.align(Alignment.Center)
                )
            }
            else -> {}
        }
        
        // 空数据状态
        if (articles.loadState.refresh is LoadState.NotLoading && articles.itemCount == 0) {
            EmptyContent(modifier = Modifier.align(Alignment.Center))
        }
    }
}

@Composable
fun ErrorItem(message: String, onRetry: () -> Unit) {
    Row(
        modifier = Modifier.fillMaxWidth().padding(16.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text(message, color = MaterialTheme.colorScheme.error)
        TextButton(onClick = onRetry) {
            Text("重试")
        }
    }
}
```

## 六、下拉刷新

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RefreshableArticleList(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()
    val isRefreshing = articles.loadState.refresh is LoadState.Loading
    
    val pullToRefreshState = rememberPullToRefreshState()
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .nestedScroll(pullToRefreshState.nestedScrollConnection)
    ) {
        LazyColumn(modifier = Modifier.fillMaxSize()) {
            items(
                count = articles.itemCount,
                key = articles.itemKey { it.id }
            ) { index ->
                articles[index]?.let { ArticleCard(it) }
            }
        }
        
        if (pullToRefreshState.isRefreshing) {
            LaunchedEffect(true) {
                articles.refresh()
            }
        }
        
        LaunchedEffect(isRefreshing) {
            if (!isRefreshing) {
                pullToRefreshState.endRefresh()
            }
        }
        
        PullToRefreshContainer(
            state = pullToRefreshState,
            modifier = Modifier.align(Alignment.TopCenter)
        )
    }
}
```

## 七、数据转换

### map 转换

```kotlin
val articles: Flow<PagingData<ArticleUiModel>> = repository
    .getArticleStream(query)
    .map { pagingData ->
        pagingData.map { article ->
            ArticleUiModel(
                id = article.id,
                title = article.title,
                formattedDate = formatDate(article.publishedAt),
                thumbnailUrl = article.imageUrl
            )
        }
    }
    .cachedIn(viewModelScope)
```

### 插入分隔符

```kotlin
val articlesWithSeparators: Flow<PagingData<UiModel>> = repository
    .getArticleStream(query)
    .map { pagingData ->
        pagingData.map { UiModel.ArticleItem(it) }
    }
    .map { pagingData ->
        pagingData.insertSeparators { before, after ->
            if (before == null) {
                return@insertSeparators null
            }
            if (after == null) {
                return@insertSeparators null
            }
            // 按日期插入分隔符
            if (before.article.date != after.article.date) {
                UiModel.DateSeparator(after.article.date)
            } else {
                null
            }
        }
    }
    .cachedIn(viewModelScope)

sealed class UiModel {
    data class ArticleItem(val article: Article) : UiModel()
    data class DateSeparator(val date: String) : UiModel()
}

// 在 LazyColumn 中使用
items(
    count = items.itemCount,
    key = items.itemKey { 
        when (it) {
            is UiModel.ArticleItem -> "article_${it.article.id}"
            is UiModel.DateSeparator -> "separator_${it.date}"
        }
    }
) { index ->
    when (val item = items[index]) {
        is UiModel.ArticleItem -> ArticleCard(item.article)
        is UiModel.DateSeparator -> DateHeader(item.date)
        null -> ArticlePlaceholder()
    }
}
```

## 八、RemoteMediator（离线优先）

```kotlin
@OptIn(ExperimentalPagingApi::class)
class ArticleRemoteMediator(
    private val apiService: ApiService,
    private val database: AppDatabase
) : RemoteMediator<Int, ArticleEntity>() {
    
    private val articleDao = database.articleDao()
    
    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, ArticleEntity>
    ): MediatorResult {
        return try {
            val page = when (loadType) {
                LoadType.REFRESH -> 1
                LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
                LoadType.APPEND -> {
                    val lastItem = state.lastItemOrNull()
                        ?: return MediatorResult.Success(endOfPaginationReached = true)
                    lastItem.page + 1
                }
            }
            
            val response = apiService.getArticles(page = page)
            
            database.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    articleDao.clearAll()
                }
                
                val entities = response.articles.map { article ->
                    article.toEntity(page = page)
                }
                articleDao.insertAll(entities)
            }
            
            MediatorResult.Success(endOfPaginationReached = response.articles.isEmpty())
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}

// Repository
@OptIn(ExperimentalPagingApi::class)
fun getArticleStream(): Flow<PagingData<ArticleEntity>> {
    return Pager(
        config = PagingConfig(pageSize = 20),
        remoteMediator = ArticleRemoteMediator(apiService, database),
        pagingSourceFactory = { database.articleDao().pagingSource() }
    ).flow
}
```

## 九、测试

### 测试 PagingSource

```kotlin
@Test
fun `load returns page when successful`() = runTest {
    val mockApi = mockk<ApiService>()
    coEvery { mockApi.searchArticles(any(), any(), any()) } returns ArticleResponse(
        articles = listOf(Article(id = "1", title = "Test"))
    )
    
    val pagingSource = ArticlePagingSource(mockApi, "test")
    
    val result = pagingSource.load(
        LoadParams.Refresh(key = null, loadSize = 20, placeholdersEnabled = false)
    )
    
    assertTrue(result is LoadResult.Page)
    assertEquals(1, (result as LoadResult.Page).data.size)
}
```

### 测试 ViewModel

```kotlin
@Test
fun `articles flow emits paging data`() = runTest {
    val viewModel = ArticleViewModel(FakeArticleRepository())
    
    viewModel.search("kotlin")
    
    val differ = AsyncPagingDataDiffer(
        diffCallback = ArticleDiffCallback(),
        updateCallback = NoopListCallback()
    )
    
    val job = launch {
        viewModel.articles.collectLatest { pagingData ->
            differ.submitData(pagingData)
        }
    }
    
    advanceUntilIdle()
    
    assertTrue(differ.snapshot().items.isNotEmpty())
    
    job.cancel()
}
```

## 十、最佳实践

- ✅ 使用 `cachedIn(viewModelScope)` 缓存 PagingData
- ✅ 使用 `itemKey` 提供稳定的 key
- ✅ 处理所有加载状态（Loading、Error、Empty）
- ✅ 实现下拉刷新
- ✅ 使用 RemoteMediator 实现离线优先
- ✅ 使用 `insertSeparators` 添加分隔符
- ✅ 在 `debounce` 后执行搜索
- ✅ 设置合理的 `prefetchDistance`

## 总结

Paging 3 与 Compose 集成的核心要点：

- **PagingSource**：定义数据加载逻辑
- **Pager**：配置分页参数
- **collectAsLazyPagingItems()**：在 Compose 中收集数据
- **LoadState**：处理加载、错误、空数据状态
- **RemoteMediator**：实现离线优先架构
- **cachedIn**：在配置变更后保留数据

Paging 3 是处理大量数据的最佳选择，结合 Compose 可以轻松构建流畅的无限滚动体验。

---

*© 2024 Fidroid. [返回首页](../index.html)*

