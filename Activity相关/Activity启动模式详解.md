# Activity的启动模式

在Android开发中，Activity的启动模式决定了系统如何创建和管理Activity实例在任务栈中的行为。共有四种核心启动模式：**standard**、**singleTop**、**singleTask** 和 **singleInstance**，每种模式通过不同的策略优化任务栈管理，避免重复创建实例或提升用户体验。以下是详细解析：

### 一、标准模式（standard）
- **行为**：默认模式，每次启动Activity都会创建新实例，无论任务栈中是否已有该Activity。新实例会被压入启动它的任务栈栈顶。
- **示例**：假设任务栈中有A→B→C（均为standard），再次启动C时，栈变为A→B→C→C。
- **使用场景**：无特殊需求的普通页面，如列表页跳转详情页的常规流程。
- **注意**：多次启动可能导致栈中存在多个实例，需谨慎处理返回逻辑。

### 二、栈顶复用模式（singleTop）
- **行为**：若目标Activity已位于栈顶，则复用该实例并调用`onNewIntent()`；否则创建新实例。
- **示例**：
  - 栈顶为C（singleTop），再次启动C时复用，仅触发`onNewIntent()`。
  - 栈中有A→B→C，此时启动A（singleTop），会创建新的A实例，栈变为A→B→C→A。
- **使用场景**：频繁接收新数据的页面（如新闻推送详情页），避免重复加载。
- **动态配置**：可通过`Intent.FLAG_ACTIVITY_SINGLE_TOP`临时覆盖XML配置。

### 三、栈内复用模式（singleTask）
- **行为**：
  1. 若任务栈中已存在该Activity实例，将其移至栈顶，并销毁其上所有Activity，调用`onNewIntent()`。
  2. 若不存在，则创建新实例并作为栈底元素。
- **示例**：
  - 栈中有A→B→C（均为singleTask），再次启动C时复用，栈不变。
  - 若启动A，则销毁B和C，栈变为A，并触发A的`onNewIntent()`。
- **使用场景**：
  - 应用主界面或全局唯一的功能入口（如浏览器主页），确保用户返回时直达主界面。
  - 需清空栈中其他页面的场景，如登录后跳转主页。
- **注意**：
  - 若其他应用通过隐式Intent启动该Activity，可能导致任务栈混乱，可通过设置`android:taskAffinity`指定独立任务栈。
  - 若初始化逻辑写在`onCreate()`中，复用时可能因`onNewIntent()`被调用而导致数据不一致，需在`onNewIntent()`中更新状态。

### 四、单实例模式（singleInstance）
- **行为**：
  1. 该Activity独占一个任务栈，且整个系统中仅有一个实例。
  2. 无论从哪个任务栈启动，均复用现有实例，并将其任务栈调至前台。
- **示例**：
  - 启动A（singleInstance）后，系统为其创建独立任务栈[Task1:A]。
  - 从其他应用启动A时，直接复用Task1中的实例，不会创建新栈。
- **使用场景**：
  - 系统级组件（如来电界面）或需完全隔离的功能（如视频通话），确保与其他应用无交互干扰。
- **特性**：
  - 该任务栈中无法启动其他Activity，即使由A自身启动，新Activity也会被分配到其他栈。
  - 任务栈生命周期独立，除非被系统回收，否则始终存在。

### 五、关键对比与扩展
| 特性                | standard       | singleTop      | singleTask       | singleInstance     |
|---------------------|----------------|----------------|------------------|--------------------|
| **实例数量**        | 多个           | 栈顶唯一       | 任务栈内唯一     | 全局唯一           |
| **任务栈归属**      | 启动者的栈     | 启动者的栈     | 可指定（taskAffinity） | 独立栈             |
| **复用条件**        | 每次创建       | 栈顶存在时复用 | 任务栈内存在时复用 | 全局存在时复用     |
| **栈清理行为**      | 无             | 无             | 销毁其上所有Activity | 无                 |
| **适用场景**        | 常规页面       | 频繁更新的页面 | 主界面、全局入口 | 系统级独立功能     |

### 六、配置与扩展
1. **静态配置**：在`AndroidManifest.xml`中通过`android:launchMode`设置。
2. **动态配置**：通过`Intent`标志位临时修改，如：
   - `FLAG_ACTIVITY_SINGLE_TOP`（等效singleTop）。
   - `FLAG_ACTIVITY_NEW_TASK`（结合`singleTask`逻辑）。
3. **任务亲和性（taskAffinity）**：
   - 与`singleTask`或`singleInstance`配合时，指定Activity所属的任务栈名称（默认与应用包名相同）。
   - 例如，设置`android:taskAffinity="com.example.othertask"`可使singleTask的Activity运行在独立栈中。
4. **常见问题**：
   - **singleTask导致的onCreate()不执行**：需将初始化逻辑移至`onNewIntent()`，并在`onCreate()`中处理`savedInstanceState`。
   - **任务栈混乱**：通过`taskAffinity`或`allowTaskReparenting`控制Activity的归属。

### 七、总结
合理选择启动模式可优化内存使用、提升用户体验。例如：
- **standard**用于常规页面跳转。
- **singleTop**避免栈顶重复实例，适用于通知详情页。
- **singleTask**确保全局唯一性，适合主界面。
- **singleInstance**实现完全隔离，用于系统级功能。

开发者需结合任务栈逻辑、生命周期方法（如`onNewIntent()`）及`taskAffinity`等属性，灵活配置启动模式，避免因实例复用导致的数据或UI异常。