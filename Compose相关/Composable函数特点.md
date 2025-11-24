# Composable 函数的特点是什么？

Composable 函数是 Jetpack Compose 的核心构建块，它们具有以下重要特点：

## 1. **声明式 UI**
- 描述 UI **应该是什么样子**，而不是如何构建它
- UI 是应用程序状态的函数：`UI = f(state)`

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello, $name!") // 声明要显示什么，而不是如何显示
}
```

## 2. **使用 @Composable 注解**
- 所有可组合函数都必须使用 `@Composable` 注解标记
- 这告诉 Compose 编译器这是一个用于构建 UI 的特殊函数

## 3. **可以调用其他 Composable 函数**
- 通过组合简单的 Composable 来构建复杂的 UI

```kotlin
@Composable
fun UserProfile(user: User) {
    Column {
        UserHeader(user.name, user.avatar) // 调用其他 Composable
        UserStats(user.stats)
        ActionButtons()
    }
}
```

## 4. **没有返回值**
- 不返回 View 对象，而是**直接发射 UI 到 Compose 运行时**
- 这是与常规函数的主要区别之一

## 5. **重组（Recomposition）**
- 当状态变化时，Composable 函数会被重新执行
- 这个过程称为"重组"
- 只有读取了变化状态的 Composable 才会重组

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Button(onClick = { count++ }) {
        Text("Clicked $count times") // 当 count 变化时，只有这个 Text 会重组
    }
}
```

## 6. **智能重组**
- Compose 会跳过参数未变化的 Composable
- 通过比较参数来决定是否需要重组

## 7. **可以拥有本地状态**
- 使用 `remember` API 在重组间保持状态
- 状态变化会触发重组

```kotlin
@Composable
fun ToggleButton() {
    var isOn by remember { mutableStateOf(false) }
    
    Button(
        onClick = { isOn = !isOn },
        colors = ButtonDefaults.buttonColors(
            backgroundColor = if (isOn) Color.Green else Color.Gray
        )
    ) {
        Text(if (isOn) "ON" else "OFF")
    }
}
```

## 8. **副作用管理**
- Composable 函数应该是**幂等**的，但有时需要执行副作用
- 使用 Effect API（如 `LaunchedEffect`, `DisposableEffect`）来安全地管理副作用

```kotlin
@Composable
fun TimerDisplay() {
    var time by remember { mutableStateOf(0) }
    
    LaunchedEffect(Unit) { // 副作用：启动计时器
        while (true) {
            delay(1000)
            time++
        }
    }
    
    Text("Time: $time seconds")
}
```

## 9. **接收参数**
- 可以接收参数来控制其行为和外观
- 参数变化会触发重组

```kotlin
@Composable
fun NewsArticle(
    title: String,        // 必需参数
    content: String,      // 必需参数
    isFeatured: Boolean = false  // 带默认值的可选参数
) {
    Card(
        elevation = if (isFeatured) 8.dp else 2.dp
    ) {
        Column {
            Text(title, style = MaterialTheme.typography.h5)
            Text(content)
        }
    }
}
```

## 10. **支持预览**
- 使用 `@Preview` 注解可以在 Android Studio 中预览 Composable

```kotlin
@Preview(showBackground = true)
@Composable
fun PreviewGreeting() {
    Greeting(name = "Android")
}
```

## 关键行为特点：

### **幂等性**
- 使用相同的参数多次调用应该产生相同的 UI
- 不应该依赖外部状态或随机值

### **无副作用（理想情况下）**
- 不应该修改全局状态、进行网络请求或文件操作
- 副作用应该通过专门的 Effect API 处理

### **快速执行**
- 可能会被频繁调用（在重组期间）
- 应该避免耗时操作

### **可以以任何顺序执行**
- Compose 运行时可以按任何顺序调用 Composable
- 不应该依赖执行顺序

这些特点使得 Composable 函数成为构建响应式、可维护 UI 的强大工具，同时自动处理了 UI 更新的复杂性。