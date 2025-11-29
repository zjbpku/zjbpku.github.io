# Compose UI 测试实战：从单元测试到端到端

> **发布日期**: 2024-04-08  
> **阅读时间**: 约 18 分钟  
> **标签**: Testing, ComposeTestRule, Semantics, Screenshot Test

Compose 的声明式特性让 UI 测试变得更加直观。本文将介绍如何使用 Compose Testing API 编写可靠的 UI 测试。

## 一、测试环境配置

```kotlin
// build.gradle.kts
dependencies {
    // Compose 测试
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")

    // 可选：Robolectric 支持本地测试
    testImplementation("org.robolectric:robolectric:4.11")
}
```

## 二、基础测试结构

```kotlin
class LoginScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun loginButton_displaysCorrectText() {
        // 设置要测试的 Composable
        composeTestRule.setContent {
            LoginScreen()
        }

        // 查找并断言
        composeTestRule
            .onNodeWithText("登录")
            .assertIsDisplayed()
    }
}
```

## 三、查找节点的方式

```kotlin
// 通过文本查找
composeTestRule.onNodeWithText("Submit")

// 通过 contentDescription 查找
composeTestRule.onNodeWithContentDescription("Close button")

// 通过 testTag 查找（推荐）
composeTestRule.onNodeWithTag("login_button")

// 组合查找条件
composeTestRule.onNode(
    hasText("Submit") and hasClickAction()
)

// 查找多个节点
composeTestRule.onAllNodesWithTag("list_item")
```

### 添加 testTag

```kotlin
Button(
    onClick = { },
    modifier = Modifier.testTag("login_button")
) {
    Text("登录")
}
```

## 四、常用断言

```kotlin
// 显示状态
.assertIsDisplayed()
.assertIsNotDisplayed()
.assertExists()
.assertDoesNotExist()

// 启用状态
.assertIsEnabled()
.assertIsNotEnabled()

// 选中状态
.assertIsSelected()
.assertIsOn()  // for toggles
.assertIsOff()

// 文本断言
.assertTextEquals("Expected text")
.assertTextContains("partial")

// 数量断言
composeTestRule.onAllNodesWithTag("item").assertCountEquals(5)
```

## 五、模拟用户交互

```kotlin
// 点击
.performClick()

// 输入文本
.performTextInput("Hello")
.performTextClearance()
.performTextReplacement("New text")

// 滚动
.performScrollTo()
.performScrollToIndex(10)

// 手势
.performTouchInput {
    swipeLeft()
    swipeUp()
    longClick()
}
```

## 六、完整测试示例

```kotlin
class TodoListTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun addTodo_showsInList() {
        composeTestRule.setContent {
            TodoApp()
        }

        // 输入新待办
        composeTestRule
            .onNodeWithTag("todo_input")
            .performTextInput("Buy milk")

        // 点击添加按钮
        composeTestRule
            .onNodeWithTag("add_button")
            .performClick()

        // 验证列表中显示新项
        composeTestRule
            .onNodeWithText("Buy milk")
            .assertIsDisplayed()
    }

    @Test
    fun completeTodo_showsStrikethrough() {
        composeTestRule.setContent {
            TodoItem(
                todo = Todo("Test", completed = false),
                onToggle = {}
            )
        }

        // 点击复选框
        composeTestRule
            .onNodeWithTag("todo_checkbox")
            .performClick()

        // 验证状态变化
        composeTestRule
            .onNodeWithTag("todo_checkbox")
            .assertIsOn()
    }
}
```

## 七、等待异步操作

```kotlin
@Test
fun loadData_showsContent() {
    composeTestRule.setContent {
        DataScreen()
    }

    // 等待加载完成
    composeTestRule.waitUntil(timeoutMillis = 5000) {
        composeTestRule
            .onAllNodesWithTag("data_item")
            .fetchSemanticsNodes()
            .isNotEmpty()
    }

    // 或使用 IdlingResource
    composeTestRule.registerIdlingResource(dataLoadingIdlingResource)
}
```

## 八、测试 ViewModel 集成

```kotlin
@Test
fun screenWithViewModel_displaysData() {
    val fakeViewModel = FakeProfileViewModel()
    fakeViewModel.setProfile(
        Profile(name = "John", email = "john@example.com")
    )

    composeTestRule.setContent {
        ProfileScreen(viewModel = fakeViewModel)
    }

    composeTestRule
        .onNodeWithText("John")
        .assertIsDisplayed()

    composeTestRule
        .onNodeWithText("john@example.com")
        .assertIsDisplayed()
}
```

## 九、截图测试

```kotlin
@Test
fun button_matchesGolden() {
    composeTestRule.setContent {
        PrimaryButton(text = "Click me", onClick = {})
    }

    composeTestRule
        .onNodeWithTag("primary_button")
        .captureToImage()
        .assertAgainstGolden(goldenIdentifier = "primary_button")
}
```

> 💡 **截图测试工具**  
> 推荐使用 Paparazzi（本地截图测试）或 Shot（设备截图测试）库来实现截图比对。

## 十、测试最佳实践

- ✅ 使用 `testTag` 而非文本查找，避免国际化问题
- ✅ 测试用户可见的行为，而非实现细节
- ✅ 保持测试独立，每个测试只验证一件事
- ✅ 使用 Fake/Mock 隔离外部依赖
- ✅ 为关键用户流程编写端到端测试
- ✅ 使用截图测试保证视觉一致性

## 总结

Compose 测试的核心流程：

1. **设置**：使用 `setContent` 渲染 Composable
2. **查找**：通过语义树查找节点
3. **交互**：模拟用户操作
4. **断言**：验证预期结果

编写好的测试能让你更有信心地重构和迭代 UI 代码。

---

*© 2024 Fidroid. [返回首页](../index.html)*

