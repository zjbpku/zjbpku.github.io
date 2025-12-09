# Compose + Room：数据库集成与 Flow 观察

> **发布日期**: 2024-06-05  
> **阅读时间**: 约 24 分钟  
> **标签**: Room, Database, Flow, DAO

Room 是 Android 官方的 SQLite 抽象层，提供编译时验证和流畅的 API。结合 Kotlin Flow，可以在 Compose 中实现响应式的数据库观察。本文将深入讲解 Room 在 Compose 项目中的最佳实践。

## 一、基础配置

### 添加依赖

```kotlin
// build.gradle.kts
plugins {
    id("com.google.devtools.ksp")
}

dependencies {
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")
    
    // 可选：Paging 集成
    implementation("androidx.room:room-paging:2.6.1")
}
```

## 二、定义实体和 DAO

### Entity

```kotlin
@Entity(tableName = "articles")
data class ArticleEntity(
    @PrimaryKey
    val id: String,
    val title: String,
    val content: String,
    val author: String,
    @ColumnInfo(name = "published_at")
    val publishedAt: Long,
    @ColumnInfo(name = "is_bookmarked")
    val isBookmarked: Boolean = false,
    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis()
)
```

### DAO with Flow

```kotlin
@Dao
interface ArticleDao {
    
    // 返回 Flow，自动观察变化
    @Query("SELECT * FROM articles ORDER BY published_at DESC")
    fun getAllArticles(): Flow<List<ArticleEntity>>
    
    // 带参数的查询
    @Query("SELECT * FROM articles WHERE id = :id")
    fun getArticleById(id: String): Flow<ArticleEntity?>
    
    // 搜索
    @Query("SELECT * FROM articles WHERE title LIKE '%' || :query || '%'")
    fun searchArticles(query: String): Flow<List<ArticleEntity>>
    
    // 书签文章
    @Query("SELECT * FROM articles WHERE is_bookmarked = 1")
    fun getBookmarkedArticles(): Flow<List<ArticleEntity>>
    
    // 一次性查询（不返回 Flow）
    @Query("SELECT * FROM articles WHERE id = :id")
    suspend fun getArticleByIdOnce(id: String): ArticleEntity?
    
    // 插入
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(article: ArticleEntity)
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(articles: List<ArticleEntity>)
    
    // 更新
    @Update
    suspend fun update(article: ArticleEntity)
    
    // 更新书签状态
    @Query("UPDATE articles SET is_bookmarked = :isBookmarked WHERE id = :id")
    suspend fun updateBookmark(id: String, isBookmarked: Boolean)
    
    // 删除
    @Delete
    suspend fun delete(article: ArticleEntity)
    
    @Query("DELETE FROM articles")
    suspend fun deleteAll()
    
    // 计数
    @Query("SELECT COUNT(*) FROM articles")
    fun getArticleCount(): Flow<Int>
}
```

### Database

```kotlin
@Database(
    entities = [ArticleEntity::class],
    version = 1,
    exportSchema = true
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun articleDao(): ArticleDao
}

// 类型转换器
class Converters {
    @TypeConverter
    fun fromTimestamp(value: Long?): Date? {
        return value?.let { Date(it) }
    }
    
    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? {
        return date?.time
    }
}
```

## 三、Repository 层

```kotlin
class ArticleRepository(
    private val articleDao: ArticleDao,
    private val apiService: ApiService
) {
    // 观察所有文章
    val allArticles: Flow<List<ArticleEntity>> = articleDao.getAllArticles()
    
    // 观察书签文章
    val bookmarkedArticles: Flow<List<ArticleEntity>> = articleDao.getBookmarkedArticles()
    
    // 获取单篇文章
    fun getArticle(id: String): Flow<ArticleEntity?> = articleDao.getArticleById(id)
    
    // 搜索文章
    fun searchArticles(query: String): Flow<List<ArticleEntity>> {
        return articleDao.searchArticles(query)
    }
    
    // 刷新数据（从网络获取并存入数据库）
    suspend fun refreshArticles() {
        try {
            val response = apiService.getArticles()
            val entities = response.articles.map { it.toEntity() }
            articleDao.insertAll(entities)
        } catch (e: Exception) {
            // 错误处理
            throw e
        }
    }
    
    // 切换书签
    suspend fun toggleBookmark(id: String) {
        val article = articleDao.getArticleByIdOnce(id)
        article?.let {
            articleDao.updateBookmark(id, !it.isBookmarked)
        }
    }
    
    // 删除文章
    suspend fun deleteArticle(article: ArticleEntity) {
        articleDao.delete(article)
    }
}
```

