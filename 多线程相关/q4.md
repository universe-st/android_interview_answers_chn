# synchronied和Lock在实现原理上有哪些区别？

`synchronized`和`Lock`（以最常用的`ReentrantLock`为例）在实现原理上有本质区别，前者是JVM底层内置实现，后者是基于Java API的代码级实现。以下从**底层机制**、**锁状态管理**、**等待队列**、**锁的获取与释放**四个核心维度详细解析：


### 一、底层实现主体不同
- **`synchronized`**：  
  是Java语言的**关键字**，其实现完全依赖JVM底层（C++代码），通过字节码指令（`monitorenter`和`monitorexit`）与对象监视器（`monitor`）交互完成同步。  
  - 当编译Java代码时，`synchronized`代码块会被转换为`monitorenter`（进入同步块）和`monitorexit`（退出同步块）指令，分别对应锁的获取和释放。  
  - 异常场景下，JVM会自动插入`monitorexit`指令确保锁释放，无需开发者干预。

- **`Lock`**：  
  是Java类库（`java.util.concurrent.locks`）中的**接口**，其实现（如`ReentrantLock`）完全由Java代码完成，核心依赖**AQS（AbstractQueuedSynchronizer，抽象队列同步器）** 框架。  
  - AQS是一个模板类，通过维护**同步状态（`state`变量）** 和**双向等待队列**实现锁的获取与释放逻辑。  
  - `ReentrantLock`等实现类通过重写AQS的`tryAcquire()`、`tryRelease()`等方法定义具体的锁逻辑。


### 二、锁状态的存储与管理不同
- **`synchronized`**：  
  锁状态与**对象绑定**，存储在对象头的`Mark Word`中（对象在内存中的布局包括：对象头、实例数据、对齐填充）。  
  - `Mark Word`是一个动态结构，根据对象状态（无锁、偏向锁、轻量级锁、重量级锁）存储不同信息：  
    - **无锁状态**：存储对象哈希码、GC分代年龄。  
    - **偏向锁**：存储偏向的线程ID（同一线程多次获取锁时无需竞争）。  
    - **轻量级锁**：存储指向线程栈中锁记录（`Lock Record`）的指针（通过CAS竞争锁）。  
    - **重量级锁**：存储指向操作系统互斥量（`mutex`）的指针（依赖OS内核实现阻塞）。  
  - 锁状态会**自适应升级**：从偏向锁→轻量级锁→重量级锁，逐步提升并发能力（JDK 1.6后优化）。

- **`Lock`**：  
  锁状态由**AQS的`state`变量**维护（`state`是一个`volatile int`类型的成员变量）。  
  - 以`ReentrantLock`为例：  
    - `state=0`：锁未被持有。  
    - 线程获取锁时，通过CAS将`state`从0改为1；重入时`state`递增（如重入2次则`state=2`）。  
    - 线程释放锁时，`state`递减，直至`state=0`表示锁完全释放。  
  - 锁状态不依赖对象头，而是由AQS实例独立管理，因此一个`Lock`可以关联多个对象的同步逻辑。


### 三、等待队列的实现不同
当线程获取锁失败时，会进入等待队列等待被唤醒。两者的等待队列实现机制完全不同：

- **`synchronized`**：  
  依赖**对象监视器（`monitor`）** 内部的**等待集（WaitSet）** 和**-entryset**：  
  - `monitor`是JVM内部的一个C++对象，包含三个核心结构：  
    - **_owner**：指向当前持有锁的线程。  
    - **_WaitSet**：存储调用`wait()`后进入等待状态的线程。  
    - **_EntryList**：存储正在竞争锁的阻塞线程。  
  - 线程获取锁失败时，会被放入`_EntryList`阻塞；持有锁的线程调用`wait()`时，会释放锁并进入`_WaitSet`，等待被`notify()`唤醒后重新进入`_EntryList`竞争锁。  
  - 队列是**无序的**（非公平锁），唤醒时随机选择线程（`notify()`）或唤醒所有线程（`notifyAll()`）。

