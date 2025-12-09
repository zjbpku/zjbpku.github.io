# Compose + DataStore：数据持久化最佳实践

> **发布日期**: 2024-06-07  
> **阅读时间**: 约 22 分钟  
> **标签**: DataStore, Preferences, Proto, 持久化

DataStore 是 Android 官方推荐的数据持久化方案，用于替代 SharedPreferences。它提供异步、一致性的 API，并与 Kotlin 协程和 Flow 无缝集成。本文将深入讲解 DataStore 在 Compose 项目中的最佳实践。

## 一、DataStore 简介

DataStore 提供两种实现：

| 类型 | 适用场景 | 数据格式 |
|-----|---------|---------|
| Preferences DataStore | 键值对存储 | 类似 Map |
| Proto DataStore | 类型安全的复杂对象 | Protocol Buffers |

## 二、Preferences DataStore

### 添加依赖

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.datastore:datastore-preferences:1.0.0")
}
```

### 创建 DataStore

```kotlin
// 扩展属性创建 DataStore（推荐在顶层声明）
val Context.settingsDataStore: DataStore<Preferences> by preferencesDataStore(
    name = "settings"
)

// 定义键
object PreferencesKeys {
    val THEME_MODE = stringPreferencesKey("theme_mode")
    val NOTIFICATIONS_ENABLED = booleanPreferencesKey("notifications_enabled")
    val FONT_SIZE = intPreferencesKey("font_size")
    val USERNAME = stringPreferencesKey("username")
}
```

### 读写数据

```kotlin
class SettingsRepository(
    private val dataStore: DataStore<Preferences>
) {
    // 读取数据（返回 Flow）
    val themeMode: Flow<String> = dataStore.data
        .catch { exception ->
            if (exception is IOException) {
                emit(emptyPreferences())
            } else {
                throw exception
            }
        }
        .map { preferences ->
            preferences[PreferencesKeys.THEME_MODE] ?: "system"
        }
    
    val notificationsEnabled: Flow<Boolean> = dataStore.data
        .map { preferences ->
            preferences[PreferencesKeys.NOTIFICATIONS_ENABLED] ?: true
        }
    
    val fontSize: Flow<Int> = dataStore.data
        .map { preferences ->
            preferences[PreferencesKeys.FONT_SIZE] ?: 16
        }
    
    // 写入数据
    suspend fun setThemeMode(mode: String) {
        dataStore.edit { preferences ->
            preferences[PreferencesKeys.THEME_MODE] = mode
        }
    }
    
    suspend fun setNotificationsEnabled(enabled: Boolean) {
        dataStore.edit { preferences ->
            preferences[PreferencesKeys.NOTIFICATIONS_ENABLED] = enabled
        }
    }
    
    suspend fun setFontSize(size: Int) {
        dataStore.edit { preferences ->
            preferences[PreferencesKeys.FONT_SIZE] = size
        }
    }
    
    // 清除所有数据
    suspend fun clearAll() {
        dataStore.edit { preferences ->
            preferences.clear()
        }
    }
}
```

### ViewModel 集成

```kotlin
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val settingsRepository: SettingsRepository
) : ViewModel() {
    
    val themeMode: StateFlow<String> = settingsRepository.themeMode
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = "system"
        )
    
    val notificationsEnabled: StateFlow<Boolean> = settingsRepository.notificationsEnabled
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = true
        )
    
    val fontSize: StateFlow<Int> = settingsRepository.fontSize
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = 16
        )
    
    fun setThemeMode(mode: String) {
        viewModelScope.launch {
            settingsRepository.setThemeMode(mode)
        }
    }
    
    fun setNotificationsEnabled(enabled: Boolean) {
        viewModelScope.launch {
            settingsRepository.setNotificationsEnabled(enabled)
        }
    }
    
    fun setFontSize(size: Int) {
        viewModelScope.launch {
            settingsRepository.setFontSize(size)
        }
    }
}
```

### 在 Compose 中使用

```kotlin
@Composable
fun SettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val themeMode by viewModel.themeMode.collectAsState()
    val notificationsEnabled by viewModel.notificationsEnabled.collectAsState()
    val fontSize by viewModel.fontSize.collectAsState()
    
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp)
    ) {
        item {
            Text(
                text = "设置",
                style = MaterialTheme.typography.headlineMedium,
                modifier = Modifier.padding(bottom = 16.dp)
            )
        }
        
        // 主题设置
        item {
            SettingsSection(title = "外观") {
                ThemeModeSelector(
                    currentMode = themeMode,
                    onModeSelected = { viewModel.setThemeMode(it) }
                )
            }
        }
        
        // 通知设置
        item {
            SettingsSection(title = "通知") {
                SwitchSetting(
                    title = "启用通知",
                    checked = notificationsEnabled,
                    onCheckedChange = { viewModel.setNotificationsEnabled(it) }
                )
            }
        }
        
        // 字体大小
        item {
            SettingsSection(title = "显示") {
                SliderSetting(
                    title = "字体大小",
                    value = fontSize.toFloat(),
                    valueRange = 12f..24f,
                    onValueChange = { viewModel.setFontSize(it.toInt()) }
                )
            }
        }
    }
}

