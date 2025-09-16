# Android EventBus 面试问题及答案

## 基础概念

### 1. 什么是EventBus？
**答**：EventBus是一个基于发布/订阅模式的事件总线框架，用于Android应用中组件间的通信。它允许组件通过事件进行解耦通信，简化了组件之间的交互。

### 2. EventBus的主要优点是什么？
**答**：
- 简化组件间通信
- 实现代码解耦
- 避免复杂的接口和回调
- 支持线程切换
- 代码简洁易维护

### 3. EventBus的三要素是什么？
**答**：
- 事件(Event)：可以是任意类型的对象
- 发布者(Publisher)：通过post()方法发布事件
- 订阅者(Subscriber)：通过@Subscribe注解声明订阅方法

## 使用方式

### 4. 如何在项目中引入EventBus？
**答**：在Gradle文件中添加依赖：
```gradle
implementation 'org.greenrobot:eventbus:3.2.0'
```

### 5. 如何注册和注销EventBus？
**答**：
```java
// 注册
@Override
protected void onStart() {
    super.onStart();
    EventBus.getDefault().register(this);
}

// 注销
@Override
protected void onStop() {
    super.onStop();
    EventBus.getDefault().unregister(this);
}
```

### 6. 如何定义和发布一个事件？
**答**：
```java
// 定义事件类
public class MessageEvent {
    public final String message;
    public MessageEvent(String message) {
        this.message = message;
    }
}

// 发布事件
EventBus.getDefault().post(new MessageEvent("Hello EventBus!"));
```

### 7. 如何订阅事件？
**答**：
```java
@Subscribe(threadMode = ThreadMode.MAIN)
public void onMessageEvent(MessageEvent event) {
    // 处理事件
    textView.setText(event.message);
}
```

## 线程模式

### 8. EventBus有哪些线程模式？
**答**：
- **ThreadMode.POSTING**：在发布事件的同一线程执行（默认）
- **ThreadMode.MAIN**：在主线程（UI线程）执行
- **ThreadMode.MAIN_ORDERED**：在主线程顺序执行
- **ThreadMode.BACKGROUND**：在后台线程执行
- **ThreadMode.ASYNC**：在单独的异步线程执行

### 9. 不同线程模式的使用场景是什么？
**答**：
- POSTING：轻量级操作，不需要线程切换
- MAIN：更新UI操作
- BACKGROUND：耗时但不需要异步的操作
- ASYNC：网络请求、数据库读写等耗时操作

## 高级特性

### 10. 什么是粘性事件(Sticky Event)？
**答**：粘性事件是在订阅者注册之前发布的事件，订阅者注册后仍然可以接收到这些事件。

### 11. 如何发布和接收粘性事件？
**答**：
```java
// 发布粘性事件
EventBus.getDefault().postSticky(new MessageEvent("Sticky Message"));

// 接收粘性事件
@Subscribe(sticky = true, threadMode = ThreadMode.MAIN)
public void onMessageEvent(MessageEvent event) {
    // 处理事件
}
```

### 12. 如何移除粘性事件？
**答**：
```java
// 移除特定类型的粘性事件
MessageEvent stickyEvent = EventBus.getDefault().getStickyEvent(MessageEvent.class);
if (stickyEvent != null) {
    EventBus.getDefault().removeStickyEvent(stickyEvent);
}

// 移除所有粘性事件
EventBus.getDefault().removeAllStickyEvents();
```

### 13. EventBus如何设置优先级？
**答**：通过@Subscribe注解的priority参数设置：
```java
@Subscribe(priority = 1)
public void onHighPriorityEvent(MessageEvent event) {
    // 高优先级处理
}
```

### 14. 事件优先级的作用是什么？
**答**：高优先级的订阅者会先接收到事件，并且可以调用`EventBus.getDefault().cancelEventDelivery(event)`来取消事件传递，阻止低优先级的订阅者接收到该事件。

## 性能与优化

### 15. EventBus如何通过索引优化性能？
**答**：EventBus支持在编译时生成订阅者索引，避免运行时通过反射查找订阅方法，提高性能。

### 16. 如何配置EventBus索引？
**答**：在Gradle中配置：
```gradle
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [ eventBusIndex : 'com.example.MyEventBusIndex' ]
            }
        }
    }
}

dependencies {
    implementation 'org.greenrobot:eventbus:3.2.0'
    annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.2.0'
}
```

然后在Application中初始化：
```java
EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();
```

### 17. EventBus会导致内存泄漏吗？如何避免？
**答**：可能会，如果订阅者没有正确注销。避免方法：
- 在合适的生命周期方法中注册和注销
- 使用弱引用或自动管理订阅的库
- 避免在长时间存在的对象中订阅短生命周期的事件

## 与其他技术对比

### 18. EventBus与LocalBroadcastManager有什么区别？
**答**：
- EventBus更轻量级，性能更好
- EventBus支持类型安全的事件
- EventBus支持线程切换和粘性事件
- LocalBroadcastManager是Android系统组件，跨进程通信更安全

### 19. EventBus与RxJava的区别？
**答**：
- EventBus专注于事件总线模式，简单易用
- RxJava功能更强大，支持复杂的异步数据流处理
- RxJava学习曲线更陡峭，EventBus更简单直接

## 实践问题

### 20. 如何处理事件类型冲突？
**答**：使用不同的事件类或给事件类添加类型字段来区分不同类型的事件。

### 21. 如何在多个模块中使用EventBus？
**答**：可以创建多个EventBus实例，但通常建议使用默认的单例实例，通过良好设计的事件类来避免冲突。

### 22. EventBus在跨模块通信中的最佳实践是什么？
**答**：
- 定义清晰的事件类结构
- 使用接口或基类定义通用事件
- 在不同模块中定义特定的事件子类
- 使用包名来组织事件类

### 23. 如何调试EventBus事件流？
**答**：
- 启用EventBus的调试模式：`EventBus.builder().logNoSubscriberMessages(false).sendNoSubscriberEvent(false)`
- 添加日志记录订阅和发布操作
- 使用Android Studio的调试功能

### 24. EventBus在ProGuard中如何配置？
**答**：在proguard-rules.pro中添加：
```proguard
-keepattributes *Annotation*
-keepclassmembers class * {
    @org.greenrobot.eventbus.Subscribe <methods>;
}
-keep enum org.greenrobot.eventbus.ThreadMode { *; }

# 如果使用索引
-keep class com.example.MyEventBusIndex { *; }
```

### 25. EventBus的最新版本有哪些新特性？
**答**：最新版本通常包含性能优化、bug修复和新功能，如：
- 更好的索引支持
- 改进的线程处理
- 新的API简化使用
- 与最新Android版本的兼容性改进

### 26. 什么情况下不应该使用EventBus？
**答**：
- 简单的回调场景使用接口更合适
- 需要严格类型检查的场景
- 项目很小，组件间通信简单
- 需要跨进程通信的场景

