# Compose 表单处理完全指南：验证、状态管理与最佳实践

> **发布日期**: 2024-05-26  
> **阅读时间**: 约 26 分钟  
> **标签**: Form, Validation, TextField, 表单

表单是应用中最常见的交互形式之一。从简单的登录表单到复杂的多步骤注册流程，正确处理表单状态和验证是构建良好用户体验的关键。本文将深入讲解 Compose 中的表单处理最佳实践。

## 一、表单状态管理基础

### 单字段状态

```kotlin
@Composable
fun SimpleTextField() {
    var text by remember { mutableStateOf("") }
    
    OutlinedTextField(
        value = text,
        onValueChange = { text = it },
        label = { Text("用户名") },
        modifier = Modifier.fillMaxWidth()
    )
}
```

### 多字段状态类

```kotlin
data class LoginFormState(
    val email: String = "",
    val password: String = "",
    val rememberMe: Boolean = false
)

@Composable
fun LoginForm() {
    var formState by remember { mutableStateOf(LoginFormState()) }
    
    Column(modifier = Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = formState.email,
            onValueChange = { formState = formState.copy(email = it) },
            label = { Text("邮箱") },
            modifier = Modifier.fillMaxWidth()
        )
        
        Spacer(modifier = Modifier.height(8.dp))
        
        OutlinedTextField(
            value = formState.password,
            onValueChange = { formState = formState.copy(password = it) },
            label = { Text("密码") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )
        
        Row(
            verticalAlignment = Alignment.CenterVertically
        ) {
            Checkbox(
                checked = formState.rememberMe,
                onCheckedChange = { formState = formState.copy(rememberMe = it) }
            )
            Text("记住我")
        }
    }
}
```

## 二、表单验证

### 基础验证

```kotlin
data class FormField(
    val value: String = "",
    val error: String? = null
)

data class RegistrationFormState(
    val username: FormField = FormField(),
    val email: FormField = FormField(),
    val password: FormField = FormField(),
    val confirmPassword: FormField = FormField()
)

class RegistrationFormValidator {
    
    fun validateUsername(username: String): String? {
        return when {
            username.isBlank() -> "用户名不能为空"
            username.length < 3 -> "用户名至少3个字符"
            username.length > 20 -> "用户名最多20个字符"
            !username.matches(Regex("^[a-zA-Z0-9_]+$")) -> "用户名只能包含字母、数字和下划线"
            else -> null
        }
    }
    
    fun validateEmail(email: String): String? {
        return when {
            email.isBlank() -> "邮箱不能为空"
            !android.util.Patterns.EMAIL_ADDRESS.matcher(email).matches() -> "邮箱格式不正确"
            else -> null
        }
    }
    
    fun validatePassword(password: String): String? {
        return when {
            password.isBlank() -> "密码不能为空"
            password.length < 8 -> "密码至少8个字符"
            !password.any { it.isDigit() } -> "密码必须包含数字"
            !password.any { it.isUpperCase() } -> "密码必须包含大写字母"
            else -> null
        }
    }
    
    fun validateConfirmPassword(password: String, confirmPassword: String): String? {
        return when {
            confirmPassword.isBlank() -> "请确认密码"
            confirmPassword != password -> "两次输入的密码不一致"
            else -> null
        }
    }
}
```

### 实时验证 vs 提交时验证

```kotlin
@Composable
fun RegistrationForm() {
    var formState by remember { mutableStateOf(RegistrationFormState()) }
    val validator = remember { RegistrationFormValidator() }
    
    Column(modifier = Modifier.padding(16.dp)) {
        // 实时验证 - 用户输入时立即验证
        OutlinedTextField(
            value = formState.username.value,
            onValueChange = { newValue ->
                formState = formState.copy(
                    username = FormField(
                        value = newValue,
                        error = validator.validateUsername(newValue)
                    )
                )
            },
            label = { Text("用户名") },
            isError = formState.username.error != null,
            supportingText = formState.username.error?.let { { Text(it) } },
            modifier = Modifier.fillMaxWidth()
        )
        
        // 失去焦点时验证
        var emailFocused by remember { mutableStateOf(false) }
        
        OutlinedTextField(
            value = formState.email.value,
            onValueChange = { newValue ->
                formState = formState.copy(
                    email = formState.email.copy(value = newValue)
                )
            },
            label = { Text("邮箱") },
            isError = !emailFocused && formState.email.error != null,
            supportingText = if (!emailFocused) formState.email.error?.let { { Text(it) } } else null,
            modifier = Modifier
                .fillMaxWidth()
                .onFocusChanged { focusState ->
                    if (emailFocused && !focusState.isFocused) {
                        // 失去焦点时验证
                        formState = formState.copy(
                            email = formState.email.copy(
                                error = validator.validateEmail(formState.email.value)
                            )
                        )
                    }
                    emailFocused = focusState.isFocused
                }
        )
        
        Button(
            onClick = {
                // 提交时验证所有字段
                formState = formState.copy(
                    username = formState.username.copy(
                        error = validator.validateUsername(formState.username.value)
                    ),
                    email = formState.email.copy(
                        error = validator.validateEmail(formState.email.value)
                    ),
                    password = formState.password.copy(
                        error = validator.validatePassword(formState.password.value)
                    )
                )
                
                // 如果没有错误，提交表单
                if (formState.isValid()) {
                    // 提交逻辑
                }
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("注册")
        }
    }
}

fun RegistrationFormState.isValid(): Boolean {
    return username.error == null && 
           email.error == null && 
           password.error == null &&
           confirmPassword.error == null
}
```

