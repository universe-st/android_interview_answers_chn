# 什么是BroadcastReceiver？它的主要作用是什么？

在Android开发中，**BroadcastReceiver（广播接收器）** 是Android四大组件之一，它是一种用于接收和处理“广播事件”的组件。


### 核心概念：
广播事件（Broadcast）是Android系统中一种跨组件、跨应用的通信机制，类似于“事件通知”。当系统或应用发生某些特定事件（如网络状态变化、电量低、应用安装/卸载等）时，会发送一条广播，而`BroadcastReceiver`则负责“监听”并“响应”这些广播。


### 主要作用：
1. **接收系统事件通知**  
   响应系统级别的状态变化，例如：  
   - 网络连接状态改变（Wi-Fi/移动数据切换）  
   - 电池电量过低或充电状态变化  
   - 屏幕点亮/熄灭、设备重启  
   - 时区变化、语言设置修改等  

2. **实现跨组件/跨应用通信**  
   - 同一应用内的不同组件（如Activity、Service）可以通过发送广播传递消息。  
   - 不同应用之间也能通过广播进行数据交互（需注意权限控制，避免安全问题）。  

3. **触发特定业务逻辑**  
   当接收到指定广播时，执行预设操作，例如：  
   - 网络恢复后自动同步数据  
   - 收到推送消息时显示通知  
   - 设备启动完成后自动启动后台服务  


### 特点：
- **生命周期短暂**：`BroadcastReceiver`的`onReceive()`方法执行完毕后，组件会被系统回收，不宜执行耗时操作（否则可能导致ANR）。  
- **两种注册方式**：  
  - 静态注册：在`AndroidManifest.xml`中声明，应用未启动时也能接收广播（部分系统广播已限制此方式）。  
  - 动态注册：在代码中通过`registerReceiver()`注册，需在组件生命周期内（如`onDestroy()`）调用`unregisterReceiver()`注销，仅在应用运行时有效。  


总之，`BroadcastReceiver`是Android中实现“事件驱动”和“跨组件通信”的重要机制，尤其适合处理系统状态变化和松耦合的消息传递场景。

# 如何使用BroadcastReceiver？

使用`BroadcastReceiver`通常分为**创建接收器**、**注册接收器**（指定接收哪些广播）、**发送广播**三个核心步骤。以下是详细的使用方法及示例：


### 一、创建BroadcastReceiver子类
首先需要自定义一个类继承`BroadcastReceiver`，并重写`onReceive(Context, Intent)`方法——这是接收广播后执行逻辑的核心方法。

```java
public class MyBroadcastReceiver extends BroadcastReceiver {
    // 接收广播时触发
    @Override
    public void onReceive(Context context, Intent intent) {
        // 从Intent中获取广播携带的数据
        String action = intent.getAction();
        if (action != null) {
            switch (action) {
                case "com.example.MY_CUSTOM_ACTION":
                    // 处理自定义广播
                    String message = intent.getStringExtra("key");
                    Toast.makeText(context, "收到自定义广播：" + message, Toast.LENGTH_SHORT).show();
                    break;
                case ConnectivityManager.CONNECTIVITY_ACTION:
                    // 处理网络状态变化广播（需注意：Android 7.0后部分系统广播限制静态注册）
                    checkNetwork(context);
                    break;
            }
        }
    }

    // 示例：检查网络状态
    private void checkNetwork(Context context) {
        ConnectivityManager cm = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        NetworkInfo activeNetwork = cm.getActiveNetworkInfo();
        boolean isConnected = activeNetwork != null && activeNetwork.isConnectedOrConnecting();
        Toast.makeText(context, isConnected ? "网络已连接" : "网络已断开", Toast.LENGTH_SHORT).show();
    }
}
```

**注意**：  
- `onReceive()`方法运行在主线程，执行时间不能超过10秒，否则会触发ANR（应用无响应）。  
- 若需执行耗时操作，应通过`IntentService`或`JobScheduler`等方式处理。  


### 二、注册BroadcastReceiver
注册的目的是告诉系统：“我的接收器要监听哪些广播”。分为**静态注册**和**动态注册**两种方式。


#### 1. 静态注册（在AndroidManifest.xml中声明）
在清单文件中注册，应用**未启动时也能接收广播**（但Android 8.0+对大部分系统广播限制了静态注册，需注意）。