## 四、ViewModel 集成

```kotlin
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val repository: ArticleRepository
) : ViewModel() {
    
    // 观察文章列表
    val articles: StateFlow<List<ArticleEntity>> = repository.allArticles
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
    
    // 搜索状态
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    // 搜索结果
    val searchResults: StateFlow<List<ArticleEntity>> = _searchQuery
        .debounce(300)
        .flatMapLatest { query ->
            if (query.isBlank()) {
                repository.allArticles
            } else {
                repository.searchArticles(query)
            }
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
    
    // 刷新状态
    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing: StateFlow<Boolean> = _isRefreshing.asStateFlow()
    
    fun search(query: String) {
        _searchQuery.value = query
    }
    
    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            try {
                repository.refreshArticles()
            } finally {
                _isRefreshing.value = false
            }
        }
    }
    
    fun toggleBookmark(id: String) {
        viewModelScope.launch {
            repository.toggleBookmark(id)
        }
    }
}
```

## 五、在 Compose 中使用

### 基础列表

```kotlin
@Composable
fun ArticleListScreen(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles by viewModel.articles.collectAsState()
    val isRefreshing by viewModel.isRefreshing.collectAsState()
    
    SwipeRefresh(
        state = rememberSwipeRefreshState(isRefreshing),
        onRefresh = { viewModel.refresh() }
    ) {
        LazyColumn(
            modifier = Modifier.fillMaxSize(),
            contentPadding = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(
                items = articles,
                key = { it.id }
            ) { article ->
                ArticleCard(
                    article = article,
                    onBookmarkClick = { viewModel.toggleBookmark(article.id) }
                )
            }
        }
    }
}

@Composable
fun ArticleCard(
    article: ArticleEntity,
    onBookmarkClick: () -> Unit
) {
    Card(
        modifier = Modifier.fillMaxWidth()
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = article.title,
                style = MaterialTheme.typography.titleMedium
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                text = article.content.take(100) + "...",
                style = MaterialTheme.typography.bodyMedium
            )
            Spacer(modifier = Modifier.height(8.dp))
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(
                    text = article.author,
                    style = MaterialTheme.typography.labelMedium
                )
                IconButton(onClick = onBookmarkClick) {
                    Icon(
                        imageVector = if (article.isBookmarked) 
                            Icons.Filled.Bookmark 
                        else 
                            Icons.Outlined.BookmarkBorder,
                        contentDescription = "书签"
                    )
                }
            }
        }
    }
}
```

### 搜索功能

```kotlin
@Composable
fun SearchableArticleList(viewModel: ArticleViewModel = hiltViewModel()) {
    val searchQuery by viewModel.searchQuery.collectAsState()
    val searchResults by viewModel.searchResults.collectAsState()
    
    Column(modifier = Modifier.fillMaxSize()) {
        OutlinedTextField(
            value = searchQuery,
            onValueChange = { viewModel.search(it) },
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            placeholder = { Text("搜索文章...") },
            leadingIcon = { Icon(Icons.Default.Search, "搜索") },
            trailingIcon = {
                if (searchQuery.isNotEmpty()) {
                    IconButton(onClick = { viewModel.search("") }) {
                        Icon(Icons.Default.Clear, "清除")
                    }
                }
            },
            singleLine = true
        )
        
        LazyColumn {
            items(searchResults, key = { it.id }) { article ->
                ArticleCard(article = article, onBookmarkClick = {})
            }
        }
    }
}
```

### 详情页面

