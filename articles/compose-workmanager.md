# Compose + WorkManager：后台任务处理最佳实践

> **发布日期**: 2024-06-01  
> **阅读时间**: 约 24 分钟  
> **标签**: WorkManager, Background, Worker, 后台任务

WorkManager 是 Android 官方推荐的后台任务解决方案，用于处理需要保证执行的延迟任务。本文将深入讲解如何在 Compose 应用中集成 WorkManager，实现可靠的后台任务处理。

## 一、WorkManager 简介

WorkManager 适用于：

- **可延迟的任务**：不需要立即执行
- **需要保证执行的任务**：即使应用退出或设备重启
- **需要约束条件的任务**：如需要网络连接

```kotlin
// 适合 WorkManager
- 上传日志到服务器
- 同步本地数据到云端
- 定期清理缓存
- 下载离线内容

// 不适合 WorkManager
- 需要立即完成的支付
- 实时聊天消息
- 播放音乐
```

## 二、基础配置

### 添加依赖

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.work:work-runtime-ktx:2.9.0")
    
    // 可选：Hilt 集成
    implementation("androidx.hilt:hilt-work:1.2.0")
    ksp("androidx.hilt:hilt-compiler:1.2.0")
}
```

### 创建 Worker

```kotlin
class UploadLogsWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            // 获取输入数据
            val userId = inputData.getString("user_id") ?: return Result.failure()
            
            // 执行后台任务
            uploadLogs(userId)
            
            // 返回成功
            Result.success()
        } catch (e: Exception) {
            // 返回重试或失败
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }
    
    private suspend fun uploadLogs(userId: String) {
        // 上传日志逻辑
    }
}
```

### 调度任务

```kotlin
class LogUploadScheduler(private val context: Context) {
    
    fun scheduleUpload(userId: String) {
        // 创建输入数据
        val inputData = workDataOf("user_id" to userId)
        
        // 创建任务请求
        val uploadRequest = OneTimeWorkRequestBuilder<UploadLogsWorker>()
            .setInputData(inputData)
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .setRequiresBatteryNotLow(true)
                    .build()
            )
            .setBackoffCriteria(
                BackoffPolicy.EXPONENTIAL,
                10, TimeUnit.SECONDS
            )
            .build()
        
        // 提交任务
        WorkManager.getInstance(context)
            .enqueue(uploadRequest)
    }
}
```

## 三、在 Compose 中观察任务状态

### 使用 LiveData

```kotlin
@Composable
fun WorkStatusScreen(workId: UUID) {
    val context = LocalContext.current
    val workManager = remember { WorkManager.getInstance(context) }
    
    // 观察任务状态
    val workInfoLiveData = remember(workId) {
        workManager.getWorkInfoByIdLiveData(workId)
    }
    val workInfo by workInfoLiveData.observeAsState()
    
    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        when (workInfo?.state) {
            WorkInfo.State.ENQUEUED -> {
                Text("任务已排队")
                CircularProgressIndicator()
            }
            WorkInfo.State.RUNNING -> {
                Text("任务执行中...")
                CircularProgressIndicator()
            }
            WorkInfo.State.SUCCEEDED -> {
                Icon(Icons.Default.Check, "成功", tint = Color.Green)
                Text("任务完成！")
            }
            WorkInfo.State.FAILED -> {
                Icon(Icons.Default.Error, "失败", tint = Color.Red)
                Text("任务失败")
            }
            WorkInfo.State.CANCELLED -> {
                Text("任务已取消")
            }
            else -> {
                Text("未知状态")
            }
        }
    }
}
```

### 使用 Flow

```kotlin
@Composable
fun WorkStatusWithFlow(workId: UUID) {
    val context = LocalContext.current
    val workManager = remember { WorkManager.getInstance(context) }
    
    // 使用 Flow 观察
    val workInfo by workManager
        .getWorkInfoByIdFlow(workId)
        .collectAsState(initial = null)
    
    WorkStatusContent(workInfo)
}

