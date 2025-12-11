# Compose 键盘处理：IME 与 WindowInsets

> **发布日期**: 2024-06-13  
> **阅读时间**: 约 20 分钟  
> **标签**: Keyboard, IME, WindowInsets, TextField

键盘处理是移动应用开发中的常见挑战。Compose 提供了完善的 WindowInsets API 来处理键盘显示、隐藏和布局调整。本文将深入讲解键盘处理的最佳实践。

## 一、WindowInsets 基础

### 启用 Edge-to-Edge

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 启用 Edge-to-Edge
        enableEdgeToEdge()
        
        setContent {
            MyApp()
        }
    }
}
```

### 常用 Insets 类型

```kotlin
@Composable
fun InsetsExample() {
    // 状态栏高度
    val statusBars = WindowInsets.statusBars
    
    // 导航栏高度
    val navigationBars = WindowInsets.navigationBars
    
    // 键盘高度（IME）
    val ime = WindowInsets.ime
    
    // 系统手势区域
    val systemGestures = WindowInsets.systemGestures
    
    // 安全区域（刘海屏等）
    val displayCutout = WindowInsets.displayCutout
    
    // 组合多个 Insets
    val safeDrawing = WindowInsets.safeDrawing
    val safeContent = WindowInsets.safeContent
}
```

## 二、键盘自适应布局

### 使用 imePadding

```kotlin
@Composable
fun ChatScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .statusBarsPadding()     // 状态栏内边距
            .navigationBarsPadding() // 导航栏内边距
            .imePadding()            // 键盘内边距
    ) {
        // 消息列表
        LazyColumn(
            modifier = Modifier.weight(1f),
            reverseLayout = true
        ) {
            items(messages) { message ->
                MessageItem(message)
            }
        }
        
        // 输入框
        ChatInput(
            onSend = { /* 发送消息 */ }
        )
    }
}