```kotlin
@Composable
fun ArticleDetailScreen(
    articleId: String,
    viewModel: ArticleDetailViewModel = hiltViewModel()
) {
    val article by viewModel.getArticle(articleId).collectAsState(initial = null)
    
    article?.let { art ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .verticalScroll(rememberScrollState())
                .padding(16.dp)
        ) {
            Text(
                text = art.title,
                style = MaterialTheme.typography.headlineMedium
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                text = "作者: ${art.author}",
                style = MaterialTheme.typography.labelLarge
            )
            Spacer(modifier = Modifier.height(16.dp))
            Text(
                text = art.content,
                style = MaterialTheme.typography.bodyLarge
            )
        }
    } ?: run {
        Box(
            modifier = Modifier.fillMaxSize(),
            contentAlignment = Alignment.Center
        ) {
            CircularProgressIndicator()
        }
    }
}
```

## 六、数据库迁移

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL(
            "ALTER TABLE articles ADD COLUMN category TEXT NOT NULL DEFAULT ''"
        )
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // 创建新表
        database.execSQL("""
            CREATE TABLE articles_new (
                id TEXT NOT NULL PRIMARY KEY,
                title TEXT NOT NULL,
                content TEXT NOT NULL,
                author TEXT NOT NULL,
                category TEXT NOT NULL,
                published_at INTEGER NOT NULL
            )
        """)
        
        // 复制数据
        database.execSQL("""
            INSERT INTO articles_new (id, title, content, author, category, published_at)
            SELECT id, title, content, author, category, published_at FROM articles
        """)
        
        // 删除旧表
        database.execSQL("DROP TABLE articles")
        
        // 重命名新表
        database.execSQL("ALTER TABLE articles_new RENAME TO articles")
    }
}

// 构建数据库
val database = Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
    .build()
```

## 七、Hilt 集成

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app.db"
        )
            .addMigrations(MIGRATION_1_2)
            .build()
    }
    
    @Provides
    fun provideArticleDao(database: AppDatabase): ArticleDao {
        return database.articleDao()
    }
}
```

## 八、测试

### 内存数据库测试

```kotlin
@RunWith(AndroidJUnit4::class)
class ArticleDaoTest {
    
    private lateinit var database: AppDatabase
    private lateinit var articleDao: ArticleDao
    
    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).allowMainThreadQueries().build()
        
        articleDao = database.articleDao()
    }
    
    @After
    fun tearDown() {
        database.close()
    }
    
    @Test
    fun insertAndRead() = runTest {
        val article = ArticleEntity(
            id = "1",
            title = "Test",
            content = "Content",
            author = "Author",
            publishedAt = System.currentTimeMillis()
        )
        
        articleDao.insert(article)
        
        val result = articleDao.getAllArticles().first()
        assertEquals(1, result.size)
        assertEquals("Test", result[0].title)
    }
    
    @Test
    fun updateBookmark() = runTest {
        val article = ArticleEntity(
            id = "1",
            title = "Test",
            content = "Content",
            author = "Author",
            publishedAt = System.currentTimeMillis(),
            isBookmarked = false
        )
        
        articleDao.insert(article)
        articleDao.updateBookmark("1", true)
        
        val result = articleDao.getArticleByIdOnce("1")
        assertTrue(result?.isBookmarked == true)
    }
}
```

## 九、最佳实践

- ✅ DAO 方法返回 Flow 实现响应式观察
- ✅ 使用 `stateIn` 转换为 StateFlow
- ✅ Repository 层封装数据库和网络操作
- ✅ 使用 Hilt 注入数据库依赖
- ✅ 实现数据库迁移而非破坏性重建
- ✅ 使用内存数据库进行单元测试
- ✅ 使用 `OnConflictStrategy.REPLACE` 处理冲突
- ✅ 搜索时使用 `debounce` 减少查询

## 总结

Room 与 Compose 集成的核心要点：

- **Flow 返回值**：DAO 方法返回 Flow 自动观察变化
- **stateIn**：转换为 StateFlow 在 Compose 中使用
- **Repository 模式**：封装数据访问逻辑
- **Migration**：版本升级时保留用户数据
- **Hilt 集成**：依赖注入简化数据库访问
- **测试**：使用内存数据库进行单元测试

Room + Flow + Compose 是构建响应式数据层的完美组合。

---

*© 2024 Fidroid. [返回首页](../index.html)*