@Composable
fun SettingsSection(
    title: String,
    content: @Composable () -> Unit
) {
    Column(modifier = Modifier.padding(vertical = 8.dp)) {
        Text(
            text = title,
            style = MaterialTheme.typography.titleSmall,
            color = MaterialTheme.colorScheme.primary
        )
        Spacer(modifier = Modifier.height(8.dp))
        content()
    }
}

@Composable
fun SwitchSetting(
    title: String,
    checked: Boolean,
    onCheckedChange: (Boolean) -> Unit
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(vertical = 8.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text(text = title)
        Switch(checked = checked, onCheckedChange = onCheckedChange)
    }
}
```

## 三、Proto DataStore

Proto DataStore 提供类型安全的数据存储。

### 添加依赖

```kotlin
// build.gradle.kts
plugins {
    id("com.google.protobuf") version "0.9.4"
}

dependencies {
    implementation("androidx.datastore:datastore:1.0.0")
    implementation("com.google.protobuf:protobuf-javalite:3.21.12")
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.21.12"
    }
    generateProtoTasks {
        all().forEach { task ->
            task.builtins {
                create("java") {
                    option("lite")
                }
            }
        }
    }
}
```

### 定义 Proto 文件

```protobuf
// src/main/proto/user_preferences.proto
syntax = "proto3";

option java_package = "com.example.app";
option java_multiple_files = true;

message UserPreferences {
    string theme_mode = 1;
    bool notifications_enabled = 2;
    int32 font_size = 3;
    
    message User {
        string id = 1;
        string name = 2;
        string email = 3;
    }
    
    User current_user = 4;
}
```

### 创建 Serializer

```kotlin
object UserPreferencesSerializer : Serializer<UserPreferences> {
    override val defaultValue: UserPreferences = UserPreferences.getDefaultInstance()
    
    override suspend fun readFrom(input: InputStream): UserPreferences {
        try {
            return UserPreferences.parseFrom(input)
        } catch (exception: InvalidProtocolBufferException) {
            throw CorruptionException("Cannot read proto.", exception)
        }
    }
    
