# Compose Navigation 最佳实践：路由、参数与深层链接

> **发布日期**: 2024-03-30  
> **阅读时间**: 约 22 分钟  
> **标签**: Navigation, Deep Link, Type-Safe Routes, ViewModel

导航是每个 Android 应用的核心组成部分。Navigation Compose 提供了一套声明式的导航 API，与 Compose 的编程模型完美契合。本文将深入探讨如何设计清晰的路由结构、传递参数、处理深层链接，以及如何与 ViewModel 优雅集成。

## 一、Navigation Compose 基础设置

首先添加依赖并创建基本的导航结构：

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.navigation:navigation-compose:2.7.7")
}
```

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(
                onNavigateToDetail = { id ->
                    navController.navigate("detail/$id")
                }
            )
        }

        composable(
            route = "detail/{itemId}",
            arguments = listOf(
                navArgument("itemId") { type = NavType.StringType }
            )
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getString("itemId")
            DetailScreen(itemId = itemId)
        }
    }
}
```

## 二、类型安全的路由设计

使用字符串拼接路由容易出错。推荐使用 sealed class 或 object 来定义类型安全的路由：

```kotlin
sealed class Screen(
    val route: String
) {
    data object Home : Screen("home")
    data object Profile : Screen("profile")
    data object Settings : Screen("settings")

    // 带参数的路由
    data class Detail(val itemId: String) : Screen("detail/$itemId") {
        companion object {
            const val ROUTE_PATTERN = "detail/{itemId}"
            const val ARG_ITEM_ID = "itemId"
        }
    }

    // 带可选参数的路由
    data class Search(
        val query: String? = null,
        val category: String? = null
    ) : Screen(
        buildString {
            append("search")
            val params = mutableListOf<String>()
            query?.let { params.add("query=$it") }
            category?.let { params.add("category=$it") }
            if (params.isNotEmpty()) {
                append("?")
                append(params.joinToString("&"))
            }
        }
    ) {
        companion object {
            const val ROUTE_PATTERN = "search?query={query}&category={category}"
        }
    }
}
```

### 使用类型安全路由

```kotlin
NavHost(
    navController = navController,
    startDestination = Screen.Home.route
) {
    composable(Screen.Home.route) {
        HomeScreen(
            onItemClick = { itemId ->
                navController.navigate(Screen.Detail(itemId).route)
            }
        )
    }

    composable(
        route = Screen.Detail.ROUTE_PATTERN,
        arguments = listOf(
            navArgument(Screen.Detail.ARG_ITEM_ID) {
                type = NavType.StringType
            }
        )
    ) { backStackEntry ->
        val itemId = backStackEntry.arguments
            ?.getString(Screen.Detail.ARG_ITEM_ID) ?: return@composable
        DetailScreen(itemId = itemId)
    }
}
```

## 三、参数传递的多种方式

| 方式 | 语法 | 适用场景 |
|-----|------|---------|
| **路径参数** | `detail/{id}` | 必需参数，如详情页 ID |
| **查询参数** | `search?q={query}` | 可选参数，如筛选条件 |
| **SavedStateHandle** | ViewModel 中获取 | 复杂数据，配合 ViewModel |

### 路径参数（必需）

```kotlin
composable(
    route = "user/{userId}/post/{postId}",
    arguments = listOf(
        navArgument("userId") { type = NavType.LongType },
        navArgument("postId") { type = NavType.LongType }
    )
) { backStackEntry ->
    val userId = backStackEntry.arguments?.getLong("userId") ?: 0L
    val postId = backStackEntry.arguments?.getLong("postId") ?: 0L
    PostDetailScreen(userId, postId)
}
```

### 查询参数（可选）

