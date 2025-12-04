# Compose 无障碍开发指南：Semantics 与屏幕阅读器

> **发布日期**: 2024-05-16  
> **阅读时间**: 约 22 分钟  
> **标签**: Accessibility, Semantics, TalkBack, 无障碍

无障碍（Accessibility）是现代应用开发的重要组成部分。Compose 提供了强大的语义系统（Semantics），让你可以轻松构建对屏幕阅读器和辅助服务友好的 UI。本文将深入讲解 Compose 的无障碍开发最佳实践。

## 一、为什么需要无障碍？

全球约有 15% 的人口有某种形式的残障，包括视觉、听觉、运动和认知障碍。构建无障碍应用不仅是道德责任，在很多地区也是法律要求。

> 💡 **无障碍带来的好处**
> - 扩大用户群体
> - 提升所有用户的体验
> - 符合法规要求（如 ADA、WCAG）
> - SEO 和可测试性的提升

## 二、Compose 语义系统

### Semantics 是什么？

Semantics（语义）是 Compose 向辅助服务描述 UI 元素的方式。每个 Composable 都可以附加语义信息，这些信息会被 TalkBack 等屏幕阅读器使用。

```kotlin
// 基本语义
Text(
    text = "欢迎",
    modifier = Modifier.semantics {
        contentDescription = "欢迎信息"
    }
)
```

### 内置组件的语义

Compose 的内置组件已经提供了合理的默认语义：

```kotlin
// Button 自动包含 Role.Button 语义
Button(onClick = { }) {
    Text("提交")  // 自动作为按钮的标签
}

// Checkbox 自动包含状态语义
Checkbox(
    checked = isChecked,
    onCheckedChange = { isChecked = it }
)

// TextField 自动提供输入相关语义
TextField(
    value = text,
    onValueChange = { text = it },
    label = { Text("用户名") }
)
```

## 三、Modifier.semantics

### contentDescription

为图像和图标提供描述：

```kotlin
// 有意义的图标
Icon(
    imageVector = Icons.Default.Share,
    contentDescription = "分享"  // 屏幕阅读器会朗读这个
)

// 纯装饰性图像
Image(
    painter = painterResource(R.drawable.decorative),
    contentDescription = null  // null 表示装饰性，会被忽略
)

// 复杂图像需要更详细的描述
Image(
    painter = painterResource(R.drawable.chart),
    contentDescription = "销售趋势图：1月100万，2月150万，3月200万",
    modifier = Modifier.semantics {
        // 可以添加更多语义
    }
)
```

### 设置角色（Role）

```kotlin
Box(
    modifier = Modifier
        .clickable { onItemClick() }
        .semantics { 
            role = Role.Button 
        }
) {
    Text("自定义按钮")
}

// 常用角色
// Role.Button - 按钮
// Role.Checkbox - 复选框
// Role.Switch - 开关
// Role.RadioButton - 单选按钮
// Role.Tab - 标签页
// Role.Image - 图像
// Role.DropdownList - 下拉列表
```

### 状态描述

```kotlin
@Composable
fun ExpandableCard(
    title: String,
    expanded: Boolean,
    onToggle: () -> Unit
) {
    Column(
        modifier = Modifier
            .clickable(onClick = onToggle)
            .semantics {
                // 设置展开/折叠状态
                stateDescription = if (expanded) "已展开" else "已折叠"
            }
    ) {
        Text(title)
        if (expanded) {
            Text("详细内容...")
        }
    }
}
```

### 自定义操作

```kotlin
@Composable
fun SwipeableItem(
    item: Item,
    onDelete: () -> Unit,
    onArchive: () -> Unit
) {
    Box(
        modifier = Modifier.semantics {
            // 添加自定义操作，替代滑动手势
            customActions = listOf(
                CustomAccessibilityAction("删除") {
                    onDelete()
                    true
                },
                CustomAccessibilityAction("归档") {
                    onArchive()
                    true
                }
            )
        }
    ) {
        // 可滑动的内容
    }
}
```

