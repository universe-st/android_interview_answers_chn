# Android开发中，自定义View执行invalidate()方法,为什么有时候不会回调onDraw()？

在Android开发中，调用`invalidate()`后`onDraw()`未被回调，通常与View的状态、绘制流程或调用环境有关，主要原因如下：


### 1. **View不可见（Visibility影响）**
当View的可见性为`GONE`或`INVISIBLE`时，系统会跳过其绘制流程：
- `GONE`：View完全不可见且不占用布局空间，调用`invalidate()`不会触发重绘。
- `INVISIBLE`：View不可见但占用空间，此时系统也会优化掉绘制步骤（因为绘制了也看不到）。

**解决**：确保View可见（`VISIBLE`）时再调用`invalidate()`。


### 2. **View尺寸为0（无绘制区域）**
如果View的宽高均为0（如`layout_width`/`layout_height`设为`0dp`，或未经过正确的`layout`计算），此时没有可绘制的区域，`invalidate()`不会触发`onDraw()`。

**解决**：检查View的布局参数和测量逻辑，确保`onMeasure()`正确计算了宽高，且`layout`后有实际尺寸。


### 3. **在非UI线程调用`invalidate()`**
`invalidate()`必须在**UI线程**调用，若在子线程调用，系统不会处理重绘请求，自然不会触发`onDraw()`。（子线程需用`postInvalidate()`替代，它会自动切换到UI线程）

**错误示例**：
```java
new Thread(() -> {
    // 子线程调用invalidate()无效
    myCustomView.invalidate(); 
}).start();
```

**正确示例**：
```java
new Thread(() -> {
    // 子线程使用postInvalidate()
    myCustomView.postInvalidate(); 
}).start();
```


### 4. **无效化区域为空或超出View范围**
`invalidate()`有多个重载方法，若调用的是带区域参数的`invalidate(Rect)`或`invalidate(int l, int t, int r, int b)`，且指定的区域：
- 为空（如`l >= r`或`t >= b`）；
- 完全超出View的边界（如区域在View之外）。

系统会判断“无需重绘”，因此不触发`onDraw()`。

**解决**：确保无效化区域在View的`[0,0,width,height]`范围内，且区域有效（`l < r && t < b`）。


### 5. **View尚未附加到视图树**
若View未经过`onAttachedToWindow()`（即未添加到`Window`中），此时View不在绘制流程中，`invalidate()`无法触发重绘。

常见场景：
- View刚创建但未通过`addView()`添加到父容器；
- View被`removeView()`移除后调用`invalidate()`。

**解决**：确保`invalidate()`在View附加到视图树后调用（可在`onAttachedToWindow()`中做标记判断）。


### 6. **硬件加速与图层缓存影响**
当View开启硬件加速（默认开启）时，若绘制逻辑涉及不支持硬件加速的操作，或View被缓存到独立图层（如通过`setLayerType()`设置），可能导致`invalidate()`无法正确触发重绘。

**解决**：
- 尝试关闭硬件加速（`setLayerType(View.LAYER_TYPE_SOFTWARE, null)`）排查是否与此相关；
- 若使用图层缓存，需配合`invalidate()`手动更新缓存（如`setLayerType()`后调用`invalidate()`）。


### 7. **父容器拦截绘制请求**
某些特殊的父容器（如自定义`ViewGroup`）可能重写了`onInterceptDraw()`、`dispatchDraw()`等方法，或设置了`willNotDraw = true`（表示不需要绘制自身，仅绘制子View），可能间接影响子View的重绘触发。

**解决**：检查父容器的绘制逻辑，确保未拦截子View的重绘请求。


### 总结
排查时可按以下步骤验证：
1. 确认View处于`VISIBLE`状态且有实际尺寸；
2. 确保`invalidate()`在UI线程调用（或用`postInvalidate()`）；
3. 检查无效化区域是否有效且在View范围内；
4. 确认View已添加到视图树中。

通过以上排查，通常能定位`onDraw()`未回调的原因。