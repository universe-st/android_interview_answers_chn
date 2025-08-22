# Context

### 一、Context 是什么？—— 定义与核心理解

**核心定义**：`Context` 是一个抽象类，它提供了应用程序环境的全局信息的接口。它是一个抽象的概念，可以理解为 **应用程序的当前状态** 或 **应用程序与系统交互的桥梁**。

**通俗比喻**：
你可以把 `Context` 想象成：
*   **一个应用程序的“身份证”或“工作证”**：有了它，你才能证明自己是这个应用的一部分，从而有权限去访问应用的资源（如图片、字符串）、启动组件、操作数据库等。
*   **一个应用程序的“环境”或“宇宙”**：所有组件（Activity, Service 等）都生存在这个宇宙中，通过 `Context` 来感知和与这个宇宙交互。

**它的主要作用体现在以下四个方面**：
1.  **访问应用资源**：通过 `Context` 可以访问到 `res` 目录下的所有资源（字符串、颜色、尺寸、布局等），以及 `Assets` 中的文件。
2.  **启动系统组件**：启动 Activity、启动/绑定 Service、发送广播、注册广播接收器等，这些操作都需要通过 `Context` 来完成。
3.  **获取系统服务**：通过 `Context` 可以获取到各种系统级的服务（称为 `Manager`），如 `WindowManager`, `LayoutInflater`, `LocationManager`, `ActivityManager` 等。
4.  **操作应用文件和数据**：访问应用私有目录（`/data/data/<package_name>/`）下的文件和 SharedPreferences。

---

### 二、Context 的类型 —— 主要有两种

`Context` 的具体实现主要分为两种，它们的作用域和生命周期不同，这是理解 `Context` 用法的关键。

#### 1. Application Context（应用上下文）

*   **获取方式**：`context.getApplicationContext()`
*   **生命周期**：**与应用进程的生命周期相同**。从应用启动到应用进程被杀死，它一直存在。它是**单例的**。
*   **作用域**：**全局的**。它代表整个应用的上下文。
*   **使用场景**：适用于需要与应用生命周期相关的、全局性的操作。例如：
    *   初始化一个全局的、单例的库（如图片加载库、网络库）。
    *   获取一个需要应用级别 `Context` 的系统服务。
    *   在非 Activity 的类（如一个工具类或一个自定义的 `ViewModel`）中需要 `Context` 时。

> **注意**：不要用它来启动一个 Activity 或创建对话框，因为这需要任务栈（Task）的支持，而 `Application Context` 不具备这个能力，会导致异常。

#### 2. Activity Context（活动上下文）

*   **获取方式**：在 `Activity` 内部直接使用 `this`。
*   **生命周期**：**与所属的 Activity 的生命周期相同**。当 Activity 被销毁时，它的 `Context` 也随之失效。
*   **作用域**：**局部的**。它代表当前 Activity 的上下文，包含了 Activity 的主题（Theme）、窗口（Window）等信息。
*   **使用场景**：适用于所有与 UI 相关的操作，因为它是“有界面的”。例如：
    *   启动另一个 Activity（`startActivity`）。
    *   显示一个对话框（`Dialog`）。
    *   加载布局（`LayoutInflater`）。
    *   操作当前 Activity 的 View。

---

### 三、其他形式的 Context

除了上述两种核心类型，你还会在其他地方见到 `Context`：

*   **Service**：`Service` 本身也继承自 `Context`，所以你在 Service 内部可以使用 `this`。它的生命周期与 Service 相同。**注意**：Service 的 `Context` 也是**应用级别**的，不能用它来启动 Activity 或绑定到 UI 相关的操作。
*   **BroadcastReceiver**：在 `onReceive()` 方法中提供的 `Context` 是一个 `ReceiverRestrictedContext`，它对某些操作（如 `registerReceiver()`）进行了限制。
*   **ContentProvider**：在 `onCreate()` 中获取到的是 `Application Context`。

---

### 四、重要注意事项与最佳实践

#### 1. 避免内存泄漏（最重要！）

这是处理 `Context` 时最常见的陷阱。

*   **问题根源**：如果一个长生命周期的对象（如一个静态变量、一个单例）持有了一个 `Activity Context` 的引用，那么即使这个 Activity 已经被关闭（onDestroy），由于垃圾回收器（GC）无法回收它，导致这个 Activity 实例及其关联的 View 和资源都无法被释放，从而造成**内存泄漏**。
*   **经典案例**：在单例模式中，错误地传入了 Activity 的 `Context`。
    ```java
    public class BadSingleton {
        private static BadSingleton instance;
        private Context mContext; // 危险！可能持有 Activity 的引用

        private BadSingleton(Context context) {
            this.mContext = context; // 如果传入的是 Activity Context，就泄漏了
        }

        public static BadSingleton getInstance(Context context) {
            if (instance == null) {
                instance = new BadSingleton(context);
            }
            return instance;
        }
    }
    ```
*   **解决方案**：在需要应用级别 `Context` 的地方，**优先使用 `Application Context`**。
    ```java
    public class GoodSingleton {
        private static GoodSingleton instance;
        private Context mAppContext; // 安全

        private GoodSingleton(Context context) {
            // 使用 getApplicationContext() 确保是 Application Context
            this.mAppContext = context.getApplicationContext();
        }

        public static GoodSingleton getInstance(Context context) {
            if (instance == null) {
                instance = new GoodSingleton(context);
            }
            return instance;
        }
    }
    ```

#### 2. 选择合适的 Context

遵循一个简单原则：
*   **做与 UI 无关的事情** -> 优先使用 `Application Context`。
*   **做与 UI 相关的事情**（启动 Activity、显示 Dialog、Inflate Layout） -> 必须使用 `Activity Context`。

错误示例：使用 `Application Context` 去显示一个 Toast 或 Dialog，虽然有时不会崩溃，但可能会丢失主题样式。用它去 `startActivity` 则会直接抛出异常。

---

### 五、总结

| 特性 | Application Context | Activity Context |
| :--- | :--- | :--- |
| **生命周期** | 整个应用（漫长） | 当前 Activity（较短） |
| **作用域** | 全局 | 局部（当前界面） |
| **使用场景** | 获取全局资源、初始化全局库、系统服务 | 启动 Activity、显示 UI、操作 View |
| **内存风险** | 低（不会直接导致 Activity 泄漏） | **高**（容易被长生命周期对象持有导致泄漏） |
| **获取方式** | `getApplicationContext()` | `Activity.this` |

**核心要点**：
1.  `Context` 是 Android 应用的运行环境，是访问资源和服务的中枢。
2.  深刻理解 **Application Context** 和 **Activity Context** 在**生命周期**和**使用场景**上的区别。
3.  **时刻警惕内存泄漏**：谨慎对待静态引用和单例模式中的 `Context`，非 UI 操作优先使用 `Application Context`。
4.  **谁调用，谁是 Context**：在 Activity 和 Service 中，`this` 就是它们自身的 Context。