## 四、合并语义

### clearAndSetSemantics

完全替换子元素的语义：

```kotlin
@Composable
fun UserCard(user: User) {
    Row(
        modifier = Modifier
            .clickable { onUserClick(user) }
            .clearAndSetSemantics {
                // 将整个卡片作为一个语义单元
                contentDescription = "${user.name}，${user.role}，点击查看详情"
            }
    ) {
        Avatar(user.avatarUrl)
        Column {
            Text(user.name)
            Text(user.role)
        }
    }
}
```

### mergeDescendants

合并子元素的语义：

```kotlin
@Composable
fun ListItem(title: String, subtitle: String) {
    Row(
        modifier = Modifier.semantics(mergeDescendants = true) {
            // 子元素的语义会被合并
        }
    ) {
        Icon(Icons.Default.Person, contentDescription = null)
        Column {
            Text(title)
            Text(subtitle)
        }
    }
    // 屏幕阅读器会将整行作为一个单元朗读
}
```

## 五、焦点管理

### 焦点顺序

```kotlin
@Composable
fun LoginForm() {
    val (usernameFocus, passwordFocus, submitFocus) = remember { FocusRequester.createRefs() }
    
    Column {
        TextField(
            value = username,
            onValueChange = { username = it },
            modifier = Modifier
                .focusRequester(usernameFocus)
                .focusProperties {
                    next = passwordFocus
                }
        )
        
        TextField(
            value = password,
            onValueChange = { password = it },
            modifier = Modifier
                .focusRequester(passwordFocus)
                .focusProperties {
                    previous = usernameFocus
                    next = submitFocus
                }
        )
        
        Button(
            onClick = { },
            modifier = Modifier
                .focusRequester(submitFocus)
                .focusProperties {
                    previous = passwordFocus
                }
        ) {
            Text("登录")
        }
    }
}
```

### 跳过装饰性元素

```kotlin
@Composable
fun ContentWithDecoration() {
    Column {
        // 装饰性分隔线，跳过焦点
        Divider(
            modifier = Modifier.semantics {
                invisibleToUser()
            }
        )
        
        // 主要内容
        Text("重要信息")
    }
}
```

## 六、Live Region（实时区域）

当内容动态变化时，通知屏幕阅读器：

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableIntStateOf(0) }
    
    Column {
        Text(
            text = "计数: $count",
            modifier = Modifier.semantics {
                // 内容变化时自动朗读
                liveRegion = LiveRegionMode.Polite
            }
        )
        
        Button(onClick = { count++ }) {
            Text("增加")
        }
    }
}

