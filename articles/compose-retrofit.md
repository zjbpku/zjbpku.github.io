# Compose + Retrofit：网络请求与状态处理

> **发布日期**: 2024-06-09  
> **阅读时间**: 约 24 分钟  
> **标签**: Retrofit, 网络请求, UiState, Error Handling

Retrofit 是 Android 最流行的网络请求库，与 Kotlin 协程和 Compose 完美配合。本文将深入讲解如何在 Compose 项目中优雅地处理网络请求和状态管理。

## 一、基础配置

### 添加依赖

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    
    // 可选：Kotlin Serialization
    implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
}
```

### 定义 API 接口

```kotlin
interface ApiService {
    
    @GET("articles")
    suspend fun getArticles(
        @Query("page") page: Int = 1,
        @Query("limit") limit: Int = 20
    ): ArticleResponse
    
    @GET("articles/{id}")
    suspend fun getArticle(@Path("id") id: String): Article
    
    @POST("articles")
    suspend fun createArticle(@Body article: CreateArticleRequest): Article
    
    @PUT("articles/{id}")
    suspend fun updateArticle(
        @Path("id") id: String,
        @Body article: UpdateArticleRequest
    ): Article
    
    @DELETE("articles/{id}")
    suspend fun deleteArticle(@Path("id") id: String)
}

@Serializable
data class ArticleResponse(
    val articles: List<Article>,
    val total: Int,
    val page: Int
)

@Serializable
data class Article(
    val id: String,
    val title: String,
    val content: String,
    val author: String,
    val createdAt: String
)
```

### 创建 Retrofit 实例

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .addInterceptor { chain ->
                val request = chain.request().newBuilder()
                    .addHeader("Authorization", "Bearer ${getToken()}")
                    .build()
                chain.proceed(request)
            }
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

## 二、UI 状态封装

### UiState 密封类

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String, val cause: Throwable? = null) : UiState<Nothing>
}

// 扩展函数
fun <T> UiState<T>.isLoading() = this is UiState.Loading
fun <T> UiState<T>.isSuccess() = this is UiState.Success
fun <T> UiState<T>.isError() = this is UiState.Error

fun <T> UiState<T>.getOrNull(): T? = (this as? UiState.Success)?.data
fun <T> UiState<T>.getOrDefault(default: T): T = getOrNull() ?: default
```

### Result 包装

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
    
    fun getOrNull(): T? = (this as? Success)?.data
    
    fun <R> map(transform: (T) -> R): Result<R> = when (this) {
        is Success -> Success(transform(data))
        is Error -> this
    }
}

suspend fun <T> safeApiCall(apiCall: suspend () -> T): Result<T> {
    return try {
        Result.Success(apiCall())
    } catch (e: Exception) {
        Result.Error(e)
    }
}
```

## 三、Repository 层

```kotlin
class ArticleRepository @Inject constructor(
    private val apiService: ApiService
) {
    suspend fun getArticles(page: Int = 1): Result<List<Article>> {
        return safeApiCall {
            apiService.getArticles(page).articles
        }
    }
    
    suspend fun getArticle(id: String): Result<Article> {
        return safeApiCall {
            apiService.getArticle(id)
        }
    }
    
    suspend fun createArticle(title: String, content: String): Result<Article> {
        return safeApiCall {
            apiService.createArticle(CreateArticleRequest(title, content))
        }
    }
    
    suspend fun deleteArticle(id: String): Result<Unit> {
        return safeApiCall {
            apiService.deleteArticle(id)
        }
    }
}
```

## 四、ViewModel 实现

```kotlin
@HiltViewModel
class ArticleListViewModel @Inject constructor(
    private val repository: ArticleRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState<List<Article>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Article>>> = _uiState.asStateFlow()
    
    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing: StateFlow<Boolean> = _isRefreshing.asStateFlow()
    
    init {
        loadArticles()
    }
    
    fun loadArticles() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            
            when (val result = repository.getArticles()) {
                is Result.Success -> {
                    _uiState.value = UiState.Success(result.data)
                }
                is Result.Error -> {
                    _uiState.value = UiState.Error(
                        message = result.exception.toUserMessage(),
                        cause = result.exception
                    )
                }
            }
        }
    }
    
    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            
            when (val result = repository.getArticles()) {
                is Result.Success -> {
                    _uiState.value = UiState.Success(result.data)
                }
                is Result.Error -> {
                    // 刷新失败时可以显示 Snackbar 而不替换现有数据
                }
            }
            
            _isRefreshing.value = false
        }
    }
    
    fun deleteArticle(id: String) {
        viewModelScope.launch {
            when (repository.deleteArticle(id)) {
                is Result.Success -> {
                    // 从列表中移除
                    val currentList = (_uiState.value as? UiState.Success)?.data ?: return@launch
                    _uiState.value = UiState.Success(currentList.filter { it.id != id })
                }
                is Result.Error -> {
                    // 显示错误
                }
            }
        }
    }
}

