# Compose + ViewModel：状态管理与最佳实践

> **发布日期**: 2024-06-15  
> **阅读时间**: 约 24 分钟  
> **标签**: ViewModel, StateFlow, UiState, SavedStateHandle

ViewModel 是 Android 架构组件的核心，负责管理 UI 相关数据和业务逻辑。本文将深入讲解 ViewModel 在 Compose 项目中的最佳实践。

## 一、ViewModel 基础

### 创建 ViewModel

```kotlin
class ArticleViewModel : ViewModel() {
    
    private val _uiState = MutableStateFlow(ArticleUiState())
    val uiState: StateFlow<ArticleUiState> = _uiState.asStateFlow()
    
    fun loadArticles() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            
            try {
                val articles = repository.getArticles()
                _uiState.update { 
                    it.copy(isLoading = false, articles = articles) 
                }
            } catch (e: Exception) {
                _uiState.update { 
                    it.copy(isLoading = false, error = e.message) 
                }
            }
        }
    }
}

data class ArticleUiState(
    val isLoading: Boolean = false,
    val articles: List<Article> = emptyList(),
    val error: String? = null
)
```

### 在 Compose 中使用

```kotlin
@Composable
fun ArticleScreen(
    viewModel: ArticleViewModel = viewModel()
) {
    val uiState by viewModel.uiState.collectAsState()
    
    LaunchedEffect(Unit) {
        viewModel.loadArticles()
    }
    
    when {
        uiState.isLoading -> LoadingScreen()
        uiState.error != null -> ErrorScreen(uiState.error!!)
        else -> ArticleList(uiState.articles)
    }
}
```

## 二、Hilt 集成

### 配置 Hilt

```kotlin
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val repository: ArticleRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    // ...
}

// 在 Compose 中使用
@Composable
fun ArticleScreen(
    viewModel: ArticleViewModel = hiltViewModel()
) {
    // ...
}
```

### 作用域控制

```kotlin
// Activity 作用域
@Composable
fun ScreenA(
    viewModel: SharedViewModel = hiltViewModel()
) { }

@Composable
fun ScreenB(
    viewModel: SharedViewModel = hiltViewModel()
) { }

// 共享 ViewModel（使用 activityViewModels 等效）
@Composable
fun SharedViewModelExample() {
    val navController = rememberNavController()
    val parentEntry = remember { navController.getBackStackEntry("parent") }
    
    NavHost(navController, startDestination = "child") {
        composable("child") {
            val sharedViewModel: SharedViewModel = hiltViewModel(parentEntry)
            ChildScreen(sharedViewModel)
        }
    }
}
```

## 三、状态管理模式

### 单一 UiState

```kotlin
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(ProfileUiState())
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()
    
    init {
        loadProfile()
    }
    
    fun loadProfile() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            
            userRepository.getProfile()
                .onSuccess { user ->
                    _uiState.update { 
                        it.copy(isLoading = false, user = user, error = null) 
                    }
                }
                .onFailure { e ->
                    _uiState.update { 
                        it.copy(isLoading = false, error = e.message) 
                    }
                }
        }
    }
    
    fun updateName(name: String) {
        _uiState.update { it.copy(user = it.user?.copy(name = name)) }
    }
    
    fun saveProfile() {
        val user = _uiState.value.user ?: return
        
        viewModelScope.launch {
            _uiState.update { it.copy(isSaving = true) }
            
            userRepository.updateProfile(user)
                .onSuccess {
                    _uiState.update { it.copy(isSaving = false, saveSuccess = true) }
                }
                .onFailure { e ->
                    _uiState.update { it.copy(isSaving = false, error = e.message) }
                }
        }
    }
    
    fun clearError() {
        _uiState.update { it.copy(error = null) }
    }
    
    fun resetSaveSuccess() {
        _uiState.update { it.copy(saveSuccess = false) }
    }
}

data class ProfileUiState(
    val isLoading: Boolean = false,
    val isSaving: Boolean = false,
    val user: User? = null,
    val error: String? = null,
    val saveSuccess: Boolean = false
)
```