## 三、使用 ViewModel 管理表单

```kotlin
class RegistrationViewModel : ViewModel() {
    
    private val validator = RegistrationFormValidator()
    
    private val _formState = MutableStateFlow(RegistrationFormState())
    val formState: StateFlow<RegistrationFormState> = _formState.asStateFlow()
    
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()
    
    sealed class RegistrationResult {
        data object Success : RegistrationResult()
        data class Error(val message: String) : RegistrationResult()
    }
    
    private val _registrationResult = MutableSharedFlow<RegistrationResult>()
    val registrationResult: SharedFlow<RegistrationResult> = _registrationResult.asSharedFlow()
    
    fun onUsernameChange(username: String) {
        _formState.update { state ->
            state.copy(
                username = FormField(
                    value = username,
                    error = validator.validateUsername(username)
                )
            )
        }
    }
    
    fun onEmailChange(email: String) {
        _formState.update { state ->
            state.copy(
                email = FormField(
                    value = email,
                    error = null  // 延迟验证
                )
            )
        }
    }
    
    fun onEmailFocusLost() {
        _formState.update { state ->
            state.copy(
                email = state.email.copy(
                    error = validator.validateEmail(state.email.value)
                )
            )
        }
    }
    
    fun onPasswordChange(password: String) {
        _formState.update { state ->
            state.copy(
                password = FormField(
                    value = password,
                    error = validator.validatePassword(password)
                )
            )
        }
    }
    
    fun submit() {
        viewModelScope.launch {
            val currentState = _formState.value
            
            // 验证所有字段
            val validatedState = currentState.copy(
                username = currentState.username.copy(
                    error = validator.validateUsername(currentState.username.value)
                ),
                email = currentState.email.copy(
                    error = validator.validateEmail(currentState.email.value)
                ),
                password = currentState.password.copy(
                    error = validator.validatePassword(currentState.password.value)
                )
            )
            
            _formState.value = validatedState
            
            if (validatedState.isValid()) {
                _isLoading.value = true
                try {
                    // 调用注册 API
                    delay(2000)  // 模拟网络请求
                    _registrationResult.emit(RegistrationResult.Success)
                } catch (e: Exception) {
                    _registrationResult.emit(RegistrationResult.Error(e.message ?: "注册失败"))
                } finally {
                    _isLoading.value = false
                }
            }
        }
    }
}
```

### 在 Composable 中使用

```kotlin
@Composable
fun RegistrationScreen(
    viewModel: RegistrationViewModel = viewModel(),
    onRegistrationSuccess: () -> Unit
) {
    val formState by viewModel.formState.collectAsState()
    val isLoading by viewModel.isLoading.collectAsState()
    val context = LocalContext.current
    
    LaunchedEffect(Unit) {
        viewModel.registrationResult.collect { result ->
            when (result) {
                is RegistrationViewModel.RegistrationResult.Success -> {
                    onRegistrationSuccess()
                }
                is RegistrationViewModel.RegistrationResult.Error -> {
                    Toast.makeText(context, result.message, Toast.LENGTH_SHORT).show()
                }
            }
        }
    }
    
    Column(modifier = Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = formState.username.value,
            onValueChange = viewModel::onUsernameChange,
            label = { Text("用户名") },
            isError = formState.username.error != null,
            supportingText = formState.username.error?.let { { Text(it) } },
            enabled = !isLoading,
            modifier = Modifier.fillMaxWidth()
        )
        
        // ... 其他字段
        
        Button(
            onClick = viewModel::submit,
            enabled = !isLoading,
            modifier = Modifier.fillMaxWidth()
        ) {
            if (isLoading) {
                CircularProgressIndicator(
                    modifier = Modifier.size(24.dp),
                    color = MaterialTheme.colorScheme.onPrimary
                )
            } else {
                Text("注册")
            }
        }
    }
}
```

