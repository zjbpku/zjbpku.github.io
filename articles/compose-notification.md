# Compose 通知：Notification 与 PendingIntent

> **发布日期**: 2024-06-25  
> **阅读时间**: 约 24 分钟  
> **标签**: Notification, PendingIntent, NotificationChannel

通知是 Android 应用与用户交互的重要方式。本文将深入讲解如何在 Compose 应用中正确使用通知系统，包括创建通知、处理点击事件和通知渠道管理。

## 一、基础配置

### 权限声明

```xml
<manifest>
    <!-- Android 13+ 需要通知权限 -->
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
</manifest>
```

### 请求通知权限

```kotlin
@Composable
fun RequestNotificationPermission() {
    val context = LocalContext.current
    
    val hasPermission = remember {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            ContextCompat.checkSelfPermission(
                context,
                Manifest.permission.POST_NOTIFICATIONS
            ) == PackageManager.PERMISSION_GRANTED
        } else true
    }
    
    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            // 权限已授予
        }
    }
    
    LaunchedEffect(Unit) {
        if (!hasPermission && Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            permissionLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
        }
    }
}
```

## 二、创建通知渠道

```kotlin
object NotificationChannels {
    const val MESSAGES = "messages"
    const val UPDATES = "updates"
    const val REMINDERS = "reminders"
    
    fun createChannels(context: Context) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val notificationManager = context.getSystemService(
                Context.NOTIFICATION_SERVICE
            ) as NotificationManager
            
            val channels = listOf(
                NotificationChannel(
                    MESSAGES,
                    "消息通知",
                    NotificationManager.IMPORTANCE_HIGH
                ).apply {
                    description = "接收新消息通知"
                    enableLights(true)
                    enableVibration(true)
                },
                NotificationChannel(
                    UPDATES,
                    "更新通知",
                    NotificationManager.IMPORTANCE_DEFAULT
                ).apply {
                    description = "应用更新和公告"
                },
                NotificationChannel(
                    REMINDERS,
                    "提醒",
                    NotificationManager.IMPORTANCE_LOW
                ).apply {
                    description = "日程和任务提醒"
                }
            )
            
            channels.forEach { channel ->
                notificationManager.createNotificationChannel(channel)
            }
        }
    }
}

// 在 Application 中调用
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        NotificationChannels.createChannels(this)
    }
}
```

## 三、发送基础通知

```kotlin
class NotificationHelper(private val context: Context) {
    
    private val notificationManager = NotificationManagerCompat.from(context)
    
    fun showSimpleNotification(
        title: String,
        message: String,
        notificationId: Int = System.currentTimeMillis().toInt()
    ) {
        val notification = NotificationCompat.Builder(context, NotificationChannels.MESSAGES)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(message)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .build()
        
        if (ActivityCompat.checkSelfPermission(
            context, 
            Manifest.permission.POST_NOTIFICATIONS
        ) == PackageManager.PERMISSION_GRANTED) {
            notificationManager.notify(notificationId, notification)
        }
    }
    
    fun showNotificationWithAction(
        title: String,
        message: String
    ) {
        // 点击通知打开应用
        val intent = Intent(context, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
            putExtra("from_notification", true)
        }
        
        val pendingIntent = PendingIntent.getActivity(
            context,
            0,
            intent,
            PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
        )
        
        // 操作按钮
        val replyIntent = Intent(context, NotificationReceiver::class.java).apply {
            action = "ACTION_REPLY"
        }
        val replyPendingIntent = PendingIntent.getBroadcast(
            context,
            1,
            replyIntent,
            PendingIntent.FLAG_IMMUTABLE
        )
        
        val dismissIntent = Intent(context, NotificationReceiver::class.java).apply {
            action = "ACTION_DISMISS"
        }
        val dismissPendingIntent = PendingIntent.getBroadcast(
            context,
            2,
            dismissIntent,
            PendingIntent.FLAG_IMMUTABLE
        )
        
        val notification = NotificationCompat.Builder(context, NotificationChannels.MESSAGES)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(message)
            .setContentIntent(pendingIntent)
            .addAction(R.drawable.ic_reply, "回复", replyPendingIntent)
            .addAction(R.drawable.ic_dismiss, "忽略", dismissPendingIntent)
            .setAutoCancel(true)
            .build()
        
        notificationManager.notify(1, notification)
    }
}
```

## 四、带图片的通知

```kotlin
fun showBigPictureNotification(
    title: String,
    message: String,
    bitmap: Bitmap
) {
    val bigPictureStyle = NotificationCompat.BigPictureStyle()
        .bigPicture(bitmap)
        .bigLargeIcon(null as Bitmap?)
    
    val notification = NotificationCompat.Builder(context, NotificationChannels.UPDATES)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle(title)
        .setContentText(message)
        .setLargeIcon(bitmap)
        .setStyle(bigPictureStyle)
        .build()
    
    notificationManager.notify(2, notification)
}
```

## 五、进度通知

