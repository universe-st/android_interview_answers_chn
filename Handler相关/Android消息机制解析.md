# Android 消息机制的原理及源码解析

Android消息机制是实现线程间通信和任务调度的核心框架，其核心组件包括 **Handler**（消息发送与处理）、**Looper**（消息循环驱动）、**MessageQueue**（消息队列管理）和 **Message**（任务载体）。以下从原理概述和源码解析两方面展开说明：


### 一、核心原理概述
#### 1. 组件作用与协作流程
- **Handler**：负责发送消息（`sendMessage`/`post`）和处理消息（`handleMessage`）。发送消息时，将任务包装为 `Message` 并插入队列；处理消息时，根据 `Message` 的来源（`target` 字段）回调对应方法。
- **Looper**：绑定线程的消息循环器，通过 `loop()` 方法不断从 `MessageQueue` 中取出消息并分发。每个线程最多有一个 `Looper`，通过 `ThreadLocal` 实现线程隔离。
- **MessageQueue**：存储消息的单链表，按消息触发时间（`when` 字段）排序。支持插入、删除和阻塞式获取消息，内部通过 `nativePollOnce` 实现线程休眠与唤醒。
- **Message**：任务载体，包含 `callback`（`Runnable`）、`target`（`Handler`）、`when`（触发时间）等字段。通过消息池复用机制（`obtain()`/`recycle()`）减少内存开销。

协作流程可概括为：
```
Handler 发送消息 → MessageQueue 入队（按 when 排序） → Looper 循环取出消息 → Handler 处理消息
```

#### 2. 线程绑定与初始化
- **Looper 绑定线程**：通过 `Looper.prepare()` 方法创建 `Looper` 实例，并通过 `ThreadLocal` 存储到当前线程。主线程的 `Looper` 由 `ActivityThread.main()` 自动初始化（`prepareMainLooper()`）。
- **Handler 关联 Looper**：`Handler` 构造时默认获取当前线程的 `Looper`（`Looper.myLooper()`），若未绑定则抛出异常。子线程需手动调用 `Looper.prepare()` 和 `Looper.loop()` 才能使用 `Handler`。

#### 3. 消息发送与队列管理
- **消息包装**：`post(Runnable)` 方法将 `Runnable` 包装为 `Message`，赋值给 `Message.callback`；`sendMessage(Message)` 直接使用 `Message` 对象。
- **队列插入**：`MessageQueue.enqueueMessage()` 根据 `when` 字段将消息插入链表的正确位置。若新消息触发时间更早或为异步消息（同步屏障场景），则唤醒 `Looper` 立即处理。
- **阻塞与唤醒**：`MessageQueue.next()` 通过 `nativePollOnce` 阻塞线程，直到消息触发时间到达或有新消息插入。同步屏障会阻塞同步消息，优先处理异步消息（如UI绘制事件）。

#### 4. 消息执行与复用
- **消息分发**：`Looper.loop()` 取出消息后，调用 `Handler.dispatchMessage()`。若 `Message.callback` 存在（`post` 场景），直接执行 `Runnable.run()`；否则回调 `Handler.handleMessage()`。
- **消息池复用**：`Message.obtain()` 从消息池（最多缓存50个）获取复用对象，`recycle()` 方法将消息重置并归还池，避免频繁创建对象。


### 二、源码解析关键实现
#### 1. Looper 初始化与循环
- **`Looper.prepare()`**：
  ```java
  public static void prepare() {
      prepare(true);
  }
  private static void prepare(boolean quitAllowed) {
      if (sThreadLocal.get() != null) {
          throw new RuntimeException("Only one Looper may be created per thread");
      }
      sThreadLocal.set(new Looper(quitAllowed)); // 存储到 ThreadLocal
  }
  ```
  - 通过 `ThreadLocal` 确保每个线程唯一 `Looper`。

- **`Looper.loop()`**：
  ```java
  public static void loop() {
      final Looper me = myLooper();
      final MessageQueue queue = me.mQueue;
      for (;;) {
          Message msg = queue.next(); // 阻塞获取消息
          if (msg == null) return; // quit() 调用后退出循环
          msg.target.dispatchMessage(msg); // 分发消息
          msg.recycleUnchecked(); // 回收消息到池
      }
  }
  ```
  - 无限循环处理消息，直到 `quit()` 被调用。

#### 2. MessageQueue 核心方法
- **`enqueueMessage()`**：
  ```java
  boolean enqueueMessage(Message msg, long when) {
      synchronized (this) {
          // 插入链表逻辑（按 when 排序）
          if (p == null || when < p.when) {
              msg.next = p;
              mMessages = msg;
              needWake = mBlocked; // 若当前阻塞，唤醒 Looper
          } else {
              // 遍历链表找到插入位置
              while (p != null && p.when <= when) {
                  prev = p;
                  p = p.next;
              }
              prev.next = msg;
              msg.next = p;
          }
          if (needWake) nativeWake(mPtr); // 唤醒线程
      }
      return true;
  }
  ```
  - 通过同步锁保证线程安全，根据 `when` 维护有序链表。

