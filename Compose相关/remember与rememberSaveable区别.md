# remember 和 rememberSaveable有什么区别？

`remember` 和 `rememberSaveable` 都是 Jetpack Compose 中用于在重组间保持状态的函数，但它们在工作机制和使用场景上有重要区别。

## 核心区别

| 特性 | `remember` | `rememberSaveable` |
|------|------------|-------------------|
| **生命周期** | 只在当前组合期间有效 | 跨配置更改（如旋转屏幕）保持 |
| **数据持久化** | 不持久化，进程死亡后丢失 | 通过 Bundle 自动保存/恢复 |
| **使用场景** | UI 临时状态、计算缓存 | 需要持久化的用户数据 |
| **性能开销** | 较低 | 稍高（需要序列化/反序列化） |

## 详细解释

### 1. `remember` - 组合期记忆

`remember` 只在**当前组合的生命周期内**保持状态。当 Activity/Fragment 被重建时（如屏幕旋转），状态会丢失。

```kotlin
@Composable
fun RememberExample() {
    // 这个状态在屏幕旋转时会重置为 0
    var count by remember { mutableStateOf(0) }
    
    Column {
        Button(onClick = { count++ }) {
            Text("Count: $count")
        }
        Text("旋转屏幕后计数会重置!")
    }
}
```

**使用场景：**
- 临时 UI 状态（如动画进度、展开/收起状态）
- 昂贵的计算结果的缓存
- 不需要持久化的中间状态

### 2. `rememberSaveable` - 跨配置记忆

`rememberSaveable` 通过 Android 的 `Bundle` 系统自动保存和恢复状态，**能够幸存于配置更改**。

```kotlin
@Composable
fun RememberSaveableExample() {
    // 这个状态在屏幕旋转后仍然保持
    var count by rememberSaveable { mutableStateOf(0) }
    
    Column {
        Button(onClick = { count++ }) {
            Text("Count: $count")
        }
        Text("旋转屏幕后计数保持不变!")
    }
}
```

## 实际示例对比

### 场景：计数器应用
```kotlin
@Composable
fun CounterApp() {
    var tempCount by remember { mutableStateOf(0) }        // 临时计数
    var persistentCount by rememberSaveable { mutableStateOf(0) } // 持久计数
    
    Column {
        // 临时计数器 - 旋转屏幕会重置
        Card {
            Column {
                Text("临时计数器 (会重置)")
                Button(onClick = { tempCount++ }) {
                    Text("临时: $tempCount")
                }
            }
        }
        
        // 持久计数器 - 旋转屏幕保持
        Card {
            Column {
                Text("持久计数器 (保持状态)")
                Button(onClick = { persistentCount++ }) {
                    Text("持久: $persistentCount")
                }
            }
        }
    }
}
```

## `rememberSaveable` 的高级用法

### 1. 自定义 Saver

当数据类型无法自动保存时，可以自定义 `Saver`：

```kotlin
// 自定义数据类，无法自动保存
data class UserSettings(
    val theme: String,
    val fontSize: Int
)

// 创建自定义 Saver
val UserSettingsSaver = run {
    val themeKey = "theme"
    val fontSizeKey = "fontSize"
    
    mapSaver(
        save = { settings ->
            mapOf(
                themeKey to settings.theme,
                fontSizeKey to settings.fontSize
            )
        },
        restore = { map ->
            UserSettings(
                theme = map[themeKey] as String,
                fontSize = map[fontSizeKey] as Int
            )
        }
    )
}

@Composable
fun UserSettingsScreen() {
    var settings by rememberSaveable(stateSaver = UserSettingsSaver) {
        mutableStateOf(UserSettings("Dark", 16))
    }
    
    // 使用 settings...
}
```

### 2. 使用 `@Parcelize` 注解

对于复杂对象，可以使用 Parcelable：

```kotlin
@Parcelize
data class AppConfig(
    val serverUrl: String,
    val timeout: Int,
    val features: List<String>
) : Parcelable

@Composable
fun ConfigScreen() {
    var config by rememberSaveable {
        mutableStateOf(AppConfig("https://api.example.com", 30, emptyList()))
    }
    
    // rememberSaveable 会自动处理 Parcelable 对象
}
```