```xml
<manifest ...>
    <application ...>
        <!-- 声明BroadcastReceiver -->
        <receiver
            android:name=".MyBroadcastReceiver"
            android:enabled="true"  <!-- 是否启用 -->
            android:exported="true"> <!-- 是否允许接收其他应用的广播 -->
            <!-- 配置要接收的广播类型（通过action过滤） -->
            <intent-filter>
                <!-- 自定义广播action -->
                <action android:name="com.example.MY_CUSTOM_ACTION" />
                <!-- 系统广播action（如网络状态变化，需申请权限） -->
                <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
            </intent-filter>
        </receiver>
    </application>

    <!-- 接收网络状态广播需声明权限 -->
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
</manifest>
```

**限制**：  
- Android 8.0（API 26）后，大部分系统广播（如`CONNECTIVITY_ACTION`）不再支持静态注册，必须动态注册。  
- 静态注册的接收器生命周期由系统管理，不受应用组件生命周期影响。  


#### 2. 动态注册（在代码中注册）
在Activity/Fragment/Service等组件中通过代码注册，**仅在应用运行时有效**，需手动注销以避免内存泄漏。

```java
public class MainActivity extends AppCompatActivity {
    private MyBroadcastReceiver myReceiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 1. 实例化接收器
        myReceiver = new MyBroadcastReceiver();

        // 2. 创建IntentFilter，指定要接收的广播action
        IntentFilter filter = new IntentFilter();
        filter.addAction("com.example.MY_CUSTOM_ACTION"); // 自定义广播
        filter.addAction(ConnectivityManager.CONNECTIVITY_ACTION); // 系统广播

        // 3. 注册接收器（上下文、接收器、过滤器）
        registerReceiver(myReceiver, filter);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 4. 必须注销接收器，否则会导致内存泄漏
        if (myReceiver != null) {
            unregisterReceiver(myReceiver);
        }
    }
}
```

**特点**：  
- 灵活性高，可根据业务需求动态开关监听（如在`onStart()`注册，`onStop()`注销）。  
- 不受Android 8.0+静态注册限制，适合接收系统广播。  


### 三、发送广播
通过`Context.sendBroadcast()`等方法发送广播，可携带数据供接收器处理。


#### 1. 发送普通广播（无序广播）
所有接收器同时接收，无法拦截或修改数据。

```java
// 在Activity中发送自定义广播
Intent intent = new Intent("com.example.MY_CUSTOM_ACTION");
intent.putExtra("key", "这是一条自定义广播"); // 携带数据
sendBroadcast(intent); // 发送广播
```


#### 2. 发送有序广播
接收器按优先级依次接收，高优先级接收器可拦截广播或修改数据。

```java
// 发送有序广播（需权限，可选）
Intent intent = new Intent("com.example.ORDERED_ACTION");
sendOrderedBroadcast(
    intent, 
    "com.example.PERMISSION", // 接收者需声明的权限（可选）
    null, // 结果接收器（最后接收，即使被拦截）
    null, // Handler（指定回调线程）
    Activity.RESULT_OK, // 初始代码
    "初始数据", // 初始数据
    null // 额外数据
);
```

在有序广播中，高优先级接收器可通过`setResultData()`修改数据，低优先级接收器通过`getResultData()`获取；也可通过`abortBroadcast()`拦截广播，后续接收器将无法接收。


#### 3. 发送本地广播（仅应用内可见）
通过`LocalBroadcastManager`发送的广播仅在当前应用内传播，安全性高（避免其他应用监听）。  
**注意**：AndroidX中已推荐使用`LiveData`或`ViewModel`替代，若需使用可引入相关依赖。

```java
// 获取LocalBroadcastManager实例
LocalBroadcastManager localManager = LocalBroadcastManager.getInstance(this);

// 发送本地广播
Intent intent = new Intent("com.example.LOCAL_ACTION");
localManager.sendBroadcast(intent);

// 注册本地广播接收器（需用localManager注册）
localManager.registerReceiver(myReceiver, filter);
```


### 四、注意事项
1. **权限控制**：  
   - 发送敏感广播时，可通过`sendBroadcast(intent, permission)`要求接收者必须声明特定权限。  
   - 接收外部广播时，可在`AndroidManifest.xml`的`<receiver>`中添加`android:permission`，限制只有拥有该权限的应用才能发送广播给它。

2. **避免ANR**：`onReceive()`中严禁执行耗时操作，如需网络请求、数据库读写等，应启动Service处理。

3. **静态注册限制**：Android 8.0+对系统广播的静态注册限制严格，建议优先使用动态注册。

4. **安全性**：自定义广播建议使用应用包名作为action前缀（如`com.example.app.ACTION_X`），避免与其他应用冲突；敏感数据传递优先用本地广播。


通过以上步骤，即可完成`BroadcastReceiver`的创建、注册和使用，实现跨组件或跨应用的事件通信。