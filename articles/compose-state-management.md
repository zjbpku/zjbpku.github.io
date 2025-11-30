# Compose 状态管理完全指南：从 remember 到 StateFlow

> **发布日期**: 2024-04-05  
> **阅读时间**: 约 22 分钟  
> **标签**: State, remember, StateFlow, ViewModel

状态管理是 Compose 应用的核心。理解何时使用 `remember`、`mutableStateOf`、`rememberSaveable`、`StateFlow`，是写出健壮 Compose 代码的关键。

## 一、Compose 中的状态是什么？

在 Compose 中，**状态是随时间变化并驱动 UI 更新的数据**。当状态变化时，Compose 会自动重组（Recomposition）读取该状态的 Composable。

```kotlin
@Composable
fun Counter() {
    // count 是状态，变化时 UI 自动更新
    var count by remember { mutableStateOf(0) }

    Button(onClick = { count++ }) {
        Text("Clicked $count times")
    }
}
```

## 二、状态 API 速查表

| API | 作用 | 生命周期 |
|-----|------|---------|
| `mutableStateOf` | 创建可观察的状态 | 需配合 remember |
| `remember` | 在重组间保持值 | Composable 在组合中存在期间 |
| `rememberSaveable` | 保存到 Bundle | 跨配置变更、进程死亡 |
| `StateFlow` | ViewModel 中的状态流 | ViewModel 生命周期 |
| `collectAsState` | 将 Flow 转为 Compose State | 订阅期间 |

## 三、remember vs rememberSaveable

```kotlin
// remember：配置变更（如旋转屏幕）后会丢失
var count by remember { mutableStateOf(0) }

// rememberSaveable：配置变更后保留
var count by rememberSaveable { mutableStateOf(0) }
```

> 💡 **何时用 rememberSaveable？**  
> 用户输入的表单数据、滚动位置、展开/折叠状态等需要跨配置变更保留的 UI 状态。

### 自定义 Saver

对于复杂对象，需要提供自定义 Saver：

```kotlin
data class City(val name: String, val country: String)

val CitySaver = Saver<City, Bundle>(
    save = { city ->
        Bundle().apply {
            putString("name", city.name)
            putString("country", city.country)
        }
    },
    restore = { bundle ->
        City(
            bundle.getString("name") ?: "",
            bundle.getString("country") ?: ""
        )
    }
)

@Composable
fun CityPicker() {
    var selectedCity by rememberSaveable(stateSaver = CitySaver) {
        mutableStateOf(City("Beijing", "China"))
    }
}
```

## 四、状态提升（State Hoisting）

状态提升是 Compose 的核心模式：将状态移到调用方，让 Composable 变成无状态的纯函数。

```kotlin
// ❌ 有状态组件：难以复用和测试
@Composable
fun ExpandableCard() {
    var expanded by remember { mutableStateOf(false) }
    Card(onClick = { expanded = !expanded }) { ... }
}

// ✅ 无状态组件：状态由调用方控制
@Composable
fun ExpandableCard(
    expanded: Boolean,
    onExpandChange: (Boolean) -> Unit
) {
    Card(onClick = { onExpandChange(!expanded) }) { ... }
}

// 调用方控制状态
@Composable
fun CardList() {
    var expandedIndex by remember { mutableStateOf(-1) }

    items.forEachIndexed { index, item ->
        ExpandableCard(
            expanded = index == expandedIndex,
            onExpandChange = { isExpanded ->
                expandedIndex = if (isExpanded) index else -1
            }
        )
    }
}
```

## 五、ViewModel 与 StateFlow

对于屏幕级状态和业务逻辑，使用 ViewModel + StateFlow：

```kotlin
class ProfileViewModel : ViewModel() {

    private val _uiState = MutableStateFlow(ProfileUiState())
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()

    fun loadProfile() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }

            val profile = repository.getProfile()

            _uiState.update {
                it.copy(isLoading = false, profile = profile)
            }
        }
    }
}

data class ProfileUiState(
    val isLoading: Boolean = false,
    val profile: Profile? = null,
    val error: String? = null
)

@Composable
fun ProfileScreen(viewModel: ProfileViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsState()

    when {
        uiState.isLoading -> LoadingIndicator()
        uiState.error != null -> ErrorMessage(uiState.error)
        uiState.profile != null -> ProfileContent(uiState.profile)
    }
}
```

## 六、状态分层：UI State vs Domain State

| 类型 | 示例 | 存放位置 |
|-----|------|---------|
| **UI 元素状态** | TextField 输入、动画进度 | remember / rememberSaveable |
| **屏幕 UI 状态** | 加载状态、列表数据、错误信息 | ViewModel + StateFlow |
| **业务/领域状态** | 用户登录状态、购物车 | Repository / DataStore |

## 七、处理副作用

状态变化时需要执行副作用（如网络请求、日志），使用 Effect API：

```kotlin
@Composable
fun UserProfile(userId: String) {
    var user by remember { mutableStateOf<User?>(null) }

    // userId 变化时重新加载
    LaunchedEffect(userId) {
        user = repository.getUser(userId)
    }

    user?.let { UserContent(it) }
}
```

### 常用 Effect API

- `LaunchedEffect` - 在协程中执行副作用
- `DisposableEffect` - 需要清理的副作用
- `SideEffect` - 每次重组后执行
- `rememberCoroutineScope` - 获取与 Composable 绑定的 CoroutineScope

## 八、常见陷阱与解决方案

### 1. 忘记使用 remember

```kotlin
// ❌ 每次重组都创建新状态，点击无效
@Composable
fun BrokenCounter() {
    var count = mutableStateOf(0)  // 缺少 remember！
    Button(onClick = { count.value++ }) {
        Text("${count.value}")
    }
}

// ✅ 正确
var count by remember { mutableStateOf(0) }
```

### 2. 在 remember 中使用不稳定的 key

```kotlin
// ❌ 每次重组 listOf 都是新对象
val items = remember(listOf(1, 2, 3)) { ... }

// ✅ 使用稳定的 key 或无 key
val items = remember { listOf(1, 2, 3) }
```

### 3. collectAsState 在错误位置

```kotlin
// ❌ 在 LazyColumn item 中 collect，可能导致问题
LazyColumn {
    items(ids) { id ->
        val item by viewModel.getItemFlow(id).collectAsState(null)
    }
}

// ✅ 在外层 collect，传递数据给 item
val items by viewModel.itemsFlow.collectAsState()
LazyColumn {
    items(items) { item -> ItemRow(item) }
}
```

## 九、状态管理最佳实践

- ✅ UI 元素状态用 `remember`，需要持久化用 `rememberSaveable`
- ✅ 屏幕级状态放 ViewModel，用 `StateFlow`
- ✅ 尽量使用状态提升，让组件无状态
- ✅ 单一数据源：一个状态只在一个地方修改
- ✅ 状态不可变：使用 `copy()` 创建新状态
- ✅ 派生状态用 `derivedStateOf`

## 总结

Compose 状态管理的核心原则：

- **状态驱动 UI**：UI 是状态的函数
- **状态提升**：让组件可复用、可测试
- **单向数据流**：状态向下流动，事件向上传递
- **选择合适的 API**：根据状态的作用域和生命周期选择

---

*© 2024 Fidroid. [返回首页](../index.html)*


