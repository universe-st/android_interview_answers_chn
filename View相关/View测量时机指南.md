# 在onResume中是否可以测量宽高？

在Android开发中，**不建议在`onResume()`中直接测量View的宽高**，因为此时视图可能尚未完成布局（layout）和测量（measure）流程，直接获取宽高很可能得到`0`或不准确的值。


### 原因分析：
Android的Activity生命周期与View的绘制流程是异步的：
1. `onResume()`表示Activity处于可见可交互状态，但此时视图树（View Tree）可能还未完成测量（`onMeasure()`）和布局（`onLayout()`）。
2. View的宽高计算依赖于`onMeasure()`和`onLayout()`的执行，这两个方法在`onResume()`之后才会触发。


### 正确的测量时机：
如果需要在页面可见时获取View的宽高，推荐以下两种方式：

#### 1. 使用`ViewTreeObserver.OnGlobalLayoutListener`
监听视图树的布局完成事件，在布局完成后获取宽高：
```java
@Override
protected void onResume() {
    super.onResume();
    
    final View targetView = findViewById(R.id.target_view);
    ViewTreeObserver vto = targetView.getViewTreeObserver();
    vto.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
        @Override
        public void onGlobalLayout() {
            // 移除监听器，避免重复回调
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
                targetView.getViewTreeObserver().removeOnGlobalLayoutListener(this);
            } else {
                // 兼容低版本
                targetView.getViewTreeObserver().removeGlobalOnLayoutListener(this);
            }
            
            // 此时可正确获取宽高
            int width = targetView.getMeasuredWidth();
            int height = targetView.getMeasuredHeight();
        }
    });
}
```

#### 2. 使用`View.post()`
通过`post()`将测量任务添加到消息队列，等待View绘制完成后执行：
```java
@Override
protected void onResume() {
    super.onResume();
    
    final View targetView = findViewById(R.id.target_view);
    targetView.post(new Runnable() {
        @Override
        public void run() {
            // 此时View已完成绘制，可获取宽高
            int width = targetView.getWidth();
            int height = targetView.getHeight();
        }
    });
}
```


### 总结：
- `onResume()`中直接测量宽高不可靠，因为View可能尚未完成布局。
- 应通过`ViewTreeObserver`监听布局完成事件，或使用`View.post()`延迟执行测量任务，以确保获取准确的宽高。