# Compose 图片加载最佳实践：Coil 集成与缓存策略

> **发布日期**: 2024-05-24  
> **阅读时间**: 约 22 分钟  
> **标签**: Coil, AsyncImage, 图片加载, 缓存

图片加载是移动应用中最常见的需求之一。Coil（Coroutine Image Loader）是专为 Kotlin 和 Compose 设计的现代图片加载库，提供了简洁的 API 和出色的性能。本文将深入讲解 Coil 在 Compose 项目中的最佳实践。

## 一、为什么选择 Coil？

| 特性 | Coil | Glide | Picasso |
|-----|------|-------|---------|
| Kotlin 优先 | ✅ | ❌ | ❌ |
| Compose 支持 | 原生 | 需要扩展 | 需要扩展 |
| 协程支持 | ✅ | ❌ | ❌ |
| 包大小 | ~1500 方法 | ~8000 方法 | ~800 方法 |
| 内存优化 | 优秀 | 优秀 | 一般 |

## 二、基础配置

### 添加依赖

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.coil-kt:coil-compose:2.5.0")
    
    // 可选：GIF 支持
    implementation("io.coil-kt:coil-gif:2.5.0")
    
    // 可选：SVG 支持
    implementation("io.coil-kt:coil-svg:2.5.0")
    
    // 可选：视频帧
    implementation("io.coil-kt:coil-video:2.5.0")
}
```

### 配置 ImageLoader

```kotlin
// Application.kt
class MyApplication : Application(), ImageLoaderFactory {
    
    override fun newImageLoader(): ImageLoader {
        return ImageLoader.Builder(this)
            .crossfade(true)
            .crossfade(300)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .diskCache {
                DiskCache.Builder()
                    .directory(cacheDir.resolve("image_cache"))
                    .maxSizePercent(0.02)
                    .build()
            }
            .memoryCache {
                MemoryCache.Builder(this)
                    .maxSizePercent(0.25)
                    .build()
            }
            .logger(DebugLogger())  // 调试时使用
            .respectCacheHeaders(false)
            .build()
    }
}
```

## 三、AsyncImage 基础用法

### 简单加载

```kotlin
@Composable
fun SimpleImage(url: String) {
    AsyncImage(
        model = url,
        contentDescription = "图片描述",
        modifier = Modifier.size(200.dp)
    )
}
```

### 完整配置

```kotlin
@Composable
fun FullFeaturedImage(url: String) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .crossfade(true)
            .build(),
        contentDescription = "图片",
        modifier = Modifier
            .fillMaxWidth()
            .aspectRatio(16f / 9f)
            .clip(RoundedCornerShape(12.dp)),
        contentScale = ContentScale.Crop,
        placeholder = painterResource(R.drawable.placeholder),
        error = painterResource(R.drawable.error),
        onLoading = { /* 加载中 */ },
        onSuccess = { /* 加载成功 */ },
        onError = { /* 加载失败 */ }
    )
}
```

### 使用 SubcomposeAsyncImage

更精细的加载状态控制：

```kotlin
@Composable
fun CustomLoadingImage(url: String) {
    SubcomposeAsyncImage(
        model = url,
        contentDescription = "图片",
        modifier = Modifier.size(200.dp)
    ) {
        when (painter.state) {
            is AsyncImagePainter.State.Loading -> {
                Box(
                    modifier = Modifier.fillMaxSize(),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            }
            is AsyncImagePainter.State.Error -> {
                Icon(
                    imageVector = Icons.Default.BrokenImage,
                    contentDescription = "加载失败",
                    modifier = Modifier.size(48.dp),
                    tint = Color.Gray
                )
            }
            else -> {
                SubcomposeAsyncImageContent()
            }
        }
    }
}
```

## 四、图片变换

### 内置变换

```kotlin
@Composable
fun TransformedImage(url: String) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .transformations(
                CircleCropTransformation(),
                // 或者
                RoundedCornersTransformation(16f),
                // 或者
                BlurTransformation(LocalContext.current, radius = 10f, sampling = 2f)
            )
            .build(),
        contentDescription = null,
        modifier = Modifier.size(100.dp)
    )
}
```

### 自定义变换

```kotlin
class GrayscaleTransformation : Transformation {
    override val cacheKey: String = "grayscale"
    
