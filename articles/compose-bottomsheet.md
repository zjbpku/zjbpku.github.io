# Compose Bottom Sheet 与 Dialog：模态组件详解

> **发布日期**: 2024-06-11  
> **阅读时间**: 约 22 分钟  
> **标签**: BottomSheet, Dialog, Modal, Scaffold

模态组件是移动应用中常见的交互模式，用于展示额外内容或请求用户确认。Compose 提供了丰富的模态组件支持。本文将深入讲解 Bottom Sheet 和 Dialog 的使用技巧。

## 一、Modal Bottom Sheet

### 基础用法

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ModalBottomSheetExample() {
    var showBottomSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState()
    
    Scaffold { padding ->
        Button(
            onClick = { showBottomSheet = true },
            modifier = Modifier.padding(padding)
        ) {
            Text("显示 Bottom Sheet")
        }
    }
    
    if (showBottomSheet) {
        ModalBottomSheet(
            onDismissRequest = { showBottomSheet = false },
            sheetState = sheetState
        ) {
            // Sheet 内容
            Column(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp)
            ) {
                Text(
                    "选择操作",
                    style = MaterialTheme.typography.titleLarge
                )
                Spacer(modifier = Modifier.height(16.dp))
                
                ListItem(
                    headlineContent = { Text("分享") },
                    leadingContent = { Icon(Icons.Default.Share, null) },
                    modifier = Modifier.clickable { 
                        showBottomSheet = false
                        // 处理分享
                    }
                )
                ListItem(
                    headlineContent = { Text("编辑") },
                    leadingContent = { Icon(Icons.Default.Edit, null) },
                    modifier = Modifier.clickable { 
                        showBottomSheet = false
                    }
                )
                ListItem(
                    headlineContent = { Text("删除") },
                    leadingContent = { Icon(Icons.Default.Delete, null) },
                    modifier = Modifier.clickable { 
                        showBottomSheet = false
                    }
                )
                
                Spacer(modifier = Modifier.height(32.dp))
            }
        }
    }
}
```

### 编程控制

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ProgrammaticBottomSheet() {
    val scope = rememberCoroutineScope()
    val sheetState = rememberModalBottomSheetState(
        skipPartiallyExpanded = true  // 跳过半展开状态
    )
    var showSheet by remember { mutableStateOf(false) }
    
    // 编程方式隐藏
    fun hideSheet() {
        scope.launch {
            sheetState.hide()
        }.invokeOnCompletion {
            if (!sheetState.isVisible) {
                showSheet = false
            }
        }
    }
    
    Button(onClick = { showSheet = true }) {
        Text("显示")
    }
    
    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            Button(onClick = { hideSheet() }) {
                Text("关闭")
            }
        }
    }
}
```

### 自定义样式

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun StyledBottomSheet() {
    var showSheet by remember { mutableStateOf(false) }
    
    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            containerColor = MaterialTheme.colorScheme.primaryContainer,
            contentColor = MaterialTheme.colorScheme.onPrimaryContainer,
            dragHandle = {
                // 自定义拖动手柄
                Box(
                    modifier = Modifier
                        .padding(vertical = 12.dp)
                        .width(48.dp)
                        .height(4.dp)
                        .background(
                            MaterialTheme.colorScheme.onPrimaryContainer.copy(alpha = 0.4f),
                            RoundedCornerShape(2.dp)
                        )
                )
            },
            shape = RoundedCornerShape(topStart = 24.dp, topEnd = 24.dp)
        ) {
            // 内容
        }
    }
}
```

## 二、Standard Bottom Sheet

嵌入在 Scaffold 中的底部抽屉：

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun StandardBottomSheetExample() {
    val scaffoldState = rememberBottomSheetScaffoldState()
    val scope = rememberCoroutineScope()
    
    BottomSheetScaffold(
        scaffoldState = scaffoldState,
        sheetContent = {
            Column(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp)
            ) {
                Text(
                    "底部面板",
                    style = MaterialTheme.typography.titleMedium
                )
                Spacer(modifier = Modifier.height(16.dp))
                Text("这是一个标准底部面板的内容")
                // 更多内容...
            }
        },
        sheetPeekHeight = 128.dp,
        sheetShape = RoundedCornerShape(topStart = 16.dp, topEnd = 16.dp)
    ) { padding ->
        Column(modifier = Modifier.padding(padding)) {
            Button(
                onClick = {
                    scope.launch {
                        scaffoldState.bottomSheetState.expand()
                    }
                }
            ) {
                Text("展开")
            }
            
            Button(
                onClick = {
                    scope.launch {
                        scaffoldState.bottomSheetState.partialExpand()
                    }
                }
            ) {
                Text("半展开")
            }
        }
    }
}
```