- **`next()`**：
  ```java
  Message next() {
      int nextPollTimeoutMillis = 0;
      for (;;) {
          nativePollOnce(mPtr, nextPollTimeoutMillis); // 阻塞线程
          synchronized (this) {
              // 检查队列头部消息
              final long now = SystemClock.uptimeMillis();
              Message msg = mMessages;
              if (msg != null && msg.target == null) { // 同步屏障
                  msg = getAsyncMessage(); // 查找异步消息
              }
              if (msg != null) {
                  if (now < msg.when) {
                      nextPollTimeoutMillis = (int) (msg.when - now); // 计算阻塞时间
                  } else {
                      mMessages = msg.next; // 取出消息
                      return msg;
                  }
              } else {
                  nextPollTimeoutMillis = -1; // 永久阻塞直到新消息插入
              }
          }
      }
  }
  ```
  - 通过 `nativePollOnce` 实现阻塞，减少CPU空转。同步屏障场景下优先处理异步消息。

#### 3. Handler 消息发送与分发
- **`post(Runnable)`**：
  ```java
  public final boolean post(Runnable r) {
      return sendMessageDelayed(getPostMessage(r), 0);
  }
  private static Message getPostMessage(Runnable r) {
      Message m = Message.obtain();
      m.callback = r; // Runnable 赋值给 callback
      return m;
  }
  ```
  - 将 `Runnable` 包装为 `Message`，调用 `sendMessageDelayed` 发送。

- **`dispatchMessage(Message msg)`**：
  ```java
  public void dispatchMessage(Message msg) {
      if (msg.callback != null) { // post 场景
          handleCallback(msg); // 执行 Runnable.run()
      } else if (mCallback != null) { // 自定义回调
          if (mCallback.handleMessage(msg)) return;
      }
      handleMessage(msg); // sendMessage 场景
  }
  private static void handleCallback(Message message) {
      message.callback.run(); // 执行 Runnable
  }
  ```
  - 按优先级处理消息：`callback` → `mCallback` → `handleMessage`。


### 三、高级特性与优化机制
#### 1. 同步屏障（Sync Barrier）
- **作用**：临时阻塞同步消息，优先处理异步消息（如UI绘制、触摸事件）。
- **实现**：
  ```java
  // 插入屏障（隐藏API）
  private int postSyncBarrier() {
      synchronized (this) {
          final int token = mNextBarrierToken++;
          Message msg = Message.obtain();
          msg.target = null; // target 为 null 表示屏障
          msg.when = SystemClock.uptimeMillis();
          Message prev = null;
          Message p = mMessages;
          if (p == null || p.when > msg.when) {
              msg.next = p;
              mMessages = msg;
          } else {
              // 插入到合适位置
          }
          return token;
      }
  }
  ```
  - 屏障消息 `target` 为 `null`，`next()` 方法遇到屏障时跳过同步消息，寻找异步消息。

#### 2. 消息池复用
- **机制**：通过 `Message.obtain()` 从缓存池获取 `Message`，避免频繁创建对象。
  ```java
  // Message.java
  private static Message sPool;
  public static Message obtain() {
      synchronized (sPoolSync) {
          if (sPool != null) {
              Message m = sPool;
              sPool = m.next;
              m.next = null;
              m.flags = 0; // 重置状态
              return m;
          }
      }
      return new Message(); // 池空时新建
  }
  ```
  - 消息执行后通过 `recycle()` 归还池，最大缓存50个。

#### 3. 内存泄漏防范
- **问题**：非静态内部类 `Handler` 持有外部类（如 `Activity`）引用，导致泄漏。
- **解决方案**：
  ```java
  static class WeakHandler extends Handler {
      private final WeakReference<Activity> mActivityRef;
      public WeakHandler(Activity activity) {
          mActivityRef = new WeakReference<>(activity);
      }
      @Override
      public void handleMessage(Message msg) {
          Activity activity = mActivityRef.get();
          if (activity != null) {
              // 处理消息
          }
      }
  }
  ```
  - 使用弱引用避免强引用链。


### 四、典型场景与最佳实践
1. **主线程更新UI**：
   ```java
   // 子线程发送消息到主线程 Handler
   new Thread(() -> {
       // 耗时操作
       runOnUiThread(() -> textView.setText("更新内容")); // 本质调用主线程 Handler.post
   }).start();
   ```

2. **延迟任务**：
   ```java
   handler.postDelayed(() -> {
       // 延迟执行
   }, 1000);
   ```

3. **异步消息优先级提升**：
   ```java
   // 插入同步屏障，优先处理异步消息
   MessageQueue queue = Looper.myQueue();
   int token = queue.postSyncBarrier();
   try {
       // 发送异步消息
       Message msg = Message.obtain();
       msg.isAsynchronous = true;
       handler.sendMessage(msg);
   } finally {
       queue.removeSyncBarrier(token); // 必须成对移除
   }
   ```


### 五、总结
Android消息机制通过 **Handler发送消息**、**MessageQueue管理队列**、**Looper驱动循环** 和 **Message承载任务** 实现线程间通信与任务调度。其核心设计包括：
- **线程隔离性**：通过 `ThreadLocal` 确保每个线程唯一 `Looper`。
- **消息优先级调度**：同步屏障优先处理异步消息，保障UI流畅。
- **性能优化**：消息池复用减少内存开销，阻塞唤醒机制降低CPU占用。

理解该机制有助于正确使用 `Handler` 进行线程通信，避免内存泄漏和ANR，并优化复杂场景下的任务调度（如动画、输入响应）。