- **`Lock`**：  
  依赖AQS的**双向同步队列（CLH队列的变种）**：  
  - 队列由节点（`Node`）组成，每个节点包含线程引用、等待状态（`waitStatus`）、前驱和后继指针。  
  - 线程获取锁失败时，会通过CAS原子操作加入队列尾部，并通过`LockSupport.park()`阻塞。  
  - 持有锁的线程释放锁时，会唤醒队列头部的线程，被唤醒的线程会再次尝试获取锁（可能存在竞争）。  
  - 队列是**有序的**（FIFO），支持公平锁（按入队顺序获取）和非公平锁（新线程可能插队）。  


### 四、锁的获取与释放机制不同
- **`synchronized`**：  
  - **获取锁**：线程执行`monitorenter`指令时，尝试获取对象的`monitor`所有权：  
    - 若`monitor`的`_owner`为`null`，当前线程直接持有锁（`_owner`设为当前线程）。  
    - 若当前线程已持有`monitor`（可重入），则增加重入次数。  
    - 否则，线程进入`_EntryList`阻塞等待。  
  - **释放锁**：线程执行`monitorexit`指令时，减少重入次数；当重入次数为0时，释放`monitor`（`_owner`设为`null`），并唤醒`_EntryList`或`_WaitSet`中的线程。  
  - **自动性**：无论正常执行完毕还是抛出异常，JVM都会确保`monitorexit`被执行，避免锁泄漏。

- **`Lock`**：  
  - **获取锁**：通过`lock()`方法触发AQS的`acquire(1)`：  
    1. 调用自定义实现的`tryAcquire(1)`（如`ReentrantLock`中通过CAS修改`state`尝试获取锁）。  
    2. 若失败，调用`addWaiter()`将线程包装为`Node`加入等待队列。  
    3. 调用`acquireQueued()`阻塞当前线程（通过`LockSupport.park()`）。  
  - **释放锁**：通过`unlock()`方法触发AQS的`release(1)`：  
    1. 调用自定义实现的`tryRelease(1)`（如`ReentrantLock`中递减`state`，直至`state=0`）。  
    2. 若释放成功，调用`unparkSuccessor()`唤醒队列中等待的线程。  
  - **手动性**：必须在`finally`中显式调用`unlock()`，否则可能因异常导致锁未释放，引发死锁。  


### 五、条件变量（等待/通知）的实现不同
- **`synchronized`**：  
  依赖`Object`类的`wait()`、`notify()`、`notifyAll()`方法，与对象的`monitor`绑定：  
  - 所有等待线程共享同一个`_WaitSet`，无法区分等待条件（如生产者-消费者模型中，"队列满"和"队列空"的等待无法分离）。  
  - 调用`notify()`时，JVM随机唤醒一个等待线程，可能唤醒不符合条件的线程（需重新检查条件）。

- **`Lock`**：  
  通过`Condition`接口实现，每个`Condition`对应AQS的一个**条件队列**：  
  - 调用`lock.newCondition()`可创建多个`Condition`，每个`Condition`有独立的等待队列（分离不同条件的等待）。  
  - 线程调用`condition.await()`时，会释放锁并加入该`Condition`的等待队列；调用`condition.signal()`时，仅唤醒该队列中的线程。  
  - 实现更精细的线程通信（如区分"满"和"空"的等待队列，避免无效唤醒）。  


### 总结
| 维度               | `synchronized`                          | `Lock`（`ReentrantLock`）               |
|--------------------|-----------------------------------------|-----------------------------------------|
| 实现主体           | JVM底层（C++）                          | Java代码（基于AQS框架）                 |
| 锁状态存储         | 对象头的`Mark Word`                     | AQS的`state`变量                        |
| 等待队列           | `monitor`的`_EntryList`和`_WaitSet`     | AQS的双向同步队列                       |
| 获取/释放方式      | 自动（`monitorenter`/`monitorexit`）     | 手动（`lock()`/`unlock()`，依赖`finally`） |
| 条件变量           | 共享`_WaitSet`（`Object`的`wait`/`notify`） | 独立条件队列（`Condition`接口）          |

简言之，`synchronized`是JVM深度优化的"黑盒"同步机制，依赖对象监视器和自动管理；`Lock`是基于AQS的"白盒"实现，通过代码逻辑灵活控制锁的行为，两者从底层机制到使用方式都体现了"自动简化"与"手动灵活"的设计差异。