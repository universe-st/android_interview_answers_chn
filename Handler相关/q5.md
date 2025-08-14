# Handler postDelay方法的原理？

`Handler.postDelayed(Runnable r, long delayMillis)` 方法用于**延迟指定时间后执行任务**，其核心原理是基于 `Handler` 消息机制，通过设置 `Message` 的触发时间（`when` 字段），让消息队列（`MessageQueue`）在指定时间后才将消息分发给 `Handler` 处理。


### 一、`postDelayed` 方法的完整流程
#### 1. 包装 `Runnable` 为 `Message`
与 `post()` 方法类似，`postDelayed` 会先将 `Runnable` 任务包装成 `Message` 对象，将 `Runnable` 赋值给 `Message` 的 `callback` 字段（用于后续执行）。

源码中通过 `getPostMessage(Runnable r)` 完成包装：
```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r; // Runnable 存储在 Message 的 callback 中
    return m;
}
```


#### 2. 计算消息的触发时间（`when` 字段）
`postDelayed` 的核心是设置消息的延迟执行时间。方法会计算**当前时间 + 延迟毫秒数**，作为消息的触发时间（`when` 字段，单位：毫秒，基于系统开机时间，`SystemClock.uptimeMillis()`）。

源码中 `postDelayed` 的实现：
```java
public final boolean postDelayed(Runnable r, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0; // 延迟时间不能为负，否则按 0 处理
    }
    // 计算触发时间：当前时间 + 延迟时间
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}

// sendMessageDelayed 进一步处理
public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    // 核心：计算 when = 当前时间 + 延迟时间
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

这里的 `SystemClock.uptimeMillis()` 是系统开机到现在的毫秒数（不包含深度睡眠的时间），确保时间计算不受系统时钟调整的影响。


#### 3. 将消息插入 `MessageQueue` 并按 `when` 排序
`sendMessageAtTime` 会调用 `MessageQueue.enqueueMessage()` 方法，将消息插入消息队列。`MessageQueue` 是按 `when` 字段排序的**单链表**，新消息会被插入到“第一个 `when` 大于当前消息 `when`”的位置，确保队列始终按执行时间从小到大排列。

例如：若队列中已有消息的 `when` 为 1000ms、3000ms，新消息的 `when` 为 2000ms，则新消息会插入到两者之间。


#### 4. `Looper` 循环等待并处理消息
`Looper.loop()` 会不断从 `MessageQueue` 中获取消息，核心逻辑如下：
- 调用 `MessageQueue.next()` 方法获取下一个需要执行的消息。
- `next()` 方法会检查队列头部的消息：
  - 若消息的 `when` 小于等于当前时间（已到执行时间），则取出该消息并返回。
  - 若消息的 `when` 大于当前时间（未到执行时间），则通过 `nativePollOnce()` 方法**阻塞等待**（阻塞时间 = 消息 `when` - 当前时间），避免CPU空转。
- 当阻塞时间结束（或有新消息插入唤醒），`next()` 返回消息，`Looper` 调用 `Handler.dispatchMessage()` 处理消息，最终执行 `Runnable.run()`。


### 二、关键细节：延迟的“不精确性”
`postDelayed` 的延迟时间并非绝对精确，可能存在偏差，原因如下：
1. **消息队列阻塞**：若队列中存在更早执行的消息（`when` 更小），且该消息的处理耗时较长，会推迟后续消息的执行时间。
   
   例如：队列中先有一个 `when=1000ms` 但执行耗时 2000ms 的消息，那么 `when=2000ms` 的消息会被推迟到 3000ms 才执行。

2. **系统调度延迟**：`nativePollOnce()` 的阻塞唤醒可能受系统进程调度影响，存在微小偏差。

3. **新消息插入**：若在阻塞等待期间插入了一个 `when` 更小的新消息，`MessageQueue` 会唤醒阻塞，优先处理新消息，可能推迟原消息的执行。


### 三、与 `post()` 的核心区别
- `post(Runnable r)`：等价于 `postDelayed(r, 0)`，消息的 `when` 为当前时间，会被尽快处理（但仍需排队等待前面的消息）。
- `postDelayed(Runnable r, long delayMillis)`：消息的 `when` 为“当前时间 + 延迟时间”，会在指定时间后才可能被处理。


### 总结
`Handler.postDelayed` 的原理可概括为：  
**将 `Runnable` 包装成 `Message`，设置其 `when` 为“当前时间 + 延迟时间”，插入按 `when` 排序的消息队列；`Looper` 循环时会阻塞等待到 `when` 时间，再取出消息并执行 `Runnable`**。  

其延迟机制依赖于 `Message` 的 `when` 字段和 `MessageQueue` 的排序+阻塞逻辑，是 Android 中实现延迟任务的基础方式（如倒计时、定时刷新等）。