```kotlin
composable(
    route = "products?category={category}&sort={sort}",
    arguments = listOf(
        navArgument("category") {
            type = NavType.StringType
            nullable = true
            defaultValue = null
        },
        navArgument("sort") {
            type = NavType.StringType
            defaultValue = "newest"
        }
    )
) { backStackEntry ->
    val category = backStackEntry.arguments?.getString("category")
    val sort = backStackEntry.arguments?.getString("sort") ?: "newest"
    ProductListScreen(category, sort)
}
```

## 四、与 ViewModel 集成

Navigation Compose 与 ViewModel 的集成非常自然。每个导航目的地可以拥有独立的 ViewModel，其生命周期与 BackStack Entry 绑定：

```kotlin
composable(
    route = Screen.Detail.ROUTE_PATTERN,
    arguments = listOf(
        navArgument(Screen.Detail.ARG_ITEM_ID) { type = NavType.StringType }
    )
) { backStackEntry ->
    // ViewModel 自动获取路由参数
    val viewModel: DetailViewModel = hiltViewModel()
    DetailScreen(viewModel = viewModel)
}

// DetailViewModel.kt
@HiltViewModel
class DetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repository: ItemRepository
) : ViewModel() {

    // 从 SavedStateHandle 获取导航参数
    private val itemId: String = checkNotNull(
        savedStateHandle[Screen.Detail.ARG_ITEM_ID]
    )

    val uiState: StateFlow<DetailUiState> = repository
        .getItemFlow(itemId)
        .map { item -> DetailUiState(item = item) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = DetailUiState(isLoading = true)
        )
}
```

## 五、嵌套导航与 Bottom Navigation

复杂应用通常需要嵌套导航图，比如底部导航栏的每个 Tab 都有独立的导航栈：

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()

    Scaffold(
        bottomBar = {
            NavigationBar {
                val navBackStackEntry by navController.currentBackStackEntryAsState()
                val currentDestination = navBackStackEntry?.destination

                listOf(
                    BottomNavItem.Home,
                    BottomNavItem.Search,
                    BottomNavItem.Profile
                ).forEach { item ->
                    NavigationBarItem(
                        icon = { Icon(item.icon, contentDescription = item.label) },
                        label = { Text(item.label) },
                        selected = currentDestination?.hierarchy?.any {
                            it.route == item.route
                        } == true,
                        onClick = {
                            navController.navigate(item.route) {
                                // 避免重复创建相同目的地
                                popUpTo(navController.graph.findStartDestination().id) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        }
                    )
                }
            }
        }
    ) { innerPadding ->
        NavHost(
            navController = navController,
            startDestination = "home_graph",
            modifier = Modifier.padding(innerPadding)
        ) {
            // Home Tab 的嵌套导航图
            navigation(
                startDestination = "home",
                route = "home_graph"
            ) {
                composable("home") { HomeScreen(navController) }
                composable("home/detail/{id}") { HomeDetailScreen() }
            }

            // Search Tab 的嵌套导航图
            navigation(
                startDestination = "search",
                route = "search_graph"
            ) {
                composable("search") { SearchScreen(navController) }
                composable("search/results") { SearchResultsScreen() }
            }

            // Profile Tab
            composable("profile") { ProfileScreen() }
        }
    }
}
```

## 六、深层链接（Deep Links）

Navigation Compose 原生支持深层链接，让用户可以通过 URL 直接跳转到应用内的特定页面：

```kotlin
composable(
    route = "product/{productId}",
    arguments = listOf(
        navArgument("productId") { type = NavType.StringType }
    ),
    deepLinks = listOf(
        navDeepLink {
            uriPattern = "https://fidroid.com/product/{productId}"
        },
        navDeepLink {
            uriPattern = "fidroid://product/{productId}"
        }
    )
) { backStackEntry ->
    val productId = backStackEntry.arguments?.getString("productId")
    ProductScreen(productId)
}
```

### AndroidManifest.xml 配置

```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="fidroid.com" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="fidroid" />
    </intent-filter>
