# Compose 权限处理：请求与状态管理

> **发布日期**: 2024-06-21  
> **阅读时间**: 约 22 分钟  
> **标签**: Permissions, ActivityResult, Runtime Permission

Android 运行时权限是应用开发中的重要环节。Compose 提供了简洁的 API 来处理权限请求。本文将深入讲解 Compose 中权限处理的最佳实践。

## 一、基础配置

### 添加依赖

```kotlin
// build.gradle.kts
dependencies {
    // Accompanist Permissions（可选，提供更丰富的 API）
    implementation("com.google.accompanist:accompanist-permissions:0.34.0")
}
```

### AndroidManifest 声明

```xml
<manifest>
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
</manifest>
```

## 二、使用 ActivityResultLauncher

### 单个权限请求

```kotlin
@Composable
fun SinglePermissionExample() {
    val context = LocalContext.current
    var hasCameraPermission by remember {
        mutableStateOf(
            ContextCompat.checkSelfPermission(
                context,
                Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED
        )
    }
    
    val permissionLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        hasCameraPermission = isGranted
        if (!isGranted) {
            // 权限被拒绝
        }
    }
    
    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        if (hasCameraPermission) {
            Text("相机权限已授予 ✓")
            // 显示相机功能
        } else {
            Text("需要相机权限才能使用此功能")
            Spacer(modifier = Modifier.height(16.dp))
            Button(
                onClick = {
                    permissionLauncher.launch(Manifest.permission.CAMERA)
                }
            ) {
                Text("授予相机权限")
            }
        }
    }
}
```

### 多个权限请求

```kotlin
@Composable
fun MultiplePermissionsExample() {
    val context = LocalContext.current
    
    val permissions = arrayOf(
        Manifest.permission.CAMERA,
        Manifest.permission.RECORD_AUDIO
    )
    
    var permissionsGranted by remember {
        mutableStateOf(
            permissions.all {
                ContextCompat.checkSelfPermission(context, it) == 
                    PackageManager.PERMISSION_GRANTED
            }
        )
    }
    
    val permissionLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestMultiplePermissions()
    ) { permissionsMap ->
        permissionsGranted = permissionsMap.values.all { it }
    }
    
    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        if (permissionsGranted) {
            Text("所有权限已授予")
            // 显示功能
        } else {
            Text("需要相机和麦克风权限")
            Button(
                onClick = { permissionLauncher.launch(permissions) }
            ) {
                Text("授予权限")
            }
        }
    }
}
```

## 三、使用 Accompanist Permissions

### 单个权限

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun AccompanistSinglePermission() {
    val cameraPermissionState = rememberPermissionState(
        permission = Manifest.permission.CAMERA
    )
    
    when {
        cameraPermissionState.status.isGranted -> {
            Text("相机权限已授予")
            CameraPreview()
        }
        cameraPermissionState.status.shouldShowRationale -> {
            // 需要解释为什么需要权限
            RationaleDialog(
                onConfirm = { cameraPermissionState.launchPermissionRequest() },
                onDismiss = { /* 用户拒绝 */ }
            )
        }
        else -> {
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                Text("此功能需要相机权限")
                Button(onClick = { cameraPermissionState.launchPermissionRequest() }) {
                    Text("请求权限")
                }
            }
        }
    }
}

