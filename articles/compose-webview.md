# Compose WebView：网页集成与 JS 交互

> **发布日期**: 2024-07-01  
> **阅读时间**: 约 22 分钟  
> **标签**: WebView, JavaScript, 混合开发

WebView 允许在应用中展示网页内容，实现原生与 Web 的混合开发。本文将深入讲解如何在 Compose 中集成 WebView 并实现与 JavaScript 的双向通信。

## 一、基础 WebView

```kotlin
@Composable
fun BasicWebView(url: String) {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                settings.javaScriptEnabled = true
                webViewClient = WebViewClient()
                loadUrl(url)
            }
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

## 二、封装 WebView 状态

```kotlin
class WebViewState(
    initialUrl: String
) {
    var webView: WebView? by mutableStateOf(null)
        internal set
    
    var url by mutableStateOf(initialUrl)
        internal set
    
    var title by mutableStateOf("")
        internal set
    
    var isLoading by mutableStateOf(true)
        internal set
    
    var progress by mutableIntStateOf(0)
        internal set
    
    var canGoBack by mutableStateOf(false)
        internal set
    
    var canGoForward by mutableStateOf(false)
        internal set
    
    fun loadUrl(url: String) {
        this.url = url
        webView?.loadUrl(url)
    }
    
    fun goBack() {
        webView?.goBack()
    }
    
    fun goForward() {
        webView?.goForward()
    }
    
    fun reload() {
        webView?.reload()
    }
    
    fun evaluateJavascript(script: String, callback: ((String) -> Unit)? = null) {
        webView?.evaluateJavascript(script) { result ->
            callback?.invoke(result)
        }
    }
}

@Composable
fun rememberWebViewState(url: String): WebViewState {
    return remember { WebViewState(url) }
}
```

## 三、完整 WebView 组件

```kotlin
@SuppressLint("SetJavaScriptEnabled")
@Composable
fun ComposeWebView(
    state: WebViewState,
    modifier: Modifier = Modifier,
    onPageStarted: ((String) -> Unit)? = null,
    onPageFinished: ((String) -> Unit)? = null,
    onError: ((Int, String) -> Unit)? = null
) {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                state.webView = this
                
                settings.apply {
                    javaScriptEnabled = true
                    domStorageEnabled = true
                    databaseEnabled = true
                    cacheMode = WebSettings.LOAD_DEFAULT
                    setSupportZoom(true)
                    builtInZoomControls = true
                    displayZoomControls = false
                    loadWithOverviewMode = true
                    useWideViewPort = true
                }
                
                webViewClient = object : WebViewClient() {
                    override fun onPageStarted(view: WebView?, url: String?, favicon: Bitmap?) {
                        state.isLoading = true
                        state.url = url ?: ""
                        onPageStarted?.invoke(url ?: "")
                    }
                    
                    override fun onPageFinished(view: WebView?, url: String?) {
                        state.isLoading = false
                        state.canGoBack = view?.canGoBack() ?: false
                        state.canGoForward = view?.canGoForward() ?: false
                        onPageFinished?.invoke(url ?: "")
                    }
                    
                    override fun onReceivedError(
                        view: WebView?,
                        request: WebResourceRequest?,
                        error: WebResourceError?
                    ) {
                        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                            onError?.invoke(
                                error?.errorCode ?: -1,
                                error?.description?.toString() ?: "Unknown error"
                            )
                        }
                    }
                }
                
                webChromeClient = object : WebChromeClient() {
                    override fun onProgressChanged(view: WebView?, newProgress: Int) {
                        state.progress = newProgress
                    }
                    
                    override fun onReceivedTitle(view: WebView?, title: String?) {
                        state.title = title ?: ""
                    }
                }
                
                loadUrl(state.url)
            }
        },
        modifier = modifier,
        update = { webView ->
            if (webView.url != state.url) {
                webView.loadUrl(state.url)
            }
        }
    )
}
```

## 四、带导航栏的 WebView

```kotlin
@Composable
fun WebViewScreen(
    initialUrl: String,
    onClose: () -> Unit
) {
    val state = rememberWebViewState(initialUrl)
    
    // 处理返回键
    BackHandler(enabled = state.canGoBack) {
        state.goBack()
    }
    
    Scaffold(
        topBar = {
            TopAppBar(
                title = {
                    Column {
                        Text(
                            state.title.ifEmpty { "加载中..." },
                            maxLines = 1,
                            overflow = TextOverflow.Ellipsis
                        )
                        Text(
                            state.url,
                            style = MaterialTheme.typography.bodySmall,
                            maxLines = 1,
                            overflow = TextOverflow.Ellipsis
                        )
                    }
                },
                navigationIcon = {
                    IconButton(onClick = onClose) {
                        Icon(Icons.Default.Close, "关闭")
                    }
                },
                actions = {
                    IconButton(
                        onClick = { state.goBack() },
                        enabled = state.canGoBack
                    ) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, "后退")
                    }
                    IconButton(
                        onClick = { state.goForward() },
                        enabled = state.canGoForward
                    ) {
                        Icon(Icons.AutoMirrored.Filled.ArrowForward, "前进")
                    }
                    IconButton(onClick = { state.reload() }) {
                        Icon(Icons.Default.Refresh, "刷新")
                    }
                }
            )
        }
    ) { padding ->
        Box(modifier = Modifier.padding(padding)) {
            ComposeWebView(
                state = state,
                modifier = Modifier.fillMaxSize()
            )
            
            // 加载进度条
            if (state.isLoading) {
                LinearProgressIndicator(
                    progress = { state.progress / 100f },
                    modifier = Modifier.fillMaxWidth()
                )
            }
        }
    }
}
```

## 五、JavaScript 交互

### Compose 调用 JavaScript

```kotlin
@Composable
fun CallJavaScriptExample() {
    val state = rememberWebViewState("file:///android_asset/index.html")
    var result by remember { mutableStateOf("") }
    
    Column {
        ComposeWebView(
            state = state,
            modifier = Modifier
                .fillMaxWidth()
                .weight(1f)
        )
        
        Row(modifier = Modifier.padding(16.dp)) {
            Button(
                onClick = {
                    state.evaluateJavascript("document.title") { value ->
                        result = value
                    }
                }
            ) {
                Text("获取标题")
            }
            
            Spacer(modifier = Modifier.width(8.dp))
            
            Button(
                onClick = {
                    state.evaluateJavascript("showAlert('Hello from Compose!')")
                }
            ) {
                Text("调用 JS 函数")
            }
        }
        
        Text("结果: $result", modifier = Modifier.padding(16.dp))
    }
}
```

### JavaScript 调用 Compose

```kotlin
class JsInterface(
    private val onMessage: (String) -> Unit
) {
    @JavascriptInterface
    fun postMessage(message: String) {
        onMessage(message)
    }
    
    @JavascriptInterface
    fun showToast(message: String) {
        // 在主线程显示 Toast
    }
}