## 三、AlertDialog

### 基础对话框

```kotlin
@Composable
fun BasicAlertDialog() {
    var showDialog by remember { mutableStateOf(false) }
    
    Button(onClick = { showDialog = true }) {
        Text("显示对话框")
    }
    
    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false },
            icon = { Icon(Icons.Default.Warning, null) },
            title = { Text("确认删除") },
            text = { Text("删除后将无法恢复，确定要删除吗？") },
            confirmButton = {
                TextButton(
                    onClick = {
                        // 执行删除
                        showDialog = false
                    }
                ) {
                    Text("删除", color = MaterialTheme.colorScheme.error)
                }
            },
            dismissButton = {
                TextButton(onClick = { showDialog = false }) {
                    Text("取消")
                }
            }
        )
    }
}
```

### 自定义对话框

```kotlin
@Composable
fun CustomDialog() {
    var showDialog by remember { mutableStateOf(false) }
    
    if (showDialog) {
        Dialog(onDismissRequest = { showDialog = false }) {
            Card(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp),
                shape = RoundedCornerShape(16.dp)
            ) {
                Column(
                    modifier = Modifier.padding(24.dp),
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    Icon(
                        imageVector = Icons.Default.CheckCircle,
                        contentDescription = null,
                        modifier = Modifier.size(64.dp),
                        tint = MaterialTheme.colorScheme.primary
                    )
                    
                    Spacer(modifier = Modifier.height(16.dp))
                    
                    Text(
                        text = "操作成功",
                        style = MaterialTheme.typography.headlineSmall
                    )
                    
                    Spacer(modifier = Modifier.height(8.dp))
                    
                    Text(
                        text = "您的请求已成功处理",
                        style = MaterialTheme.typography.bodyMedium,
                        textAlign = TextAlign.Center
                    )
                    
                    Spacer(modifier = Modifier.height(24.dp))
                    
                    Button(
                        onClick = { showDialog = false },
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Text("确定")
                    }
                }
            }
        }
    }
}
```

### 输入对话框

```kotlin
@Composable
fun InputDialog(
    onDismiss: () -> Unit,
    onConfirm: (String) -> Unit
) {
    var text by remember { mutableStateOf("") }
    
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("输入名称") },
        text = {
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                label = { Text("名称") },
                singleLine = true,
                modifier = Modifier.fillMaxWidth()
            )
        },
        confirmButton = {
            TextButton(
                onClick = { onConfirm(text) },
                enabled = text.isNotBlank()
            ) {
                Text("确定")
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text("取消")
            }
        }
    )
}
```

## 四、日期/时间选择器