## 四、键盘操作

### 焦点管理

```kotlin
@Composable
fun FormWithFocusManagement() {
    val focusManager = LocalFocusManager.current
    var username by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    
    Column {
        OutlinedTextField(
            value = username,
            onValueChange = { username = it },
            label = { Text("用户名") },
            keyboardOptions = KeyboardOptions(
                imeAction = ImeAction.Next
            ),
            keyboardActions = KeyboardActions(
                onNext = { focusManager.moveFocus(FocusDirection.Down) }
            ),
            modifier = Modifier.fillMaxWidth()
        )
        
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("密码") },
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Password,
                imeAction = ImeAction.Done
            ),
            keyboardActions = KeyboardActions(
                onDone = {
                    focusManager.clearFocus()
                    // 提交表单
                }
            ),
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

### 使用 FocusRequester

```kotlin
@Composable
fun FormWithFocusRequester() {
    val (usernameFocus, passwordFocus) = remember { FocusRequester.createRefs() }
    var username by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    
    LaunchedEffect(Unit) {
        usernameFocus.requestFocus()  // 自动聚焦第一个字段
    }
    
    Column {
        OutlinedTextField(
            value = username,
            onValueChange = { username = it },
            label = { Text("用户名") },
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(
                onNext = { passwordFocus.requestFocus() }
            ),
            modifier = Modifier
                .fillMaxWidth()
                .focusRequester(usernameFocus)
        )
        
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("密码") },
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
            modifier = Modifier
                .fillMaxWidth()
                .focusRequester(passwordFocus)
        )
    }
}
```

## 五、多步骤表单

```kotlin
enum class RegistrationStep {
    ACCOUNT_INFO,
    PERSONAL_INFO,
    PREFERENCES
}

data class MultiStepFormState(
    val currentStep: RegistrationStep = RegistrationStep.ACCOUNT_INFO,
    // Step 1
    val email: String = "",
    val password: String = "",
    // Step 2
    val firstName: String = "",
    val lastName: String = "",
    val birthDate: String = "",
    // Step 3
    val newsletter: Boolean = false,
    val notifications: Boolean = true
)

