# Compose + Media3：视频播放与 ExoPlayer

> **发布日期**: 2024-06-29  
> **阅读时间**: 约 24 分钟  
> **标签**: Media3, ExoPlayer, 视频播放, 音频

Media3 是 Google 推出的新一代媒体库，整合了 ExoPlayer、MediaSession 等组件。本文将深入讲解如何在 Compose 中集成 Media3 实现视频播放功能。

## 一、基础配置

### 添加依赖

```kotlin
// build.gradle.kts
dependencies {
    val media3Version = "1.2.1"
    implementation("androidx.media3:media3-exoplayer:$media3Version")
    implementation("androidx.media3:media3-exoplayer-dash:$media3Version")
    implementation("androidx.media3:media3-exoplayer-hls:$media3Version")
    implementation("androidx.media3:media3-ui:$media3Version")
    implementation("androidx.media3:media3-session:$media3Version")
}
```

## 二、基础视频播放器

```kotlin
@Composable
fun VideoPlayer(
    videoUrl: String,
    modifier: Modifier = Modifier
) {
    val context = LocalContext.current
    
    val exoPlayer = remember {
        ExoPlayer.Builder(context).build().apply {
            val mediaItem = MediaItem.fromUri(videoUrl)
            setMediaItem(mediaItem)
            prepare()
        }
    }
    
    DisposableEffect(Unit) {
        onDispose {
            exoPlayer.release()
        }
    }
    
    AndroidView(
        factory = { ctx ->
            PlayerView(ctx).apply {
                player = exoPlayer
                useController = true
            }
        },
        modifier = modifier
    )
}

// 使用
@Composable
fun VideoScreen() {
    VideoPlayer(
        videoUrl = "https://example.com/video.mp4",
        modifier = Modifier
            .fillMaxWidth()
            .aspectRatio(16f / 9f)
    )
}
```

## 三、封装播放器状态

```kotlin
class VideoPlayerState(
    private val context: Context
) {
    val player: ExoPlayer = ExoPlayer.Builder(context).build()
    
    var isPlaying by mutableStateOf(false)
        private set
    
    var currentPosition by mutableLongStateOf(0L)
        private set
    
    var duration by mutableLongStateOf(0L)
        private set
    
    var bufferedPercentage by mutableIntStateOf(0)
        private set
    
    var playbackState by mutableIntStateOf(Player.STATE_IDLE)
        private set
    
    private val listener = object : Player.Listener {
        override fun onIsPlayingChanged(playing: Boolean) {
            isPlaying = playing
        }
        
        override fun onPlaybackStateChanged(state: Int) {
            playbackState = state
            if (state == Player.STATE_READY) {
                duration = player.duration
            }
        }
    }
    
    init {
        player.addListener(listener)
    }
    
    fun setMediaItem(url: String) {
        player.setMediaItem(MediaItem.fromUri(url))
        player.prepare()
    }
    
    fun play() {
        player.play()
    }
    
    fun pause() {
        player.pause()
    }
    
    fun seekTo(position: Long) {
        player.seekTo(position)
    }
    
    fun updatePosition() {
        currentPosition = player.currentPosition
        bufferedPercentage = player.bufferedPercentage
    }
    
    fun release() {
        player.removeListener(listener)
        player.release()
    }
}

@Composable
fun rememberVideoPlayerState(): VideoPlayerState {
    val context = LocalContext.current
    return remember { VideoPlayerState(context) }
}
```

## 四、自定义播放器 UI