// LiveRegionMode.Polite - 等待当前朗读完成后通知
// LiveRegionMode.Assertive - 立即中断并通知
```

### 错误提示

```kotlin
@Composable
fun FormField(
    value: String,
    error: String?,
    onValueChange: (String) -> Unit
) {
    Column {
        TextField(
            value = value,
            onValueChange = onValueChange,
            isError = error != null,
            modifier = Modifier.semantics {
                if (error != null) {
                    error(error)
                }
            }
        )
        
        if (error != null) {
            Text(
                text = error,
                color = Color.Red,
                modifier = Modifier.semantics {
                    liveRegion = LiveRegionMode.Assertive
                }
            )
        }
    }
}
```

## 七、标题和层级

### 设置标题

```kotlin
@Composable
fun ArticleScreen() {
    Column {
        Text(
            text = "文章标题",
            style = MaterialTheme.typography.headlineLarge,
            modifier = Modifier.semantics {
                heading()  // 标记为标题
            }
        )
        
        Text(
            text = "第一章",
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier.semantics {
                heading()
            }
        )
        
        Text("正文内容...")
    }
}
```

## 八、触摸目标大小

确保触摸目标足够大（建议至少 48dp）：

```kotlin
@Composable
fun SmallIconButton(
    onClick: () -> Unit,
    icon: ImageVector
) {
    IconButton(
        onClick = onClick,
        modifier = Modifier.sizeIn(minWidth = 48.dp, minHeight = 48.dp)
    ) {
        Icon(
            imageVector = icon,
            contentDescription = "操作",
            modifier = Modifier.size(24.dp)  // 图标本身可以小
        )
    }
}
```

## 九、测试无障碍

### 使用语义树测试

```kotlin
@Test
fun testButtonAccessibility() {
    composeTestRule.setContent {
        MyButton(text = "提交", onClick = { })
    }
    
    // 验证语义
    composeTestRule
        .onNodeWithText("提交")
        .assertHasClickAction()
        .assert(hasRole(Role.Button))
    
    // 打印语义树（调试用）
    composeTestRule.onRoot().printToLog("Semantics")
}
```

### 使用 TalkBack 测试

1. 开启 TalkBack：设置 → 无障碍 → TalkBack
2. 使用手势浏览应用
3. 确认所有交互元素都可以被发现和操作
4. 验证朗读内容是否有意义

## 十、常见模式

### 可访问的图片轮播

```kotlin
@Composable
fun AccessibleCarousel(
    images: List<ImageItem>,
    currentIndex: Int,
    onIndexChange: (Int) -> Unit
) {
    Box(
        modifier = Modifier.semantics {
            contentDescription = "图片轮播，共 ${images.size} 张，当前第 ${currentIndex + 1} 张"
            
            customActions = listOf(
                CustomAccessibilityAction("上一张") {
                    if (currentIndex > 0) {
                        onIndexChange(currentIndex - 1)
                        true
                    } else false
                },
                CustomAccessibilityAction("下一张") {
                    if (currentIndex < images.size - 1) {
                        onIndexChange(currentIndex + 1)
                        true
                    } else false
                }
            )
        }
    ) {
        Image(
            painter = painterResource(images[currentIndex].resId),
            contentDescription = images[currentIndex].description
        )
    }
}
```

### 可访问的评分组件

```kotlin
@Composable
fun AccessibleRating(
    rating: Int,
    maxRating: Int = 5,
    onRatingChange: (Int) -> Unit
) {
    Row(
        modifier = Modifier.semantics {
            stateDescription = "$rating 星，共 $maxRating 星"
            
            customActions = (1..maxRating).map { stars ->
                CustomAccessibilityAction("设置为 $stars 星") {
                    onRatingChange(stars)
                    true
                }
            }
        }
    ) {
        repeat(maxRating) { index ->
            Icon(
                imageVector = if (index < rating) Icons.Filled.Star else Icons.Outlined.Star,
                contentDescription = null,  // 由父元素处理
                modifier = Modifier.clickable { onRatingChange(index + 1) }
            )
        }
    }
}
```

## 十一、最佳实践清单

- ✅ 为所有有意义的图标和图像提供 `contentDescription`
- ✅ 装饰性图像使用 `contentDescription = null`
- ✅ 使用 `mergeDescendants` 合并相关元素
- ✅ 为复杂手势提供替代的 `customActions`
- ✅ 确保触摸目标至少 48dp
- ✅ 使用 `heading()` 标记标题
- ✅ 使用 `liveRegion` 通知动态内容变化
- ✅ 为表单字段提供清晰的错误提示
- ✅ 测试焦点顺序是否合理
- ✅ 使用 TalkBack 实际测试

## 总结

Compose 无障碍开发的核心要点：

- **Semantics**：向辅助服务描述 UI 的方式
- **contentDescription**：为图像提供文字描述
- **Role**：明确元素的交互类型
- **clearAndSetSemantics / mergeDescendants**：控制语义合并
- **customActions**：为手势提供替代操作
- **liveRegion**：通知动态内容变化
- **heading()**：标记文档结构

构建无障碍应用不仅帮助残障用户，也能提升所有用户的体验。

---

*© 2024 Fidroid. [返回首页](../index.html)*


