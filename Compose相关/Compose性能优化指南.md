# 如何优化 Compose 程序的性能？

优化 Jetpack Compose 程序的性能需要从**重组机制、状态管理、布局设计、资源使用**等多个核心维度入手，结合 Compose 的特性针对性优化。以下是关键优化方向及具体实践：


### 一、避免不必要的重组
Compose 的核心是“重组”（Recomposition）—— 当状态变化时，仅重新执行依赖该状态的组合函数。但不合理的代码可能导致“过度重组”，浪费资源。


#### 1. 减少组合阶段的耗时操作
组合阶段（Composition）的职责是**描述 UI 结构**，应保持轻量。避免在组合函数中执行：
- 耗时计算（如复杂循环、解析）；
- IO 操作（如网络请求、文件读写）；
- 创建临时对象（如每次重组都新建 `List`、`lambda` 或自定义对象）。

**解决方案**：
- 用 `remember` 缓存计算结果（仅在依赖变化时重新计算）：
  ```kotlin
  @Composable
  fun HeavyCalculation(data: List<Int>) {
      // 缓存计算结果，仅 data 变化时重新计算
      val result by remember(data) {
          derivedStateOf { data.sumOf { it * 2 } } // 模拟耗时计算
      }
      Text("Result: $result")
  }
  ```
- 耗时操作放在协程（`LaunchedEffect`）或后台线程，结果通过状态传递：
  ```kotlin
  @Composable
  fun LoadData() {
      var data by remember { mutableStateOf<List<String>?>(null) }
      LaunchedEffect(Unit) {
          // 后台加载数据（不阻塞组合阶段）
          data = api.fetchData() 
      }
      data?.let { Text(it.joinToString()) }
  }
  ```


#### 2. 确保参数和 lambda 的稳定性
Compose 会通过“稳定性”（Stability）判断是否需要重组：**稳定类型**（如 `Int`、`String`、`data class`）的参数变化时才触发重组；**不稳定类型**（如普通 `class`、匿名 `lambda`）可能导致频繁重组。

**优化方案**：
- 用 `@Stable` 或 `@Immutable` 标记自定义类型，告诉 Compose 其稳定性：
  ```kotlin
  @Immutable // 不可变类型（所有属性都是 val 且稳定）
  data class User(val id: Int, val name: String)
  
  @Stable // 稳定类型（属性变化时可被感知）
  class Config(var theme: String) {
      // 确保属性变化时通过状态通知（如用 mutableStateOf）
  }
  ```
- 避免在组合函数中创建匿名 `lambda`（每次重组都会生成新对象，被视为不稳定），改为顶层函数或用 `remember` 缓存：
  ```kotlin
  // 不好：每次重组创建新 lambda
  Button(onClick = { /* 逻辑 */ }) { ... }
  
  // 好：顶层函数（稳定）
  private fun onButtonClick() { /* 逻辑 */ }
  Button(onClick = ::onButtonClick) { ... }
  
  // 好：用 remember 缓存（依赖不变时复用）
  Button(onClick = remember { { /* 逻辑 */ } }) { ... }
  ```


#### 3. 拆分组合函数，缩小重组范围
大而全的组合函数会导致“一触即发”的大面积重组。应按职责拆分小型、独立的函数，让重组仅影响必要部分。

**示例**：
```kotlin
// 不好：一个函数处理所有逻辑，任何状态变化都会整体重组
@Composable
fun UserProfile(user: User, onEdit: () -> Unit) {
    Column {
        Text("Name: ${user.name}") // 依赖 user.name
        Text("Age: ${user.age}")   // 依赖 user.age
        Button(onClick = onEdit) { Text("Edit") } // 依赖 onEdit
    }
}

// 好：拆分为独立函数，仅依赖变化的部分重组
@Composable
fun UserProfile(user: User, onEdit: () -> Unit) {
    Column {
        UserName(user.name) // 仅 user.name 变化时重组
        UserAge(user.age)   // 仅 user.age 变化时重组
        EditButton(onEdit)  // 仅 onEdit 变化时重组
    }
}

@Composable private fun UserName(name: String) { Text("Name: $name") }
@Composable private fun UserAge(age: Int) { Text("Age: $age") }
@Composable private fun EditButton(onClick: () -> Unit) { Button(onClick) { ... } }
```


### 二、优化状态管理
状态是重组的“触发器”，状态设计不合理会导致频繁、无意义的重组。


#### 1. 状态“局部化”，避免过度提升
状态提升（State Hoisting）是推荐的模式，但过度提升（将状态放到过高层级）会导致无关组件重组。应仅提升“必要共享”的状态。

**示例**：
```kotlin
// 不好：将子组件的局部状态提升到顶层，导致整个页面重组
@Composable
fun Parent() {
    var childCount by remember { mutableStateOf(0) } // 仅子组件用的状态
    Column {
        Header() // 无关组件，却会因 childCount 变化重组
        Child(count = childCount, onIncrement = { childCount++ })
    }
}

// 好：状态留在子组件，仅子组件重组
@Composable
fun Parent() {
    Column {
        Header() // 不受影响
        Child() // 内部管理 count 状态
    }
}

@Composable
private fun Child() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) { Text("Count: $count") }
}
```


