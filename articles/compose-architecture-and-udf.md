# 用 Jetpack Compose 搭建可维护的应用架构

> **发布日期**: 2024-03-22  
> **阅读时间**: 约 18 分钟  
> **标签**: Jetpack Compose, 架构设计, UDF / MVI, 状态管理

Compose 自带声明式 UI 能力，但并不会替你解决「状态如何在应用中流动」这一根本问题。如果仍然把所有逻辑塞进 Activity / Fragment，那么换了 UI 框架也很难获得长期可维护的架构。本文试图结合 ViewModel、单向数据流与 Navigation，给出一套适合在中大型项目中落地的 Compose 架构思路。

## 1. 为什么在 Compose 时代更需要架构

在经典 View 时代，很多「架构不太优雅」的项目仍然能靠经验和约定运行下去，因为 UI 更新是显式的命令式调用。而在 Compose 中，如果没有清晰的状态边界与数据流，**Recomposition 很容易放大代码异味**：

- 同一份状态被多个地方修改，导致 UI 行为不可预测。
- 不同 Composable 之间通过回调层层传递，形成新的「回调地狱」。
- 导航、权限、错误提示等「横切逻辑」散落在各个 Composable 内部。

> 💡 简单说：Compose 把 UI 写得更快了，但如果没有合理的架构，技术债也会累积得更快。

## 2. UDF：用单向数据流串起状态变化

单向数据流（Unidirectional Data Flow，简称 UDF）并不是 Compose 专属概念，但与声明式 UI 的心智模型天然契合。它通常包含三个元素：

- **State**：当前 UI 所需的所有数据快照。
- **Event / Intent**：用户交互或系统回调产生的「意图」。
- **Reducer / Handler**：根据事件更新状态、触发副作用的逻辑。

### 2.1 一个典型的 UDF 状态模型

```kotlin
data class ArticleListState(
    val isLoading: Boolean = false,
    val articles: List<Article> = emptyList(),
    val error: String? = null,
    val filterKeyword: String = ""
)

sealed interface ArticleListEvent {
    data class OnKeywordChange(val keyword: String) : ArticleListEvent
    data object OnRefresh : ArticleListEvent
    data class OnArticleClick(val id: String) : ArticleListEvent
}
```

Compose 端只关心 `ArticleListState` 如何映射成 UI，而不关心状态是如何被修改的；所有修改都统一通过 `ArticleListEvent` 进入 ViewModel。

### 2.2 ViewModel 中的事件处理

```kotlin
class ArticleListViewModel(
    private val repository: ArticleRepository,
    private val navigator: ArticleNavigator
) : ViewModel() {

    private val _state = MutableStateFlow(ArticleListState())
    val state: StateFlow<ArticleListState> = _state.asStateFlow()

    fun onEvent(event: ArticleListEvent) {
        when (event) {
            is ArticleListEvent.OnKeywordChange -> {
                _state.update { it.copy(filterKeyword = event.keyword) }
            }
            ArticleListEvent.OnRefresh -> loadArticles()
            is ArticleListEvent.OnArticleClick -> {
                navigator.openArticleDetail(event.id)
            }
        }
    }

    private fun loadArticles() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            runCatching { repository.fetchArticles() }
                .onSuccess { list ->
                    _state.update { it.copy(isLoading = false, articles = list) }
                }
                .onFailure { t ->
                    _state.update { it.copy(isLoading = false, error = t.message) }
                }
        }
    }
}
```

注意几件事：

- 所有状态更新集中在 ViewModel 内部，Compose 只负责展示。
- 导航通过 `navigator` 这一抽象来实现，避免 Composable 直接依赖 NavController。
- 网络请求等副作用都放在 ViewModel 中，便于测试与重用。

## 3. Compose 层：无状态 UI + 状态入口

从 Compose 的角度来看，理想的结构是：

- **无状态 UI**：仅通过参数接收状态与回调，不直接持有 ViewModel。
- **状态入口**：少量「桥接层」负责从 ViewModel 收集状态，并调用无状态 UI。

### 3.1 状态入口 Composable