### 多数据流组合

```kotlin
@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val articleRepository: ArticleRepository
) : ViewModel() {
    
    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing: StateFlow<Boolean> = _isRefreshing.asStateFlow()
    
    val uiState: StateFlow<DashboardUiState> = combine(
        userRepository.currentUser,
        articleRepository.recentArticles,
        _isRefreshing
    ) { user, articles, isRefreshing ->
        DashboardUiState(
            user = user,
            recentArticles = articles,
            isRefreshing = isRefreshing
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = DashboardUiState()
    )
    
    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            try {
                userRepository.refresh()
                articleRepository.refresh()
            } finally {
                _isRefreshing.value = false
            }
        }
    }
}
```

## 四、SavedStateHandle

### 保存和恢复状态

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: ArticleRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    // 自动保存到 SavedState
    val searchQuery = savedStateHandle.getStateFlow("query", "")
    
    private val _searchResults = MutableStateFlow<List<Article>>(emptyList())
    val searchResults: StateFlow<List<Article>> = _searchResults.asStateFlow()
    
    init {
        // 观察搜索词变化
        viewModelScope.launch {
            searchQuery
                .debounce(300)
                .filter { it.isNotBlank() }
                .collectLatest { query ->
                    search(query)
                }
        }
    }
    
    fun updateQuery(query: String) {
        savedStateHandle["query"] = query
    }
    
    private suspend fun search(query: String) {
        _searchResults.value = repository.search(query)
    }
}
```

### 接收导航参数

```kotlin
@HiltViewModel
class ArticleDetailViewModel @Inject constructor(
    private val repository: ArticleRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    // 从导航参数获取
    private val articleId: String = savedStateHandle.get<String>("articleId") 
        ?: throw IllegalArgumentException("articleId is required")
    
    private val _uiState = MutableStateFlow<ArticleDetailUiState>(ArticleDetailUiState.Loading)
    val uiState: StateFlow<ArticleDetailUiState> = _uiState.asStateFlow()
    
    init {
        loadArticle()
    }
    
    private fun loadArticle() {
        viewModelScope.launch {
            repository.getArticle(articleId)
                .onSuccess { article ->
                    _uiState.value = ArticleDetailUiState.Success(article)
                }
                .onFailure { e ->
                    _uiState.value = ArticleDetailUiState.Error(e.message ?: "Unknown error")
                }
        }
    }
}

sealed interface ArticleDetailUiState {
    data object Loading : ArticleDetailUiState
    data class Success(val article: Article) : ArticleDetailUiState
    data class Error(val message: String) : ArticleDetailUiState
}
```

## 五、事件处理

### 一次性事件

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val authRepository: AuthRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState: StateFlow<LoginUiState> = _uiState.asStateFlow()
    
    // 使用 Channel 发送一次性事件
    private val _events = Channel<LoginEvent>()
    val events = _events.receiveAsFlow()
    
    fun login(email: String, password: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            
            authRepository.login(email, password)
                .onSuccess {
                    _events.send(LoginEvent.NavigateToHome)
                }
                .onFailure { e ->
                    _events.send(LoginEvent.ShowError(e.message ?: "Login failed"))
                }
            
            _uiState.update { it.copy(isLoading = false) }
        }
    }
}

sealed interface LoginEvent {
    data object NavigateToHome : LoginEvent
    data class ShowError(val message: String) : LoginEvent
}

// 在 Compose 中处理事件
@Composable
fun LoginScreen(
    viewModel: LoginViewModel = hiltViewModel(),
    onNavigateToHome: () -> Unit
) {
    val uiState by viewModel.uiState.collectAsState()
    val snackbarHostState = remember { SnackbarHostState() }
    
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is LoginEvent.NavigateToHome -> onNavigateToHome()
                is LoginEvent.ShowError -> {
                    snackbarHostState.showSnackbar(event.message)
                }
            }
        }
    }
    
    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        LoginContent(
            uiState = uiState,
            onLogin = viewModel::login,
            modifier = Modifier.padding(padding)
        )
    }
}
```

