# 当我从一个A Activity启动另外一个B Activity，请写出生命周期方法的调用顺序

当从 **Activity A** 启动 **Activity B** 时，生命周期方法的调用顺序如下（默认启动模式为 `standard`）：

### 一、核心调用顺序（默认场景）
1. **Activity A 的 `onPause()`**  
   - **触发时机**：A 即将失去焦点（例如用户点击按钮启动 B），但此时 A 仍部分可见（如 B 是透明主题）或完全可见（如 B 是全屏 Activity）。
   - **作用**：暂停 A 的交互操作（如停止动画），保存临时数据（如用户输入未提交的内容）。

2. **Activity B 的 `onCreate(Bundle savedInstanceState)`**  
   - **触发时机**：B 首次被创建时调用，仅执行一次。
   - **作用**：初始化 B 的界面（如 `setContentView()`）、绑定数据、注册监听器等。

3. **Activity B 的 `onStart()`**  
   - **触发时机**：B 变为可见但尚未进入前台（用户无法交互）。
   - **作用**：准备显示 B 的 UI（如加载可见但非交互的资源）。

4. **Activity B 的 `onResume()`**  
   - **触发时机**：B 进入前台，用户可交互。
   - **作用**：恢复 B 的交互能力（如启动传感器、请求网络数据）。

5. **Activity A 的 `onStop()`**  
   - **触发时机**：B 完全覆盖 A 后，A 不再可见。
   - **作用**：释放与可见性相关的资源（如停止后台任务、注销广播接收器）。

**完整顺序示例**：  
```
A.onPause() → B.onCreate() → B.onStart() → B.onResume() → A.onStop()
```

### 二、特殊场景下的差异
#### 1. **B 为透明主题或对话框样式**  
- **调用顺序**：  
  ```
  A.onPause() → B.onCreate() → B.onStart() → B.onResume()
  ```
  - **原因**：B 未完全覆盖 A，A 仍部分可见，因此 **A 的 `onStop()` 不会被调用**。

#### 2. **启动模式为 `singleTop` 或 `singleTask`**  
- **若 B 已存在于栈顶**：  
  - **B 的生命周期**：仅调用 `onNewIntent(Intent)`，不触发 `onCreate()`、`onStart()`、`onResume()`。
  - **A 的生命周期**：仍执行 `onPause()` → `onStop()`（若 B 完全覆盖 A）。

- **若 B 不在栈顶**：  
  - **调用顺序**与默认场景一致。

#### 3. **内存不足导致 A 被回收**  
- **调用顺序**：  
  ```
  A.onPause() → B.onCreate() → B.onStart() → B.onResume() → A.onStop() → A.onDestroy()
  ```
  - **说明**：若系统因内存不足销毁 A，后续返回 A 时需重新创建实例，触发 `onCreate()`、`onStart()`、`onResume()`。

### 三、关键说明
1. **执行优先级**：  
   - **B 的 `onResume()` 必须在 A 的 `onStop()` 之前完成**，以确保 B 可见后 A 才进入停止状态，避免界面闪烁。
   - **A 的 `onPause()` 执行时间需严格控制**（建议 < 500ms），否则会阻塞 B 的启动流程。

2. **数据传递与恢复**：  
   - **通过 `Intent` 传递参数**：在 A 的 `startActivity()` 中携带数据，在 B 的 `onCreate()` 或 `onNewIntent()` 中解析。
   - **保存 A 的状态**：在 A 的 `onPause()` 或 `onSaveInstanceState()` 中保存临时数据，在 `onRestart()` 或 `onRestoreInstanceState()` 中恢复。

3. **与启动模式的关联**：  
   - **`singleTask` 或 `singleInstance`**：若 B 已存在，系统会将其移至栈顶并销毁其上所有 Activity，此时 A 的 `onStop()` 可能被跳过（取决于 A 是否在 B 之上）。

### 四、扩展场景：返回 A 时的生命周期
1. **用户按返回键从 B 回到 A**：  
   - **B 的生命周期**：  
     ```
     B.onPause() → B.onStop() → B.onDestroy()
     ```
   - **A 的生命周期**：  
     ```
     A.onRestart() → A.onStart() → A.onResume()
     ```
   - **完整顺序**：  
     ```
     B.onPause() → A.onRestart() → A.onStart() → A.onResume() → B.onStop() → B.onDestroy()
     ```

2. **B 调用 `finish()` 主动关闭**：  
   - 顺序与用户按返回键一致。

### 五、总结
- **默认场景**：A 的 `onPause()` 先于 B 的创建，B 完全显示后 A 进入停止状态。
- **特殊场景**：透明主题、启动模式、内存回收等会改变调用顺序，需结合具体需求处理资源释放与状态恢复。
- **开发建议**：  
  - 在 `onPause()` 中保存临时数据，在 `onStop()` 中释放重量级资源。
  - 避免在 `onPause()` 中执行耗时操作（如文件写入、复杂计算）。
  - 结合 `startActivityForResult()` 和 `onActivityResult()` 处理 B 返回的数据。

通过理解这些顺序和规则，开发者可精准控制 Activity 状态，优化用户体验并避免内存泄漏。