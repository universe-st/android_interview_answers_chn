# 什么是副作用？如何处理副作用？

在 Jetpack Compose 中，**副作用**是指在可组合函数范围之外发生的状态变化。由于 Compose 的声明式特性，直接在执行可组合函数时修改外部状态会导致不可预测的行为。

## 什么是副作用？

副作用包括：
- 修改共享对象的状态
- 更新 ViewModel 中的状态
- 共享 UI 状态（如 `SharedFlow`）
- 访问外部资源（数据库、网络）
- 执行一次性操作（如显示 Snackbar）

**问题示例**：
```kotlin
@Composable
fun ProblematicCounter() {
    var count = 0 // 每次重组都会重置为 0
    
    Button(onClick = { count++ }) {
        Text("Count: $count") // 永远显示 1
    }
}
```

## 处理副作用的常用 API

### 1. LaunchedEffect - 协程副作用
```kotlin
@Composable
fun TimerExample() {
    var time by remember { mutableStateOf(0) }
    
    // 当 key 变化时重启，无 key 则只运行一次
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            time++
        }
    }
    
    Text("Time: $time seconds")
}
```

### 2. rememberCoroutineScope - 响应事件的副作用
```kotlin
@Composable
fun EventHandlerExample() {
    val scope = rememberCoroutineScope()
    var data by remember { mutableStateOf("") }
    
    Column {
        Button(onClick = {
            scope.launch {
                // 处理点击事件
                data = fetchData()
            }
        }) {
            Text("Load Data")
        }
        Text(data)
    }
}

suspend fun fetchData(): String {
    delay(1000) // 模拟网络请求
    return "Loaded data"
}
```

### 3. DisposableEffect - 需要清理的副作用
```kotlin
@Composable
fun SensorExample() {
    val sensorManager = remember { 
        context.getSystemService(Context.SENSOR_SERVICE) as SensorManager 
    }
    
    DisposableEffect(Unit) {
        val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent?) {
                // 处理传感器数据
            }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }
        
        sensorManager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_NORMAL)
        
        onDispose {
            sensorManager.unregisterListener(listener) // 清理资源
        }
    }
}
```

### 4. SideEffect - 每次成功重组后执行
```kotlin
@Composable
fun AnalyticsExample(userId: String) {
    var screenName by remember { mutableStateOf("") }
    
    SideEffect {
        // 每次重组后记录 analytics
        analyticsTracker.trackScreenView(userId, screenName)
    }
    
    // UI 内容
}
```

### 5. produceState - 将异步数据流转换为状态
```kotlin
@Composable
fun UserProfile(userId: String) {
    val userState = produceState<User?>(initialValue = null, userId) {
        // 在后台协程中获取数据
        val user = userRepository.getUser(userId)
        value = user
    }
    
    if (userState.value == null) {
        CircularProgressIndicator()
    } else {
        Text("User: ${userState.value?.name}")
    }
}
```

### 6. derivedStateOf - 优化性能的派生状态
```kotlin
@Composable
fun TodoList(todos: List<Todo>) {
    val highPriorityVisible by remember {
        derivedStateOf {
            todos.any { it.priority == Priority.HIGH }
        }
    }
    
    if (highPriorityVisible) {
        Text("You have high priority tasks!")
    }
}
```

## 最佳实践

### 正确的状态提升
```kotlin
@Composable
fun Counter(
    count: Int,
    onIncrement: () -> Unit
) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}

@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }
    
    Counter(
        count = count,
        onIncrement = { count++ }
    )
}
```

### 使用 ViewModel 处理业务逻辑
```kotlin
class UserViewModel : ViewModel() {
    private val _users = mutableStateOf<List<User>>(emptyList())
    val users: State<List<User>> = _users
    
    init {
        viewModelScope.launch {
            _users.value = userRepository.getUsers()
        }
    }
}

@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val users by viewModel.users
    
    LazyColumn {
        items(users) { user ->
            UserItem(user = user)
        }
    }
}
```

## 总结

Jetpack Compose 提供了一套完整的副作用处理 API，关键点：
- **LaunchedEffect**: 协程作用域内的副作用
- **rememberCoroutineScope**: 事件触发的副作用  
- **DisposableEffect**: 需要资源清理的副作用
- **SideEffect**: 每次重组后的同步副作用
- 始终将状态提升到合适的层级
- 使用 ViewModel 处理业务逻辑

这些工具帮助我们在声明式 UI 中安全地管理副作用，确保应用的可预测性和稳定性。