@Composable
fun ChatInput(onSend: (String) -> Unit) {
    var text by remember { mutableStateOf("") }
    
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier.weight(1f),
            placeholder = { Text("输入消息...") }
        )
        
        IconButton(
            onClick = {
                if (text.isNotBlank()) {
                    onSend(text)
                    text = ""
                }
            }
        ) {
            Icon(Icons.AutoMirrored.Filled.Send, "发送")
        }
    }
}
```

### 键盘动画

```kotlin
@Composable
fun AnimatedImeContent() {
    val imeVisible = WindowInsets.isImeVisible
    val imeHeight = WindowInsets.ime.getBottom(LocalDensity.current)
    
    // 键盘高度动画
    val animatedImeHeight by animateDpAsState(
        targetValue = with(LocalDensity.current) { imeHeight.toDp() },
        animationSpec = spring(stiffness = Spring.StiffnessMediumLow)
    )
    
    Column(
        modifier = Modifier.fillMaxSize()
    ) {
        // 主内容
        Box(modifier = Modifier.weight(1f)) {
            // 内容
        }
        
        // 底部区域随键盘动画
        Spacer(modifier = Modifier.height(animatedImeHeight))
    }
}
```

## 三、TextField 焦点管理

### 基础焦点控制

```kotlin
@Composable
fun FocusExample() {
    val focusManager = LocalFocusManager.current
    var text by remember { mutableStateOf("") }
    
    Column(modifier = Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier.fillMaxWidth(),
            keyboardOptions = KeyboardOptions(
                imeAction = ImeAction.Done
            ),
            keyboardActions = KeyboardActions(
                onDone = {
                    focusManager.clearFocus() // 关闭键盘
                }
            )
        )
        
        Button(
            onClick = { focusManager.clearFocus() },
            modifier = Modifier.padding(top = 8.dp)
        ) {
            Text("收起键盘")
        }
    }
}
```

### 焦点请求

```kotlin
@Composable
fun RequestFocusExample() {
    val focusRequester = remember { FocusRequester() }
    var text by remember { mutableStateOf("") }
    
    Column(modifier = Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier
                .fillMaxWidth()
                .focusRequester(focusRequester)
        )
        
        Button(
            onClick = { focusRequester.requestFocus() }
        ) {
            Text("聚焦输入框")
        }
    }
    
    // 自动聚焦
    LaunchedEffect(Unit) {
        focusRequester.requestFocus()
    }
}
```

### 焦点顺序

```kotlin
@Composable
fun FocusOrderExample() {
    val (emailFocus, passwordFocus, confirmFocus) = remember { FocusRequester.createRefs() }
    val focusManager = LocalFocusManager.current
    
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var confirm by remember { mutableStateOf("") }
    
    Column(modifier = Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("邮箱") },
            modifier = Modifier
                .fillMaxWidth()
                .focusRequester(emailFocus),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(
                onNext = { passwordFocus.requestFocus() }
            )
        )
        
        Spacer(modifier = Modifier.height(8.dp))
        
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("密码") },
            modifier = Modifier
                .fillMaxWidth()
                .focusRequester(passwordFocus),
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(
                onNext = { confirmFocus.requestFocus() }
            )
        )
        
        Spacer(modifier = Modifier.height(8.dp))
        
        OutlinedTextField(
            value = confirm,
            onValueChange = { confirm = it },
            label = { Text("确认密码") },
            modifier = Modifier
                .fillMaxWidth()
                .focusRequester(confirmFocus),
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
            keyboardActions = KeyboardActions(
                onDone = { focusManager.clearFocus() }
            )
        )
    }
}
```

## 四、软键盘控制

### 显示/隐藏键盘

```kotlin
@Composable
fun KeyboardControlExample() {
    val keyboardController = LocalSoftwareKeyboardController.current
    var text by remember { mutableStateOf("") }
    
    Column(modifier = Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier.fillMaxWidth()
        )
        
        Row(modifier = Modifier.padding(top = 8.dp)) {
            Button(onClick = { keyboardController?.show() }) {
                Text("显示键盘")
            }
            
            Spacer(modifier = Modifier.width(8.dp))
            
            Button(onClick = { keyboardController?.hide() }) {
                Text("隐藏键盘")
            }
        }
    }
}
```

### 键盘类型

```kotlin
@Composable
fun KeyboardTypesExample() {
    Column(modifier = Modifier.padding(16.dp)) {
        // 文本键盘
        OutlinedTextField(
            value = "",
            onValueChange = {},
            label = { Text("文本") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Text)
        )
        
        // 数字键盘
        OutlinedTextField(
            value = "",
            onValueChange = {},
            label = { Text("数字") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number)
        )
        
        // 手机号键盘
        OutlinedTextField(
            value = "",
            onValueChange = {},
            label = { Text("手机号") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Phone)
        )
        
        // 邮箱键盘
        OutlinedTextField(
            value = "",
            onValueChange = {},
            label = { Text("邮箱") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email)
        )
        
        // 密码键盘
        OutlinedTextField(
            value = "",
            onValueChange = {},
            label = { Text("密码") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password),
            visualTransformation = PasswordVisualTransformation()
        )
    }
}
```

## 五、点击外部收起键盘

```kotlin
@Composable
fun DismissKeyboardOnTap() {
    val focusManager = LocalFocusManager.current
    var text by remember { mutableStateOf("") }
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectTapGestures(
                    onTap = { focusManager.clearFocus() }
                )
            }
    ) {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier
                .padding(16.dp)
                .fillMaxWidth()
        )
    }
}
```

## 六、Scaffold 中的 Insets

```kotlin
@Composable
fun ScaffoldWithInsets() {
    Scaffold(
        modifier = Modifier.fillMaxSize(),
        contentWindowInsets = WindowInsets(0), // 禁用默认 insets
        topBar = {
            TopAppBar(
                title = { Text("标题") },
                modifier = Modifier.statusBarsPadding()
            )
        },
        bottomBar = {
            NavigationBar(
                modifier = Modifier.navigationBarsPadding()
            ) {
                // 导航项
            }
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .padding(padding)
                .imePadding()
        ) {
            // 内容
        }
    }
}
```

## 七、常见问题解决

### TextField 被键盘遮挡

```kotlin
@Composable
fun ScrollableFormWithKeyboard() {
    val scrollState = rememberScrollState()
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .verticalScroll(scrollState)
            .imePadding()
            .padding(16.dp)
    ) {
        repeat(10) { index ->
            OutlinedTextField(
                value = "",
                onValueChange = {},
                label = { Text("字段 $index") },
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(vertical = 4.dp)
            )
        }
    }
}
```

## 八、最佳实践

- ✅ 使用 `enableEdgeToEdge()` 启用全屏
- ✅ 使用 `imePadding()` 处理键盘
- ✅ 使用 `FocusRequester` 管理焦点
- ✅ 设置正确的 `ImeAction`
- ✅ 实现点击外部收起键盘
- ✅ 使用 `KeyboardOptions` 配置键盘类型
- ✅ 处理焦点顺序（Next/Done）
- ✅ 确保输入框可见

## 总结

Compose 键盘处理的核心要点：

- **WindowInsets**：处理系统 UI 区域
- **imePadding**：自动适应键盘高度
- **FocusManager**：管理焦点和键盘
- **KeyboardController**：编程控制键盘
- **KeyboardOptions**：配置键盘类型和动作
- **FocusRequester**：请求和管理焦点

正确处理键盘可以显著提升用户输入体验。

---

*© 2024 Fidroid. [返回首页](../index.html)*


