# onSaveInstanceState() 和 onRestoreInstanceState()

在 Android 开发中，`onSaveInstanceState()` 和 `onRestoreInstanceState()` 是两个关键的生命周期回调方法，用于处理 **Activity 状态的保存与恢复**，尤其在系统因内存不足或配置变更（如屏幕旋转）导致 Activity 被销毁重建时至关重要。以下是基于系统机制和实际开发场景的深度解析：

### 一、核心作用与触发条件
#### 1. **`onSaveInstanceState()`**
- **作用**：在 Activity 即将被系统销毁前，将临时状态数据保存到 `Bundle` 中，以便后续重建时恢复。
- **触发条件**：
  - **系统主动销毁**：当系统因内存不足、用户按下 HOME 键、启动新 Activity 或屏幕旋转等原因可能销毁 Activity 时触发。
  - **配置变更**：屏幕旋转、语言切换等场景下，系统会先销毁当前 Activity 再重建，此时必定调用该方法。
  - **不触发场景**：用户主动按返回键退出或调用 `finish()` 销毁 Activity 时，系统认为无需恢复状态，因此不调用。

#### 2. **`onRestoreInstanceState()`**
- **作用**：在 Activity 重建后，从 `Bundle` 中恢复之前保存的状态数据。
- **触发条件**：
  - **仅在重建时调用**：只有当 Activity 被系统销毁后重新创建时才会触发，例如屏幕旋转后重建或进程被回收后恢复。
  - **调用时机**：在 `onStart()` 之后、`onPostCreate()` 之前执行，此时所有 View 已初始化完毕，可安全访问界面组件。

### 二、关键参数与默认行为
#### 1. **参数 `Bundle`**
- **数据载体**：通过 `Bundle` 的键值对存储状态数据，支持基本类型、`Parcelable` 对象及 `Serializable` 对象（不推荐）。
  ```java
  @Override
  protected void onSaveInstanceState(Bundle outState) {
      super.onSaveInstanceState(outState); // 必须调用父类，否则系统自动保存的 View 状态会丢失
      outState.putInt("current_score", currentScore); // 保存自定义数据
  }
  ```

#### 2. **系统自动保存的 View 状态**
- **默认实现**：
  - 系统会自动保存所有设置了 `android:id` 的 View 的状态，如 `EditText` 的输入内容、`CheckBox` 的选中状态等。
  - 若未设置 `android:id`，View 的状态将无法自动保存，需手动处理。
- **禁用自动保存**：在 View 中设置 `android:saveEnabled="false"` 可关闭自动保存。

#### 3. **自定义 View 的状态处理**
- 若需保存自定义 View 的状态，需重写其 `onSaveInstanceState()` 和 `onRestoreInstanceState()` 方法：
  ```java
  @Override
  protected Parcelable onSaveInstanceState() {
      Bundle bundle = new Bundle();
      bundle.putParcelable("super_state", super.onSaveInstanceState());
      bundle.putInt("custom_value", customValue);
      return bundle;
  }

  @Override
  protected void onRestoreInstanceState(Parcelable state) {
      if (state instanceof Bundle) {
          Bundle bundle = (Bundle) state;
          customValue = bundle.getInt("custom_value");
          super.onRestoreInstanceState(bundle.getParcelable("super_state"));
          return;
      }
      super.onRestoreInstanceState(state);
  }
  ```

### 三、生命周期关联与调用顺序
#### 1. **正常销毁与重建流程**
- **屏幕旋转时的调用顺序**：
  ```
  onPause() → onSaveInstanceState() → onStop() → onDestroy() → onCreate() → onStart() → onRestoreInstanceState() → onResume()
  ```
- **内存不足导致的进程回收**：
  ```
  onPause() → onStop() → onSaveInstanceState() → onDestroy() → onCreate() → onStart() → onRestoreInstanceState() → onResume()
  ```

#### 2. **与 `onCreate()` 的对比**
| **特性**               | `onCreate()`                          | `onRestoreInstanceState()`          |
|------------------------|---------------------------------------|---------------------------------------|
| **调用时机**           | Activity 首次创建或重建时            | 仅在 Activity 重建时调用             |
| **`savedInstanceState` 状态** | 可能为 `null`（首次创建）           | 必定非 `null`（仅在重建时调用）     |
| **数据恢复场景**       | 需先判断 `savedInstanceState != null` | 直接使用 `savedInstanceState`       |
| **推荐场景**           | 初始化 View 和非状态依赖的逻辑       | 恢复复杂 UI 状态或依赖 View 的数据   |