#### 2. 使用不可变数据和快照状态集合
- **不可变数据**：优先用 `val` 和不可变集合（如 `List`、`Map`），Compose 对不可变对象的变化检测更高效（直接对比引用）。
- **快照状态集合**：若需可变集合，用 Compose 提供的 `SnapshotStateList`（`mutableStateListOf`）或 `SnapshotStateMap`（`mutableStateMapOf`），它们会自动通知状态变化：
  ```kotlin
  // 错误：普通 mutableList 变化时，Compose 无法感知
  val list = remember { mutableListOf<String>() } 
  list.add("item") // 不会触发重组
  
  // 正确：用 SnapshotStateList
  val list = remember { mutableStateListOf<String>() }
  list.add("item") // 自动触发重组
  ```


#### 3. 用 `derivedStateOf` 减少高频状态的重组
当一个状态由另一个高频变化的状态派生而来（如滚动位置 → 导航栏透明度），用 `derivedStateOf` 限制重组频率（仅派生结果变化时才触发）。

**示例**：
```kotlin
@Composable
fun ScrollableContent() {
    val listState = rememberLazyListState()
    // 滚动位置 > 100 时显示导航栏（仅位置过阈值时才变化）
    val showToolbar by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }
    LazyColumn(state = listState) { ... }
    if (showToolbar) Toolbar()
}
```


### 三、布局与绘制优化
布局和绘制是性能消耗的重灾区，需减少层级、避免过度测量和绘制。


#### 1. 减少布局嵌套和层级
Compose 的布局系统（如 `Column`、`Row`）会递归测量子组件，层级过深（如 10 层以上）会显著增加测量成本。

**优化方案**：
- 用 `Modifier` 链式调用替代嵌套容器（如 `Modifier.padding().size()` 替代 `Box` 套 `Box`）；
- 避免无意义的容器（如仅用于包裹的 `Box` 可直接移除）；
- 复杂布局用 `Layout` 自定义测量逻辑，减少冗余测量：
  ```kotlin
  // 自定义布局：一次测量所有子组件，避免重复测量
  @Composable
  fun CustomLayout(content: @Composable () -> Unit) {
      Layout(content = content) { measurables, constraints ->
          // 测量所有子组件（仅一次）
          val placeables = measurables.map { it.measure(constraints) }
          // 计算布局大小并放置子组件
          layout(constraints.maxWidth, constraints.maxHeight) {
              placeables.forEach { it.place(0, 0) }
          }
      }
  }
  ```


#### 2. 优化列表性能（LazyColumn/LazyRow）
长列表（如 100 项以上）必须用 `LazyColumn`（垂直）或 `LazyRow`（水平），它们会**懒加载可见项**并回收不可见项，避免一次性渲染所有内容。

**关键优化**：
- 为列表项设置稳定的 `key`：确保数据变化时可复用已有项，减少重组和重新布局：
  ```kotlin
  LazyColumn {
      items(users, key = { user -> user.id }) { user -> // 用唯一 id 作为 key
          UserItem(user)
      }
  }
  ```
- 避免列表项内部的“重计算”：列表项的 `lambda` 会频繁执行，内部避免创建临时对象或耗时操作（用 `remember` 缓存）。


#### 3. 减少过度绘制（Overdraw）
过度绘制指同一区域被多次绘制（如多层半透明背景），会浪费 GPU 资源。

**优化方案**：
- 移除不必要的背景（如父容器和子组件重复设置背景）；
- 减少透明效果（如 `alpha`、`shadow`）的使用，必要时用 `Modifier.clip` 限制绘制范围；
- 通过 Android Studio 的 **Layout Inspector** 查看过度绘制（开启“Overdraw”高亮）。


### 四、资源与动画优化
#### 1. 图片加载优化
- 用 `Coil` 或 `Glide` 等库加载图片（自动处理缓存、尺寸适配、内存管理），避免直接用 `painterResource` 加载大图片；
- 列表中加载图片时，设置固定尺寸并启用缓存：
  ```kotlin
  AsyncImage(
      model = ImageRequest.Builder(LocalContext.current)
          .data(user.avatarUrl)
          .size(200) // 限制尺寸，避免加载过大图片
          .crossfade(true)
          .build(),
      contentDescription = null,
      modifier = Modifier.size(100.dp)
  )
  ```


#### 2. 动画优化
动画会频繁触发重组和绘制，需控制影响范围：
- 用 `Animatable` 或 `rememberInfiniteTransition` 时，确保动画值仅传递给必要组件；
- 避免在动画中修改布局尺寸（如 `width`、`height`），优先用 `scale`、`alpha` 等不影响布局的属性。


### 五、利用工具定位性能问题
- **Layout Inspector**：查看重组次数（“Compositions”列）、布局层级、过度绘制；
- **Profiler**：通过 CPU 记录分析耗时函数，Memory 记录检测内存泄漏；
- **重组高亮**：在开发者选项中开启“Show Compositions”，屏幕上闪烁的区域即为重组部分，频繁闪烁的区域需优化。


### 总结
Jetpack Compose 性能优化的核心是：**让重组只发生在必要的地方，让布局和绘制尽可能高效**。通过合理拆分组件、优化状态管理、减少冗余操作，并结合工具定位瓶颈，可显著提升应用流畅度。