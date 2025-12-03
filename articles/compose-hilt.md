# Hilt 与 Compose 集成：依赖注入最佳实践

> **发布日期**: 2024-05-20  
> **阅读时间**: 约 24 分钟  
> **标签**: Hilt, 依赖注入, ViewModel, Navigation

Hilt 是 Android 官方推荐的依赖注入框架，与 Compose 配合使用可以构建可测试、可维护的应用架构。本文将深入讲解 Hilt 在 Compose 项目中的最佳实践。

## 一、Hilt 基础配置

### 添加依赖

```kotlin
// build.gradle.kts (Project)
plugins {
    id("com.google.dagger.hilt.android") version "2.48" apply false
}

// build.gradle.kts (App)
plugins {
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.48")
    ksp("com.google.dagger:hilt-android-compiler:2.48")
    
    // Hilt 与 Compose Navigation 集成
    implementation("androidx.hilt:hilt-navigation-compose:1.1.0")
}
```

### Application 配置

```kotlin
@HiltAndroidApp
class MyApplication : Application()
```

### Activity 配置

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyAppTheme {
                AppNavigation()
            }
        }
    }
}
```

## 二、hiltViewModel

### 基本用法

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()
    
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            val users = repository.getUsers()
            _uiState.update { it.copy(isLoading = false, users = users) }
        }
    }
}

data class HomeUiState(
    val isLoading: Boolean = false,
    val users: List<User> = emptyList()
)

@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()
    
    LaunchedEffect(Unit) {
        viewModel.loadUsers()
    }
    
    when {
        uiState.isLoading -> LoadingIndicator()
        else -> UserList(users = uiState.users)
    }
}
```

### SavedStateHandle 注入

```kotlin
@HiltViewModel
class DetailViewModel @Inject constructor(
    private val repository: ItemRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    // 从 Navigation 参数获取
    private val itemId: String = savedStateHandle.get<String>("itemId") ?: ""
    
    // 或使用 StateFlow
    val itemIdFlow: StateFlow<String> = savedStateHandle.getStateFlow("itemId", "")
    
    private val _item = MutableStateFlow<Item?>(null)
    val item: StateFlow<Item?> = _item.asStateFlow()
    
    init {
        loadItem()
    }
    
    private fun loadItem() {
        viewModelScope.launch {
            _item.value = repository.getItem(itemId)
        }
    }
}
```

## 三、与 Navigation Compose 集成

### NavGraph 中使用 hiltViewModel

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            // 每个 destination 自动获取正确作用域的 ViewModel
            HomeScreen(
                onNavigateToDetail = { id ->
                    navController.navigate("detail/$id")
                }
            )
        }
        
        composable(
            route = "detail/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.StringType })
        ) {
            DetailScreen(
                onNavigateBack = { navController.popBackStack() }
            )
        }
    }
}

@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel(),
    onNavigateToDetail: (String) -> Unit
) {
    // ...
}

@Composable
fun DetailScreen(
    viewModel: DetailViewModel = hiltViewModel(),
    onNavigateBack: () -> Unit
) {
    // itemId 自动从 SavedStateHandle 获取
}
```

### 共享 ViewModel（嵌套导航）

```kotlin
@Composable
fun ParentScreen(
    // 获取与当前 NavBackStackEntry 关联的 ViewModel
    viewModel: SharedViewModel = hiltViewModel()
) {
    val navController = rememberNavController()
    
    NavHost(navController, startDestination = "tab1") {
        composable("tab1") { entry ->
            // 在子页面中获取父级的 ViewModel
            val parentEntry = remember(entry) {
                navController.getBackStackEntry("parent_route")
            }
            val sharedViewModel: SharedViewModel = hiltViewModel(parentEntry)
            
            Tab1Screen(sharedViewModel)
        }
        
        composable("tab2") { entry ->
            val parentEntry = remember(entry) {
                navController.getBackStackEntry("parent_route")
            }
            val sharedViewModel: SharedViewModel = hiltViewModel(parentEntry)
            
            Tab2Screen(sharedViewModel)
        }
    }
}
```

## 四、Module 定义

### 提供 Repository

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}

// 或使用 @Provides
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
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

### 不同作用域

```kotlin
// Singleton - 应用全局单例
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(context, AppDatabase::class.java, "app.db").build()
    }
}

// ActivityRetainedComponent - Activity 重建后保留
@Module
@InstallIn(ActivityRetainedComponent::class)
object ActivityRetainedModule {
    @Provides
    @ActivityRetainedScoped
    fun provideAnalytics(): Analytics {
        return Analytics()
    }
}

// ViewModelComponent - ViewModel 作用域
@Module
@InstallIn(ViewModelComponent::class)
object ViewModelModule {
    @Provides
    @ViewModelScoped
    fun provideUseCases(repository: Repository): UseCases {
        return UseCases(repository)
    }
}
```

## 五、Assisted Inject

当需要同时注入依赖和运行时参数时：

```kotlin
class UserDetailViewModel @AssistedInject constructor(
    private val repository: UserRepository,
    @Assisted private val userId: String
) : ViewModel() {
    
    @AssistedFactory
    interface Factory {
        fun create(userId: String): UserDetailViewModel
    }
    
    // ViewModel 逻辑...
}

