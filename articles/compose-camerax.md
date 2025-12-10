# Compose + CameraX：相机集成与拍照

> **发布日期**: 2024-06-23  
> **阅读时间**: 约 26 分钟  
> **标签**: CameraX, 相机, 拍照, 预览

CameraX 是 Jetpack 提供的相机库，简化了相机开发的复杂性。本文将深入讲解如何在 Compose 中集成 CameraX 实现相机预览、拍照和录像功能。

## 一、基础配置

### 添加依赖

```kotlin
// build.gradle.kts
dependencies {
    val cameraxVersion = "1.3.1"
    implementation("androidx.camera:camera-core:$cameraxVersion")
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")
    implementation("androidx.camera:camera-extensions:$cameraxVersion")
}
```

### 权限声明

```xml
<manifest>
    <uses-feature android:name="android.hardware.camera" android:required="true" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
</manifest>
```

## 二、相机预览

### 基础预览

```kotlin
@Composable
fun CameraPreview(
    modifier: Modifier = Modifier
) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    
    val previewView = remember { PreviewView(context) }
    val cameraProviderFuture = remember { ProcessCameraProvider.getInstance(context) }
    
    DisposableEffect(lifecycleOwner) {
        val cameraProvider = cameraProviderFuture.get()
        
        val preview = Preview.Builder().build().also {
            it.setSurfaceProvider(previewView.surfaceProvider)
        }
        
        val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA
        
        try {
            cameraProvider.unbindAll()
            cameraProvider.bindToLifecycle(
                lifecycleOwner,
                cameraSelector,
                preview
            )
        } catch (e: Exception) {
            Log.e("CameraPreview", "Binding failed", e)
        }
        
        onDispose {
            cameraProvider.unbindAll()
        }
    }
    
    AndroidView(
        factory = { previewView },
        modifier = modifier
    )
}
```

### 封装相机状态

```kotlin
class CameraState(
    private val context: Context,
    private val lifecycleOwner: LifecycleOwner
) {
    private var cameraProvider: ProcessCameraProvider? = null
    private var imageCapture: ImageCapture? = null
    private var videoCapture: VideoCapture<Recorder>? = null
    private var camera: Camera? = null
    
    var isFlashEnabled by mutableStateOf(false)
        private set
    
    var lensFacing by mutableStateOf(CameraSelector.LENS_FACING_BACK)
        private set
    
    suspend fun initialize(previewView: PreviewView) {
        cameraProvider = ProcessCameraProvider.getInstance(context).await()
        bindCamera(previewView)
    }
    
    private fun bindCamera(previewView: PreviewView) {
        val cameraProvider = cameraProvider ?: return
        
        val preview = Preview.Builder().build().also {
            it.setSurfaceProvider(previewView.surfaceProvider)
        }
        
        imageCapture = ImageCapture.Builder()
            .setCaptureMode(ImageCapture.CAPTURE_MODE_MAXIMIZE_QUALITY)
            .build()
        
        val recorder = Recorder.Builder()
            .setQualitySelector(QualitySelector.from(Quality.HD))
            .build()
        videoCapture = VideoCapture.withOutput(recorder)
        
        val cameraSelector = CameraSelector.Builder()
            .requireLensFacing(lensFacing)
            .build()
        
        try {
            cameraProvider.unbindAll()
            camera = cameraProvider.bindToLifecycle(
                lifecycleOwner,
                cameraSelector,
                preview,
                imageCapture,
                videoCapture
            )
        } catch (e: Exception) {
            Log.e("CameraState", "Binding failed", e)
        }
    }
    
    fun toggleFlash() {
        isFlashEnabled = !isFlashEnabled
        camera?.cameraControl?.enableTorch(isFlashEnabled)
    }
    
    fun switchCamera(previewView: PreviewView) {
        lensFacing = if (lensFacing == CameraSelector.LENS_FACING_BACK) {
            CameraSelector.LENS_FACING_FRONT
        } else {
            CameraSelector.LENS_FACING_BACK
        }
        bindCamera(previewView)
    }
    
    suspend fun takePhoto(): Uri? {
        val imageCapture = imageCapture ?: return null
        
        val photoFile = File(
            context.cacheDir,
            "photo_${System.currentTimeMillis()}.jpg"
        )
        
        val outputOptions = ImageCapture.OutputFileOptions.Builder(photoFile).build()
        
        return suspendCancellableCoroutine { continuation ->
            imageCapture.takePicture(
                outputOptions,
                ContextCompat.getMainExecutor(context),
                object : ImageCapture.OnImageSavedCallback {
                    override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                        continuation.resume(output.savedUri ?: Uri.fromFile(photoFile))
                    }
                    
                    override fun onError(exception: ImageCaptureException) {
                        continuation.resumeWithException(exception)
                    }
                }
            )
        }
    }
    
    fun release() {
        cameraProvider?.unbindAll()
    }
}

@Composable
fun rememberCameraState(): CameraState {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    
    return remember {
        CameraState(context, lifecycleOwner)
    }
}
```

## 三、完整相机界面