@Composable
fun MultiStepRegistration() {
    var formState by remember { mutableStateOf(MultiStepFormState()) }
    
    Column(modifier = Modifier.padding(16.dp)) {
        // 步骤指示器
        StepIndicator(
            currentStep = formState.currentStep,
            steps = RegistrationStep.entries
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        // 步骤内容
        when (formState.currentStep) {
            RegistrationStep.ACCOUNT_INFO -> {
                AccountInfoStep(
                    email = formState.email,
                    password = formState.password,
                    onEmailChange = { formState = formState.copy(email = it) },
                    onPasswordChange = { formState = formState.copy(password = it) }
                )
            }
            RegistrationStep.PERSONAL_INFO -> {
                PersonalInfoStep(
                    firstName = formState.firstName,
                    lastName = formState.lastName,
                    onFirstNameChange = { formState = formState.copy(firstName = it) },
                    onLastNameChange = { formState = formState.copy(lastName = it) }
                )
            }
            RegistrationStep.PREFERENCES -> {
                PreferencesStep(
                    newsletter = formState.newsletter,
                    notifications = formState.notifications,
                    onNewsletterChange = { formState = formState.copy(newsletter = it) },
                    onNotificationsChange = { formState = formState.copy(notifications = it) }
                )
            }
        }
        
        Spacer(modifier = Modifier.height(24.dp))
        
        // 导航按钮
        Row(
            horizontalArrangement = Arrangement.SpaceBetween,
            modifier = Modifier.fillMaxWidth()
        ) {
            if (formState.currentStep != RegistrationStep.ACCOUNT_INFO) {
                OutlinedButton(
                    onClick = {
                        formState = formState.copy(
                            currentStep = RegistrationStep.entries[formState.currentStep.ordinal - 1]
                        )
                    }
                ) {
                    Text("上一步")
                }
            } else {
                Spacer(modifier = Modifier.width(1.dp))
            }
            
            Button(
                onClick = {
                    if (formState.currentStep == RegistrationStep.PREFERENCES) {
                        // 提交表单
                    } else {
                        formState = formState.copy(
                            currentStep = RegistrationStep.entries[formState.currentStep.ordinal + 1]
                        )
                    }
                }
            ) {
                Text(
                    if (formState.currentStep == RegistrationStep.PREFERENCES) "完成" else "下一步"
                )
            }
        }
    }
}

@Composable
fun StepIndicator(
    currentStep: RegistrationStep,
    steps: List<RegistrationStep>
) {
    Row(
        horizontalArrangement = Arrangement.SpaceBetween,
        modifier = Modifier.fillMaxWidth()
    ) {
        steps.forEachIndexed { index, step ->
            val isActive = step == currentStep
            val isCompleted = step.ordinal < currentStep.ordinal
            
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                Box(
                    contentAlignment = Alignment.Center,
                    modifier = Modifier
                        .size(32.dp)
                        .background(
                            color = when {
                                isActive -> MaterialTheme.colorScheme.primary
                                isCompleted -> MaterialTheme.colorScheme.primary.copy(alpha = 0.6f)
                                else -> MaterialTheme.colorScheme.surfaceVariant
                            },
                            shape = CircleShape
                        )
                ) {
                    if (isCompleted) {
                        Icon(
                            Icons.Default.Check,
                            contentDescription = null,
                            tint = Color.White,
                            modifier = Modifier.size(16.dp)
                        )
                    } else {
                        Text(
                            text = "${index + 1}",
                            color = if (isActive) Color.White else Color.Gray
                        )
                    }
                }
            }
            
            if (index < steps.size - 1) {
                Divider(
                    modifier = Modifier
                        .weight(1f)
                        .padding(horizontal = 8.dp)
                        .align(Alignment.CenterVertically)
                )
            }
        }
    }
}
```

## 六、常用表单组件

### 下拉选择

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DropdownField(
    label: String,
    options: List<String>,
    selectedOption: String,
    onOptionSelected: (String) -> Unit
) {
    var expanded by remember { mutableStateOf(false) }
    
    ExposedDropdownMenuBox(
        expanded = expanded,
        onExpandedChange = { expanded = it }
    ) {
        OutlinedTextField(
            value = selectedOption,
            onValueChange = {},
            readOnly = true,
            label = { Text(label) },
            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded = expanded) },
            modifier = Modifier
                .menuAnchor()
                .fillMaxWidth()
        )
        
        ExposedDropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false }
        ) {
            options.forEach { option ->
                DropdownMenuItem(
                    text = { Text(option) },
                    onClick = {
                        onOptionSelected(option)
                        expanded = false
                    }
                )
            }
        }
    }
}
```

### 日期选择

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DatePickerField(
    label: String,
    selectedDate: Long?,
    onDateSelected: (Long) -> Unit
) {
    var showDatePicker by remember { mutableStateOf(false) }
    val dateFormatter = remember { SimpleDateFormat("yyyy-MM-dd", Locale.getDefault()) }
    
    OutlinedTextField(
        value = selectedDate?.let { dateFormatter.format(Date(it)) } ?: "",
        onValueChange = {},
        readOnly = true,
        label = { Text(label) },
        trailingIcon = {
            IconButton(onClick = { showDatePicker = true }) {
                Icon(Icons.Default.DateRange, contentDescription = "选择日期")
            }
        },
        modifier = Modifier.fillMaxWidth()
    )
    
    if (showDatePicker) {
        val datePickerState = rememberDatePickerState(initialSelectedDateMillis = selectedDate)
        
        DatePickerDialog(
            onDismissRequest = { showDatePicker = false },
            confirmButton = {
                TextButton(
                    onClick = {
                        datePickerState.selectedDateMillis?.let { onDateSelected(it) }
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

## 七、最佳实践

- ✅ 使用数据类封装表单状态
- ✅ 将验证逻辑与 UI 分离
- ✅ 提供清晰的错误提示
- ✅ 合理使用实时验证和提交验证
- ✅ 管理键盘焦点流转
- ✅ 在 ViewModel 中处理表单逻辑
- ✅ 使用 rememberSaveable 保存表单状态
- ✅ 禁用提交按钮直到表单有效

## 总结

Compose 表单处理的核心要点：

- **状态管理**：使用数据类封装表单字段
- **验证策略**：实时验证、失焦验证、提交验证
- **ViewModel 集成**：业务逻辑与 UI 分离
- **焦点管理**：FocusManager、FocusRequester
- **多步骤表单**：步骤指示器、状态保持
- **常用组件**：下拉选择、日期选择等

良好的表单处理是构建优秀用户体验的基础。

---

*© 2024 Fidroid. [返回首页](../index.html)*


