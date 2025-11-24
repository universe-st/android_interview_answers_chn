# 如何在 Compose 中实现导航？

在 Jetpack Compose 中实现导航，官方推荐使用 **Jetpack Navigation Compose** 库（基于 Jetpack Navigation 框架适配 Compose）。它提供了类型安全的导航管理、返回栈控制、参数传递等核心功能，能轻松处理单页面内的多屏幕切换、深层链接等场景。


### 核心概念
在使用前，先了解几个核心概念：
- **`NavController`**：导航控制器，管理导航状态（如返回栈、当前屏幕），是导航的核心入口。
- **`NavHost`**：导航容器，负责承载所有“目的地”（屏幕），并根据 `NavController` 的状态显示对应的屏幕。
- **`Destination`**：导航目的地，即单个屏幕（Composable 函数），通过唯一的 `route`（路由）标识。
- **`NavGraph`**：导航图，定义所有目的地及其关系（如起始页、跳转规则）。


### 实现步骤

#### 1. 添加依赖
在 `app/build.gradle` 中添加 Navigation Compose 依赖（确保版本与 Compose 兼容，可参考 [官方文档](https://developer.android.com/jetpack/compose/navigation) 获取最新版本）：
```kotlin
dependencies {
    implementation "androidx.navigation:navigation-compose:2.7.7"
}
```


#### 2. 创建 `NavController`
`NavController` 是导航的核心，需在组件树的合适层级（通常是顶层）创建并持有，使用 `rememberNavController()` 确保其在重组时保持状态：
```kotlin
@Composable
fun MyApp() {
    // 创建并记住 NavController
    val navController = rememberNavController()
    
    // 整个应用的内容（包含导航容器）
    NavHost(
        navController = navController,
        startDestination = "home" // 起始目的地的路由
    ) {
        // 定义导航图（所有目的地）
        // ...
    }
}
```


#### 3. 定义导航目的地（屏幕）
每个屏幕是一个 `@Composable` 函数，需在 `NavHost` 的导航图中通过 `composable(route)` 注册，`route` 是该屏幕的唯一标识（如 `"home"`、`"detail"`）。

**示例：定义两个屏幕**
```kotlin
// 首页
@Composable
fun HomeScreen(navController: NavController) {
    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("Home Screen")
        Button(
            onClick = { 
                // 跳转到详情页（后面会讲导航逻辑）
                navController.navigate("detail") 
            }
        ) {
            Text("Go to Detail")
        }
    }
}

// 详情页
@Composable
fun DetailScreen(navController: NavController) {
    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("Detail Screen")
        Button(
            onClick = { 
                // 返回上一页
                navController.popBackStack() 
            }
        ) {
            Text("Go Back")
        }
    }
}
```


#### 4. 配置导航图（`NavHost`）
在 `NavHost` 中通过 `composable` 函数注册所有目的地，并关联对应的 `Composable` 屏幕：
```kotlin
@Composable
fun MyApp() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = "home" // 启动时显示首页
    ) {
        // 注册首页：route 为 "home"，关联 HomeScreen
        composable(route = "home") {
            HomeScreen(navController = navController) // 传递 navController 用于导航
        }
        
        // 注册详情页：route 为 "detail"，关联 DetailScreen
        composable(route = "detail") {
            DetailScreen(navController = navController)
        }
    }
}
```

此时，运行应用会显示 `HomeScreen`，点击按钮可跳转到 `DetailScreen`，点击返回按钮可回到首页——基础导航已实现。


### 进阶功能

#### 1. 传递参数
实际开发中，常需要在导航时传递参数（如从列表页传递 `id` 到详情页）。可通过在 `route` 中定义参数占位符（如 `detail/{itemId}`），并在目标屏幕中解析。

**步骤：**  
- 在目标路由中声明参数：`"detail/{itemId}"`（`itemId` 为参数名）。  
- 导航时传递具体值：`navController.navigate("detail/123")`。  
- 目标屏幕中通过 `backStackEntry.arguments` 解析参数。  


**示例：带参数的导航**
```kotlin
// 1. 定义带参数的详情页路由
composable(route = "detail/{itemId}") { backStackEntry ->
    // 解析参数（itemId 是字符串，需手动转换类型）
    val itemId = backStackEntry.arguments?.getString("itemId") ?: "0"
    DetailScreenWithParam(
        navController = navController,
        itemId = itemId.toInt()
    )
}

// 带参数的详情页
@Composable
fun DetailScreenWithParam(navController: NavController, itemId: Int) {
    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("Detail for item: $itemId")
        Button(onClick = { navController.popBackStack() }) {
            Text("Go Back")
        }
    }
}

// 首页中跳转（传递参数 123）
HomeScreen(navController) {
    Button(onClick = { navController.navigate("detail/123") }) {
        Text("Go to Detail with Param")
    }
}
```


#### 2. 类型安全的参数传递（推荐）
上述方式需手动解析参数，容易出错。可通过 `NavType` 定义参数类型（如 `IntType`、`StringType`），并开启类型安全模式。

**示例：指定参数类型**
```kotlin
composable(
    route = "detail/{itemId}",
    arguments = listOf(
        navArgument("itemId") {
            type = NavType.IntType // 声明为 Int 类型
            defaultValue = 0 // 可选：设置默认值
        }
    )
) { backStackEntry ->
    // 直接获取 Int 类型参数
    val itemId = backStackEntry.arguments?.getInt("itemId") ?: 0
    DetailScreenWithParam(navController, itemId)
}
```


#### 3. 控制返回栈
有时需要清除返回栈（如登录后跳转到主页，不希望返回登录页），可通过 `navigate` 的配置实现：
```kotlin
// 登录后跳转到主页，并清除登录页之前的所有页面
navController.navigate("home") {
    // 清除到起始页（home）的所有页面（包含自身）
    popUpTo("login") { inclusive = true }
}
```


#### 4. 结合底部导航栏（`BottomNavigation`）
底部导航栏（如首页、分类、我的）是常见场景，可将 `BottomNavigation` 的选项与导航路由关联：

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    // 底部导航选项（路由 + 图标 + 文字）
    val items = listOf(
        "home" to Icons.Default.Home,
        "category" to Icons.Default.Category,
        "profile" to Icons.Default.Person
    )

    Scaffold(
        bottomBar = {
            BottomNavigation {
                // 获取当前路由（用于高亮选中项）
                val currentRoute = currentRoute(navController)
                items.forEach { (route, icon) ->
                    BottomNavigationItem(
                        icon = { Icon(icon, contentDescription = null) },
                        label = { Text(route) },
                        selected = currentRoute == route,
                        onClick = {
                            // 点击时导航到对应路由，避免重复点击同一页
                            if (currentRoute != route) {
                                navController.navigate(route) {
                                    // 重新选择时，清除该路由之上的所有页面
                                    popUpTo(navController.graph.startDestinationId) {
                                        saveState = true
                                    }
                                    // 避免添加重复的目的地到返回栈
                                    launchSingleTop = true
                                    // 恢复之前的状态（如滚动位置）
                                    restoreState = true
                                }
                            }
                        }
                    )
                }
            }
        }
    ) { innerPadding ->
        // 导航容器（使用内边距避免被底部导航遮挡）
        NavHost(
            navController = navController,
            startDestination = "home",
            modifier = Modifier.padding(innerPadding)
        ) {
            composable("home") { HomeScreen() }
            composable("category") { CategoryScreen() }
            composable("profile") { ProfileScreen() }
        }
    }
}

