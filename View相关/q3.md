# 请详细讲解Android中View的绘制原理。

在Android中，View的绘制是一个由**ViewRootImpl**协调的、递归遍历整个View树的过程，核心分为三大步骤：**测量（measure）**、**布局（layout）**、**绘制（draw）**。这三个步骤依次执行，最终将View树渲染到屏幕上。


### 一、绘制的触发起点：ViewRootImpl
所有View的绘制都从**ViewRootImpl**开始。ViewRootImpl是连接WindowManager和View树的桥梁，当需要刷新界面时（如首次加载、调用`invalidate()`、屏幕旋转等），ViewRootImpl会调用其`performTraversals()`方法，依次触发measure、layout、draw三个流程。

`performTraversals()`的简化逻辑如下：
```java
private void performTraversals() {
    // 1. 触发测量（measure）
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    // 2. 触发布局（layout）
    performLayout(lp, mWidth, mHeight);
    // 3. 触发绘制（draw）
    performDraw();
}
```


### 二、第一步：测量（measure）—— 确定View的宽高
**作用**：计算View的测量宽高（`measuredWidth`、`measuredHeight`），为后续布局提供尺寸依据。  
测量是一个**递归过程**：从顶层ViewGroup开始，依次测量所有子View，最终确定每个View的大小。


#### 1. 核心工具：MeasureSpec
测量时，父View会通过**MeasureSpec**（测量规格）限制子View的大小。MeasureSpec由**模式（Mode）** 和**尺寸（Size）** 组成（32位int值，高2位为模式，低30位为尺寸）。

- **模式（Mode）**：
  - `EXACTLY`：精确尺寸。父View已确定子View的大小（如`match_parent`或固定值`100dp`），子View必须使用该尺寸。
  - `AT_MOST`：最大尺寸。子View的大小不能超过父View给出的最大限制（如`wrap_content`），子View可自行决定大小，但不能超过上限。
  - `UNSPECIFIED`：无限制。父View不限制子View的大小（如ScrollView的子View，可无限扩展）。

- **MeasureSpec的生成**：  
  父View的MeasureSpec + 子View的`LayoutParams`（如`layout_width`）共同决定子View的MeasureSpec。例如：  
  - 父View是`EXACTLY(400dp)`，子View是`match_parent` → 子View的MeasureSpec为`EXACTLY(400dp)`。  
  - 父View是`EXACTLY(400dp)`，子View是`wrap_content` → 子View的MeasureSpec为`AT_MOST(400dp)`。  


#### 2. 测量的执行流程
- **View的measure()方法**：  
  触发测量的入口，由父View调用。View的`measure()`会调用`onMeasure()`（需要子类重写），在`onMeasure()`中计算测量宽高，最终通过`setMeasuredDimension()`保存结果（必须调用，否则会报错）。

  普通View（如TextView）的默认`onMeasure()`逻辑：使用`getDefaultSize()`根据MeasureSpec计算宽高（例如`wrap_content`在`AT_MOST`模式下会使用自身内容的大小）。

- **ViewGroup的measure()方法**：  
  ViewGroup除了测量自身，还需遍历所有子View，为每个子View生成MeasureSpec，然后调用子View的`measure()`方法完成递归测量。

  不同ViewGroup（如LinearLayout、RelativeLayout）的测量逻辑不同：  
  - 例如LinearLayout会根据方向（水平/垂直）计算剩余空间，依次为子View分配MeasureSpec。  
  - 复杂布局（如RelativeLayout）可能需要多次测量子View（因为子View的位置可能相互依赖）。  


### 三、第二步：布局（layout）—— 确定View的位置
**作用**：根据测量得到的宽高，确定View在父容器中的具体位置（左上角和右下角坐标）。  
布局也是**递归过程**：从顶层ViewGroup开始，依次确定每个子View的位置。


#### 1. 布局的核心参数
每个View的位置由4个参数决定：`left`（左边界，相对于父View）、`top`（上边界）、`right`（右边界）、`bottom`（下边界）。  
宽高与位置的关系：`width = right - left`，`height = bottom - top`。


#### 2. 布局的执行流程
- **View的layout()方法**：  
  由父View调用，接收`left`、`top`、`right`、`bottom`四个参数，通过`setFrame()`方法保存位置，然后调用`onLayout()`（空实现，供子类重写）。

- **ViewGroup的onLayout()方法**：  
  ViewGroup的`onLayout()`是抽象方法，必须由子类实现（因为不同布局的子View摆放逻辑不同）。在`onLayout()`中，ViewGroup会根据自身布局规则（如LinearLayout的水平排列、RelativeLayout的相对位置）计算每个子View的`left`、`top`、`right`、`bottom`，然后调用子View的`layout()`方法完成递归布局。

  例如：LinearLayout（垂直方向）会依次累加子View的高度，为每个子View设置`top`为上一个子View的`bottom`，从而实现垂直排列。


### 四、第三步：绘制（draw）—— 将View绘制到屏幕
**作用**：根据测量的宽高和布局的位置，将View的内容（背景、文本、图像等）绘制到屏幕上。  
绘制同样是**递归过程**：从顶层View开始，依次绘制自身和子View。


#### 1. 绘制的核心流程（View.draw()方法定义）
`draw()`方法规定了绘制的固定顺序（不可修改），步骤如下：
1. **绘制背景**：`background.draw(canvas)`（绘制View的背景，如`background`属性设置的Drawable）。  
2. **绘制自身内容**：`onDraw(canvas)`（子类重写，绘制View的主体内容，如TextView的文字、ImageView的图片）。  
3. **绘制子View**：`dispatchDraw(canvas)`（ViewGroup重写，遍历子View并调用其`draw()`方法，实现子View的绘制）。  
4. **绘制装饰**：`onDrawForeground(canvas)`（绘制前景，如滚动条、前景色、高程阴影等）。  


#### 2. 关键细节
- **Canvas（画布）**：绘制的载体，提供绘制图形（矩形、圆形）、文本、位图等方法。每个View的Canvas会被裁剪到自身的边界（`left`、`top`、`right`、`bottom`），避免绘制到其他区域。  
- **硬件加速**：默认开启，绘制操作会转换为GPU命令执行（效率更高），但部分绘制操作（如`Canvas.clipPath()`）不支持硬件加速，需通过`setLayerType(LAYER_TYPE_SOFTWARE, null)`关闭。  
- **绘制优化**：若View无需绘制（如完全透明），可重写`isOpaque()`返回`true`，减少绘制步骤；避免在`onDraw()`中创建对象（会触发GC，影响性能）。  


### 五、三个步骤的关联与总结
- **measure** → 确定“多大”（`measuredWidth`/`measuredHeight`）。  
- **layout** → 确定“在哪”（`left`/`top`/`right`/`bottom`），并得到最终宽高（`width`/`height`，通常与测量宽高一致）。  
- **draw** → 确定“画什么”，根据大小和位置将内容渲染到屏幕。  

整个过程从ViewRootImpl开始，递归遍历View树，任何一步的改变（如调用`requestLayout()`触发重新测量和布局，`invalidate()`触发重新绘制）都会重新执行对应步骤，最终保证界面的正确显示。


**关键点**：  
- 测量依赖MeasureSpec（父View限制+子View布局参数）。  
- 布局是ViewGroup摆放子View的核心逻辑。  
- 绘制有固定顺序，需避免不必要的绘制开销。  
- 所有操作默认在主线程执行，耗时会导致ANR。