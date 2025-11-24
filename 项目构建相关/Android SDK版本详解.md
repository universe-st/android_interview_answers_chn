# minSdkVersion、compileSdkVersion、targetSdkVersion各有什么作用？

在Android开发中，`minSdkVersion`、`compileSdkVersion`、`targetSdkVersion` 是控制应用兼容性和行为的三个核心配置参数，它们的作用和影响各不相同：


### 1. `minSdkVersion`（最小SDK版本）
- **作用**：指定应用能够运行的**最低Android系统版本**（以API级别表示，如Android 5.0对应API 21）。
- **影响**：
  - 低于此版本的设备无法安装或运行该应用（Google Play会直接过滤掉不兼容设备）。
  - 开发时，编译器会检查代码中是否使用了高于`minSdkVersion`的API，如果直接使用未做兼容处理，会报错提醒。
- **场景**：决定应用的兼容性范围。例如，若`minSdkVersion=21`，则应用无法在Android 4.4（API 19）及以下设备上运行。


### 2. `compileSdkVersion`（编译SDK版本）
- **作用**：指定编译应用时使用的**Android SDK版本**（即使用该版本的API和工具链进行编译）。
- **影响**：
  - 决定了编译时可访问的API范围（只能使用该版本及以下的API，除非通过兼容库扩展）。
  - 不影响应用的运行时行为，仅影响编译阶段（例如，若使用了`compileSdkVersion=33`的新API，需确保在低版本设备上有降级处理）。
- **最佳实践**：建议使用**最新的稳定版本**（如当前最新的API 34），以支持最新的语言特性和编译优化，同时确保代码兼容性。


### 3. `targetSdkVersion`（目标SDK版本）
- **作用**：指定应用**针对哪个Android版本进行了优化适配**（以API级别表示）。
- **影响**：
  - 决定了应用在高版本系统上的**行为模式**。Android系统会根据此版本判断是否启用新特性或行为变更。
  - 例如：若`targetSdkVersion=33`（Android 13），则应用会遵循Android 13的权限机制（如通知权限需显式申请）；若`targetSdkVersion<33`，即使在Android 13设备上，系统也会兼容旧的权限行为。
- **最佳实践**：应尽可能接近`compileSdkVersion`（通常相等），并随着系统版本更新而升级，以确保应用在新系统上表现符合用户预期。


### 三者关系总结
- `minSdkVersion ≤ targetSdkVersion ≤ compileSdkVersion`（编译版本必须≥目标版本，目标版本必须≥最低版本）。
- `minSdkVersion`控制兼容性下限，`compileSdkVersion`控制编译环境，`targetSdkVersion`控制运行时行为适配。

合理配置这三个参数，可以在兼容性、功能支持和系统适配之间取得平衡。