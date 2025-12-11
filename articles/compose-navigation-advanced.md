# Compose Navigation 进阶：深层链接与嵌套导航

> **发布日期**: 2024-06-17  
> **阅读时间**: 约 26 分钟  
> **标签**: Navigation, DeepLink, 嵌套导航, BottomNavigation

Compose Navigation 提供了强大的导航功能，支持深层链接、嵌套导航、类型安全参数等高级特性。本文将深入讲解这些进阶用法。

## 一、类型安全导航（Navigation 2.8+）

### 定义路由

```kotlin
// 使用 Kotlin Serialization
@Serializable
data object Home

@Serializable
data object ArticleList

@Serializable
data class ArticleDetail(val id: String)

@Serializable
data class Search(val query: String = "")
```

### 创建 NavHost

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = Home
    ) {
        composable<Home> {
            HomeScreen(
                onNavigateToArticles = { navController.navigate(ArticleList) }
            )
        }
        
        composable<ArticleList> {
            ArticleListScreen(
                onArticleClick = { id -> 
                    navController.navigate(ArticleDetail(id)) 
                }
            )
        }
        
        composable<ArticleDetail> { backStackEntry ->
            val detail: ArticleDetail = backStackEntry.toRoute()
            ArticleDetailScreen(articleId = detail.id)
        }
    }
}
```

## 二、嵌套导航

### Bottom Navigation

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    
    Scaffold(
        bottomBar = {
            NavigationBar {
                val navBackStackEntry by navController.currentBackStackEntryAsState()
                val currentDestination = navBackStackEntry?.destination
                
                bottomNavItems.forEach { item ->
                    NavigationBarItem(
                        icon = { Icon(item.icon, contentDescription = item.label) },
                        label = { Text(item.label) },
                        selected = currentDestination?.hierarchy?.any { 
                            it.hasRoute(item.route::class) 
                        } == true,
                        onClick = {
                            navController.navigate(item.route) {
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
    ) { padding ->
        NavHost(
            navController = navController,
            startDestination = HomeGraph,
            modifier = Modifier.padding(padding)
        ) {
            homeGraph(navController)
            searchGraph(navController)
            profileGraph(navController)
        }
    }
}

// 定义导航图
@Serializable data object HomeGraph
@Serializable data object SearchGraph  
@Serializable data object ProfileGraph

fun NavGraphBuilder.homeGraph(navController: NavController) {
    navigation<HomeGraph>(startDestination = Home) {
        composable<Home> {
            HomeScreen(
                onArticleClick = { id -> 
                    navController.navigate(ArticleDetail(id)) 
                }
            )
        }
        composable<ArticleDetail> { backStackEntry ->
            val detail: ArticleDetail = backStackEntry.toRoute()
            ArticleDetailScreen(articleId = detail.id)
        }
    }
}
```

### 嵌套 NavHost

```kotlin
@Composable
fun AuthNavigation(onAuthSuccess: () -> Unit) {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = Login
    ) {
        composable<Login> {
            LoginScreen(
                onLoginSuccess = onAuthSuccess,
                onNavigateToRegister = { navController.navigate(Register) },
                onNavigateToForgotPassword = { navController.navigate(ForgotPassword) }
            )
        }
        
        composable<Register> {
            RegisterScreen(
                onRegisterSuccess = onAuthSuccess,
                onBack = { navController.popBackStack() }
            )
        }
        
        composable<ForgotPassword> {
            ForgotPasswordScreen(
                onBack = { navController.popBackStack() }
            )
        }
    }
}

// 主应用根据认证状态切换
@Composable
fun RootNavigation() {
    val isLoggedIn by authViewModel.isLoggedIn.collectAsState()
    
    if (isLoggedIn) {
        MainScreen()
    } else {
        AuthNavigation(onAuthSuccess = { authViewModel.setLoggedIn(true) })
    }
}
```

## 三、深层链接

### 定义深层链接

```kotlin
NavHost(navController, startDestination = Home) {
    composable<Home> { HomeScreen() }
    
    composable<ArticleDetail>(
        deepLinks = listOf(
            navDeepLink<ArticleDetail>(
                basePath = "https://example.com/article"
            )
        )
    ) { backStackEntry ->
        val detail: ArticleDetail = backStackEntry.toRoute()
        ArticleDetailScreen(articleId = detail.id)
    }
}

// AndroidManifest.xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="example.com"
            android:pathPrefix="/article" />
    </intent-filter>
</activity>
```