### DatePicker

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DatePickerExample() {
    var showDatePicker by remember { mutableStateOf(false) }
    val datePickerState = rememberDatePickerState()
    var selectedDate by remember { mutableStateOf<Long?>(null) }
    
    Button(onClick = { showDatePicker = true }) {
        Text(selectedDate?.let { 
            SimpleDateFormat("yyyy-MM-dd", Locale.getDefault()).format(Date(it))
        } ?: "选择日期")
    }
    
    if (showDatePicker) {
        DatePickerDialog(
            onDismissRequest = { showDatePicker = false },
            confirmButton = {
                TextButton(
                    onClick = {
                        selectedDate = datePickerState.selectedDateMillis
                        showDatePicker = false
                    }
                ) {
                    Text("确定")
                }
            },
            dismissButton = {
                TextButton(onClick = { showDatePicker = false }) {
                    Text("取消")
                }
            }
        ) {
            DatePicker(state = datePickerState)
        }
    }
}
```

### TimePicker

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TimePickerExample() {
    var showTimePicker by remember { mutableStateOf(false) }
    val timePickerState = rememberTimePickerState()
    var selectedTime by remember { mutableStateOf<String?>(null) }
    
    Button(onClick = { showTimePicker = true }) {
        Text(selectedTime ?: "选择时间")
    }
    
    if (showTimePicker) {
        Dialog(onDismissRequest = { showTimePicker = false }) {
            Card(shape = RoundedCornerShape(16.dp)) {
                Column(
                    modifier = Modifier.padding(24.dp),
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    TimePicker(state = timePickerState)
                    
                    Row(
                        modifier = Modifier.fillMaxWidth(),
                        horizontalArrangement = Arrangement.End
                    ) {
                        TextButton(onClick = { showTimePicker = false }) {
                            Text("取消")
                        }
                        TextButton(
                            onClick = {
                                selectedTime = String.format(
                                    "%02d:%02d",
                                    timePickerState.hour,
                                    timePickerState.minute
                                )
                                showTimePicker = false
                            }
                        ) {
                            Text("确定")
                        }
                    }
                }
            }
        }
    }
}
```

## 五、封装通用对话框

### 确认对话框

```kotlin
@Composable
fun ConfirmDialog(
    visible: Boolean,
    title: String,
    message: String,
    confirmText: String = "确定",
    dismissText: String = "取消",
    isDestructive: Boolean = false,
    onConfirm: () -> Unit,
    onDismiss: () -> Unit
) {
    if (visible) {
        AlertDialog(
            onDismissRequest = onDismiss,
            title = { Text(title) },
            text = { Text(message) },
            confirmButton = {
                TextButton(onClick = onConfirm) {
                    Text(
                        confirmText,
                        color = if (isDestructive) 
                            MaterialTheme.colorScheme.error 
                        else 
                            MaterialTheme.colorScheme.primary
                    )
                }
            },
            dismissButton = {
                TextButton(onClick = onDismiss) {
                    Text(dismissText)
                }
            }
        )
    }
}

// 使用
@Composable
fun DeleteConfirmation() {
    var showConfirm by remember { mutableStateOf(false) }
    
    Button(onClick = { showConfirm = true }) {
        Text("删除")
    }
    
    ConfirmDialog(
        visible = showConfirm,
        title = "确认删除",
        message = "删除后将无法恢复，确定要删除吗？",
        confirmText = "删除",
        isDestructive = true,
        onConfirm = {
            // 执行删除
            showConfirm = false
        },
        onDismiss = { showConfirm = false }
    )
}
```

### Loading 对话框

```kotlin
@Composable
fun LoadingDialog(
    visible: Boolean,
    message: String = "加载中..."
) {
    if (visible) {
        Dialog(onDismissRequest = { }) {
            Card(shape = RoundedCornerShape(16.dp)) {
                Row(
                    modifier = Modifier.padding(24.dp),
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    CircularProgressIndicator(
                        modifier = Modifier.size(32.dp),
                        strokeWidth = 3.dp
                    )
                    Spacer(modifier = Modifier.width(16.dp))
                    Text(message)
                }
            }
        }
    }
}
```

## 六、最佳实践

- ✅ 使用 `remember` 管理显示状态
- ✅ Modal Bottom Sheet 用于临时操作面板
- ✅ Standard Bottom Sheet 用于持久内容
- ✅ AlertDialog 用于确认操作
- ✅ 自定义 Dialog 用于复杂内容
- ✅ 封装通用对话框组件
- ✅ 使用 `skipPartiallyExpanded` 控制展开行为
- ✅ 正确处理返回键和背景点击

## 总结

Compose 模态组件的核心要点：

- **ModalBottomSheet**：临时操作面板
- **BottomSheetScaffold**：嵌入式底部抽屉
- **AlertDialog**：确认对话框
- **Dialog**：自定义对话框
- **DatePickerDialog/TimePicker**：日期时间选择
- **状态管理**：使用 remember 控制显示

模态组件是构建良好用户体验的重要工具。

---

*© 2024 Fidroid. [返回首页](../index.html)*


