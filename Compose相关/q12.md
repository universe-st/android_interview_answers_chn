# 如何调试 Compose 应用？

调试 Jetpack Compose 应用需要结合其**声明式 UI 特性**（如重组、状态驱动）和传统 Android 调试工具，重点关注布局结构、重组行为、状态变化等核心场景。以下是常用的调试方法和工具：


### 一、布局调试：可视化组件结构与属性
Compose 的布局是通过组合函数构建的，传统的 `View` 调试工具（如 `Hierarchy Viewer`）已不再适用，需使用专门针对 Compose 的工具。


#### 1. Layout Inspector（布局检查器）
Android Studio 内置的 **Layout Inspector** 是调试 Compose 布局的核心工具，可实时查看组件树、尺寸、位置及属性。

**使用步骤**：  
1. 运行应用，确保设备/模拟器处于活跃状态。  
2. 在 Android Studio 中点击菜单栏 `Tools > Layout Inspector`，选择当前运行的应用进程。  
3. 在左侧面板查看 **Compose 组件树**（标记为 `@Composable`），右侧可查看组件的 `Modifier` 属性（如尺寸、边距、对齐方式）和状态。  

**关键功能**：  
- **组件树定位**：点击 UI 上的元素，左侧树会自动定位到对应的 Compose 组件（如 `Column`、`Text`）。  
- **测量信息**：查看组件的 `measuredSize`、`position` 等，排查布局偏移、重叠问题。  
- **重组次数**：在组件树中，`Compositions` 列显示该组件的重组次数，频繁重组的组件需优化（见下文“重组调试”）。  


#### 2. 预览功能（`@Preview`）
`@Preview` 注解可在不运行应用的情况下快速预览 Compose UI，适合调试布局细节（如尺寸、样式）。

**示例**：  
```kotlin
// 预览多个配置（不同尺寸、主题）
@Preview(name = "Small Screen", device = "spec:width=320dp,height=480dp")
@Preview(name = "Dark Theme", uiMode = UI_MODE_NIGHT_YES)
@Composable
fun MyScreenPreview() {
    MaterialTheme {
        MyScreen() // 直接预览目标组件
    }
}
```

**优势**：  
- 快速迭代：修改代码后无需重新编译，预览窗口实时更新。  
- 隔离调试：单独预览某个子组件（如 `Button`），排除父组件干扰。  


### 二、重组调试：追踪不必要的重组
重组是 Compose 性能问题的常见根源，需定位“过度重组”的组件。


#### 1. 重组高亮（Show Compositions）
通过开发者选项开启“重组高亮”，直观观察哪些组件在频繁重组。

**开启步骤**：  
1. 打开设备的“开发者选项”（设置 → 关于手机 → 连续点击版本号）。  
2. 进入“开发者选项” → 找到“Compose 调试”（或“显示重组”）→ 开启“Show Compositions”。  

**效果**：  
组件重组时会闪烁绿色（轻度重组）或红色（频繁重组），可快速定位异常重组的区域。


#### 2. 日志打印与断点调试
在组合函数中添加日志或断点，追踪重组触发时机和次数。

**示例**：  
```kotlin
@Composable
fun UserProfile(user: User) {
    // 打印重组日志（包含参数值，判断是否因参数变化触发）
    Log.d("Recompose", "UserProfile重组，user: ${user.name}")
    
    // 断点调试：在组合函数内打断点，查看调用栈和参数变化
    Text("Name: ${user.name}")
}
```

**注意**：  
- 组合函数可能被频繁调用，断点会多次触发，建议结合条件断点（如仅当 `user.id == 1` 时暂停）。  
- 避免在组合函数中添加过多日志（可能影响性能），调试完成后移除。  


#### 3. `printToString()` 辅助函数
对于复杂状态（如列表、自定义对象），可通过 `printToString()` 追踪状态变化是否触发重组：  
```kotlin
@Composable
fun TodoList(todos: List<Todo>) {
    // 打印列表变化，判断是否因引用变化触发重组
    Log.d("StateChange", "Todos: ${todos.printToString()}")
    LazyColumn {
        items(todos) { TodoItem(it) }
    }
}
```


### 三、状态调试：追踪状态变化与传播
状态是重组的“触发器”，需确保状态变化符合预期（如是否正确触发重组、是否被正确共享）。