```kotlin
@Composable
fun CameraScreen(
    onPhotoTaken: (Uri) -> Unit,
    onClose: () -> Unit
) {
    val context = LocalContext.current
    val cameraState = rememberCameraState()
    val previewView = remember { PreviewView(context) }
    val scope = rememberCoroutineScope()
    
    var isTakingPhoto by remember { mutableStateOf(false) }
    
    LaunchedEffect(Unit) {
        cameraState.initialize(previewView)
    }
    
    DisposableEffect(Unit) {
        onDispose { cameraState.release() }
    }
    
    Box(modifier = Modifier.fillMaxSize()) {
        // 相机预览
        AndroidView(
            factory = { previewView },
            modifier = Modifier.fillMaxSize()
        )
        
        // 顶部控制栏
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .statusBarsPadding()
                .padding(16.dp),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            IconButton(onClick = onClose) {
                Icon(
                    Icons.Default.Close,
                    contentDescription = "关闭",
                    tint = Color.White
                )
            }
            
            IconButton(onClick = { cameraState.toggleFlash() }) {
                Icon(
                    if (cameraState.isFlashEnabled) 
                        Icons.Default.FlashOn 
                    else 
                        Icons.Default.FlashOff,
                    contentDescription = "闪光灯",
                    tint = Color.White
                )
            }
        }
        
        // 底部控制栏
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .align(Alignment.BottomCenter)
                .navigationBarsPadding()
                .padding(32.dp),
            horizontalArrangement = Arrangement.SpaceEvenly,
            verticalAlignment = Alignment.CenterVertically
        ) {
            // 占位
            Spacer(modifier = Modifier.size(48.dp))
            
            // 拍照按钮
            Box(
                modifier = Modifier
                    .size(72.dp)
                    .clip(CircleShape)
                    .background(Color.White)
                    .clickable(enabled = !isTakingPhoto) {
                        scope.launch {
                            isTakingPhoto = true
                            try {
                                cameraState.takePhoto()?.let { uri ->
                                    onPhotoTaken(uri)
                                }
                            } finally {
                                isTakingPhoto = false
                            }
                        }
                    },
                contentAlignment = Alignment.Center
            ) {
                if (isTakingPhoto) {
                    CircularProgressIndicator(
                        modifier = Modifier.size(32.dp),
                        color = Color.Black
                    )
                } else {
                    Box(
                        modifier = Modifier
                            .size(60.dp)
                            .clip(CircleShape)
                            .border(3.dp, Color.Black, CircleShape)
                    )
                }
            }
            
            // 切换摄像头
            IconButton(
                onClick = { cameraState.switchCamera(previewView) }
            ) {
                Icon(
                    Icons.Default.Cameraswitch,
                    contentDescription = "切换摄像头",
                    tint = Color.White,
                    modifier = Modifier.size(32.dp)
                )
            }
        }
    }
}
```

## 四、图片分析

```kotlin
@Composable
fun ImageAnalysisExample() {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    val previewView = remember { PreviewView(context) }
    
    var detectedText by remember { mutableStateOf("") }
    
    LaunchedEffect(Unit) {
        val cameraProvider = ProcessCameraProvider.getInstance(context).await()
        
        val preview = Preview.Builder().build().also {
            it.setSurfaceProvider(previewView.surfaceProvider)
        }
        
        val imageAnalyzer = ImageAnalysis.Builder()
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build()
            .also { analysis ->
                analysis.setAnalyzer(Executors.newSingleThreadExecutor()) { imageProxy ->
                    // 在这里处理图像
                    // 例如：二维码识别、文字识别等
                    processImage(imageProxy) { result ->
                        detectedText = result
                    }
                    imageProxy.close()
                }
            }
        
        cameraProvider.bindToLifecycle(
            lifecycleOwner,
            CameraSelector.DEFAULT_BACK_CAMERA,
            preview,
            imageAnalyzer
        )
    }
    
    Box(modifier = Modifier.fillMaxSize()) {
        AndroidView(
            factory = { previewView },
            modifier = Modifier.fillMaxSize()
        )
        
        Text(
            text = detectedText,
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .padding(16.dp)
                .background(Color.Black.copy(alpha = 0.7f))
                .padding(8.dp),
            color = Color.White
        )
    }
}
```

## 五、二维码扫描