## 六、表单处理

```kotlin
@HiltViewModel
class RegistrationViewModel @Inject constructor(
    private val authRepository: AuthRepository
) : ViewModel() {
    
    var email by mutableStateOf("")
        private set
    
    var password by mutableStateOf("")
        private set
    
    var confirmPassword by mutableStateOf("")
        private set
    
    private val _uiState = MutableStateFlow(RegistrationUiState())
    val uiState: StateFlow<RegistrationUiState> = _uiState.asStateFlow()
    
    val isFormValid: Boolean
        get() = email.isNotBlank() && 
                password.length >= 8 && 
                password == confirmPassword
    
    val emailError: String?
        get() = when {
            email.isBlank() -> null
            !android.util.Patterns.EMAIL_ADDRESS.matcher(email).matches() -> "无效的邮箱格式"
            else -> null
        }
    
    val passwordError: String?
        get() = when {
            password.isBlank() -> null
            password.length < 8 -> "密码至少8位"
            else -> null
        }
    
    val confirmPasswordError: String?
        get() = when {
            confirmPassword.isBlank() -> null
            confirmPassword != password -> "两次密码不一致"
            else -> null
        }
    
    fun updateEmail(value: String) { email = value }
    fun updatePassword(value: String) { password = value }
    fun updateConfirmPassword(value: String) { confirmPassword = value }
    
    fun register() {
        if (!isFormValid) return
        
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            
            authRepository.register(email, password)
                .onSuccess {
                    _uiState.update { it.copy(isLoading = false, isSuccess = true) }
                }
                .onFailure { e ->
                    _uiState.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }
}
```

## 七、测试

### 单元测试

```kotlin
class ArticleViewModelTest {
    
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()
    
    private lateinit var viewModel: ArticleViewModel
    private lateinit var repository: FakeArticleRepository
    
    @Before
    fun setup() {
        repository = FakeArticleRepository()
        viewModel = ArticleViewModel(repository)
    }
    
    @Test
    fun `loadArticles updates state with articles`() = runTest {
        // Given
        val articles = listOf(Article(id = "1", title = "Test"))
        repository.setArticles(articles)
        
        // When
        viewModel.loadArticles()
        
        // Then
        val state = viewModel.uiState.value
        assertFalse(state.isLoading)
        assertEquals(articles, state.articles)
        assertNull(state.error)
    }
    
    @Test
    fun `loadArticles handles error`() = runTest {
        // Given
        repository.setShouldFail(true)
        
        // When
        viewModel.loadArticles()
        
        // Then
        val state = viewModel.uiState.value
        assertFalse(state.isLoading)
        assertNotNull(state.error)
    }
}

class MainDispatcherRule : TestWatcher() {
    private val testDispatcher = UnconfinedTestDispatcher()
    
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

## 八、最佳实践

- ✅ 使用 StateFlow 暴露 UI 状态
- ✅ 使用 `update` 函数更新状态
- ✅ 使用 `stateIn` 组合多个 Flow
- ✅ 使用 Channel 发送一次性事件
- ✅ 使用 SavedStateHandle 保存状态
- ✅ 表单验证逻辑放在 ViewModel
- ✅ 使用 Hilt 注入依赖
- ✅ 编写单元测试

## 总结

ViewModel 与 Compose 集成的核心要点：

- **StateFlow**：响应式状态管理
- **UiState**：单一状态源
- **SavedStateHandle**：进程重建恢复
- **Channel**：一次性事件
- **Hilt**：依赖注入
- **测试**：Mock Repository 测试逻辑

正确使用 ViewModel 是构建健壮应用的基础。

---

*© 2024 Fidroid. [返回首页](../index.html)*


