# 从 View 到 Compose：重新理解 Android UI 心智模型

> **发布日期**: 2024-03-25  
> **阅读时间**: 约 15 分钟  
> **标签**: Jetpack Compose, 声明式 UI, Android, 工程实践

很多团队在引入 Jetpack Compose 时的第一反应是「把 XML 换成 Kotlin」，但真正的迁移难点从来不在语法，而在于你如何理解 UI 与状态的关系。本文希望通过与经典 View 系统的对比，帮助你建立一套清晰的 Compose 心智模型。

## 1. 经典 View 系统的隐形复杂度

在经典的 View 系统里，我们习惯通过 `findViewById` 或 ViewBinding 拿到具体 View 引用，然后在各种生命周期节点里调用 `setText`、`setVisibility`、`setEnabled` 等方法去「命令式」地修改 UI。

这种模式有几个常见问题：

- UI 与状态强耦合在 Activity / Fragment 中，逻辑容易变成「上帝类」。
- View 的中间状态很多，例如「半初始化」「已经被回收但还持有引用」。
- 随着需求增长，`if-else` 与 `setXxx` 散落在各处，很难追踪状态变化链路。

> 💡 一句话概括：经典 View 模式是「我告诉系统你现在要长成什么样」，而不是「我描述在某个状态下你应该是什么样」。

## 2. Compose 的核心：状态驱动 UI

Jetpack Compose 是一个典型的声明式 UI 框架：你不再直接操作 View 层级，而是描述「在某个状态下 UI 应该长成什么样」。当状态发生变化时，Compose 会自动触发 Recomposition，重新执行对应的 `@Composable` 函数来更新 UI。

### 2.1 从「操作 View」到「组合函数」

```kotlin
// 经典 View 写法（伪代码）
textView = findViewById(R.id.title)
button = findViewById(R.id.btn)

fun render(isLoading: Boolean) {
    if (isLoading) {
        button.isEnabled = false
        textView.text = "Loading..."
    } else {
        button.isEnabled = true
        textView.text = "Loaded"
    }
}
```

```kotlin
// Compose 写法
@Composable
fun TitleWithButton(isLoading: Boolean, onClick: () -> Unit) {
    Column {
        Text(text = if (isLoading) "Loading..." else "Loaded")
        Button(
            enabled = !isLoading,
            onClick = onClick
        ) {
            Text("刷新")
        }
    }
}
```

在 Compose 中，我们不再关心「这个 Button 已经被 disable 过几次」，而是只关心「当前 `isLoading` 为 true 时 Button 应该 disabled」。状态成为了 UI 的唯一输入来源。

### 2.2 Recomposition 并不是重新创建 View

很多人初看 Compose 时会担心：*每次状态变化都重新执行 Composable，会不会非常耗性能？*

这里需要记住一个关键点：**Recomposition 不是重新创建整个视图树，而是对比前后状态，做「差量更新」。**

Compose 内部维护了一份 Slot Table（可以粗略理解为 UI 结构的有序记录），当状态变化时，它会根据你写的 Composable 树计算出需要更新的部分，而不是暴力重建。

## 3. 状态应该放在哪里？

用好 Compose 的关键在于：**把状态放在「恰好够用」的地方**。这和经典 View 时代「尽量少地持有引用」类似，只是现在的单位从 View 变成了「状态 + Composable」。

### 3.1 UI 状态：remember 与 rememberSaveable

```kotlin
@Composable
fun SearchBar(
    modifier: Modifier = Modifier,
    onSearch: (String) -> Unit
) {
    var keyword by rememberSaveable { mutableStateOf("") }

    TextField(
        value = keyword,
        onValueChange = { newValue ->
            keyword = newValue
        },
        modifier = modifier.fillMaxWidth(),
        placeholder = { Text("搜索文章...") },
        trailingIcon = {
            IconButton(onClick = { onSearch(keyword) }) {
                Icon(Icons.Default.Search, contentDescription = null)
            }
        }
    )
}
```

这里的 `keyword` 就是典型的「UI 状态」，它只在 Composable 内部有意义，因此使用 `rememberSaveable` 即可，同时还能在配置变化（如旋转）后自动恢复。

### 3.2 屏幕级状态：ViewModel + UDF

对于一个完整的页面，推荐使用 ViewModel 来承载屏幕级状态，并通过单向数据流（UDF）暴露给 Compose：

```kotlin
data class ArticleUiState(
    val isLoading: Boolean = false,
    val articles: List<Article> = emptyList(),
    val error: String? = null
)

class ArticleViewModel(
    private val repository: ArticleRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(ArticleUiState())
    val uiState: StateFlow<ArticleUiState> = _uiState.asStateFlow()

    fun load() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            runCatching { repository.fetchArticles() }
                .onSuccess { list ->
                    _uiState.update { it.copy(isLoading = false, articles = list) }
                }
                .onFailure { throwable ->
                    _uiState.update { it.copy(isLoading = false, error = throwable.message) }
                }
        }
    }
}
```

```kotlin
@Composable
fun ArticleScreen(
    viewModel: ArticleViewModel = viewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    ArticleScreen(uiState = uiState, onRetry = { viewModel.load() })
}
```

在这个模式下，Compose 完全「订阅」于 ViewModel 暴露的状态，使得 UI 变成一个「纯函数」：相同的输入总是渲染出相同的视图。

## 4. 渐进式迁移：不必一次性「推翻重来」

引入 Compose 并不意味着你要立刻抛弃所有 XML 布局。更现实的做法是：

- 从局部组件开始，比如列表 item、弹窗、表单等可独立复用的 UI 模块。
- 在 Fragment 中通过 `ComposeView` 或 Activity 中通过 `setContent` 逐步扩展覆盖范围。
- 为新模块设计基于 Compose 的 Design System，让「新老 UI」在视觉上保持一致。

> 💡 一个简单的经验法则：**当你在 XML 里写复杂的自定义 View 时，往往就是可以考虑用 Compose 重写的好时机。**

## 5. 总结：用正确的心智模型拥抱 Compose

可以用三句话来总结本文的核心观点：

- 不要把 Compose 当成「写 UI 的另一种语法」，而是一个完整的声明式 UI 框架。
- UI 不再是「被命令驱动的对象」，而是「被状态描述的结果」。
- 通过合理划分状态边界（Composable 内部 / 屏幕级 / 应用级），可以让复杂 UI 在 Compose 下保持清晰可维护。

后续文章会进一步展开如何在真实项目中基于 Compose 搭建架构（包含 Navigation、状态容器、Design System 等主题），欢迎在首页继续浏览 Compose 系列内容。

---

*本文首发于 Fidroid · Android & Jetpack Compose 博客。*

