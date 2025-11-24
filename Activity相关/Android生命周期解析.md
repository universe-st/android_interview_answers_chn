# onNewIntent()和 onConfigurationChanged()

在 Android 开发中，`onNewIntent()` 和 `onConfigurationChanged()` 是两个关键的生命周期回调方法，分别用于处理 **Intent 数据更新** 和 **设备配置变化**。以下是基于系统机制和实际开发场景的详细解析：

### 一、`onNewIntent()` 方法
#### 1. **核心作用**
当已存在的 Activity 实例接收到新的启动 Intent 时触发，用于更新当前 Activity 的 Intent 数据。该方法仅在 **`singleTop`、`singleTask`、`singleInstance` 启动模式** 或 **显式设置 `FLAG_ACTIVITY_SINGLE_TOP` 标志位** 时调用。

#### 2. **调用时机**
- **`singleTop` 模式**：若目标 Activity 位于任务栈顶，直接复用实例并调用 `onNewIntent()`；若不在栈顶，则创建新实例，不触发该方法。
- **`singleTask` 模式**：无论 Activity 是否在栈顶，只要存在实例，就复用并调用 `onNewIntent()`，同时清空其上方的所有 Activity。
- **`singleInstance` 模式**：与 `singleTask` 类似，但该 Activity 单独运行在一个任务栈中。

#### 3. **关键参数与使用方法**
- **参数 `intent`**：携带新的启动数据，需通过 `setIntent(intent)` 方法更新当前 Activity 的 Intent，否则后续调用 `getIntent()` 仍返回旧数据。
  ```java
  @Override
  protected void onNewIntent(Intent intent) {
      super.onNewIntent(intent);
      setIntent(intent); // 必须调用，否则旧 Intent 会被保留
      handleNewIntent(intent); // 自定义数据处理逻辑
  }
  ```
- **数据处理场景**：
  - **通知栏点击**：同一 Activity 处理不同通知携带的参数（如新闻详情页显示不同新闻）。
  - **深度链接**：处理外部链接跳转时的动态内容（如电商应用根据链接展示特定商品）。
  - **任务恢复**：从后台返回时更新界面状态（如社交应用刷新未读消息数）。

#### 4. **生命周期关联**
- **调用顺序**：
  - `singleTop` 模式下栈顶复用：`onPause() → onNewIntent() → onResume()`。
  - `singleTask` 模式下栈内复用时：`onRestart() → onStart() → onNewIntent() → onResume()`。
- **内存回收场景**：若 Activity 被系统销毁后重建，`onNewIntent()` 不会调用，需在 `onCreate()` 和 `onNewIntent()` 中统一处理数据初始化。

#### 5. **最佳实践**
- **数据更新逻辑**：将 Intent 处理代码封装到独立方法（如 `handleNewIntent()`），同时在 `onCreate()` 和 `onNewIntent()` 中调用，确保数据一致性。
- **避免重复操作**：通过 `getIntent().getAction()` 判断 Intent 类型，避免重复执行相同逻辑。
- **权限校验**：对外部传入的 Intent 进行安全校验，防止恶意数据注入。

### 二、`onConfigurationChanged()` 方法
#### 1. **核心作用**
当设备配置（如屏幕方向、语言、键盘状态）发生变化时触发，允许开发者手动处理配置变更，避免 Activity 重新创建。

#### 2. **调用条件**
- **声明 `configChanges` 属性**：在 `AndroidManifest.xml` 的 `<activity>` 标签中声明需监听的配置类型，例如：
  ```xml
  <activity
      android:name=".MainActivity"
      android:configChanges="orientation|screenSize|keyboardHidden" />
  ```
  - **API 13 及以上**：屏幕旋转需同时声明 `orientation` 和 `screenSize`，否则系统仍会重启 Activity。
  - **分屏模式**：需添加 `screenLayout|smallestScreenSize` 以适配多窗口场景。
- **权限声明**：若需动态修改配置（如切换语言），需在 Manifest 中添加 `<uses-permission android:name="android.permission.CHANGE_CONFIGURATION" />`。