```kotlin
@Composable
fun CustomVideoPlayer(
    videoUrl: String,
    modifier: Modifier = Modifier
) {
    val playerState = rememberVideoPlayerState()
    var showControls by remember { mutableStateOf(true) }
    
    LaunchedEffect(videoUrl) {
        playerState.setMediaItem(videoUrl)
    }
    
    // 更新播放位置
    LaunchedEffect(playerState.isPlaying) {
        while (playerState.isPlaying) {
            playerState.updatePosition()
            delay(1000)
        }
    }
    
    // 自动隐藏控制栏
    LaunchedEffect(showControls) {
        if (showControls && playerState.isPlaying) {
            delay(3000)
            showControls = false
        }
    }
    
    DisposableEffect(Unit) {
        onDispose { playerState.release() }
    }
    
    Box(
        modifier = modifier
            .background(Color.Black)
            .clickable { showControls = !showControls }
    ) {
        AndroidView(
            factory = { ctx ->
                PlayerView(ctx).apply {
                    player = playerState.player
                    useController = false  // 使用自定义控制器
                }
            },
            modifier = Modifier.fillMaxSize()
        )
        
        // 自定义控制栏
        AnimatedVisibility(
            visible = showControls,
            enter = fadeIn(),
            exit = fadeOut(),
            modifier = Modifier.fillMaxSize()
        ) {
            VideoControls(
                isPlaying = playerState.isPlaying,
                currentPosition = playerState.currentPosition,
                duration = playerState.duration,
                bufferedPercentage = playerState.bufferedPercentage,
                onPlayPause = {
                    if (playerState.isPlaying) playerState.pause()
                    else playerState.play()
                },
                onSeek = { playerState.seekTo(it) }
            )
        }
        
        // 加载指示器
        if (playerState.playbackState == Player.STATE_BUFFERING) {
            CircularProgressIndicator(
                modifier = Modifier.align(Alignment.Center),
                color = Color.White
            )
        }
    }
}

@Composable
fun VideoControls(
    isPlaying: Boolean,
    currentPosition: Long,
    duration: Long,
    bufferedPercentage: Int,
    onPlayPause: () -> Unit,
    onSeek: (Long) -> Unit
) {
    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(Color.Black.copy(alpha = 0.3f))
    ) {
        // 播放/暂停按钮
        IconButton(
            onClick = onPlayPause,
            modifier = Modifier.align(Alignment.Center)
        ) {
            Icon(
                imageVector = if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                contentDescription = if (isPlaying) "暂停" else "播放",
                tint = Color.White,
                modifier = Modifier.size(64.dp)
            )
        }
        
        // 进度条
        Column(
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .fillMaxWidth()
                .padding(16.dp)
        ) {
            // 时间显示
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                Text(
                    text = formatDuration(currentPosition),
                    color = Color.White,
                    fontSize = 12.sp
                )
                Text(
                    text = formatDuration(duration),
                    color = Color.White,
                    fontSize = 12.sp
                )
            }
            
            Spacer(modifier = Modifier.height(4.dp))
            
            // 进度条
            Slider(
                value = if (duration > 0) currentPosition.toFloat() / duration else 0f,
                onValueChange = { onSeek((it * duration).toLong()) },
                colors = SliderDefaults.colors(
                    thumbColor = Color.White,
                    activeTrackColor = Color.White,
                    inactiveTrackColor = Color.White.copy(alpha = 0.3f)
                )
            )
        }
    }
}

fun formatDuration(millis: Long): String {
    val seconds = (millis / 1000) % 60
    val minutes = (millis / (1000 * 60)) % 60
    val hours = millis / (1000 * 60 * 60)
    
    return if (hours > 0) {
        String.format("%d:%02d:%02d", hours, minutes, seconds)
    } else {
        String.format("%02d:%02d", minutes, seconds)
    }
}
```

## 五、播放列表

```kotlin
@Composable
fun PlaylistPlayer(
    playlist: List<MediaItem>,
    modifier: Modifier = Modifier
) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItems(playlist)
            prepare()
        }
    }
    
    var currentIndex by remember { mutableIntStateOf(0) }
    
    LaunchedEffect(Unit) {
        player.addListener(object : Player.Listener {
            override fun onMediaItemTransition(mediaItem: MediaItem?, reason: Int) {
                currentIndex = player.currentMediaItemIndex
            }
        })
    }
    
    DisposableEffect(Unit) {
        onDispose { player.release() }
    }
    
    Column(modifier = modifier) {
        // 视频播放器
        AndroidView(
            factory = { ctx ->
                PlayerView(ctx).apply {
                    this.player = player
                }
            },
            modifier = Modifier
                .fillMaxWidth()
                .aspectRatio(16f / 9f)
        )
        
        // 播放列表
        LazyColumn {
            itemsIndexed(playlist) { index, item ->
                ListItem(
                    headlineContent = { Text(item.mediaMetadata.title?.toString() ?: "视频 $index") },
                    leadingContent = {
                        if (index == currentIndex) {
                            Icon(Icons.Default.PlayArrow, contentDescription = null)
                        }
                    },
                    modifier = Modifier.clickable {
                        player.seekTo(index, 0)
                        player.play()
                    }
                )
            }
        }
    }
}
```

