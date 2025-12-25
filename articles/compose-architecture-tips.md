# Compose 代码组织与架构实战技巧

> **发布日期**: 2024-12-23  
> **阅读时间**: 约 40-45 分钟  
> **标签**: Compose, 代码组织, 架构设计, Clean Architecture, 最佳实践, Hilt

随着 Compose 项目规模的增长，如何组织代码、设计合理的架构以及确保代码的可维护性成为了核心问题。本文总结了在实际大型项目中使用 Compose 的架构设计和代码组织模式，涵盖从组件拆分到整体架构设计的全方位技巧。

## 📚 官方参考

- [Guide to app architecture - Android Developers](https://developer.android.com/topic/architecture)
- [Architecture in Jetpack Compose - Android Developers](https://developer.android.com/jetpack/compose/architecture)
- [Modern Android Development (MAD) skills - Architecture](https://developer.android.com/modern-android-development/architecture)

## 目录

- [I. 模块化与包结构](#i-模块化与包结构)
- [II. UI 组件化与复用](#ii-ui-组件化与复用)
- [III. ViewModel 与状态管理架构](#iii-viewmodel-与状态管理架构)
- [IV. 依赖注入 (Hilt) 实战](#iv-依赖注入-hilt-实战)
- [V. 导航与路由架构](#v-导航与路由架构)
- [VI. 业务逻辑与 UI 分离](#vi-业务逻辑与-ui-分离)
- [VII. 主题与样式管理](#vii-主题与样式管理)
- [VIII. 测试与可维护性](#viii-测试与可维护性)

## I. 模块化与包结构

合理的模块化和包结构是大型项目的基础。

### 1.1 按功能模块化 (Feature-based Modularization)

推荐按功能划分模块，而非按组件类型划分。

```text
// ✅ 推荐的包结构
com.example.myapp
├── common              // 公共模块
│   ├── ui              // 通用组件
│   ├── theme           // 主题
│   └── util            // 工具类
├── data                // 数据层
│   ├── repository      // 仓库
│   ├── source          // 数据源
│   └── model           // 数据模型
├── domain              // 领域层
│   ├── usecase         // 用例
│   └── model           // 领域模型
└── features            // 功能模块
    ├── home            // 首页功能
    │   ├── ui          // UI 组件
    │   ├── viewmodel   // ViewModel
    │   └── model       // UI 模型
    └── profile         // 个人中心功能
```

**为什么重要？**
- 降低耦合度：模块之间职责清晰
- 提升构建速度：按需编译
- 方便并行开发：不同团队负责不同模块

### 1.2 内部/外部 API 分离

在模块内区分公开的 API 和内部实现的私有类。

```kotlin
// features/home/ui/HomeContent.kt
@Composable
fun HomeContent(...) { ... }  // 公开组件

// features/home/ui/internal/Header.kt
@Composable
internal fun HomeHeader(...) { ... }  // 内部组件，不暴露给外部
```

## II. UI 组件化与复用

Compose 的声明式特性让组件化变得非常容易，但过度拆分或不合理的拆分也会带来问题。

### 2.1 原子化组件设计

参考原子设计理论 (Atomic Design) 拆分组件。

- **Atoms (原子)**：最小单位，如 Button, Text, Icon
- **Molecules (分子)**：原子组合，如 SearchBar (Text + Icon)
- **Organisms (有机体)**：分子组合，如 NavBar, ProductCard
- **Templates (模板)**：页面结构布局
- **Pages (页面)**：具体内容的页面

```kotlin
// atoms/MyButton.kt
@Composable
fun MyButton(onClick: () -> Unit, text: String) { ... }

// molecules/SearchBar.kt
@Composable
fun SearchBar(query: String, onQueryChange: (String) -> Unit) {
    Row {
        MyIcon(Icons.Default.Search)
        MyTextField(query, onQueryChange)
    }
}
```

### 2.2 提取通用组件库

将跨功能的 UI 组件提取到独立的 `common:ui` 模块中。

```kotlin
// common-ui/src/main/java/com/example/common/ui/LoadingView.kt
@Composable
fun LoadingView(modifier: Modifier = Modifier) {
    Box(modifier = modifier.fillMaxSize()) {
        CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
    }
}
```

### 2.3 使用 CompositionLocal 传递全局配置

```kotlin
// common-ui/src/main/java/com/example/common/ui/LocalAppConfig.kt
val LocalAppConfig = compositionLocalOf { AppConfig() }

@Composable
fun MyAppTheme(content: @Composable () -> Unit) {
    val config = remember { AppConfig() }
    CompositionLocalProvider(LocalAppConfig provides config) {
        content()
    }
}
```

## III. ViewModel 与状态管理架构

ViewModel 是连接 UI 和业务逻辑的核心。

### 3.1 统一 UI State 模型

使用 Sealed Class 或 Data Class 定义完整的 UI 状态。

```kotlin
// ✅ 推荐的 UI State 定义
sealed class HomeUiState {
    object Loading : HomeUiState()
    data class Success(val products: List<Product>, val banner: List<Banner>) : HomeUiState()
    data class Error(val message: String) : HomeUiState()
}

class HomeViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<HomeUiState>(HomeUiState.Loading)
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()
    
    init {
        loadData()
    }
    
    private fun loadData() {
        // 加载数据并更新 _uiState
    }
}
```

### 3.2 状态提升 (State Hoisting) 模式

```kotlin
// ✅ 状态提升模式：Stateless Composable
@Composable
fun ProductList(
    products: List<Product>,
    onProductClick: (Product) -> Unit
) {
    LazyColumn {
        items(products) { product ->
            ProductRow(product, onClick = { onProductClick(product) })
        }
    }
}

// Stateful Composable：负责状态管理
@Composable
fun ProductScreen(viewModel: ProductViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    ProductList(
        products = (uiState as? ProductUiState.Success)?.products ?: emptyList(),
        onProductClick = { viewModel.onProductSelected(it) }
    )
}
```

## IV. 依赖注入 (Hilt) 实战

Hilt 是 Android 官方推荐的 DI 框架，在 Compose 中应用广泛。

### 4.1 注入 ViewModel

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    // ...
}

@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    // 使用注入的 ViewModel
}
```

### 4.2 注入 Repository 和 UseCase

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideUserRepository(api: ApiService): UserRepository {
        return UserRepositoryImpl(api)
    }
}
```

## V. 导航与路由架构

Compose Navigation 提供了类型安全的导航方案。

### 5.1 类型安全路由 (Type-safe Navigation)

在 Compose 2.8.0+ 中推荐使用类型安全的路由。

```kotlin
// 定义路由模型
@Serializable
object Home

@Serializable
data class Profile(val userId: String)

// 配置导航
@Composable
fun AppNavigation(navController: NavHostController) {
    NavHost(navController = navController, startDestination = Home) {
        composable<Home> {
            HomeScreen(onNavigateToProfile = { userId ->
                navController.navigate(Profile(userId))
            })
        }
        composable<Profile> { backStackEntry ->
            val profile: Profile = backStackEntry.toRoute()
            ProfileScreen(userId = profile.userId)
        }
    }
}
```

### 5.2 抽离导航逻辑

将导航路径定义在独立的模块或文件中。

```kotlin
// navigation/Screen.kt
sealed class Screen(val route: String) {
    object Home : Screen("home")
    object Profile : Screen("profile/{userId}") {
        fun createRoute(userId: String) = "profile/$userId"
    }
}
```

## VI. 业务逻辑与 UI 分离

保持 UI 层纯粹，业务逻辑下沉。

### 6.1 使用 UseCase (Interactor) 模式

```kotlin
class GetProductListUseCase @Inject constructor(
    private val repository: ProductRepository
) {
    operator fun invoke(): Flow<List<Product>> = repository.getProducts()
}

@HiltViewModel
class ProductViewModel @Inject constructor(
    private val getProductListUseCase: GetProductListUseCase
) : ViewModel() {
    // 调用 UseCase 而非直接调用 Repository
}
```

### 6.2 区分 UI Model 和 Data Model

```kotlin
// Data Model (Repository 层)
data class ProductEntity(val id: Int, val name: String, val price: Double)

// UI Model (UI 层)
data class ProductUiModel(val id: Int, val title: String, val formattedPrice: String)

// 转换逻辑
fun ProductEntity.toUiModel() = ProductUiModel(
    id = id,
    title = name,
    formattedPrice = "$${price}"
)
```

## VII. 主题与样式管理

集中管理主题和样式。

### 7.1 自定义 Design System

```kotlin
// theme/MyTheme.kt
object MyTheme {
    val colors: MyColors
        @Composable
        get() = LocalMyColors.current
        
    val typography: MyTypography
        @Composable
        get() = LocalMyTypography.current
}

@Composable
fun MyTheme(
    colors: MyColors = MyDefaultColors,
    content: @Composable () -> Unit
) {
    CompositionLocalProvider(
        LocalMyColors provides colors,
        // ...
    ) {
        content()
    }
}
```

### 7.2 样式扩展

```kotlin
// UI 组件中使用自定义主题
@Composable
fun StyledButton(text: String) {
    Button(
        colors = ButtonDefaults.buttonColors(
            backgroundColor = MyTheme.colors.primary
        )
    ) {
        Text(text, style = MyTheme.typography.button)
    }
}
```

## VIII. 测试与可维护性

架构设计的最终目标是可测试和可维护。

### 8.1 UI 组件的可测试性

```kotlin
@Test
fun testProductRow() {
    composeTestRule.setContent {
        ProductRow(
            product = ProductUiModel(1, "Test", "$10"),
            onClick = { }
        )
    }
    
    composeTestRule.onNodeWithText("Test").assertIsDisplayed()
}
```

### 8.2 ViewModel 的单元测试

```kotlin
@Test
fun testViewModelLoadData() = runTest {
    val viewModel = ProductViewModel(fakeUseCase)
    viewModel.loadData()
    
    val state = viewModel.uiState.value
    assert(state is ProductUiState.Success)
}
```

## 总结

Compose 代码组织与架构设计的核心原则：

- ✅ **按功能模块化**：保持模块职责单一
- ✅ **状态提升**：让组件变得无状态、易测试
- ✅ **单一数据源**：UI State 是 UI 的唯一真理
- ✅ **DI 实战**：利用 Hilt 简化依赖管理
- ✅ **逻辑下沉**：业务逻辑放在 UseCase 和 ViewModel
- ✅ **统一主题**：集中管理 Design System

掌握这些技巧，可以构建出结构清晰、易于扩展和维护的大型 Compose 应用。

---

## 推荐阅读

- [Guide to app architecture - Android Developers](https://developer.android.com/topic/architecture)
- [Architecture in Jetpack Compose - Android Developers](https://developer.android.com/jetpack/compose/architecture)
- [Modern Android Development (MAD) skills - Architecture](https://developer.android.com/modern-android-development/architecture)
