# 生命周期方法

在Android开发中，Activity的生命周期是指Activity从创建到销毁的整个过程，系统会在不同阶段回调对应的生命周期方法。这些方法帮助开发者管理Activity的状态（如资源初始化、释放、数据保存等），确保应用行为符合用户预期。

Activity的核心生命周期方法按调用顺序可分为**完整生命周期**、**可见生命周期**和**前台生命周期**三个阶段，具体如下：


### 一、核心生命周期方法及调用时机

#### 1. `onCreate(Bundle savedInstanceState)`
- **调用时机**：Activity第一次被创建时调用（仅调用一次）。
- **作用**：完成Activity的初始化工作，如设置布局（`setContentView()`）、初始化控件、绑定数据、注册广播接收器等。
- **参数**：`savedInstanceState` 用于恢复之前保存的状态（如屏幕旋转时保存的数据），若为首次创建则为`null`。
- **示例**：启动一个新Activity时，第一个被调用的方法。


#### 2. `onStart()`
- **调用时机**：Activity由“不可见”变为“可见”时调用（但尚未进入前台，用户无法交互）。
- **作用**：可在此处执行与“可见性”相关的操作，如启动动画、加载可见但非交互的资源。
- **关联场景**：
  - 首次创建Activity时，在`onCreate()`之后调用。
  - 从停止状态（`onStop()`之后）恢复时，在`onRestart()`之后调用。


#### 3. `onResume()`
- **调用时机**：Activity进入前台，用户可以与之交互时调用（处于“运行状态”）。
- **作用**：恢复交互相关资源，如启动传感器、开始播放动画、获取焦点等。此时Activity位于任务栈顶，是用户当前操作的页面。
- **关联场景**：
  - 首次创建时，在`onStart()`之后调用。
  - 从暂停状态（`onPause()`之后）恢复时直接调用（如关闭覆盖它的对话框）。


#### 4. `onPause()`
- **调用时机**：Activity即将失去焦点（仍可能部分可见）时调用（如被新的Activity覆盖、屏幕锁屏、按Home键）。
- **作用**：
  - 暂停当前交互操作（如暂停视频播放、停止动画）。
  - 保存临时数据（避免因系统回收导致数据丢失）。
  - 释放轻量级资源（如注销广播接收器的临时监听）。
- **注意**：此方法执行速度需快，否则会阻塞新Activity的启动（系统会等待当前`onPause()`完成后才会调用新Activity的`onResume()`）。


#### 5. `onStop()`
- **调用时机**：Activity完全不可见时调用（如被新的Activity完全覆盖）。
- **作用**：释放与“可见性”相关的重量级资源，如停止网络请求、释放非必要的内存缓存、暂停后台任务等。
- **注意**：若Activity只是部分可见（如被对话框遮挡），不会触发`onStop()`，只会触发`onPause()`。


#### 6. `onRestart()`
- **调用时机**：Activity从“停止状态”（`onStop()`之后）重新变为“可见状态”前调用（仅在Activity未被销毁时触发）。
- **作用**：准备恢复Activity状态，可在此处重新加载数据或恢复之前的配置。
- **关联场景**：用户按返回键从新Activity回到当前Activity时，会先调用`onRestart()`，再依次执行`onStart()`→`onResume()`。


#### 7. `onDestroy()`
- **调用时机**：Activity被销毁前最后调用（仅调用一次）。
- **触发原因**：
  - 主动调用`finish()`方法。
  - 系统因内存不足回收（低优先级Activity可能被回收）。
  - 配置变更（如屏幕旋转）导致Activity重建（先销毁旧实例，再创建新实例）。
- **作用**：彻底释放所有资源，如解绑服务、注销所有广播接收器、释放线程等，避免内存泄漏。


### 二、生命周期调用流程示例

#### 1. 正常启动到销毁
```
onCreate() → onStart() → onResume()  // 启动并进入运行状态
（用户操作...）
onPause() → onStop() → onDestroy()  // 关闭Activity
```


#### 2. 按Home键（进入后台）再返回
```
// 按Home键（进入后台）
onPause() → onStop()  

// 从后台返回前台
onRestart() → onStart() → onResume()
```


#### 3. 打开新Activity（覆盖当前Activity）再返回
```
// 打开新Activity
当前Activity: onPause()  
新Activity: onCreate() → onStart() → onResume()  
当前Activity: onStop()  

// 从新Activity返回
新Activity: onPause()  
当前Activity: onRestart() → onStart() → onResume()  
新Activity: onStop() → onDestroy()
```


#### 4. 屏幕旋转（配置变更导致重建）
```
// 旧实例销毁
onPause() → onStop() → onDestroy()  

// 新实例创建
onCreate()（可通过savedInstanceState恢复数据）→ onStart() → onResume()
```


### 三、关键说明
- **生命周期的“不可逆”与“可逆”**：
  - `onCreate()`→`onDestroy()`是“完整生命周期”，不可逆（一旦销毁，实例不再存在）。
  - `onStart()`→`onStop()`是“可见生命周期”，可逆（可通过`onRestart()`恢复可见）。
  - `onResume()`→`onPause()`是“前台生命周期”，可逆（可恢复交互状态）。
- **数据保存**：在`onPause()`或`onSaveInstanceState()`中保存临时数据，在`onCreate()`或`onRestoreInstanceState()`中恢复（适用于配置变更场景）。
- **内存管理**：在`onStop()`中释放非必要资源，在`onDestroy()`中彻底清理，避免内存泄漏（如未注销的监听器、未终止的线程）。

理解Activity生命周期是Android开发的基础，合理利用这些方法可确保应用在各种场景下（如切换页面、屏幕旋转、内存不足）都能正确响应，提升用户体验。