#### 3. **关键参数与使用方法**
- **参数 `newConfig`**：包含新的配置信息，可通过以下方式获取具体变化：
  ```java
  @Override
  public void onConfigurationChanged(Configuration newConfig) {
      super.onConfigurationChanged(newConfig); // 必须调用父类方法
      // 处理屏幕方向变化
      if (newConfig.orientation == Configuration.ORIENTATION_LANDSCAPE) {
          adjustLayoutForLandscape();
      } else {
          adjustLayoutForPortrait();
      }
      // 处理分屏状态
      if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
          boolean isSplitScreen = newConfig.windowConfiguration.getWindowingMode() ==
              WindowConfiguration.WINDOWING_MODE_SPLIT_SCREEN_PRIMARY;
          if (isSplitScreen) {
              resizeViewsForSplitScreen();
          }
      }
  }
  ```
- **资源更新**：通过 `getResources().getConfiguration()` 获取最新配置，手动加载适配资源（如不同方向的布局文件）。

#### 4. **生命周期关联**
- **配置变化时的调用顺序**：
  - 仅触发 `onConfigurationChanged()`，跳过 `onDestroy()` 和 `onCreate()`。
  - 若未声明 `configChanges`，系统会销毁并重建 Activity，依次调用 `onSaveInstanceState()` → `onDestroy()` → `onCreate()` → `onRestoreInstanceState()`。

#### 5. **最佳实践**
- **避免复杂逻辑**：该方法应仅处理与配置直接相关的 UI 调整，耗时操作（如网络请求）仍需在其他生命周期方法中执行。
- **兼容多设备**：针对折叠屏、平板等特殊设备，结合 `smallestScreenSize` 和 `screenLayout` 动态调整布局。
- **状态保存与恢复**：若需保留用户输入或临时状态，可在 `onSaveInstanceState()` 中保存，并在 `onConfigurationChanged()` 中恢复。

### 三、对比与关联场景
| **特性**                | `onNewIntent()`                                | `onConfigurationChanged()`                   |
|-------------------------|-----------------------------------------------|-----------------------------------------------|
| **触发条件**            | 启动模式为 `singleTop`/`singleTask` 或设置标志位 | 声明的配置变化且未重启 Activity               |
| **参数类型**            | `Intent`（携带新数据）                         | `Configuration`（携带新配置）                 |
| **核心目的**            | 更新 Activity 的 Intent 数据                  | 手动处理配置变化，避免 Activity 重建          |
| **典型场景**            | 处理通知、深度链接、任务恢复                  | 屏幕旋转、语言切换、分屏模式                  |

#### 1. **组合使用场景**
- **折叠屏适配**：当设备折叠状态变化时，可能同时触发配置变化和新 Intent（如从分屏模式切换回全屏时接收新数据）。此时需在 `onConfigurationChanged()` 中调整布局，并在 `onNewIntent()` 中更新数据。
- **多语言切换**：用户切换语言后，Activity 可能接收到新 Intent（如重新加载本地化内容），同时触发配置变化，需同时处理两种回调。

#### 2. **注意事项**
- **性能影响**：过度依赖 `onConfigurationChanged()` 可能导致代码复杂度增加，建议仅在必要时使用（如复杂动画或无法快速重建的 UI）。
- **兼容性问题**：部分配置变化（如 `locale`）即使声明 `configChanges`，系统仍可能重启 Activity，需结合 `onSaveInstanceState()` 确保状态正确恢复。

### 四、总结
- **`onNewIntent()`** 是启动模式相关的回调，用于复用 Activity 实例时更新数据，需配合 `setIntent()` 使用。
- **`onConfigurationChanged()`** 是配置变化的回调，通过声明 `configChanges` 避免 Activity 重建，需手动处理资源更新。
- **开发建议**：
  - 优先使用系统默认的生命周期流程（如屏幕旋转时重建 Activity），仅在性能敏感或特殊场景下覆盖默认行为。
  - 结合 **`onSaveInstanceState()`** 和 **`onRestoreInstanceState()`** 确保配置变化时状态不丢失。
  - 通过 **`adb shell dumpsys activity activities`** 命令调试 Activity 实例状态，验证启动模式和配置变化的处理逻辑。