@Composable
fun RationaleDialog(
    onConfirm: () -> Unit,
    onDismiss: () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("需要相机权限") },
        text = { Text("此应用需要访问相机来拍摄照片。请授予相机权限。") },
        confirmButton = {
            TextButton(onClick = onConfirm) { Text("授予") }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("取消") }
        }
    )
}
```

### 多个权限

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun AccompanistMultiplePermissions() {
    val permissionsState = rememberMultiplePermissionsState(
        permissions = listOf(
            Manifest.permission.CAMERA,
            Manifest.permission.RECORD_AUDIO,
            Manifest.permission.ACCESS_FINE_LOCATION
        )
    )
    
    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        // 显示每个权限状态
        permissionsState.permissions.forEach { perm ->
            val status = when {
                perm.status.isGranted -> "已授予 ✓"
                perm.status.shouldShowRationale -> "需要解释"
                else -> "未授予"
            }
            Text("${perm.permission.split(".").last()}: $status")
        }
        
        Spacer(modifier = Modifier.height(16.dp))
        
        when {
            permissionsState.allPermissionsGranted -> {
                Text("所有权限已授予！")
            }
            permissionsState.shouldShowRationale -> {
                Text("某些权限需要解释")
                Button(onClick = { permissionsState.launchMultiplePermissionRequest() }) {
                    Text("请求权限")
                }
            }
            else -> {
                Button(onClick = { permissionsState.launchMultiplePermissionRequest() }) {
                    Text("请求所有权限")
                }
            }
        }
    }
}
```

## 四、权限状态封装

```kotlin
sealed class PermissionStatus {
    data object Granted : PermissionStatus()
    data object Denied : PermissionStatus()
    data object PermanentlyDenied : PermissionStatus()
    data object RequiresRationale : PermissionStatus()
}

@Composable
fun rememberPermissionStatus(permission: String): PermissionStatus {
    val context = LocalContext.current
    val activity = context as? Activity
    
    return remember(permission) {
        when {
            ContextCompat.checkSelfPermission(context, permission) == 
                PackageManager.PERMISSION_GRANTED -> PermissionStatus.Granted
            activity?.shouldShowRequestPermissionRationale(permission) == true -> 
                PermissionStatus.RequiresRationale
            else -> PermissionStatus.Denied
        }
    }
}
```

## 五、处理永久拒绝

```kotlin
@Composable
fun HandlePermanentlyDenied() {
    val context = LocalContext.current
    var showSettingsDialog by remember { mutableStateOf(false) }
    
    val permissionLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (!isGranted) {
            // 检查是否永久拒绝
            val activity = context as? Activity
            val shouldShowRationale = activity?.shouldShowRequestPermissionRationale(
                Manifest.permission.CAMERA
            ) ?: false
            
            if (!shouldShowRationale) {
                // 永久拒绝，引导去设置
                showSettingsDialog = true
            }
        }
    }
    
    if (showSettingsDialog) {
        AlertDialog(
            onDismissRequest = { showSettingsDialog = false },
            title = { Text("权限被禁用") },
            text = { Text("相机权限已被永久禁用。请前往设置手动开启。") },
            confirmButton = {
                TextButton(
                    onClick = {
                        showSettingsDialog = false
                        // 打开应用设置页面
                        Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                            data = Uri.fromParts("package", context.packageName, null)
                            context.startActivity(this)
                        }
                    }
                ) {
                    Text("前往设置")
                }
            },
            dismissButton = {
                TextButton(onClick = { showSettingsDialog = false }) {
                    Text("取消")
                }
            }
        )
    }
    
    Button(onClick = { permissionLauncher.launch(Manifest.permission.CAMERA) }) {
        Text("请求相机权限")
    }
}
```

