# View的绘制流程是从Activity的哪个生命周期方法开始执行的？

在Android中，**View的绘制流程并非直接由Activity的某个生命周期方法内部执行**，而是在Activity生命周期推进到特定阶段后，由系统触发的。

具体来说，关键节点是：  
当Activity执行完`onResume()`方法后，系统会将Activity的根视图（DecorView）添加到Window中，并通过`ViewRootImpl`与DecorView建立关联。随后，`ViewRootImpl`会调用`performTraversals()`方法，该方法会依次触发View的**measure（测量）、layout（布局）、draw（绘制）** 流程，完成View的最终渲染。


因此，从Activity生命周期的角度看，View的绘制流程是在`onResume()`方法执行之后，由系统间接触发的。`onResume()`标志着Activity进入前台并可见，为后续的View绘制提供了前提条件。