    override suspend fun transform(input: Bitmap, size: Size): Bitmap {
        val output = Bitmap.createBitmap(input.width, input.height, input.config)
        val canvas = Canvas(output)
        val paint = Paint().apply {
            colorFilter = ColorMatrixColorFilter(ColorMatrix().apply {
                setSaturation(0f)
            })
        }
        canvas.drawBitmap(input, 0f, 0f, paint)
        return output
    }
}

// 使用
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(url)
        .transformations(GrayscaleTransformation())
        .build(),
    contentDescription = null
)
```

## 五、缓存策略

### 缓存级别

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(url)
        // 内存缓存
        .memoryCachePolicy(CachePolicy.ENABLED)
        // 磁盘缓存
        .diskCachePolicy(CachePolicy.ENABLED)
        // 网络缓存（HTTP 缓存头）
        .networkCachePolicy(CachePolicy.ENABLED)
        .build(),
    contentDescription = null
)
```

### 自定义缓存键

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(url)
        .memoryCacheKey("custom_key_$url")
        .diskCacheKey("custom_key_$url")
        .build(),
    contentDescription = null
)
```

### 预加载

```kotlin
@Composable
fun PreloadImages(urls: List<String>) {
    val context = LocalContext.current
    val imageLoader = LocalImageLoader.current
    
    LaunchedEffect(urls) {
        urls.forEach { url ->
            val request = ImageRequest.Builder(context)
                .data(url)
                .memoryCachePolicy(CachePolicy.ENABLED)
                .build()
            imageLoader.enqueue(request)
        }
    }
}
```

## 六、列表中的图片加载

### 基本列表

```kotlin
@Composable
fun ImageList(images: List<ImageItem>) {
    LazyColumn {
        items(images, key = { it.id }) { item ->
            AsyncImage(
                model = item.url,
                contentDescription = item.title,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(200.dp),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

### 优化大量图片

```kotlin
@Composable
fun OptimizedImageList(images: List<ImageItem>) {
    val context = LocalContext.current
    
    LazyColumn {
        items(images, key = { it.id }) { item ->
            AsyncImage(
                model = ImageRequest.Builder(context)
                    .data(item.url)
                    // 降低采样率以节省内存
                    .size(Size.ORIGINAL)
                    .scale(Scale.FILL)
                    // 禁用不必要的动画
                    .crossfade(false)
                    .build(),
                contentDescription = item.title,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(200.dp),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

### 网格布局

```kotlin
@Composable
fun ImageGrid(images: List<String>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(3),
        contentPadding = PaddingValues(4.dp),
        horizontalArrangement = Arrangement.spacedBy(4.dp),
        verticalArrangement = Arrangement.spacedBy(4.dp)
    ) {
        items(images) { url ->
            AsyncImage(
                model = url,
                contentDescription = null,
                modifier = Modifier
                    .aspectRatio(1f)
                    .clip(RoundedCornerShape(4.dp)),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

## 七、特殊图片类型

### GIF 动画

```kotlin
// 配置 ImageLoader 支持 GIF
val imageLoader = ImageLoader.Builder(context)
    .components {
        if (SDK_INT >= 28) {
            add(ImageDecoderDecoder.Factory())
        } else {
            add(GifDecoder.Factory())
        }
    }
    .build()

@Composable
fun GifImage(url: String) {
    AsyncImage(
        model = url,
        contentDescription = "GIF",
        modifier = Modifier.size(200.dp)
    )
}
```

### SVG

```kotlin
// 配置 ImageLoader 支持 SVG
val imageLoader = ImageLoader.Builder(context)
    .components {
        add(SvgDecoder.Factory())
    }
    .build()

@Composable
fun SvgImage(url: String) {
    AsyncImage(
        model = url,
        contentDescription = "SVG",
        modifier = Modifier.size(100.dp)
    )
}
```

### 视频缩略图

```kotlin
// 配置 ImageLoader 支持视频帧
val imageLoader = ImageLoader.Builder(context)
    .components {
        add(VideoFrameDecoder.Factory())
    }
    .build()

@Composable
fun VideoThumbnail(videoUrl: String) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(videoUrl)
            .videoFrameMillis(1000)  // 第 1 秒的帧
            .build(),
        contentDescription = "视频缩略图",
        modifier = Modifier.size(200.dp)
    )
}
```

## 八、错误处理与重试

### 优雅的错误处理

```kotlin
@Composable
fun ImageWithRetry(url: String) {
    var retryHash by remember { mutableIntStateOf(0) }
    
    SubcomposeAsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .setParameter("retry_hash", retryHash)
            .build(),
        contentDescription = null,
        modifier = Modifier.size(200.dp)
    ) {
        when (val state = painter.state) {
            is AsyncImagePainter.State.Loading -> {
                CircularProgressIndicator()
            }
            is AsyncImagePainter.State.Error -> {
                Column(
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    Icon(Icons.Default.Error, "错误")
                    TextButton(onClick = { retryHash++ }) {
                        Text("重试")
                    }
                }
            }
            else -> {
                SubcomposeAsyncImageContent()
            }
        }
    }
}
```

### 渐进式加载

```kotlin
@Composable
fun ProgressiveImage(
    thumbnailUrl: String,
    fullUrl: String
) {
    var showFullImage by remember { mutableStateOf(false) }
    
    Box(modifier = Modifier.size(300.dp)) {
        // 缩略图
        AsyncImage(
            model = thumbnailUrl,
            contentDescription = null,
            modifier = Modifier.fillMaxSize(),
            contentScale = ContentScale.Crop,
            onSuccess = { showFullImage = true }
        )
        
        // 高清图
        if (showFullImage) {
            AsyncImage(
                model = fullUrl,
                contentDescription = null,
                modifier = Modifier.fillMaxSize(),
                contentScale = ContentScale.Crop
            )
        }
    }
}
```

## 九、性能优化

### 1. 指定尺寸

```kotlin
// ✅ 好：指定具体尺寸
AsyncImage(
    model = ImageRequest.Builder(context)
        .data(url)
        .size(200, 200)
        .build(),
    modifier = Modifier.size(200.dp)
)

// ❌ 避免：不指定尺寸
AsyncImage(
    model = url,
    modifier = Modifier.wrapContentSize()
)
```

### 2. 使用占位符

```kotlin
// ✅ 减少布局跳动
AsyncImage(
    model = url,
    placeholder = ColorPainter(Color.LightGray),
    modifier = Modifier
        .fillMaxWidth()
        .aspectRatio(16f / 9f)
)
```

### 3. 内存管理

```kotlin
// 在列表中限制内存使用
val imageLoader = ImageLoader.Builder(context)
    .memoryCache {
        MemoryCache.Builder(context)
            .maxSizePercent(0.20)  // 使用 20% 可用内存
            .build()
    }
    .build()
```

## 十、测试

### 模拟 ImageLoader

```kotlin
@Test
fun imageLoadsSuccessfully() {
    val fakeImageLoader = FakeImageLoader(context)
    
    composeTestRule.setContent {
        CompositionLocalProvider(LocalImageLoader provides fakeImageLoader) {
            MyImageComponent(url = "https://example.com/image.jpg")
        }
    }
    
    composeTestRule.onNodeWithContentDescription("图片").assertExists()
}

class FakeImageLoader(context: Context) : ImageLoader {
    override fun enqueue(request: ImageRequest): Disposable {
        // 返回模拟结果
    }
    
    override suspend fun execute(request: ImageRequest): ImageResult {
        // 返回模拟结果
    }
}
```

## 十一、最佳实践清单

- ✅ 在 Application 中配置全局 ImageLoader
- ✅ 为列表项指定 key 以优化重组
- ✅ 使用占位符避免布局跳动
- ✅ 指定图片尺寸避免不必要的内存消耗
- ✅ 根据需求配置缓存策略
- ✅ 使用 crossfade 提升用户体验
- ✅ 处理加载错误并提供重试机制
- ✅ 大量图片时考虑分页加载

## 总结

Coil 在 Compose 中图片加载的核心要点：

- **AsyncImage**：声明式的图片加载组件
- **SubcomposeAsyncImage**：自定义加载状态
- **变换**：圆形裁剪、圆角、模糊等
- **缓存**：内存 + 磁盘三级缓存
- **预加载**：提前加载即将显示的图片
- **特殊格式**：GIF、SVG、视频帧支持
- **性能**：指定尺寸、占位符、内存管理

Coil 是 Compose 项目中图片加载的最佳选择，简洁高效，与 Compose 完美集成。

---

*© 2024 Fidroid. [返回首页](../index.html)*

