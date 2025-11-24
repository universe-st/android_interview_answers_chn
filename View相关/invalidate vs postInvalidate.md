# invalidate() 和 postInvalicate() 区别

在Android开发中，`invalidate()` 和 `postInvalidate()` 都是用于触发View重绘的方法，但它们的核心区别在于**调用线程限制**和**实现机制**，具体如下：


### 1. 调用线程限制不同
- **`invalidate()`**：**必须在UI线程（主线程）中调用**。  
  它直接标记当前View为“无效状态”，并触发View树的重绘流程（最终会调用`onDraw()`方法）。如果在子线程中调用，会抛出`CalledFromWrongThreadException`异常。

- **`postInvalidate()`**：**可以在非UI线程（子线程）中调用**。  
  它内部通过`Handler`将重绘请求发送到UI线程的消息队列，间接触发`invalidate()`，从而避免了线程安全问题。


### 2. 实现机制不同
- **`invalidate()`**：属于同步操作，直接作用于当前线程，标记View为无效后，系统会在合适的时机（通常是当前消息循环结束时）统一处理重绘。

- **`postInvalidate()`**：属于异步操作，通过`Handler.post()`将重绘任务投递到UI线程的消息队列，等待UI线程调度执行，本质上是对“子线程通知UI线程重绘”的封装。


### 3. 重载方法与扩展功能
两者都有重载方法，支持指定重绘区域（减少不必要的绘制，优化性能）：
- `invalidate(Rect dirty)` / `invalidate(int l, int t, int r, int b)`：仅重绘指定区域。
- `postInvalidate(int left, int top, int right, int bottom)`：在子线程中指定重绘区域。
- `postInvalidateDelayed(long delayMillis)`：延迟指定时间后在子线程中触发重绘。


### 总结
| 特性                | `invalidate()`               | `postInvalidate()`           |
|---------------------|------------------------------|------------------------------|
| 调用线程            | 仅UI线程                     | 可在子线程（内部切换到UI线程）|
| 实现方式            | 直接标记无效，同步触发重绘流程 | 通过Handler异步投递到UI线程  |
| 作用                | 触发View重绘                 | 触发View重绘                 |


**使用场景**：  
- 在UI线程中更新View后（如修改了颜色、文本等），直接调用`invalidate()`。  
- 在子线程中处理数据后需要更新UI时，使用`postInvalidate()`（或通过`Handler`/`runOnUiThread`切换到UI线程后调用`invalidate()`）。