## 六、位置权限特殊处理

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun LocationPermissionExample() {
    // Android 10+ 需要分步请求位置权限
    val fineLocationState = rememberPermissionState(
        Manifest.permission.ACCESS_FINE_LOCATION
    )
    
    val backgroundLocationState = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        rememberPermissionState(Manifest.permission.ACCESS_BACKGROUND_LOCATION)
    } else null
    
    Column(modifier = Modifier.padding(16.dp)) {
        // 前台位置权限
        when {
            fineLocationState.status.isGranted -> {
                Text("前台位置权限已授予 ✓")
                
                // 仅在前台权限授予后才请求后台权限
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                    Spacer(modifier = Modifier.height(16.dp))
                    
                    if (backgroundLocationState?.status?.isGranted == true) {
                        Text("后台位置权限已授予 ✓")
                    } else {
                        Text("需要后台位置权限以在后台追踪位置")
                        Button(
                            onClick = { 
                                backgroundLocationState?.launchPermissionRequest() 
                            }
                        ) {
                            Text("授予后台位置权限")
                        }
                    }
                }
            }
            else -> {
                Button(onClick = { fineLocationState.launchPermissionRequest() }) {
                    Text("授予位置权限")
                }
            }
        }
    }
}
```

## 七、通知权限（Android 13+）

```kotlin
@Composable
fun NotificationPermissionExample() {
    val context = LocalContext.current
    
    // Android 13+ 需要请求通知权限
    var hasNotificationPermission by remember {
        mutableStateOf(
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                ContextCompat.checkSelfPermission(
                    context,
                    Manifest.permission.POST_NOTIFICATIONS
                ) == PackageManager.PERMISSION_GRANTED
            } else true
        )
    }
    
    val permissionLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        hasNotificationPermission = isGranted
    }
    
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU && !hasNotificationPermission) {
        Card(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp)
        ) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("开启通知", style = MaterialTheme.typography.titleMedium)
                Text("开启通知以接收重要更新", style = MaterialTheme.typography.bodyMedium)
                Spacer(modifier = Modifier.height(8.dp))
                Button(
                    onClick = {
                        permissionLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
                    }
                ) {
                    Text("开启通知")
                }
            }
        }
    }
}
```

## 八、权限请求 UI 模式

```kotlin
@Composable
fun PermissionRequiredScreen(
    permission: String,
    rationaleTitle: String,
    rationaleMessage: String,
    content: @Composable () -> Unit
) {
    val context = LocalContext.current
    var hasPermission by remember {
        mutableStateOf(
            ContextCompat.checkSelfPermission(context, permission) == 
                PackageManager.PERMISSION_GRANTED
        )
    }
    var showRationale by remember { mutableStateOf(false) }
    
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        hasPermission = granted
        if (!granted) {
            showRationale = true
        }
    }
    
    if (hasPermission) {
        content()
    } else {
        PermissionRequestUI(
            onRequest = { launcher.launch(permission) }
        )
    }
    
    if (showRationale) {
        AlertDialog(
            onDismissRequest = { showRationale = false },
            title = { Text(rationaleTitle) },
            text = { Text(rationaleMessage) },
            confirmButton = {
                TextButton(onClick = { 
                    showRationale = false
                    launcher.launch(permission) 
                }) {
                    Text("重试")
                }
            }
        )
    }
}

// 使用
@Composable
fun CameraScreen() {
    PermissionRequiredScreen(
        permission = Manifest.permission.CAMERA,
        rationaleTitle = "需要相机权限",
        rationaleMessage = "此功能需要访问相机才能正常工作。"
    ) {
        // 相机功能内容
        CameraPreview()
    }
}
```

## 九、最佳实践

- ✅ 在需要时才请求权限（延迟请求）
- ✅ 解释为什么需要权限（shouldShowRationale）
- ✅ 处理永久拒绝情况（引导到设置）
- ✅ 分步请求敏感权限（位置权限）
- ✅ Android 13+ 单独处理通知权限
- ✅ 使用封装组件简化权限请求
- ✅ 提供无权限时的降级体验
- ✅ 记住用户选择，避免重复请求

## 总结

Compose 权限处理的核心要点：

- **rememberLauncherForActivityResult**：请求权限的核心 API
- **shouldShowRationale**：判断是否需要解释
- **永久拒绝处理**：引导到应用设置
- **Accompanist Permissions**：更丰富的权限 API
- **权限分组**：位置、通知等特殊处理
- **UI 模式**：封装可复用的权限请求组件

正确处理权限可以提升用户体验和应用信任度。

---

*© 2024 Fidroid. [返回首页](../index.html)*