// 使用
@Composable
fun UserDetailScreen(userId: String) {
    val viewModel = assistedViewModel(userId) { factory: UserDetailViewModel.Factory ->
        factory.create(userId)
    }
    // ...
}

// 辅助函数
@Composable
inline fun <reified VM : ViewModel, reified F> assistedViewModel(
    key: Any,
    crossinline factory: (F) -> VM
): VM {
    val factoryProducer = EntryPointAccessors.fromActivity(
        LocalContext.current as Activity,
        AssistedFactoryEntryPoint::class.java
    )
    
    return viewModel(key = key.toString()) {
        @Suppress("UNCHECKED_CAST")
        factory(factoryProducer.getFactory() as F)
    }
}
```

## 六、测试

### ViewModel 单元测试

```kotlin
@HiltAndroidTest
class HomeViewModelTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @Inject
    lateinit var repository: FakeUserRepository
    
    private lateinit var viewModel: HomeViewModel
    
    @Before
    fun setup() {
        hiltRule.inject()
        viewModel = HomeViewModel(repository)
    }
    
    @Test
    fun `loadUsers updates state correctly`() = runTest {
        // Given
        val expectedUsers = listOf(User("1", "Alice"), User("2", "Bob"))
        repository.setUsers(expectedUsers)
        
        // When
        viewModel.loadUsers()
        
        // Then
        assertEquals(expectedUsers, viewModel.uiState.value.users)
    }
}

// 测试模块替换
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [RepositoryModule::class]
)
abstract class FakeRepositoryModule {
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: FakeUserRepository
    ): UserRepository
}
```

### Compose UI 测试

```kotlin
@HiltAndroidTest
class HomeScreenTest {
    
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule(order = 1)
    val composeRule = createAndroidComposeRule<MainActivity>()
    
    @Inject
    lateinit var repository: FakeUserRepository
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @Test
    fun homeScreen_displaysUsers() {
        // Given
        repository.setUsers(listOf(User("1", "Alice")))
        
        // When
        composeRule.setContent {
            HomeScreen()
        }
        
        // Then
        composeRule.onNodeWithText("Alice").assertIsDisplayed()
    }
}
```

## 七、最佳实践

### 1. ViewModel 作用域

```kotlin
// ✅ 好：使用 hiltViewModel() 自动管理作用域
@Composable
fun MyScreen(viewModel: MyViewModel = hiltViewModel()) {
    // ...
}

// ❌ 避免：手动创建 ViewModel
@Composable
fun MyScreen() {
    val viewModel = remember { MyViewModel(repository) }
}
```

### 2. 避免在 Composable 中直接注入

```kotlin
// ❌ 避免：在 Composable 中直接使用 @Inject
@Composable
fun BadExample() {
    // 这样不行！Composable 不是 Hilt 组件
}

// ✅ 好：通过 ViewModel 注入依赖
@HiltViewModel
class GoodViewModel @Inject constructor(
    private val repository: Repository
) : ViewModel()

@Composable
fun GoodExample(viewModel: GoodViewModel = hiltViewModel()) {
    // 通过 ViewModel 访问依赖
}
```

### 3. 分层架构

```
UI Layer (Compose)
    ↓ hiltViewModel()
ViewModel Layer
    ↓ @Inject constructor
Domain Layer (UseCases)
    ↓ @Inject constructor
Data Layer (Repository)
    ↓ @Inject constructor
Data Source (API, Database)
```

### 4. 接口抽象

```kotlin
// 定义接口
interface UserRepository {
    suspend fun getUsers(): List<User>
}

// 实现
class UserRepositoryImpl @Inject constructor(
    private val api: ApiService,
    private val dao: UserDao
) : UserRepository {
    override suspend fun getUsers(): List<User> {
        // 实现逻辑
    }
}

// 绑定
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```

## 八、常见问题

### 1. ViewModel 创建失败

```kotlin
// 确保 ViewModel 使用 @HiltViewModel 注解
@HiltViewModel
class MyViewModel @Inject constructor(...) : ViewModel()

// 确保 Activity 使用 @AndroidEntryPoint
@AndroidEntryPoint
class MainActivity : ComponentActivity()
```

### 2. 循环依赖

```kotlin
// 使用 Provider 或 Lazy 打破循环
class ServiceA @Inject constructor(
    private val serviceBProvider: Provider<ServiceB>
) {
    fun doSomething() {
        serviceBProvider.get().method()
    }
}
```

### 3. 测试时替换依赖

```kotlin
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [ProductionModule::class]
)
object TestModule {
    @Provides
    @Singleton
    fun provideRepository(): Repository = FakeRepository()
}
```

## 总结

Hilt 与 Compose 集成的核心要点：

- **hiltViewModel()**：在 Composable 中获取 ViewModel
- **@HiltViewModel**：标记可注入的 ViewModel
- **SavedStateHandle**：自动获取 Navigation 参数
- **作用域**：Singleton、ActivityRetained、ViewModelScoped
- **测试**：使用 @TestInstallIn 替换模块
- **最佳实践**：分层架构、接口抽象、避免直接注入

正确使用 Hilt 可以让你的 Compose 应用更加模块化、可测试、可维护。

---

*© 2024 Fidroid. [返回首页](../index.html)*