// 辅助函数：获取当前路由
@Composable
private fun currentRoute(navController: NavController): String? {
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    return navBackStackEntry?.destination?.route
}
```


#### 5. 深层链接（Deep Link）
支持从外部（如通知、网页）直接跳转到 App 内的指定屏幕，需在 `composable` 中配置 `deepLinks`：

```kotlin
composable(
    route = "detail/{itemId}",
    deepLinks = listOf(
        navDeepLink {
            // 匹配的 URI（如 "myapp://example.com/detail/123"）
            uriPattern = "myapp://example.com/detail/{itemId}"
        }
    )
) { backStackEntry ->
    val itemId = backStackEntry.arguments?.getInt("itemId") ?: 0
    DetailScreenWithParam(navController, itemId)
}
```

同时需在 `AndroidManifest.xml` 中注册该链接（让系统知道 App 可处理该 URI）：
```xml
<activity
    android:name=".MainActivity"
    ...>
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- 匹配深层链接的 URI -->
        <data
            android:scheme="myapp"
            android:host="example.com"
            android:pathPattern="/detail/.*" />
    </intent-filter>
</activity>
```


### 注意事项
1. **`NavController` 的作用域**：应在需要管理导航的最高层级创建（如 `MyApp` 组件），避免在子组件中重复创建。  
2. **路由命名规范**：路由应唯一且清晰（如包含模块名：`"user/profile"`），避免冲突。  
3. **避免过度传递 `NavController`**：深层组件如需导航，可通过 `CompositionLocal` 隐式传递 `NavController`，减少参数钻取。  
4. **状态保存**：结合 `rememberSaveable` 保存屏幕状态（如输入框内容），避免导航返回时状态丢失。  


### 总结
Jetpack Navigation Compose 是 Compose 中实现导航的标准方案，通过 `NavController` 管理状态、`NavHost` 承载目的地、`route` 标识屏幕，可轻松实现基础导航、参数传递、底部导航等场景。核心是理解“导航图”的配置和 `NavController` 的操作方法，结合官方最佳实践即可高效实现导航功能。