@Composable
fun JsBridgeExample() {
    val context = LocalContext.current
    var message by remember { mutableStateOf("") }
    
    val jsInterface = remember {
        JsInterface { msg ->
            message = msg
        }
    }
    
    AndroidView(
        factory = { ctx ->
            WebView(ctx).apply {
                settings.javaScriptEnabled = true
                addJavascriptInterface(jsInterface, "Android")
                loadUrl("file:///android_asset/bridge.html")
            }
        }
    )
    
    // 显示从 JS 收到的消息
    if (message.isNotEmpty()) {
        Text("来自 JS: $message")
    }
}

// HTML 中调用
// <script>
//   Android.postMessage("Hello from JavaScript!");
//   Android.showToast("Toast message");
// </script>
```

## 六、文件选择

```kotlin
@Composable
fun WebViewWithFileChooser() {
    var filePathCallback by remember { 
        mutableStateOf<ValueCallback<Array<Uri>>?>(null) 
    }
    
    val fileChooserLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.GetMultipleContents()
    ) { uris ->
        filePathCallback?.onReceiveValue(uris.toTypedArray())
        filePathCallback = null
    }
    
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                settings.javaScriptEnabled = true
                
                webChromeClient = object : WebChromeClient() {
                    override fun onShowFileChooser(
                        webView: WebView?,
                        callback: ValueCallback<Array<Uri>>?,
                        params: FileChooserParams?
                    ): Boolean {
                        filePathCallback = callback
                        fileChooserLauncher.launch("*/*")
                        return true
                    }
                }
                
                loadUrl("https://example.com/upload")
            }
        }
    )
}
```

## 七、Cookie 管理

```kotlin
object WebViewCookieManager {
    fun setCookie(url: String, cookie: String) {
        CookieManager.getInstance().apply {
            setAcceptCookie(true)
            setCookie(url, cookie)
            flush()
        }
    }
    
    fun getCookie(url: String): String? {
        return CookieManager.getInstance().getCookie(url)
    }
    
    fun clearCookies() {
        CookieManager.getInstance().removeAllCookies(null)
    }
}

// 使用
LaunchedEffect(Unit) {
    WebViewCookieManager.setCookie(
        "https://example.com",
        "token=abc123; path=/; secure"
    )
}
```

## 八、本地 HTML

```kotlin
@Composable
fun LocalHtmlWebView() {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                settings.javaScriptEnabled = true
                
                // 从 assets 加载
                loadUrl("file:///android_asset/local.html")
                
                // 或者直接加载 HTML 字符串
                // loadDataWithBaseURL(
                //     null,
                //     "<html><body><h1>Hello</h1></body></html>",
                //     "text/html",
                //     "UTF-8",
                //     null
                // )
            }
        }
    )
}
```

## 九、最佳实践

- ✅ 启用 JavaScript 时注意安全性
- ✅ 使用 @JavascriptInterface 注解
- ✅ 在主线程更新 UI
- ✅ 处理返回键导航
- ✅ 显示加载进度
- ✅ 处理错误页面
- ✅ 管理 Cookie 和缓存
- ✅ 避免内存泄漏（及时释放）

## 总结

Compose WebView 集成的核心要点：

- **AndroidView**：在 Compose 中嵌入 WebView
- **WebViewState**：封装 WebView 状态
- **JavaScript 交互**：双向通信
- **JsInterface**：JS 调用原生代码
- **文件选择**：处理文件上传
- **Cookie 管理**：同步登录状态

WebView 是实现混合开发的重要工具。

---

*© 2024 Fidroid. [返回首页](../index.html)*

