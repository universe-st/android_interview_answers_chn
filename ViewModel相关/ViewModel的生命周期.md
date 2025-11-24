# Android中，ViewModel的生命周期与Activity和Fragment有什么关联？

在 Android 中，`ViewModel` 的核心设计目标是 **分离 UI 控制器（Activity/Fragment）与数据逻辑**，并解决「配置变化导致数据丢失」的问题。其生命周期与 Activity/Fragment 紧密关联，但**独立于 UI 控制器的配置变化**，存活时长更长。下面从「关联逻辑」「核心差异」「具体场景」三个维度详细说明：


### 一、先明确三个核心生命周期的定义
#### 1. Activity/Fragment 的生命周期（关键节点）
- **常规生命周期**：从 `onCreate()` 到 `onDestroy()`（例如启动→用户退出→`finish()`）。
- **配置变化导致的“临时销毁”**：当屏幕旋转、语言切换、权限变更等配置变化时，系统会销毁当前 Activity/Fragment 并重建（走 `onSaveInstanceState()`→`onDestroy()`→`onCreate()`），此时 UI 控制器实例被替换，但用户场景未结束（如仍在当前页面）。
- **最终销毁**：用户主动退出（如 `finish()`）或系统回收（无配置变化），此时 UI 控制器彻底消失，无法重建。

#### 2. ViewModel 的生命周期
- **创建时机**：当 UI 控制器（Activity/Fragment）首次通过 `ViewModelProvider` 获取 `ViewModel` 实例时，`ViewModel` 被创建。
- **存活时长**：从创建开始，直到 UI 控制器「最终销毁」（而非配置变化导致的临时销毁），`ViewModel` 才会被系统清除。
- **销毁时机**：UI 控制器最终销毁时，系统调用 `ViewModel.onCleared()` 方法，通知其释放资源（如取消网络请求、解绑观察者）。


### 二、ViewModel 与 Activity/Fragment 的核心关联
#### 1. 关联原则：ViewModel 依附于「宿主的生命周期所有者」
`ViewModel` 的生命周期由其「宿主」（Activity 或 Fragment）的 **LifecycleOwner** 管理，核心规则：
- 宿主是 Activity：`ViewModel` 跟随 Activity 的「最终销毁」而销毁；配置变化时 Activity 重建，新的 Activity 实例会复用之前的 `ViewModel`（而非创建新实例）。
- 宿主是 Fragment：`ViewModel` 跟随 Fragment 的「最终销毁」而销毁；即使 Fragment 依附的 Activity 发生配置变化，Fragment 重建后仍会复用原 `ViewModel`；若 Fragment 被 `replace()` 且未加入返回栈（`addToBackStack()`），则 Fragment 最终销毁，`ViewModel` 也随之清除。

#### 2. 生命周期对比图（文字示意）
| 场景                          | Activity 生命周期状态       | Fragment 生命周期状态       | ViewModel 生命周期状态       |
|-------------------------------|----------------------------|----------------------------|----------------------------|
| 首次启动                      | `onCreate()` → `onResume()` | `onCreate()` → `onResume()` | 被创建（通过 `ViewModelProvider` 获取） |
| 屏幕旋转（配置变化）          | 旧实例 `onDestroy()` → 新实例 `onCreate()` | 旧实例 `onDestroy()` → 新实例 `onCreate()` | 存活（复用原实例，不重建） |
| 用户点击返回键（Activity finish） | `onPause()` → `onDestroy()`（最终销毁） | `onPause()` → `onDestroy()`（最终销毁） | 调用 `onCleared()` → 销毁 |
| Fragment 被 replace 且未入栈   | 无变化（Activity 存活）     | `onDestroy()`（最终销毁）   | 调用 `onCleared()` → 销毁 |
| 系统回收（无配置变化）        | `onDestroy()`（最终销毁）   | `onDestroy()`（最终销毁）   | 调用 `onCleared()` → 销毁 |


### 三、关键场景验证：为什么 ViewModel 能解决配置变化数据丢失？
#### 场景 1：Activity 旋转时的数据保留
假设 Activity 中有一个网络请求后的列表数据，存储在 `ViewModel` 中：
- 首次启动：Activity `onCreate()` → 通过 `ViewModelProvider(this).get(MyViewModel::class.java)` 创建 `ViewModel` → 加载数据并存储在 `ViewModel` 中。
- 旋转屏幕：Activity 旧实例销毁、新实例创建 → 新 Activity 再次通过 `ViewModelProvider(this)` 获取 `ViewModel` → 直接复用原实例中的数据（无需重新请求网络）。

核心原因：`ViewModelProvider` 会根据 Activity 的「生命周期所有者」（`this` 即 Activity 的 `LifecycleOwner`）判断是否已有对应的 `ViewModel` 实例，配置变化时所有者未“最终销毁”，因此复用实例。

#### 场景 2：Fragment 与 Activity 共享 ViewModel
若多个 Fragment 需共享数据（如同一 Activity 下的两个 Fragment 通信），可让它们以「Activity 为宿主」获取 `ViewModel`：
```kotlin
// Fragment A
val viewModel = ViewModelProvider(requireActivity()).get(SharedViewModel::class.java)

// Fragment B
val viewModel = ViewModelProvider(requireActivity()).get(SharedViewModel::class.java)
```
此时 `ViewModel` 的生命周期与 Activity 绑定：只要 Activity 未最终销毁（即使 Fragment 重建、切换），`ViewModel` 始终存活，实现数据共享。


### 四、注意事项：避免生命周期关联导致的问题
#### 1. 禁止 ViewModel 持有 Activity/Fragment 引用
`ViewModel` 的生命周期可能长于 UI 控制器（如配置变化时 UI 控制器销毁但 `ViewModel` 存活），若 `ViewModel` 持有 Activity/Fragment 或其控件引用，会导致 UI 控制器无法被 GC 回收，引发内存泄漏。

✅ 正确做法：如需上下文，使用 `Application` 上下文（通过 `AndroidViewModel` 获取）：
```kotlin
class MyViewModel(application: Application) : AndroidViewModel(application) {
    val appContext = application // 安全，Application 生命周期与应用一致
}
```

#### 2. `onCleared()` 的正确使用
`ViewModel` 的 `onCleared()` 是唯一的销毁回调，需在此释放非内存资源（如取消 Retrofit 网络请求、解绑 EventBus、关闭数据库连接），避免资源泄漏。

#### 3. 不同宿主的 ViewModel 隔离性
- 以 Activity 为宿主的 `ViewModel`，对其下所有 Fragment 可见（可共享）。
- 以 Fragment 为宿主的 `ViewModel`，仅对该 Fragment 及其子 Fragment 可见（与其他 Fragment 隔离）。


### 总结
ViewModel 与 Activity/Fragment 的生命周期关联核心是：**ViewModel 独立于 UI 控制器的配置变化，仅依附于 UI 控制器的“最终销毁”**。这种设计让数据在屏幕旋转等场景下得以保留，同时分离了 UI 逻辑与数据逻辑，提升代码可维护性。

关键记忆点：
- 配置变化时，UI 控制器重建，ViewModel 复用。
- 最终销毁时，UI 控制器消失，ViewModel 销毁（调用 `onCleared()`）。
- ViewModel 不能持有 UI 控制器引用，需用 Application 上下文。