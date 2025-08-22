# RequestLayout、onLayout、onDraw 和 drawChild 这几个方法的区别与联系

在Android开发中，`requestLayout()`、`onLayout()`、`onDraw()`和`drawChild()`是与View绘制流程密切相关的核心方法，它们分别作用于布局流程的不同阶段，共同支撑起View的测量、布局和绘制逻辑。以下从**作用、调用时机、使用场景**三个维度详细说明它们的区别与联系。


### 一、核心区别


#### 1. `requestLayout()`：触发布局流程的"请求"方法
- **作用**：通知系统当前View的布局参数（如宽高、位置）发生了变化，需要重新执行**测量（measure）** 和**布局（layout）** 流程，但**不一定触发绘制（draw）**。
- **调用时机**：当View的尺寸、位置依赖的条件改变时（如动态修改`layout_width`、`padding`等），开发者手动调用，或系统在布局参数变化时自动调用。
- **执行流程**：
  - 方法会向上传递请求至父容器，最终到达`ViewRootImpl`，触发整个视图树的`measure()`和`layout()`。
  - 仅当布局变化导致View的视觉外观改变时（如位置移动），才会间接触发`draw()`；若仅尺寸变化但视觉内容不变（如背景不变），可能不会触发绘制。
- **使用场景**：动态调整View的布局参数后，确保View能正确显示新的位置和尺寸（如旋转屏幕后重新布局）。


#### 2. `onLayout()`：布局阶段的"执行"方法
- **作用**：在布局阶段确定View自身或其子View的**最终位置**（通过设置`mLeft`、`mTop`、`mRight`、`mBottom`坐标）。
- **调用时机**：在`measure()`之后，由`layout()`方法触发（`layout()`是View的公开方法，内部会调用`onLayout()`）。
- **实现差异**：
  - 对于`View`：`onLayout()`是空实现（因为单个View无子类，无需摆放子View）。
  - 对于`ViewGroup`：`onLayout()`是抽象方法，**必须重写**，用于定义子View的摆放规则（如`LinearLayout`的水平/垂直排列、`RelativeLayout`的相对位置）。
- **使用场景**：自定义`ViewGroup`时，通过重写`onLayout()`控制子View的位置（如实现流式布局、网格布局）。


#### 3. `onDraw()`：绘制阶段的"内容绘制"方法
- **作用**：在绘制阶段绘制View的**自身内容**（如文本、图形、图片等），通过`Canvas`对象实现自定义绘制。
- **调用时机**：由`draw()`方法触发（`draw()`是View绘制的总入口），在`measure()`和`layout()`完成后执行。具体时机包括：
  - 首次加载View时。
  - 调用`invalidate()`（触发重绘）或`postInvalidate()`（主线程外触发重绘）时。
  - 布局变化导致视觉外观改变时（如位置移动后需要重绘）。
- **绘制内容**：默认会绘制背景、内容、滚动条等，开发者可重写该方法添加自定义绘制逻辑（如绘制圆形、路径等）。
- **使用场景**：自定义View时，通过`Canvas`实现独特的视觉效果（如进度条、仪表盘）。


#### 4. `drawChild()`：ViewGroup绘制子View的"代理"方法
- **作用**：`ViewGroup`专用方法，用于**绘制其子View**，本质是代理调用子View的`draw()`方法。
- **调用时机**：在`ViewGroup`的`dispatchDraw()`方法中触发（`dispatchDraw()`是ViewGroup绘制子View的核心方法），遍历所有子View并调用`drawChild()`。
- **特殊能力**：`ViewGroup`可通过重写`drawChild()`控制子View的绘制过程，例如：
  - 改变子View的绘制顺序（默认按添加顺序绘制）。
  - 对子View的绘制结果进行裁剪、滤镜等处理。
  - 跳过某个子View的绘制（返回`false`）。
- **使用场景**：自定义`ViewGroup`时，需要控制子View的绘制逻辑（如实现子View重叠时的层级优先级）。


### 二、联系与整体流程


这四个方法均属于Android View的**绘制流水线**（measure → layout → draw），彼此依赖、协同工作，整体流程如下：

1. **触发阶段**：  
   当View布局参数变化时，`requestLayout()`被调用，触发整个视图树的`measure()`（测量尺寸）和`layout()`（确定位置）。

2. **布局阶段**：  
   `layout()`方法内部调用`onLayout()`，`ViewGroup`通过`onLayout()`确定子View的位置，完成布局。

3. **绘制阶段**：  
   布局完成后，`draw()`方法触发绘制流程：  
   - 先绘制View自身背景。  
   - 调用`onDraw()`绘制View自身内容。  
   - 对于`ViewGroup`，调用`dispatchDraw()`，内部通过`drawChild()`遍历绘制所有子View。  
   - 最后绘制前景、滚动条等。


### 三、关键总结

| 方法           | 核心作用                  | 所属阶段   | 调用方                | 典型场景                     |
|----------------|---------------------------|------------|-----------------------|------------------------------|
| `requestLayout()` | 触发重新测量和布局        | 触发阶段   | 开发者/系统           | 动态修改布局参数后           |
| `onLayout()`      | 确定自身/子View的位置     | 布局阶段   | 系统（由`layout()`调用）| 自定义`ViewGroup`摆放子View  |
| `onDraw()`        | 绘制自身内容              | 绘制阶段   | 系统（由`draw()`调用） | 自定义View的视觉效果         |
| `drawChild()`     | 代理绘制子View            | 绘制阶段   | `ViewGroup`自身       | 控制子View的绘制顺序或效果   |

简言之：`requestLayout()`是"发起布局请求"的开关，`onLayout()`是"执行布局"的实现，`onDraw()`是"绘制自身"的核心，`drawChild()`是"ViewGroup绘制子View"的代理，四者共同保证View从尺寸测量、位置确定到视觉呈现的完整流程。