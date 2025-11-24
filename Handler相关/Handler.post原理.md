# Handler post方法的原理？

在Android中，`Handler.post(Runnable)` 方法是一种简化的线程通信方式，其本质是基于 `Handler` 的消息机制实现的，核心原理是**将 `Runnable` 任务包装成 `Message` 消息，通过消息循环机制在 `Handler` 绑定的线程中执行**。


### 一、`post` 方法的核心流程
`post` 方法的工作步骤可以概括为：**包装任务 → 发送消息 → 循环处理 → 执行任务**，具体如下：


#### 1. 将 `Runnable` 包装为 `Message`
当调用 `handler.post(runnable)` 时，`Handler` 会先创建一个 `Message` 对象，并将 `Runnable` 赋值给 `Message` 的 `callback` 字段（`Message` 中专门用于存储 `Runnable` 的属性）。

源码中通过 `getPostMessage(Runnable r)` 方法完成包装：
```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain(); // 从消息池获取复用的Message
    m.callback = r; // 将Runnable赋值给Message的callback
    return m;
}
```


#### 2. 发送消息到 `MessageQueue`
包装完成后，`Handler` 会调用 `sendMessageDelayed` 方法将该 `Message` 发送到消息队列（`MessageQueue`），延迟时间为 `0`（表示“尽快执行”）。

`post` 方法的源码本质是调用消息发送方法：
```java
public final boolean post(Runnable r) {
    // 等价于 sendMessageDelayed(getPostMessage(r), 0)
    return sendMessageDelayed(getPostMessage(r), 0);
}
```

其他重载方法（如 `postDelayed`、`postAtTime`）的原理类似，只是指定了不同的延迟时间（`when` 字段）。


#### 3. Looper 循环取出消息并分发
`Looper` 会不断从 `MessageQueue` 中取出消息，当取出通过 `post` 方法发送的 `Message` 时，会调用 `Handler.dispatchMessage(Message msg)` 方法进行分发。


#### 4. 执行 `Runnable` 的 `run()` 方法
在 `dispatchMessage` 方法中，会优先检查 `Message` 的 `callback`（即 `Runnable`）是否为 `null`：  
- 若不为 `null`（`post` 方法的情况），则直接调用 `handleCallback(msg)` 执行 `Runnable` 的 `run()` 方法；  
- 若为 `null`（`sendMessage` 方法的情况），则调用 `Handler.handleMessage(msg)` 处理消息。

`dispatchMessage` 源码逻辑：
```java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        // 处理post的Runnable
        handleCallback(msg);
    } else {
        // 处理sendMessage发送的消息（调用handleMessage）
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

// 执行Runnable的run()方法
private static void handleCallback(Message message) {
    message.callback.run();
}
```


### 二、`post` 方法的核心特点
1. **执行线程**：  
   `Runnable` 的 `run()` 方法会在 `Handler` 绑定的 `Looper` 所在线程执行（而非新线程）。例如：  
   - 主线程的 `Handler` 调用 `post`，`Runnable` 在主线程执行（可用于更新UI）；  
   - 子线程的 `Handler` 调用 `post`，`Runnable` 在该子线程执行。

2. **与 `sendMessage` 的区别**：  
   - `post(Runnable)`：无需重写 `handleMessage`，直接通过 `Runnable` 定义任务，更简洁；  
   - `sendMessage(Message)`：需要重写 `handleMessage` 处理消息，适合传递复杂数据（通过 `Message` 的 `what`、`obj` 等字段）。  

   两者本质都是发送消息，只是任务定义方式不同。

3. **非异步执行**：  
   `Runnable` 的 `run()` 是同步执行的，会阻塞当前 `Looper` 的消息循环（直到 `run()` 执行完成，才会处理下一个消息）。因此，`post` 方法中不应执行耗时操作（尤其是主线程的 `Handler`），否则会导致UI卡顿。


### 三、典型使用场景
- **主线程更新UI**：子线程执行耗时操作后，通过主线程 `Handler.post(Runnable)` 在主线程更新UI；  
  ```java
  // 主线程创建的Handler（绑定主线程Looper）
  Handler mainHandler = new Handler(Looper.getMainLooper());
  
  // 子线程中执行耗时操作后，通过post更新UI
  new Thread(() -> {
      // 耗时操作...
      mainHandler.post(() -> {
          // 在主线程更新UI
          textView.setText("更新内容");
      });
  }).start();
  ```

- **延迟执行任务**：通过 `postDelayed` 实现延迟操作（如延迟3秒后执行）；  
  ```java
  handler.postDelayed(() -> {
      // 3秒后执行的任务
  }, 3000);
  ```


### 总结
`Handler.post` 方法的本质是**将 `Runnable` 包装为 `Message` 消息，通过消息循环机制在 `Handler` 绑定的线程中执行**。它是 `Handler` 消息机制的简化接口，减少了直接操作 `Message` 的代码量，适合快速提交简单任务。理解其原理有助于正确使用 `Handler` 进行线程通信，并避免因误用导致的性能问题（如主线程执行耗时任务）。