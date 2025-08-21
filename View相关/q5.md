# 自定义View与自定义ViewGroup有哪些区别？

在Android中，自定义View与自定义ViewGroup的核心区别在于**职责定位**和**功能侧重**，具体可从以下几个方面区分：


### 1. **核心职责不同**
- **自定义View**：  
  主要负责**自身的绘制与交互**，本质是“单个UI元素”。  
  例如：自定义一个圆形进度条、特殊形状的按钮、自定义图表等，核心是通过`onDraw()`方法绘制具体的视觉效果，并处理自身的触摸事件（如`onTouchEvent()`）。

- **自定义ViewGroup**：  
  主要负责**管理子View**（包含其他View或ViewGroup），本质是“容器”。  
  例如：自定义流式布局（子View自动换行）、卡片式布局等，核心是通过测量子View大小（`onMeasure()`）、确定子View位置（`onLayout()`）来实现特定的布局规则，同时协调子View的交互。


### 2. **继承关系与父类不同**
- **自定义View**：  
  通常继承自`View`或其直接子类（如`TextView`、`ImageView`），聚焦于单个元素的特性扩展。

- **自定义ViewGroup**：  
  通常继承自`ViewGroup`或其容器子类（如`LinearLayout`、`RelativeLayout`），具备管理子View的能力。


### 3. **测量与布局流程不同**
- **自定义View**：  
  只需在`onMeasure()`中根据父容器的约束（`MeasureSpec`）计算自身的宽高，无需关心子View；  
  `onLayout()`方法对其几乎无意义（因无子女），通常无需重写。

- **自定义ViewGroup**：  
  - `onMeasure()`：不仅要计算自身宽高，还必须遍历所有子View，调用`child.measure()`让子View测量自身大小，再根据子View的测量结果和布局规则（如排列方式）确定自身最终大小（例如`wrap_content`时需累加子View尺寸）。  
  - `onLayout()`：必须重写，负责调用每个子View的`layout()`方法，明确子View在父容器中的位置（`left/top/right/bottom`），这是容器的核心职责。


### 4. **事件处理机制不同**
- **自定义View**：  
  主要通过`onTouchEvent()`处理自身的触摸事件（如点击、滑动），无需考虑事件分发。

- **自定义ViewGroup**：  
  除了自身的`onTouchEvent()`，还需处理**事件分发逻辑**：  
  通过`onInterceptTouchEvent()`决定是否拦截事件（不传给子View），或通过`dispatchTouchEvent()`将事件分发给子View。这是容器特有的能力，用于协调子View与自身的事件响应。


### 总结
- **自定义View**：“画好自己，处理自己的交互”，是UI的“叶子节点”。  
- **自定义ViewGroup**：“管好子女，安排好布局”，是UI的“枝干节点”，本质是View的子类，但多了管理子View的核心能力。