</activity>
```

## 七、导航事件与结果回传

有时需要从目标页面返回结果给上一个页面。Navigation Compose 提供了 `previousBackStackEntry` 来实现：

```kotlin
// 发送结果的页面
@Composable
fun SelectColorScreen(navController: NavController) {
    val colors = listOf(Color.Red, Color.Green, Color.Blue)

    LazyColumn {
        items(colors) { color ->
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(60.dp)
                    .background(color)
                    .clickable {
                        // 将结果设置到上一个页面的 SavedStateHandle
                        navController.previousBackStackEntry
                            ?.savedStateHandle
                            ?.set("selected_color", color.toArgb())
                        navController.popBackStack()
                    }
            )
        }
    }
}

// 接收结果的页面
@Composable
fun SettingsScreen(navController: NavController) {
    val selectedColor = navController.currentBackStackEntry
        ?.savedStateHandle
        ?.getStateFlow<Int?>("selected_color", null)
        ?.collectAsState()

    Column {
        Text("当前选中颜色: ${selectedColor?.value}")
        Button(onClick = { navController.navigate("select_color") }) {
            Text("选择颜色")
        }
    }
}
```

## 八、导航动画

Navigation Compose 支持自定义页面切换动画：

```kotlin
composable(
    route = "detail/{id}",
    enterTransition = {
        slideInHorizontally(
            initialOffsetX = { it },
            animationSpec = tween(300)
        ) + fadeIn(animationSpec = tween(300))
    },
    exitTransition = {
        slideOutHorizontally(
            targetOffsetX = { -it / 3 },
            animationSpec = tween(300)
        ) + fadeOut(animationSpec = tween(300))
    },
    popEnterTransition = {
        slideInHorizontally(
            initialOffsetX = { -it / 3 },
            animationSpec = tween(300)
        ) + fadeIn(animationSpec = tween(300))
    },
    popExitTransition = {
        slideOutHorizontally(
            targetOffsetX = { it },
            animationSpec = tween(300)
        ) + fadeOut(animationSpec = tween(300))
    }
) {
    DetailScreen()
}
```

> 💡 **全局导航动画**  
> 如果希望所有页面使用统一的动画，可以在 NavHost 级别设置默认的 enterTransition、exitTransition 等参数。

## 九、常见问题与最佳实践

### 1. 避免在 Composable 中持有 NavController 引用

将导航操作通过回调传递，让 Screen Composable 保持无状态：

```kotlin
// ✅ 推荐：通过回调传递导航意图
@Composable
fun HomeScreen(
    onNavigateToDetail: (String) -> Unit
) {
    Button(onClick = { onNavigateToDetail("item-123") }) {
        Text("查看详情")
    }
}

// ❌ 避免：直接依赖 NavController
@Composable
fun HomeScreen(navController: NavController) {
    Button(onClick = { navController.navigate("detail/item-123") }) {
        Text("查看详情")
    }
}
```

### 2. 使用 launchSingleTop 避免重复导航

```kotlin
navController.navigate(route) {
    launchSingleTop = true  // 如果目标已在栈顶，不会重复创建
}
```

### 3. 正确处理系统返回键

Navigation Compose 会自动处理系统返回键。如果需要自定义返回行为：

```kotlin
BackHandler(enabled = hasUnsavedChanges) {
    // 显示确认对话框
    showExitDialog = true
}
```

## 总结

Navigation Compose 提供了一套现代化的声明式导航方案：

- 使用 **sealed class** 定义类型安全的路由
- 通过 **路径参数**和**查询参数**传递数据
- 利用 **SavedStateHandle** 在 ViewModel 中获取参数
- 使用**嵌套导航图**组织复杂的应用结构
- 配置**深层链接**支持外部跳转
- 自定义**页面切换动画**提升体验

下一步建议：结合 [Compose 性能优化](compose-performance-deep-dive.md)，确保导航过程中的流畅体验。

---

*© 2024 Fidroid. [返回首页](../index.html)*