```kotlin
@Composable
fun QRCodeScanner(
    onQRCodeScanned: (String) -> Unit
) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    val previewView = remember { PreviewView(context) }
    
    LaunchedEffect(Unit) {
        val cameraProvider = ProcessCameraProvider.getInstance(context).await()
        
        val preview = Preview.Builder().build().also {
            it.setSurfaceProvider(previewView.surfaceProvider)
        }
        
        val barcodeScanner = BarcodeScanning.getClient(
            BarcodeScannerOptions.Builder()
                .setBarcodeFormats(Barcode.FORMAT_QR_CODE)
                .build()
        )
        
        val imageAnalyzer = ImageAnalysis.Builder()
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build()
            .also { analysis ->
                analysis.setAnalyzer(Executors.newSingleThreadExecutor()) { imageProxy ->
                    val mediaImage = imageProxy.image
                    if (mediaImage != null) {
                        val inputImage = InputImage.fromMediaImage(
                            mediaImage,
                            imageProxy.imageInfo.rotationDegrees
                        )
                        
                        barcodeScanner.process(inputImage)
                            .addOnSuccessListener { barcodes ->
                                barcodes.firstOrNull()?.rawValue?.let { value ->
                                    onQRCodeScanned(value)
                                }
                            }
                            .addOnCompleteListener {
                                imageProxy.close()
                            }
                    } else {
                        imageProxy.close()
                    }
                }
            }
        
        cameraProvider.bindToLifecycle(
            lifecycleOwner,
            CameraSelector.DEFAULT_BACK_CAMERA,
            preview,
            imageAnalyzer
        )
    }
    
    Box(modifier = Modifier.fillMaxSize()) {
        AndroidView(
            factory = { previewView },
            modifier = Modifier.fillMaxSize()
        )
        
        // 扫描框
        Canvas(modifier = Modifier.fillMaxSize()) {
            val scanAreaSize = 250.dp.toPx()
            val left = (size.width - scanAreaSize) / 2
            val top = (size.height - scanAreaSize) / 2
            
            // 半透明背景
            drawRect(
                color = Color.Black.copy(alpha = 0.5f),
                size = size
            )
            
            // 透明扫描区域
            drawRect(
                color = Color.Transparent,
                topLeft = Offset(left, top),
                size = Size(scanAreaSize, scanAreaSize),
                blendMode = BlendMode.Clear
            )
            
            // 扫描框边框
            drawRect(
                color = Color.White,
                topLeft = Offset(left, top),
                size = Size(scanAreaSize, scanAreaSize),
                style = Stroke(width = 2.dp.toPx())
            )
        }
    }
}
```

## 六、录像功能

```kotlin
@Composable
fun VideoRecordingExample() {
    val context = LocalContext.current
    val cameraState = rememberCameraState()
    
    var isRecording by remember { mutableStateOf(false) }
    var recordingTime by remember { mutableStateOf(0) }
    var recording: Recording? by remember { mutableStateOf(null) }
    
    // 录制计时器
    LaunchedEffect(isRecording) {
        if (isRecording) {
            recordingTime = 0
            while (isRecording) {
                delay(1000)
                recordingTime++
            }
        }
    }
    
    fun startRecording() {
        val videoFile = File(
            context.cacheDir,
            "video_${System.currentTimeMillis()}.mp4"
        )
        
        val outputOptions = FileOutputOptions.Builder(videoFile).build()
        
        recording = cameraState.videoCapture?.output
            ?.prepareRecording(context, outputOptions)
            ?.withAudioEnabled()
            ?.start(ContextCompat.getMainExecutor(context)) { event ->
                when (event) {
                    is VideoRecordEvent.Start -> {
                        isRecording = true
                    }
                    is VideoRecordEvent.Finalize -> {
                        isRecording = false
                        if (event.hasError()) {
                            Log.e("Video", "Recording error: ${event.error}")
                        } else {
                            // 录制成功
                            val uri = event.outputResults.outputUri
                        }
                    }
                }
            }
    }
    
    fun stopRecording() {
        recording?.stop()
        recording = null
    }
    
    // UI
    Box(modifier = Modifier.fillMaxSize()) {
        // 相机预览...
        
        // 录制时间
        if (isRecording) {
            Text(
                text = formatTime(recordingTime),
                modifier = Modifier
                    .align(Alignment.TopCenter)
                    .padding(top = 64.dp),
                color = Color.Red,
                fontSize = 24.sp
            )
        }
        
        // 录制按钮
        IconButton(
            onClick = {
                if (isRecording) stopRecording() else startRecording()
            },
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .padding(32.dp)
        ) {
            Icon(
                imageVector = if (isRecording) 
                    Icons.Default.Stop 
                else 
                    Icons.Default.FiberManualRecord,
                contentDescription = if (isRecording) "停止" else "录制",
                tint = Color.Red,
                modifier = Modifier.size(64.dp)
            )
        }
    }
}
```

## 七、最佳实践

- ✅ 使用 CameraX 而非 Camera2（更简单）
- ✅ 正确处理生命周期（绑定到 LifecycleOwner）
- ✅ 请求权限后再初始化相机
- ✅ 使用 DisposableEffect 释放资源
- ✅ 处理相机不可用的情况
- ✅ 使用协程处理拍照操作
- ✅ 支持前后摄像头切换
- ✅ 添加闪光灯控制

## 总结

CameraX 与 Compose 集成的核心要点：

- **PreviewView + AndroidView**：显示相机预览
- **ProcessCameraProvider**：管理相机生命周期
- **ImageCapture**：拍照功能
- **VideoCapture**：录像功能
- **ImageAnalysis**：实时图像分析
- **CameraSelector**：选择前/后摄像头

CameraX 大大简化了相机开发的复杂性。

---

*© 2024 Fidroid. [返回首页](../index.html)*