```kotlin
@Composable
fun ArticleListRoute(
    viewModel: ArticleListViewModel = viewModel()
) {
    val state by viewModel.state.collectAsState()

    ArticleListScreen(
        state = state,
        onEvent = viewModel::onEvent
    )
}
```

### 3.2 无状态 UI

```kotlin
@Composable
fun ArticleListScreen(
    state: ArticleListState,
    onEvent: (ArticleListEvent) -> Unit
) {
    Column {
        SearchBar(
            keyword = state.filterKeyword,
            onKeywordChange = { onEvent(ArticleListEvent.OnKeywordChange(it)) },
            onSearch = { onEvent(ArticleListEvent.OnRefresh) }
        )

        if (state.isLoading) {
            CircularProgressIndicator()
        } else if (state.error != null) {
            ErrorView(
                message = state.error,
                onRetry = { onEvent(ArticleListEvent.OnRefresh) }
            )
        } else {
            ArticleList(
                articles = state.articles,
                onArticleClick = { id -> onEvent(ArticleListEvent.OnArticleClick(id)) }
            )
        }
    }
}
```

这样的拆分让 UI 与状态管理之间的边界极其清晰：**任何时候只要你拿到一份完整的 `ArticleListState`，就可以独立预览 / snapshot 这一屏的 UI。**

## 4. Navigation 与跨屏状态

在 Compose 世界中，推荐使用官方的 `Navigation-Compose`，并遵守几个简单规则：

- 路由字符串尽量语义化，例如 `"article/list"`、`"article/detail/{id}"`。
- 不要在 Composable 内部随意 new NavController，而是通过上层注入或抽象导航接口。
- 跨屏共享的状态（例如登录信息、全局配置）放在更高层级的 ViewModel，而不是通过 navArgs 硬传。

```kotlin
@Composable
fun AppNavHost(
    navController: NavHostController,
    modifier: Modifier = Modifier
) {
    NavHost(
        navController = navController,
        startDestination = "article/list",
        modifier = modifier
    ) {
        composable("article/list") {
            ArticleListRoute()
        }
        composable(
            route = "article/detail/{id}",
            arguments = listOf(navArgument("id") { type = NavType.StringType })
        ) { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id") ?: return@composable
            ArticleDetailRoute(articleId = id)
        }
    }
}
```

## 5. 关于性能：架构与 Recomposition 的协作

很多性能问题并不是 Compose 本身带来的，而是由于架构设计导致「状态过于粗粒度」：当一个巨大的 State 对象被修改时，整棵 UI 树都被迫 Recompose。

一些实践建议：

- 拆分 State，使局部 UI 只依赖必要字段。
- 使用 `derivedStateOf` 把计算型状态缓存下来，避免在 Recomposition 中重复做重计算。
- 合理使用 `LazyColumn` 等懒加载组件，并为列表元素提供稳定的 key。

```kotlin
val filteredArticles by remember(state.articles, state.filterKeyword) {
    derivedStateOf {
        state.articles.filter { article ->
            article.title.contains(state.filterKeyword, ignoreCase = true)
        }
    }
}
```

架构的目标不是「追求极致的 Recomposition 次数」，而是让你能清楚地知道**是谁**在更新**哪一块** UI，并在必要时有能力优化它。

## 6. 总结：一套可以长期演进的 Compose 架构

将本文内容压缩成一个 checklist，大致可以是：

- 所有屏幕都有明确的 State / Event 定义。
- ViewModel 是状态与副作用的唯一「入口」，Composable 不直接做网络 / 数据库访问。
- 大部分 UI 组件保持「无状态 + 参数驱动」，方便复用与预览。
- Navigation 有统一管理入口，路由命名语义化，跨屏状态通过更高层级的 ViewModel 管理。
- 针对性能问题，有手段定位和优化 Recomposition 热点。

在此基础上，你可以根据团队规模和项目特点，继续引入更多工程实践（例如模块化、功能边界拆分、统一 Design System 等）。Compose 提供了强大的 UI 能力，而一套清晰的架构，则能保证这些能力在未来几年里继续为你的项目稳定服务。

---

*本文首发于 Fidroid · Android & Jetpack Compose 博客。*


