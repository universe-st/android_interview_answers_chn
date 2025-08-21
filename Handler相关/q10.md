# Handler如果没有消息处理是阻塞的还是非阻塞的？

在Android中，**Handler本身并不直接决定阻塞或非阻塞**，其行为依赖于所关联的`Looper`和`MessageQueue`的工作机制。


当Handler没有消息需要处理时（即`MessageQueue`中没有消息），实际是Handler所关联的**Looper所在的线程会进入阻塞状态**，而非持续空转（非阻塞轮询）。


具体原理：
1. `Looper`的核心是一个无限循环（`loop()`方法），不断从`MessageQueue`中获取消息（通过`MessageQueue.next()`）。
2. 当`MessageQueue`中没有消息时，`next()`方法会通过**Linux的管道（pipe）或epoll机制**进入阻塞状态，释放CPU资源，等待新消息到来。
3. 当有新消息通过Handler发送到`MessageQueue`时，会唤醒阻塞的`next()`方法，Looper继续处理消息。


因此，**当Handler没有消息处理时，其关联的线程会阻塞等待**（而非非阻塞轮询），这种阻塞是高效的（不消耗CPU），直到有新消息触发唤醒。