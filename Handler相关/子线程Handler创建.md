# 如何在子线程中创建 Handler?

在子线程中创建`Handler`的核心前提是：**子线程默认没有`Looper`，必须手动初始化`Looper`**（因为`Handler`依赖`Looper`管理的消息队列）。具体步骤如下：


### 子线程创建Handler的完整流程
#### 1. 初始化Looper
通过 `Looper.prepare()` 为当前子线程创建`Looper`实例（同时会创建对应的`MessageQueue`），并将`Looper`与当前线程绑定（通过`ThreadLocal`实现线程隔离）。

#### 2. 创建Handler
在`Looper.prepare()`之后创建`Handler`，此时`Handler`会自动绑定当前线程的`Looper`（通过`Looper.myLooper()`获取）。

#### 3. 启动消息循环
调用 `Looper.loop()` 启动消息循环，`Looper`会开始从`MessageQueue`中循环取出消息并分发给`Handler`处理。


### 代码示例
```java
new Thread(new Runnable() {
    @Override
    public void run() {
        // 步骤1：为当前子线程初始化Looper
        Looper.prepare();
        
        // 步骤2：创建Handler（自动绑定当前线程的Looper）
        Handler childHandler = new Handler(Looper.myLooper()) {
            @Override
            public void handleMessage(@NonNull Message msg) {
                // 此方法在子线程中执行（因为绑定了子线程的Looper）
                switch (msg.what) {
                    case 1:
                        Log.d("ChildHandler", "收到消息：" + msg.obj + "，处理线程：" + Thread.currentThread().getName());
                        break;
                }
            }
        };
        
        // 发送消息到子线程的消息队列
        Message message = Message.obtain();
        message.what = 1;
        message.obj = "来自子线程内部的消息";
        childHandler.sendMessage(message);
        
        // 步骤3：启动消息循环（此方法会阻塞当前线程，需放在最后）
        Looper.loop();
    }
}).start();
```


### 关键注意事项
#### 1. 必须手动调用`Looper.prepare()`
- 子线程默认没有`Looper`，若直接创建`Handler`会抛出异常：`Can't create handler inside thread XXX that has not called Looper.prepare()`。
- `Looper.prepare()` 内部会判断当前线程是否已有`Looper`，若已存在则抛出异常（确保每个线程最多只有一个`Looper`）。

#### 2. `Looper.loop()`是阻塞方法
- `Looper.loop()` 会进入无限循环，不断从`MessageQueue`中取消息，因此需放在子线程逻辑的**最后**执行。
- 若需终止循环，可调用 `Looper.myLooper().quit()`（强制清空消息队列并退出）或 `quitSafely()`（处理完当前所有延迟消息后退出）。

  示例：在消息处理完成后退出循环
  ```java
  @Override
  public void handleMessage(@NonNull Message msg) {
      // 处理消息...
      // 退出循环（避免子线程一直存活）
      Looper.myLooper().quit();
  }
  ```

#### 3. 线程绑定特性
- 子线程的`Handler`发送的消息，其处理逻辑（`handleMessage()`）会在**当前子线程**执行（而非主线程）。
- 若需在子线程中向主线程发送消息，应使用主线程的`Handler`（如在Activity中创建的默认`Handler`）。

#### 4. 内存泄漏风险
- 若子线程的`Handler`持有外部对象（如Activity）的强引用，且`Looper`未退出，可能导致外部对象无法被回收。
- 解决：使用**静态内部类+弱引用**持有外部对象，或在外部对象销毁时主动调用`Looper.quit()`。


### 为什么主线程不需要手动初始化Looper？
- 主线程（UI线程）在启动时，系统会自动调用 `Looper.prepareMainLooper()` 初始化`Looper`（在`ActivityThread`的`main()`方法中），并启动`Looper.loop()`。
- 这也是主线程可以直接创建`Handler`的原因，而子线程需要手动处理，是为了避免不必要的资源消耗（并非所有子线程都需要消息循环）。


### 总结
子线程创建`Handler`的核心是“手动初始化`Looper`”，三步流程清晰：`Looper.prepare()` → 创建`Handler` → `Looper.loop()`。掌握这一机制，可灵活实现子线程内部的消息处理或跨线程通信（如子线程向主线程发送更新UI的消息）。