// 错误消息转换
fun Throwable.toUserMessage(): String {
    return when (this) {
        is IOException -> "网络连接失败，请检查网络"
        is HttpException -> when (code()) {
            401 -> "登录已过期，请重新登录"
            403 -> "没有访问权限"
            404 -> "请求的资源不存在"
            500 -> "服务器错误，请稍后重试"
            else -> "请求失败 (${code()})"
        }
        else -> message ?: "未知错误"
    }
}
```

## 五、在 Compose 中使用

### 基础列表页面

```kotlin
@Composable
fun ArticleListScreen(
    viewModel: ArticleListViewModel = hiltViewModel(),
    onArticleClick: (String) -> Unit
) {
    val uiState by viewModel.uiState.collectAsState()
    val isRefreshing by viewModel.isRefreshing.collectAsState()
    
    ArticleListContent(
        uiState = uiState,
        isRefreshing = isRefreshing,
        onRefresh = { viewModel.refresh() },
        onRetry = { viewModel.loadArticles() },
        onArticleClick = onArticleClick,
        onDeleteClick = { viewModel.deleteArticle(it) }
    )
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ArticleListContent(
    uiState: UiState<List<Article>>,
    isRefreshing: Boolean,
    onRefresh: () -> Unit,
    onRetry: () -> Unit,
    onArticleClick: (String) -> Unit,
    onDeleteClick: (String) -> Unit
) {
    val pullToRefreshState = rememberPullToRefreshState()
    
    Box(modifier = Modifier.fillMaxSize()) {
        when (uiState) {
            is UiState.Loading -> {
                LoadingContent()
            }
            is UiState.Success -> {
                if (uiState.data.isEmpty()) {
                    EmptyContent(onRetry = onRetry)
                } else {
                    PullToRefreshBox(
                        isRefreshing = isRefreshing,
                        onRefresh = onRefresh,
                        state = pullToRefreshState
                    ) {
                        LazyColumn(
                            contentPadding = PaddingValues(16.dp),
                            verticalArrangement = Arrangement.spacedBy(8.dp)
                        ) {
                            items(uiState.data, key = { it.id }) { article ->
                                ArticleCard(
                                    article = article,
                                    onClick = { onArticleClick(article.id) },
                                    onDelete = { onDeleteClick(article.id) }
                                )
                            }
                        }
                    }
                }
            }
            is UiState.Error -> {
                ErrorContent(
                    message = uiState.message,
                    onRetry = onRetry
                )
            }
        }
    }
}

@Composable
fun LoadingContent() {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        CircularProgressIndicator()
    }
}

@Composable
fun EmptyContent(onRetry: () -> Unit) {
    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = Icons.Outlined.Inbox,
            contentDescription = null,
            modifier = Modifier.size(64.dp),
            tint = MaterialTheme.colorScheme.outline
        )
        Spacer(modifier = Modifier.height(16.dp))
        Text("暂无内容")
        Spacer(modifier = Modifier.height(8.dp))
        TextButton(onClick = onRetry) {
            Text("刷新")
        }
    }
}

@Composable
fun ErrorContent(message: String, onRetry: () -> Unit) {
    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = Icons.Outlined.ErrorOutline,
            contentDescription = null,
            modifier = Modifier.size(64.dp),
            tint = MaterialTheme.colorScheme.error
        )
        Spacer(modifier = Modifier.height(16.dp))
        Text(
            text = message,
            textAlign = TextAlign.Center,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        Spacer(modifier = Modifier.height(16.dp))
        Button(onClick = onRetry) {
            Text("重试")
        }
    }
}
```

## 六、详情页面与操作

```kotlin
@HiltViewModel
class ArticleDetailViewModel @Inject constructor(
    private val repository: ArticleRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    private val articleId: String = savedStateHandle.get<String>("articleId") ?: ""
    
    private val _uiState = MutableStateFlow<UiState<Article>>(UiState.Loading)
    val uiState: StateFlow<UiState<Article>> = _uiState.asStateFlow()
    
    private val _deleteState = MutableStateFlow<DeleteState>(DeleteState.Idle)
    val deleteState: StateFlow<DeleteState> = _deleteState.asStateFlow()
    
    init {
        loadArticle()
    }
    
    fun loadArticle() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            
            when (val result = repository.getArticle(articleId)) {
                is Result.Success -> _uiState.value = UiState.Success(result.data)
                is Result.Error -> _uiState.value = UiState.Error(result.exception.toUserMessage())
            }
        }
    }
    
    fun deleteArticle() {
        viewModelScope.launch {
            _deleteState.value = DeleteState.Loading
            
            when (repository.deleteArticle(articleId)) {
                is Result.Success -> _deleteState.value = DeleteState.Success
                is Result.Error -> _deleteState.value = DeleteState.Error("删除失败")
            }
        }
    }
}