## 六、全屏播放

```kotlin
@Composable
fun FullscreenVideoPlayer(
    videoUrl: String,
    onClose: () -> Unit
) {
    val context = LocalContext.current
    val activity = context as? Activity
    
    // 进入全屏
    DisposableEffect(Unit) {
        val originalOrientation = activity?.requestedOrientation
        activity?.requestedOrientation = ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE
        
        // 隐藏系统 UI
        activity?.window?.let { window ->
            WindowCompat.setDecorFitsSystemWindows(window, false)
            WindowInsetsControllerCompat(window, window.decorView).let { controller ->
                controller.hide(WindowInsetsCompat.Type.systemBars())
                controller.systemBarsBehavior = 
                    WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
            }
        }
        
        onDispose {
            activity?.requestedOrientation = originalOrientation ?: 
                ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED
            activity?.window?.let { window ->
                WindowCompat.setDecorFitsSystemWindows(window, true)
                WindowInsetsControllerCompat(window, window.decorView)
                    .show(WindowInsetsCompat.Type.systemBars())
            }
        }
    }
    
    Box(modifier = Modifier.fillMaxSize().background(Color.Black)) {
        VideoPlayer(
            videoUrl = videoUrl,
            modifier = Modifier.fillMaxSize()
        )
        
        IconButton(
            onClick = onClose,
            modifier = Modifier
                .align(Alignment.TopEnd)
                .padding(16.dp)
        ) {
            Icon(
                Icons.Default.Close,
                contentDescription = "关闭",
                tint = Color.White
            )
        }
    }
}
```

## 七、画中画（PiP）

```kotlin
@Composable
fun PipVideoPlayer(
    videoUrl: String
) {
    val context = LocalContext.current
    val activity = context as? Activity
    val playerState = rememberVideoPlayerState()
    
    LaunchedEffect(videoUrl) {
        playerState.setMediaItem(videoUrl)
        playerState.play()
    }
    
    // 进入画中画
    fun enterPip() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val params = PictureInPictureParams.Builder()
                .setAspectRatio(Rational(16, 9))
                .build()
            activity?.enterPictureInPictureMode(params)
        }
    }
    
    Column {
        AndroidView(
            factory = { ctx ->
                PlayerView(ctx).apply {
                    player = playerState.player
                }
            },
            modifier = Modifier
                .fillMaxWidth()
                .aspectRatio(16f / 9f)
        )
        
        Button(onClick = { enterPip() }) {
            Text("画中画")
        }
    }
}
```

## 八、最佳实践

- ✅ 使用 DisposableEffect 释放播放器
- ✅ 处理生命周期（暂停/恢复）
- ✅ 缓存播放器实例
- ✅ 使用 MediaSession 支持媒体控制
- ✅ 处理音频焦点
- ✅ 支持后台播放
- ✅ 实现画中画模式
- ✅ 处理网络变化

## 总结

Media3 与 Compose 集成的核心要点：

- **ExoPlayer**：强大的媒体播放引擎
- **PlayerView**：通过 AndroidView 集成
- **自定义控制器**：完全自定义播放 UI
- **播放列表**：支持多媒体播放
- **全屏/画中画**：完整的播放体验
- **MediaSession**：系统媒体控制集成

Media3 是构建媒体应用的首选方案。

---

*© 2024 Fidroid. [返回首页](../index.html)*