@Composable
fun WorkStatusContent(workInfo: WorkInfo?) {
    val state = workInfo?.state
    val progress = workInfo?.progress?.getInt("progress", 0) ?: 0
    
    Column {
        Text("状态: ${state?.name ?: "未知"}")
        
        if (state == WorkInfo.State.RUNNING) {
            LinearProgressIndicator(
                progress = { progress / 100f },
                modifier = Modifier.fillMaxWidth()
            )
            Text("进度: $progress%")
        }
    }
}
```

## 四、带进度的 Worker

```kotlin
class DownloadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        val url = inputData.getString("url") ?: return Result.failure()
        
        return try {
            downloadFile(url) { progress ->
                // 更新进度
                setProgress(workDataOf("progress" to progress))
            }
            Result.success()
        } catch (e: Exception) {
            Result.failure()
        }
    }
    
    private suspend fun downloadFile(url: String, onProgress: suspend (Int) -> Unit) {
        // 模拟下载
        for (i in 0..100 step 10) {
            delay(500)
            onProgress(i)
        }
    }
}

// 在 Compose 中显示进度
@Composable
fun DownloadProgress(workId: UUID) {
    val context = LocalContext.current
    val workInfo by WorkManager.getInstance(context)
        .getWorkInfoByIdFlow(workId)
        .collectAsState(initial = null)
    
    val progress = workInfo?.progress?.getInt("progress", 0) ?: 0
    
    Column(modifier = Modifier.padding(16.dp)) {
        Text("下载中...")
        LinearProgressIndicator(
            progress = { progress / 100f },
            modifier = Modifier.fillMaxWidth()
        )
        Text("$progress%")
    }
}
```

## 五、定期任务

```kotlin
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        // 同步数据
        syncData()
        return Result.success()
    }
}

// 调度定期任务
fun schedulePeriodicsync(context: Context) {
    val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(
        repeatInterval = 1, TimeUnit.HOURS,
        flexInterval = 15, TimeUnit.MINUTES
    )
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .build()
    
    WorkManager.getInstance(context)
        .enqueueUniquePeriodicWork(
            "sync_work",
            ExistingPeriodicWorkPolicy.KEEP,
            syncRequest
        )
}
```

## 六、链式任务

```kotlin
fun scheduleChainedWork(context: Context) {
    val workManager = WorkManager.getInstance(context)
    
    // 创建任务
    val downloadWork = OneTimeWorkRequestBuilder<DownloadWorker>().build()
    val processWork = OneTimeWorkRequestBuilder<ProcessWorker>().build()
    val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>().build()
    
    // 链式执行
    workManager
        .beginWith(downloadWork)
        .then(processWork)
        .then(uploadWork)
        .enqueue()
}

// 并行 + 顺序
fun scheduleComplexChain(context: Context) {
    val workManager = WorkManager.getInstance(context)
    
    val download1 = OneTimeWorkRequestBuilder<DownloadWorker>()
        .setInputData(workDataOf("file" to "file1"))
        .build()
    val download2 = OneTimeWorkRequestBuilder<DownloadWorker>()
        .setInputData(workDataOf("file" to "file2"))
        .build()
    val merge = OneTimeWorkRequestBuilder<MergeWorker>().build()
    
    // 并行下载，然后合并
    workManager
        .beginWith(listOf(download1, download2))
        .then(merge)
        .enqueue()
}
```

## 七、与 Hilt 集成

```kotlin
// Worker 中注入依赖
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repository: DataRepository,
    private val apiService: ApiService
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            val localData = repository.getUnsyncedData()
            apiService.syncData(localData)
            repository.markAsSynced()
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}

// Application 配置
@HiltAndroidApp
class MyApplication : Application(), Configuration.Provider {
    
    @Inject
    lateinit var workerFactory: HiltWorkerFactory
    
    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}