sealed interface DeleteState {
    data object Idle : DeleteState
    data object Loading : DeleteState
    data object Success : DeleteState
    data class Error(val message: String) : DeleteState
}

@Composable
fun ArticleDetailScreen(
    viewModel: ArticleDetailViewModel = hiltViewModel(),
    onBack: () -> Unit,
    onDeleted: () -> Unit
) {
    val uiState by viewModel.uiState.collectAsState()
    val deleteState by viewModel.deleteState.collectAsState()
    
    // 处理删除成功
    LaunchedEffect(deleteState) {
        if (deleteState is DeleteState.Success) {
            onDeleted()
        }
    }
    
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("文章详情") },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, "返回")
                    }
                },
                actions = {
                    if (uiState is UiState.Success) {
                        IconButton(
                            onClick = { viewModel.deleteArticle() },
                            enabled = deleteState !is DeleteState.Loading
                        ) {
                            if (deleteState is DeleteState.Loading) {
                                CircularProgressIndicator(
                                    modifier = Modifier.size(24.dp),
                                    strokeWidth = 2.dp
                                )
                            } else {
                                Icon(Icons.Default.Delete, "删除")
                            }
                        }
                    }
                }
            )
        }
    ) { padding ->
        Box(modifier = Modifier.padding(padding)) {
            when (val state = uiState) {
                is UiState.Loading -> LoadingContent()
                is UiState.Success -> ArticleDetailContent(state.data)
                is UiState.Error -> ErrorContent(state.message, viewModel::loadArticle)
            }
        }
    }
}
```

## 七、创建/编辑表单

```kotlin
@HiltViewModel
class CreateArticleViewModel @Inject constructor(
    private val repository: ArticleRepository
) : ViewModel() {
    
    var title by mutableStateOf("")
        private set
    
    var content by mutableStateOf("")
        private set
    
    private val _submitState = MutableStateFlow<SubmitState>(SubmitState.Idle)
    val submitState: StateFlow<SubmitState> = _submitState.asStateFlow()
    
    val isValid: Boolean
        get() = title.isNotBlank() && content.isNotBlank()
    
    fun updateTitle(value: String) { title = value }
    fun updateContent(value: String) { content = value }
    
    fun submit() {
        if (!isValid) return
        
        viewModelScope.launch {
            _submitState.value = SubmitState.Loading
            
            when (val result = repository.createArticle(title, content)) {
                is Result.Success -> {
                    _submitState.value = SubmitState.Success(result.data)
                }
                is Result.Error -> {
                    _submitState.value = SubmitState.Error(result.exception.toUserMessage())
                }
            }
        }
    }
}

sealed interface SubmitState {
    data object Idle : SubmitState
    data object Loading : SubmitState
    data class Success(val article: Article) : SubmitState
    data class Error(val message: String) : SubmitState
}
```

## 八、错误处理与重试

### 使用 Snackbar

```kotlin
@Composable
fun ArticleListWithSnackbar(viewModel: ArticleListViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    val snackbarHostState = remember { SnackbarHostState() }
    
    // 显示错误 Snackbar
    LaunchedEffect(uiState) {
        if (uiState is UiState.Error) {
            val result = snackbarHostState.showSnackbar(
                message = (uiState as UiState.Error).message,
                actionLabel = "重试",
                duration = SnackbarDuration.Long
            )
            if (result == SnackbarResult.ActionPerformed) {
                viewModel.loadArticles()
            }
        }
    }
    
    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { padding ->
        // 内容
    }
}
```

## 九、最佳实践

- ✅ 使用密封类封装 UI 状态
- ✅ Repository 返回 Result 包装类
- ✅ 将异常转换为用户友好的消息
- ✅ 处理 Loading、Success、Error 三种状态
- ✅ 实现下拉刷新
- ✅ 使用 Snackbar 显示非阻塞错误
- ✅ 禁用按钮防止重复提交
- ✅ 使用 Hilt 注入网络依赖

## 总结

Retrofit 与 Compose 集成的核心要点：

- **UiState 密封类**：统一管理加载、成功、错误状态
- **Result 包装**：安全处理网络请求结果
- **Repository 模式**：封装网络请求逻辑
- **错误处理**：转换异常为用户消息
- **状态驱动 UI**：根据状态显示不同内容
- **Hilt 注入**：管理网络依赖

正确处理网络请求状态是构建健壮应用的关键。

---

*© 2024 Fidroid. [返回首页](../index.html)*