### 处理深层链接

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            val navController = rememberNavController()
            
            // 处理 Intent
            LaunchedEffect(Unit) {
                handleDeepLink(intent, navController)
            }
            
            AppNavigation(navController)
        }
    }
    
    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        // 处理新的深层链接
    }
    
    private fun handleDeepLink(intent: Intent, navController: NavController) {
        navController.handleDeepLink(intent)
    }
}
```

## 四、导航动画

```kotlin
NavHost(
    navController = navController,
    startDestination = Home,
    enterTransition = {
        slideIntoContainer(
            towards = AnimatedContentTransitionScope.SlideDirection.Left,
            animationSpec = tween(300)
        )
    },
    exitTransition = {
        slideOutOfContainer(
            towards = AnimatedContentTransitionScope.SlideDirection.Left,
            animationSpec = tween(300)
        )
    },
    popEnterTransition = {
        slideIntoContainer(
            towards = AnimatedContentTransitionScope.SlideDirection.Right,
            animationSpec = tween(300)
        )
    },
    popExitTransition = {
        slideOutOfContainer(
            towards = AnimatedContentTransitionScope.SlideDirection.Right,
            animationSpec = tween(300)
        )
    }
) {
    // destinations
}

// 单个目的地自定义动画
composable<ArticleDetail>(
    enterTransition = {
        fadeIn(animationSpec = tween(500)) + 
        slideInVertically(initialOffsetY = { it })
    },
    exitTransition = {
        fadeOut(animationSpec = tween(500))
    }
) {
    // content
}
```

## 五、导航结果返回

```kotlin
// 发送结果
@Composable
fun SelectColorScreen(navController: NavController) {
    val colors = listOf("Red", "Green", "Blue")
    
    LazyColumn {
        items(colors) { color ->
            TextButton(
                onClick = {
                    navController.previousBackStackEntry
                        ?.savedStateHandle
                        ?.set("selected_color", color)
                    navController.popBackStack()
                }
            ) {
                Text(color)
            }
        }
    }
}

// 接收结果
@Composable
fun SettingsScreen(navController: NavController) {
    val savedStateHandle = navController.currentBackStackEntry?.savedStateHandle
    val selectedColor = savedStateHandle
        ?.getStateFlow("selected_color", "")
        ?.collectAsState()
    
    Column {
        Text("Selected: ${selectedColor?.value}")
        
        Button(onClick = { navController.navigate(SelectColor) }) {
            Text("Choose Color")
        }
    }
}
```

## 六、共享 ViewModel

```kotlin
@Composable
fun CheckoutFlow(navController: NavController) {
    val parentEntry = remember(navController) {
        navController.getBackStackEntry(CheckoutGraph)
    }
    
    NavHost(
        navController = navController,
        startDestination = Cart
    ) {
        composable<Cart> {
            val viewModel: CheckoutViewModel = hiltViewModel(parentEntry)
            CartScreen(viewModel)
        }
        
        composable<Shipping> {
            val viewModel: CheckoutViewModel = hiltViewModel(parentEntry)
            ShippingScreen(viewModel)
        }
        
        composable<Payment> {
            val viewModel: CheckoutViewModel = hiltViewModel(parentEntry)
            PaymentScreen(viewModel)
        }
    }
}
```

## 七、条件导航

```kotlin
@Composable
fun ConditionalNavigation() {
    val navController = rememberNavController()
    val authState by authViewModel.authState.collectAsState()
    
    LaunchedEffect(authState) {
        when (authState) {
            is AuthState.Unauthenticated -> {
                navController.navigate(Login) {
                    popUpTo(0) { inclusive = true }
                }
            }
            is AuthState.Authenticated -> {
                navController.navigate(Home) {
                    popUpTo(0) { inclusive = true }
                }
            }
            else -> {}
        }
    }
    
    NavHost(navController, startDestination = Splash) {
        composable<Splash> { SplashScreen() }
        composable<Login> { LoginScreen() }
        composable<Home> { HomeScreen() }
    }
}
```

## 八、最佳实践

- ✅ 使用类型安全路由（Navigation 2.8+）
- ✅ 合理划分导航图层级
- ✅ Bottom Navigation 保存/恢复状态
- ✅ 使用深层链接增强用户体验
- ✅ 添加导航过渡动画
- ✅ 使用 savedStateHandle 返回结果
- ✅ 共享 ViewModel 管理流程状态
- ✅ 处理认证状态的条件导航

## 总结

Compose Navigation 进阶要点：

- **类型安全路由**：编译时检查
- **嵌套导航**：分层管理复杂应用
- **深层链接**：支持外部跳转
- **导航动画**：提升用户体验
- **结果返回**：页面间数据传递
- **共享 ViewModel**：流程状态管理

掌握这些进阶特性可以构建复杂的导航结构。

---

*© 2024 Fidroid. [返回首页](../index.html)*


