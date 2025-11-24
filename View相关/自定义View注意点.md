# 自定义View有哪些需要注意的点？

在Android中自定义View时，需要关注多个细节以确保功能正确、性能优良且兼容性良好。以下是核心注意点：


### 1. **构造函数的正确实现**  
自定义View需覆盖不同参数的构造函数，以支持代码创建和XML布局引用：  
- `View(Context context)`：用于代码直接创建。  
- `View(Context context, AttributeSet attrs)`：用于XML布局（必须实现，否则XML引用会崩溃）。  
- `View(Context context, AttributeSet attrs, int defStyleAttr)`：支持默认样式。  
- `View(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes)`：（API 21+）支持默认样式资源。  

**注意**：在构造函数中初始化画笔（Paint）、属性等，避免在`onDraw`中重复初始化。


### 2. **测量（onMeasure）的处理**  
`onMeasure`决定View的宽高，需正确处理`MeasureSpec`的三种模式（`EXACTLY`/`AT_MOST`/`UNSPECIFIED`）：  
- **支持`wrap_content`**：默认情况下，`wrap_content`会等同于`match_parent`（继承自`View`的默认实现）。需手动计算内容尺寸（如文本宽高、绘制区域大小），作为`wrap_content`时的默认值。  
- **处理父容器限制**：根据`MeasureSpec`的`size`和`mode`，结合自身内容尺寸，通过`setMeasuredDimension(width, height)`设置最终测量值。  


### 3. **布局（onLayout）的处理（ViewGroup专属）**  
自定义`ViewGroup`需重写`onLayout`，确定子View的位置：  
- 计算子View的`left/top/right/bottom`坐标时，需考虑父容器的`padding`和子View的`margin`，避免布局偏移。  
- 调用子View的`layout(l, t, r, b)`方法完成定位，确保子View显示在正确区域。  


### 4. **绘制（onDraw）的优化**  
`onDraw`是自定义View的核心，但频繁调用（如刷新）可能影响性能：  
- **避免创建对象**：不在`onDraw`中新建`Paint`、`Rect`等对象，否则会触发频繁GC。  
- **减少过度绘制（Overdraw）**：  
  - 移除不必要的背景（如父容器和子View背景重叠）。  
  - 使用`canvas.clipRect()`限制绘制区域，只绘制可见部分。  
- **画笔（Paint）优化**：初始化时设置`setAntiAlias(true)`（抗锯齿）、`setDither(true)`（防抖动）等，避免在`onDraw`中重复设置。  


### 5. **自定义属性的规范使用**  
通过`attrs.xml`定义属性，需注意：  
- 在`res/values/attrs.xml`中声明属性（如`dimension`/`color`/`boolean`），避免类型错误。  
- 在构造函数中通过`TypedArray`获取属性后，**必须调用`typedArray.recycle()`回收**，否则内存泄漏。  


### 6. **事件处理与冲突**  
处理触摸事件（`onTouchEvent`）或手势（`GestureDetector`）时：  
- 区分`getX()`（相对当前View的坐标）和`getRawX()`（相对屏幕的坐标），避免坐标计算错误。  
- 解决事件冲突：父View与子View事件冲突时，可通过`requestDisallowInterceptTouchEvent(true)`阻止父View拦截事件。  


### 7. **性能优化**  
- **减少刷新频率**：避免不必要的`invalidate()`/`requestLayout()`，局部刷新可用`invalidate(Rect)`。  
- **硬件加速**：对动画等场景启用硬件加速（默认开启），但注意部分`Canvas`方法（如`clipPath`）在硬件加速下不支持，需针对性关闭（`setLayerType(LAYER_TYPE_SOFTWARE, null)`）。  
- **缓存复用**：对不变的内容（如静态文本）使用`setWillNotDraw(true)`关闭绘制，或通过`setLayerType`启用图层缓存。  


### 8. **状态保存与恢复**  
当View因配置变化（如屏幕旋转）重建时，需保存关键状态：  
- 重写`onSaveInstanceState()`返回`Parcelable`（如自定义`SavedState`），保存数据。  
- 重写`onRestoreInstanceState(Parcelable state)`恢复数据，避免状态丢失（如计数器数值、选中状态）。  


### 9. **兼容性处理**  
- **API版本适配**：高版本API（如`ViewOutlineProvider`）需用`Build.VERSION.SDK_INT`判断，避免低版本崩溃。  
- **屏幕适配**：使用`dp`/`sp`而非`px`，通过`TypedValue.applyDimension()`将`dp`转为`px`，确保不同密度设备显示一致。  


### 10. **封装与接口设计**  
对外提供清晰的`setter`/`getter`方法（如`setTitle(String)`），内部修改属性后调用`invalidate()`刷新，避免外部直接操作内部变量，保证状态一致性。  


### 11. **避免耗时操作**  
`onMeasure`/`onLayout`/`onDraw`运行在UI线程，禁止执行耗时操作（如IO、复杂计算），需将耗时逻辑放到子线程，完成后通过`Handler`通知UI刷新。  


### 12. **测试验证**  
- 在不同Android版本、屏幕尺寸的设备上测试显示和交互。  
- 检查内存泄漏（如未回收的`TypedArray`、`Handler`持有Activity引用）。  


通过关注以上细节，可确保自定义View功能稳定、性能优异且兼容性强。