#### 3. **最佳实践**
- **优先使用 `onRestoreInstanceState()`**：由于该方法在 View 初始化后调用，可避免空指针问题，尤其适合恢复列表滚动位置、复杂布局状态等。
- **统一处理数据恢复**：将恢复逻辑封装到独立方法，同时在 `onCreate()` 和 `onRestoreInstanceState()` 中调用，确保数据一致性：
  ```java
  private void restoreState(Bundle savedInstanceState) {
      if (savedInstanceState != null) {
          currentScore = savedInstanceState.getInt("current_score");
          // 其他恢复逻辑
      }
  }

  @Override
  protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      restoreState(savedInstanceState);
  }

  @Override
  protected void onRestoreInstanceState(Bundle savedInstanceState) {
      super.onRestoreInstanceState(savedInstanceState);
      restoreState(savedInstanceState);
  }
  ```

### 四、数据存储策略与注意事项
#### 1. **适用场景与限制**
- **适合保存的数据**：
  - 瞬态 UI 状态（如输入框内容、复选框状态）。
  - 临时变量（如当前页码、列表滚动位置）。
- **不适合保存的数据**：
  - 大量数据（如 Bitmap）：可能导致内存泄漏或性能问题，建议使用 `ViewModel` 或文件存储。
  - 持久化数据（如用户设置）：应使用 `SharedPreferences` 或数据库（如 Room）。

#### 2. **避免重复保存**
- **配置变更时的优化**：在 `AndroidManifest.xml` 中声明 `android:configChanges`，可阻止 Activity 重建并直接触发 `onConfigurationChanged()`，减少状态保存开销：
  ```xml
  <activity
      android:name=".MainActivity"
      android:configChanges="orientation|screenSize" />
  ```
  - 此时需在 `onConfigurationChanged()` 中手动处理布局调整和数据更新。

#### 3. **测试与调试**
- **模拟进程被杀**：使用 `adb` 命令强制杀死应用进程，验证状态恢复逻辑：
  ```bash
  adb shell am kill com.example.app
  ```
- **验证 View 状态**：通过 `adb shell dumpsys activity activities` 查看 Activity 实例状态，确认 `savedInstanceState` 是否正确保存。

### 五、关联场景与高级技巧
#### 1. **Fragment 状态保存**
- Fragment 的 `onSaveInstanceState()` 与 Activity 类似，但需注意：
  - `hide()` 和 `show()` 状态不会自动保存，需通过 `setRetainInstance(true)` 保留实例（但无法加入返回栈）。
  - 复杂状态建议结合 `ViewModel` 或 `SavedStateHandle` 处理。

#### 2. **多进程与跨进程通信**
- 若 Activity 运行在多进程模式（`android:process`），需确保 `savedInstanceState` 中的数据在跨进程时可正确序列化。
- 对于跨进程传递的状态，建议使用 `Parcelable` 而非 `Serializable`，以提升性能。

#### 3. **内存泄漏防范**
- 避免在 `savedInstanceState` 中保存持有 Activity 上下文的对象（如 `Drawable`、`Handler`），防止内存泄漏。

### 六、总结与开发建议
| **核心要点**                | **操作指南**                                                                 |
|-----------------------------|-----------------------------------------------------------------------------|
| **数据保存**                | 仅保存瞬态状态，调用 `super.onSaveInstanceState()` 保留系统自动保存的 View 状态 |
| **数据恢复**                | 在 `onRestoreInstanceState()` 中处理复杂 UI 恢复，避免在 `onCreate()` 中依赖非空判断 |
| **性能优化**                | 避免保存大量数据，结合 `ViewModel` 和持久化存储处理复杂场景                     |
| **兼容性测试**              | 测试屏幕旋转、多窗口模式、低内存等场景，确保状态正确恢复                       |

通过合理使用这两个方法，开发者可确保应用在异常销毁后仍能无缝恢复用户操作，提升用户体验。同时，结合 `ViewModel`、`LiveData` 等架构组件，可进一步简化状态管理逻辑，实现更健壮的应用架构。