### 3. 带键的 rememberSaveable

与 `remember` 一样，`rememberSaveable` 也支持依赖键：

```kotlin
@Composable
fun UserProfile(userId: String) {
    val userData = rememberSaveable(userId, saver = UserSaver) {
        mutableStateOf(loadUserData(userId))
    }
    
    // 当 userId 变化时重新加载，但相同 userId 在配置更改时保持状态
}
```

## 使用场景指南

### 使用 `remember` 的情况：

```kotlin
@Composable
fun MyComponent() {
    // ✅ 动画进度 - 不需要持久化
    val animationProgress by remember { 
        Animatable(0f) 
    }
    
    // ✅ 计算昂贵的值 - 只是性能优化
    val filteredList = remember(list) {
        list.filter { it.isActive }.sorted()
    }
    
    // ✅ 临时 UI 状态
    var isExpanded by remember { mutableStateOf(false) }
    
    // ✅ 对话框显示状态
    var showDialog by remember { mutableStateOf(false) }
}
```

### 使用 `rememberSaveable` 的情况：

```kotlin
@Composable
fun UserInputScreens() {
    // ✅ 用户输入的表单数据
    var username by rememberSaveable { mutableStateOf("") }
    var email by rememberSaveable { mutableStateOf("") }
    
    // ✅ 用户选择/偏好设置
    var selectedTheme by rememberSaveable { mutableStateOf("Dark") }
    var notificationEnabled by rememberSaveable { mutableStateOf(true) }
    
    // ✅ 列表滚动位置
    var listState by rememberSaveable { 
        mutableStateOf(LazyListState()) 
    }
    
    // ✅ 分页或标签页状态
    var selectedTab by rememberSaveable { mutableStateOf(0) }
}
```

## 性能考虑

### `remember` 的性能优势：
```kotlin
@Composable
fun PerformanceExample(heavyData: List<Data>) {
    // ✅ 使用 remember 缓存昂贵计算
    val processedData = remember(heavyData) {
        heavyData
            .filter { it.isValid }
            .map { transformData(it) }
            .sortedBy { it.priority }
    }
    
    // 如果没有 remember，每次重组都会重新计算
    val inefficientData = heavyData
        .filter { it.isValid }
        .map { transformData(it) }
        .sortedBy { it.priority }
}
```

### `rememberSaveable` 的开销：
```kotlin
@Composable
fun SaveableOverheadExample() {
    // ❌ 不必要的使用 - 临时状态不需要持久化
    var temporaryUiState by rememberSaveable { mutableStateOf(false) }
    
    // ✅ 合适的用法 - 用户数据需要持久化
    var userPreference by rememberSaveable { mutableStateOf("default") }
}
```

## 最佳实践

### 1. 合理选择
```kotlin
@Composable
fun BestPractices() {
    // 根据数据的重要性选择
    val temporaryState = remember { mutableStateOf(false) } // UI 状态
    val userData = rememberSaveable { mutableStateOf("") }  // 用户数据
    
    // 根据生命周期需求选择
    val cache = remember(data) { expensiveOperation(data) } // 计算缓存
    val settings = rememberSaveable { mutableStateOf(Settings()) } // 设置
}
```

### 2. 组合使用
```kotlin
@Composable
fun CombinedUsage() {
    // 临时状态 + 持久状态组合
    var draftMessage by remember { mutableStateOf("") }        // 草稿，可丢弃
    var sentMessages by rememberSaveable { mutableStateOf(listOf<String>()) } // 已发送，需保存
    
    fun sendMessage() {
        sentMessages = sentMessages + draftMessage
        draftMessage = "" // 清空草稿
    }
}
```

## 总结

**关键区别：**
- `remember`：**组合期内**保持状态，配置更改时重置
- `rememberSaveable`：**跨配置更改**保持状态，通过 Bundle 持久化

**选择指南：**
- 使用 `remember` 用于：临时 UI 状态、动画、计算缓存
- 使用 `rememberSaveable` 用于：用户输入、应用设置、需要持久化的数据

**记住规则：** 如果用户期望某个状态在屏幕旋转后仍然保持，就使用 `rememberSaveable`；否则使用 `remember`。