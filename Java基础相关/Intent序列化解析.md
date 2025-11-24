# Intent传递对象为什么需要序列化？

在Android中，Intent传递对象时需要序列化，核心原因与**跨进程通信（IPC）的本质**和**Android组件的运行机制**密切相关，具体可以从以下几个方面理解：


### 1. 进程隔离与内存不共享
Android中，不同组件（如Activity、Service等）可能运行在**不同的进程**中（通过`AndroidManifest.xml`的`android:process`属性指定）。每个进程拥有独立的内存空间，进程间无法直接访问彼此的内存数据。

当通过Intent传递对象时，如果直接传递对象的内存引用（类似Java中的对象引用），在跨进程场景下这个引用是无效的（目标进程无法访问源进程的内存）。因此，需要将对象转换为**可传输的字节序列**，才能在进程间传递。


### 2. Intent的数据传递依赖Binder机制
Intent传递数据的底层依赖Android的**Binder机制**（Android的核心IPC机制）。Binder在传输数据时，只能处理**基本数据类型**（如int、String等）和**实现了序列化协议的对象**。

如果对象未序列化，Binder无法识别如何将其转换为可传输的格式，会直接抛出`NotSerializableException`（使用Java的Serializable时）或运行时错误（使用Android的Parcelable时）。


### 3. 序列化的本质：对象→字节流的转换
序列化的核心作用是将对象的状态（成员变量的值）转换为**有序的字节序列**，接收方可以通过反序列化将字节序列重新还原为对象。

在Intent传递对象的场景中，序列化确保了：
- 源进程的对象数据能被"拆解"为字节流，通过Binder传输到目标进程；
- 目标进程能通过反序列化，基于字节流"重建"出一个与源对象状态一致的新对象。


### 4. Android支持的两种序列化方式
为了满足Intent传递对象的需求，Android支持两种序列化协议：
- **Serializable**：Java自带的序列化接口，实现简单（只需实现`Serializable`接口），但性能较差（会产生大量临时对象，引发GC）。
- **Parcelable**：Android专用的序列化接口，需要手动实现序列化/反序列化逻辑（通过`writeToParcel()`和`CREATOR`），性能远优于Serializable，是Android推荐的方式。


### 总结
Intent传递对象需要序列化的根本原因是：**跨进程通信时，不同进程的内存不共享，必须通过序列化将对象转换为可传输的字节流，才能通过Binder机制传递，最终在目标进程中重建对象**。这是Android组件间安全、可靠通信的基础。