```

## 八、ViewModel 封装

```kotlin
@HiltViewModel
class SyncViewModel @Inject constructor(
    private val workManager: WorkManager
) : ViewModel() {
    
    private val _syncState = MutableStateFlow<SyncState>(SyncState.Idle)
    val syncState: StateFlow<SyncState> = _syncState.asStateFlow()
    
    private var currentWorkId: UUID? = null
    
    fun startSync() {
        val request = OneTimeWorkRequestBuilder<SyncWorker>()
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build()
            )
            .build()
        
        currentWorkId = request.id
        workManager.enqueue(request)
        
        // 观察状态
        viewModelScope.launch {
            workManager.getWorkInfoByIdFlow(request.id)
                .collect { workInfo ->
                    _syncState.value = when (workInfo?.state) {
                        WorkInfo.State.ENQUEUED -> SyncState.Queued
                        WorkInfo.State.RUNNING -> SyncState.Running(
                            workInfo.progress.getInt("progress", 0)
                        )
                        WorkInfo.State.SUCCEEDED -> SyncState.Success
                        WorkInfo.State.FAILED -> SyncState.Error("同步失败")
                        WorkInfo.State.CANCELLED -> SyncState.Idle
                        else -> SyncState.Idle
                    }
                }
        }
    }
    
    fun cancelSync() {
        currentWorkId?.let { workManager.cancelWorkById(it) }
    }
}

sealed class SyncState {
    data object Idle : SyncState()
    data object Queued : SyncState()
    data class Running(val progress: Int) : SyncState()
    data object Success : SyncState()
    data class Error(val message: String) : SyncState()
}
```

### 在 Compose 中使用

```kotlin
@Composable
fun SyncScreen(viewModel: SyncViewModel = hiltViewModel()) {
    val syncState by viewModel.syncState.collectAsState()
    
    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        when (val state = syncState) {
            is SyncState.Idle -> {
                Button(onClick = { viewModel.startSync() }) {
                    Text("开始同步")
                }
            }
            is SyncState.Queued -> {
                Text("等待执行...")
                CircularProgressIndicator()
            }
            is SyncState.Running -> {
                Text("同步中...")
                LinearProgressIndicator(
                    progress = { state.progress / 100f },
                    modifier = Modifier.fillMaxWidth()
                )
                Text("${state.progress}%")
                
                OutlinedButton(onClick = { viewModel.cancelSync() }) {
                    Text("取消")
                }
            }
            is SyncState.Success -> {
                Icon(
                    Icons.Default.CheckCircle,
                    contentDescription = "成功",
                    tint = Color.Green,
                    modifier = Modifier.size(48.dp)
                )
                Text("同步完成！")
                
                Button(onClick = { viewModel.startSync() }) {
                    Text("重新同步")
                }
            }
            is SyncState.Error -> {
                Icon(
                    Icons.Default.Error,
                    contentDescription = "错误",
                    tint = Color.Red,
                    modifier = Modifier.size(48.dp)
                )
                Text(state.message)
                
                Button(onClick = { viewModel.startSync() }) {
                    Text("重试")
                }
            }
        }
    }
}
```

## 九、前台服务通知

对于长时间运行的任务：

```kotlin
class LongRunningWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        // 设置为前台服务
        setForeground(createForegroundInfo())
        
        // 执行长时间任务
        performLongTask()
        
        return Result.success()
    }
    
    private fun createForegroundInfo(): ForegroundInfo {
        val notification = NotificationCompat.Builder(applicationContext, "work_channel")
            .setContentTitle("正在处理")
            .setContentText("请稍候...")
            .setSmallIcon(R.drawable.ic_sync)
            .setOngoing(true)
            .build()
        
        return ForegroundInfo(1, notification)
    }
}
```

## 十、最佳实践

- ✅ 使用 `CoroutineWorker` 支持协程
- ✅ 设置合理的约束条件（网络、电量等）
- ✅ 实现重试策略和错误处理
- ✅ 使用 `enqueueUniqueWork` 避免重复任务
- ✅ 通过 Flow/LiveData 观察任务状态
- ✅ 长时间任务使用前台服务通知
- ✅ 使用 Hilt 注入 Worker 依赖
- ✅ 在 ViewModel 中封装任务调度逻辑

## 总结

WorkManager 与 Compose 集成的核心要点：

- **CoroutineWorker**：支持协程的 Worker 实现
- **约束条件**：网络、电量、存储等限制
- **Flow 观察**：在 Compose 中实时显示任务状态
- **进度更新**：使用 setProgress 报告进度
- **链式任务**：beginWith().then() 顺序执行
- **Hilt 集成**：使用 @HiltWorker 注入依赖
- **ViewModel 封装**：统一管理任务状态

WorkManager 是处理可靠后台任务的最佳选择，结合 Compose 可以提供流畅的用户体验。

---

*© 2024 Fidroid. [返回首页](../index.html)*

