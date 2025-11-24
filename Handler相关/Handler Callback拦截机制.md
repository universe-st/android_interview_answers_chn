# Handler的Callback存在，但返回true，handleMessage是否会执行？

在 Android 中，当 Handler 的 `Callback` 存在且其 `handleMessage` 方法返回 `true` 时，**Handler 自身重写的 `handleMessage` 方法不会执行**。


### 原理说明：
Handler 处理消息的核心逻辑在 `dispatchMessage(Message msg)` 方法中，其执行顺序为：
1. 先检查是否存在 `Callback`（通过 `mCallback` 引用）。
2. 若存在 `Callback`，则调用 `Callback.handleMessage(msg)`：
   - 若返回 `true`：表示消息已被 `Callback` 处理完毕，**后续流程终止**（不会执行 Handler 自身的 `handleMessage`）。
   - 若返回 `false`：表示 `Callback` 未处理该消息，继续执行 Handler 自身的 `handleMessage(msg)`。


简单说：`Callback` 的优先级高于 Handler 自身的 `handleMessage`，`Callback.handleMessage` 返回 `true` 会“拦截”消息，阻止 Handler 自身方法执行。