    override suspend fun writeTo(t: UserPreferences, output: OutputStream) {
        t.writeTo(output)
    }
}
```

### 创建 DataStore

```kotlin
val Context.userPreferencesDataStore: DataStore<UserPreferences> by dataStore(
    fileName = "user_preferences.pb",
    serializer = UserPreferencesSerializer
)
```

### 读写数据

```kotlin
class UserPreferencesRepository(
    private val dataStore: DataStore<UserPreferences>
) {
    val userPreferences: Flow<UserPreferences> = dataStore.data
        .catch { exception ->
            if (exception is IOException) {
                emit(UserPreferences.getDefaultInstance())
            } else {
                throw exception
            }
        }
    
    val themeMode: Flow<String> = userPreferences.map { it.themeMode }
    val currentUser: Flow<UserPreferences.User?> = userPreferences.map { 
        if (it.hasCurrentUser()) it.currentUser else null 
    }
    
    suspend fun setThemeMode(mode: String) {
        dataStore.updateData { preferences ->
            preferences.toBuilder()
                .setThemeMode(mode)
                .build()
        }
    }
    
    suspend fun setCurrentUser(id: String, name: String, email: String) {
        dataStore.updateData { preferences ->
            preferences.toBuilder()
                .setCurrentUser(
                    UserPreferences.User.newBuilder()
                        .setId(id)
                        .setName(name)
                        .setEmail(email)
                        .build()
                )
                .build()
        }
    }
    
    suspend fun clearCurrentUser() {
        dataStore.updateData { preferences ->
            preferences.toBuilder()
                .clearCurrentUser()
                .build()
        }
    }
}
```

## 四、从 SharedPreferences 迁移

```kotlin
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(
    name = "settings",
    produceMigrations = { context ->
        listOf(
            SharedPreferencesMigration(
                context = context,
                sharedPreferencesName = "old_settings"
            )
        )
    }
)
```

### 自定义迁移

```kotlin
val migration = object : DataMigration<Preferences> {
    override suspend fun shouldMigrate(currentData: Preferences): Boolean {
        return currentData[PreferencesKeys.MIGRATION_VERSION] != 2
    }
    
    override suspend fun migrate(currentData: Preferences): Preferences {
        return currentData.toMutablePreferences().apply {
            // 执行迁移逻辑
            this[PreferencesKeys.MIGRATION_VERSION] = 2
        }.toPreferences()
    }
    
    override suspend fun cleanUp() {
        // 清理旧数据
    }
}
```

## 五、Hilt 集成

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {
    
    @Provides
    @Singleton
    fun provideSettingsDataStore(@ApplicationContext context: Context): DataStore<Preferences> {
        return context.settingsDataStore
    }
    
    @Provides
    @Singleton
    fun provideSettingsRepository(dataStore: DataStore<Preferences>): SettingsRepository {
        return SettingsRepository(dataStore)
    }
}
```

## 六、测试

```kotlin
@Test
fun `theme mode is saved and retrieved correctly`() = runTest {
    val testDataStore = PreferenceDataStoreFactory.create(
        scope = TestScope(UnconfinedTestDispatcher())
    ) {
        testContext.preferencesDataStoreFile("test_settings")
    }
    
    val repository = SettingsRepository(testDataStore)
    
    repository.setThemeMode("dark")
    
    val themeMode = repository.themeMode.first()
    assertEquals("dark", themeMode)
}
```

## 七、最佳实践

- ✅ 在顶层声明 DataStore（避免多实例）
- ✅ 使用 Flow 观察数据变化
- ✅ 处理 IOException 异常
- ✅ 使用 stateIn 转换为 StateFlow
- ✅ Proto DataStore 用于复杂类型
- ✅ 使用迁移从 SharedPreferences 升级
- ✅ Hilt 注入 DataStore 依赖
- ✅ 单元测试使用内存 DataStore

## 八、DataStore vs SharedPreferences

| 特性 | DataStore | SharedPreferences |
|-----|-----------|------------------|
| 异步 API | ✅ | ❌ |
| 类型安全 | ✅ | ❌ |
| 错误处理 | ✅ | ❌ |
| Flow 支持 | ✅ | ❌ |
| 无 UI 阻塞 | ✅ | ❌ |
| 事务支持 | ✅ | ❌ |

## 总结

DataStore 与 Compose 集成的核心要点：

- **Preferences DataStore**：简单键值对存储
- **Proto DataStore**：类型安全的复杂对象
- **Flow**：响应式数据观察
- **stateIn**：转换为 StateFlow 在 Compose 使用
- **Migration**：从 SharedPreferences 平滑迁移
- **Hilt**：依赖注入简化访问

DataStore 是现代 Android 应用数据持久化的首选方案。

---

*© 2024 Fidroid. [返回首页](../index.html)*