```kotlin
fun showProgressNotification(
    title: String,
    progress: Int,
    max: Int = 100
) {
    val notification = NotificationCompat.Builder(context, NotificationChannels.UPDATES)
        .setSmallIcon(R.drawable.ic_download)
        .setContentTitle(title)
        .setContentText("下载中 $progress%")
        .setProgress(max, progress, false)
        .setOngoing(true)  // 用户无法滑动删除
        .build()
    
    notificationManager.notify(3, notification)
}

fun completeProgressNotification(title: String) {
    val notification = NotificationCompat.Builder(context, NotificationChannels.UPDATES)
        .setSmallIcon(R.drawable.ic_done)
        .setContentTitle(title)
        .setContentText("下载完成")
        .setProgress(0, 0, false)
        .build()
    
    notificationManager.notify(3, notification)
}

// 在 ViewModel 中使用
class DownloadViewModel(
    private val notificationHelper: NotificationHelper
) : ViewModel() {
    
    fun startDownload() {
        viewModelScope.launch {
            for (progress in 0..100 step 10) {
                notificationHelper.showProgressNotification("下载文件", progress)
                delay(500)
            }
            notificationHelper.completeProgressNotification("下载文件")
        }
    }
}
```

## 六、分组通知

```kotlin
fun showGroupedNotifications(messages: List<Message>) {
    val groupKey = "com.example.MESSAGES"
    
    // 创建摘要通知
    val summaryNotification = NotificationCompat.Builder(context, NotificationChannels.MESSAGES)
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("${messages.size} 条新消息")
        .setContentText("来自多个对话")
        .setGroup(groupKey)
        .setGroupSummary(true)
        .setStyle(
            NotificationCompat.InboxStyle()
                .also { style ->
                    messages.take(5).forEach { message ->
                        style.addLine("${message.sender}: ${message.content}")
                    }
                }
                .setBigContentTitle("${messages.size} 条新消息")
        )
        .build()
    
    // 创建单独的通知
    messages.forEachIndexed { index, message ->
        val notification = NotificationCompat.Builder(context, NotificationChannels.MESSAGES)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(message.sender)
            .setContentText(message.content)
            .setGroup(groupKey)
            .build()
        
        notificationManager.notify(100 + index, notification)
    }
    
    notificationManager.notify(0, summaryNotification)
}
```

## 七、处理通知点击

### 使用 Deep Link

```kotlin
// 创建带 Deep Link 的 PendingIntent
fun createDeepLinkPendingIntent(articleId: String): PendingIntent {
    val deepLinkIntent = Intent(
        Intent.ACTION_VIEW,
        "https://example.com/article/$articleId".toUri(),
        context,
        MainActivity::class.java
    )
    
    return TaskStackBuilder.create(context).run {
        addNextIntentWithParentStack(deepLinkIntent)
        getPendingIntent(
            articleId.hashCode(),
            PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
        )!!
    }
}

// 在 Navigation 中处理
NavHost(navController, startDestination = "home") {
    composable(
        route = "article/{id}",
        deepLinks = listOf(
            navDeepLink { uriPattern = "https://example.com/article/{id}" }
        )
    ) { backStackEntry ->
        val articleId = backStackEntry.arguments?.getString("id")
        ArticleDetailScreen(articleId)
    }
}
```

### 处理通知操作

```kotlin
class NotificationReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        when (intent.action) {
            "ACTION_REPLY" -> {
                // 处理回复
                Toast.makeText(context, "回复", Toast.LENGTH_SHORT).show()
            }
            "ACTION_DISMISS" -> {
                // 取消通知
                val notificationManager = NotificationManagerCompat.from(context)
                notificationManager.cancel(1)
            }
        }
    }
}

// 在 AndroidManifest.xml 中注册
<receiver android:name=".NotificationReceiver" android:exported="false" />
```

## 八、在 Compose 中管理通知

```kotlin
@HiltViewModel
class NotificationViewModel @Inject constructor(
    private val notificationHelper: NotificationHelper
) : ViewModel() {
    
    fun sendNotification(title: String, message: String) {
        notificationHelper.showSimpleNotification(title, message)
    }
    
    fun cancelNotification(id: Int) {
        notificationHelper.cancel(id)
    }
    
    fun cancelAllNotifications() {
        notificationHelper.cancelAll()
    }
}

@Composable
fun NotificationTestScreen(
    viewModel: NotificationViewModel = hiltViewModel()
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        Button(
            onClick = { 
                viewModel.sendNotification("测试通知", "这是一条测试消息") 
            }
        ) {
            Text("发送通知")
        }
        
        Button(
            onClick = { viewModel.cancelAllNotifications() }
        ) {
            Text("取消所有通知")
        }
    }
}
```

## 九、前台服务通知

```kotlin
class DownloadService : Service() {
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = NotificationCompat.Builder(this, NotificationChannels.UPDATES)
            .setSmallIcon(R.drawable.ic_download)
            .setContentTitle("下载中")
            .setContentText("正在下载文件...")
            .setOngoing(true)
            .build()
        
        startForeground(NOTIFICATION_ID, notification)
        
        // 执行下载任务
        
        return START_NOT_STICKY
    }
    
    companion object {
        const val NOTIFICATION_ID = 1001
    }
}
```

## 十、最佳实践

- ✅ Android 13+ 请求通知权限
- ✅ 创建合适的通知渠道
- ✅ 使用 FLAG_IMMUTABLE 创建 PendingIntent
- ✅ 设置正确的优先级和重要性
- ✅ 使用 Deep Link 处理通知点击
- ✅ 分组相关通知
- ✅ 进度通知使用 setOngoing(true)
- ✅ 适时取消不需要的通知

## 总结

Compose 应用通知的核心要点：

- **NotificationChannel**：Android 8+ 必须创建渠道
- **NotificationCompat**：兼容不同 Android 版本
- **PendingIntent**：处理通知点击和操作
- **Deep Link**：与 Navigation 集成
- **分组通知**：管理多条相关通知
- **前台服务**：长时间运行的任务

正确使用通知可以提升用户参与度和体验。

---

*© 2024 Fidroid. [返回首页](../index.html)*