#### 1. 状态来源追踪
使用 `remember`、`mutableStateOf` 或状态容器（如 `ViewModel`）时，通过日志记录状态的修改时机：  
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Log.d("State", "初始count: $count") // 初始值
    
    Button(onClick = { 
        count++ 
        Log.d("State", "点击后count: $count") // 记录修改
    }) {
        Text("Count: $count")
    }
}
```


#### 2. `SnapshotStateList`/`SnapshotStateMap` 调试
对于可变集合（`mutableStateListOf`、`mutableStateMapOf`），需确认修改操作是否正确触发重组（普通集合修改不会触发）：  
```kotlin
@Composable
fun ShoppingCart() {
    val items = remember { mutableStateListOf<String>() }
    
    Button(onClick = { 
        items.add("Apple") 
        Log.d("StateList", "添加后: $items") // 确认集合变化被记录
    }) {
        Text("Add Item")
    }
    LazyColumn { items(items) { Text(it) } }
}
```


#### 3. 结合 ViewModel 调试
若状态由 `ViewModel` 管理，可通过日志或断点追踪 `ViewModel` 中状态的修改：  
```kotlin
class UserViewModel : ViewModel() {
    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user.asStateFlow()
    
    fun fetchUser() {
        viewModelScope.launch {
            Log.d("ViewModel", "开始请求用户数据")
            _user.value = api.getUser() // 记录状态更新
            Log.d("ViewModel", "用户数据更新: ${_user.value?.name}")
        }
    }
}
```


### 四、崩溃与异常调试
Compose 中的异常可能发生在**组合阶段**（如空指针、参数错误）或**绘制阶段**（如尺寸非法），需通过日志和堆栈定位。


#### 1. Logcat 日志分析
崩溃时，Logcat 会输出包含 `Compose` 关键字的堆栈信息，重点关注：  
- **组合阶段异常**：通常显示 `@Composable` 函数的调用栈，例如：  
  ```
  java.lang.NullPointerException: user must not be null
      at com.example.MyApp.UserProfile(UserProfile.kt:15)
      at com.example.MyApp.HomeScreen(HomeScreen.kt:20)
  ```  
  直接定位到 `UserProfile` 函数第 15 行的空指针问题。  

- **绘制阶段异常**：可能涉及 `Modifier` 尺寸设置（如负数尺寸），堆栈中会包含 `LayoutNode` 相关信息。  


#### 2. 启用 Compose 调试模式
在 `AndroidManifest.xml` 中启用 Compose 调试模式，获取更详细的错误信息：  
```xml
<application
    ...
    android:debuggable="true"> <!-- 开发环境默认开启 -->
</application>
```


### 五、动画调试：分析动画行为
Compose 动画（如 `Animatable`、`rememberInfiniteTransition`）的异常（如卡顿、值异常）可通过 **Animation Inspector** 调试。

**使用步骤**：  
1. 运行应用，打开 Layout Inspector（见上文）。  
2. 在 Layout Inspector 中点击顶部的 **Animation Inspector** 图标（播放按钮）。  
3. 选择正在运行的动画，查看动画的**时间线**、**当前值**、**目标值**等，判断是否因值突变或计算错误导致异常。  


### 六、性能调试：定位性能瓶颈
使用 Android Studio 的 **Profiler** 分析 Compose 应用的性能问题（如卡顿、高 CPU 占用）。

#### 1. CPU  Profiler
- 记录应用运行时的 CPU 活动，查看 `Compose` 相关函数（如 `recompose`、`measure`）的耗时。  
- 若 `recompose` 函数耗时过高，说明存在过度重组；若 `measure` 耗时高，可能是布局层级过深。  


#### 2. 内存 Profiler
- 检测内存泄漏：例如 `ViewModel` 未正确销毁、`LaunchedEffect` 未正确取消协程等。  
- 关注 `Compose` 相关对象（如 `LayoutNode`、`Composition`）的内存占用，异常增长可能暗示布局未被正确回收。  


### 七、其他实用技巧
1. **使用 `@Stable`/`@Immutable` 验证稳定性**：  
   若自定义类型标记为 `@Stable` 但实际不稳定（如 `var` 属性未用 `mutableStateOf`），可通过日志检查是否因稳定性问题导致过度重组。  

2. **调试 `CompositionLocal`**：  
   若 `CompositionLocal` 的值不符合预期，可在提供值和访问值的地方添加日志，追踪值的覆盖链（如是否被某个 `CompositionLocalProvider` 意外覆盖）。  

3. **混合应用调试**：  
   若应用同时使用 Compose 和传统 View（如 `XML`），可在 Layout Inspector 中切换“Compose”和“View”视图，分别调试两套 UI 结构。  


### 总结
调试 Compose 应用需结合其特性，重点关注：  
- 用 **Layout Inspector** 可视化布局结构和重组次数；  
- 开启 **重组高亮** 定位过度重组；  
- 通过日志和断点追踪 **状态变化** 和 **组合函数调用**；  
- 用 **Profiler** 分析性能瓶颈（CPU、内存）。  

熟练掌握这些工具和方法，可高效解决 Compose 开发中的布局、状态、性能等问题。