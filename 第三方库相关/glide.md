### **Android Glide 面试问题与答案大全**

#### **一、基础概念与配置**

**1. Glide 是什么？它有哪些核心优势？**
**答：** Glide 是一个由 Bumptech 开发的开源、高效、强大的 Android 媒体管理和图片加载框架。
**核心优势：**
*   **易于使用：** 链式调用，API 简洁明了。
*   **性能优异：** 支持智能缓存（内存、磁盘）、自动压缩和复用，有效减少 OOM。
*   **生命周期集成：** 与 Activity/Fragment 生命周期绑定，自动管理请求，防止内存泄漏。
*   **功能丰富：** 支持加载图片、GIF、视频缩略图，支持各种变换（裁剪、圆角等）。
*   **支持 OkHttp/Volley 集成：** 可以替换底层的网络栈，更好地与现代网络库配合。

**2. 如何在项目中引入 Glide？**
**答：** 在 `app/build.gradle` 文件中添加依赖：
```gradle
dependencies {
    implementation 'com.github.bumptech.glide:glide:4.16.0' // 请使用最新版本
    annotationProcessor 'com.github.bumptech.glide:compiler:4.16.0' // 注解处理器，用于生成 Generated API
}
```
如果需要使用 OkHttp 作为网络层，还可以添加集成库：
```gradle
dependencies {
    implementation 'com.github.bumptech.glide:okhttp3-integration:4.16.0'
}
```

**3. 最简单的 Glide 加载图片代码是怎样的？**
**答：**
```java
Glide.with(context)
     .load(url) // 可以是 URL, URI, 资源ID, File等
     .into(imageView);
```

#### **二、核心方法与流程**

**4. `Glide.with()` 方法的作用是什么？它有哪些重载？**
**答：** 该方法用于创建一个加载请求的实例，并开始管理与传入参数的生命周期绑定。
**重载形式：**
*   `with(Activity activity)`
*   `with(Fragment fragment)`
*   `with(FragmentActivity activity)`
*   `with(Context context)` - 如果传入的是 Application Context，请求将只与应用的生命周期绑定，不会在页面销毁时自动暂停/清除。

**5. `.load()` 方法可以接受哪些参数类型？**
**答：** 非常灵活，支持多种数据源：
*   `String` (网络 URL)
*   `Uri`
*   `Integer` (资源 ID，如 `R.drawable.my_image`)
*   `File`
*   `byte[]`
*   `Drawable`
*   等等。

**6. `.into()` 方法除了接受 ImageView，还能接受什么？**
**答：** 除了 `ImageView`，还可以接受自定义的 `Target`（目标）。`Target` 是一个接口，允许你在图片加载完成、失败或准备好时获得回调，从而进行更复杂的操作，例如将图片设置到自定义 View 或先进行一些处理再显示。
```java
Glide.with(context)
     .load(url)
     .into(new CustomTarget<Drawable>() {
         @Override
         public void onResourceReady(@NonNull Drawable resource, @Nullable Transition<? super Drawable> transition) {
             // 在这里处理加载成功的图片
             myCustomView.setDrawable(resource);
         }

         @Override
         public void onLoadCleared(@Nullable Drawable placeholder) {
             // 当请求被清除时调用
         }
     });
```

#### **三、占位符与错误处理**

**7. 如何设置加载中的占位符和加载失败的错误占位符？**
**答：** 使用 `placeholder()` 和 `error()` 方法。
```java
Glide.with(context)
     .load(url)
     .placeholder(R.drawable.placeholder) // 加载中显示
     .error(R.drawable.error) // 加载失败显示
     .into(imageView);
```
**注意：** `placeholder` 占位图也会在从磁盘缓存加载时显示，因为磁盘读取也可能需要时间。

**8. 还有 `fallback()` 方法，它和 `error()` 有什么区别？**
**答：**
*   `error()`: 在加载**失败**时（如网络错误、404）显示的图片。
*   `fallback()`: 当 `load()` 方法传入的数据源为 `null` 时显示的图片。它是处理空数据源的一种方式。

#### **四、转换与变换 (Transformation)**

**9. 如何对图片进行圆角或圆形裁剪？**
**答：** 使用 `transform()` 方法，并传入相应的 `Transformation` 对象。Glide 提供了常用的实现，也可以通过集成库获取更多。
```java
// 圆形裁剪
Glide.with(context)
     .load(url)
     .circleCrop() // 简便方法，等同于 .transform(new CircleCrop())
     .into(imageView);

// 圆角矩形
Glide.with(context)
     .load(url)
     .transform(new RoundedCorners(16)) // 16是圆角半径，单位像素
     .into(imageView);
```
需要添加 `glide-transformations` 库来获得更多变换效果（如模糊、灰度等）。

**10. 可以同时应用多个变换吗？如何使用？**
**答：** 可以。使用 `MultiTransformation` 将多个变换组合起来。
```java
Glide.with(context)
     .load(url)
     .transform(new MultiTransformation<>(
         new CenterCrop(),
         new RoundedCorners(16)
     ))
     .into(imageView);
```

#### **五、缓存机制（重中之重）**

**11. Glide 的缓存机制分为几层？**
**答：** 分为**三层**（在 Glide 4.x 中）：
1.  **活动资源 (Active Resources)：** 弱引用缓存，存放当前正在使用（显示）的图片。这是最快的一层，防止正在显示的图片被回收。
2.  **内存缓存 (Memory Cache)：** LRU（最近最少使用）算法的强引用缓存，存放最近被加载过且当前未在使用的图片。速度快。
3.  **磁盘缓存 (Disk Cache)：** 将图片资源持久化到设备存储中。速度慢于内存，但可以跨应用重启使用。

**12. Glide 的磁盘缓存策略 (`diskCacheStrategy`) 有哪几种？分别适用于什么场景？**
**答：** 通过 `.diskCacheStrategy()` 方法设置。
*   `DiskCacheStrategy.AUTOMATIC` (默认)：智能模式。对远程数据缓存 `DATA` + `RESOURCE`，对本地数据只缓存 `RESOURCE`。
*   `DiskCacheStrategy.DATA`: 只缓存原始未经修改的数据（例如网络下载的原始字节流）。
*   `DiskCacheStrategy.RESOURCE`: 缓存解码后且转换后的图片（例如经过裁剪、圆角处理后的 Bitmap）。
*   `DiskCacheStrategy.ALL`: 既缓存 `DATA` 也缓存 `RESOURCE`。
*   `DiskCacheStrategy.NONE`: 不进行磁盘缓存。
*   `DiskCacheStrategy.AUTOMATIC`: 默认策略，通常是最佳选择。

**13. 如何跳过内存缓存或磁盘缓存？**
**答：**
*   `skipMemoryCache(true)`: 跳过内存缓存（包括活动和内存缓存）。
*   `diskCacheStrategy(DiskCacheStrategy.NONE)`: 跳过磁盘缓存。

**14. 如何清理 Glide 的缓存？**
**答：**
```java
// 在子线程中执行
new Thread(new Runnable() {
    @Override
    public void run() {
        // 清理磁盘缓存
        Glide.get(context).clearDiskCache();
    }
}).start();

// 在主线程中清理内存缓存
Glide.get(context).clearMemory();
```

#### **六、高级特性与原理**

**15. Glide 如何实现与生命周期的绑定的？**
**答：** 这是 Glide 的核心设计。
1.  `Glide.with(activity)` 会向当前 Activity 添加一个不可见的 `RequestManagerFragment`。
2.  这个 Fragment 会监听 Activity 的生命周期方法（如 `onStart`, `onStop`, `onDestroy`）。
3.  当生命周期发生变化时，Fragment 会通知其关联的 `RequestManager`。
4.  `RequestManager` 会相应地管理所有由它发起的图片请求：`onStart` 时恢复请求，`onStop` 时暂停请求，`onDestroy` 时清除请求和资源。这有效防止了内存泄漏。

**16. 如何加载 Gif 图？如何判断一个 URL 是否是 Gif？**
**答：**
*   **加载 Gif：** 默认情况下，如果数据源是 Gif，Glide 会自动将其作为 Gif 播放。你也可以使用 `asGif()` 强制按 Gif 加载，如果不是 Gif 则会出错。
*   **判断 Gif：** 通常不需要显式判断。Glide 的 `HttpUrlConnectionFetcher` 或 `OkHttpStreamFetcher` 在获取到数据流后，会通过读取文件头部的魔数（Magic Number）来判断图片格式。

**17. `preload()` 方法有什么用？**
**答：** `preload()` 用于预加载图片到缓存中，但不显示到任何 View 上。它有两个重载：
*   `preload()`: 加载原始尺寸。
*   `preload(int width, int height)`: 加载指定尺寸。
当你确定很快会显示某张图片时（如预览图后的大图），预加载可以提升用户体验，实现无缝切换。

**18. `submit()` 和 `preload()` 有什么区别？**
**答：** 两者都用于不显示图片的加载，但目的不同。
*   `preload()`: **目的是填充缓存**，方便后续快速加载。它返回一个 `Target`，通常不需要处理。
*   `submit(int width, int height)`: **目的是获取图片对象**。它返回一个 `FutureTarget`，你可以通过 `.get()` 方法（必须在后台线程调用）同步地获取到解码后的 `Bitmap` 或 `Drawable`，以便进行其他操作。

**19. Glide 的 `Generated API` (Generated API) 是什么？有什么好处？**
**答：** 通过在项目中添加注解处理器 (`glide:compiler`)，Glide 会为你的 `AppGlideModule` 生成一个流式 API。
*   **好处：**
    *   **扩展性好：** 可以轻松添加自定义选项。
    *   **API 更简洁：** 可以将一系列常用操作（如特定变换）封装成一个方法调用。
    *   **类型安全：** 生成的 API 方法是类型安全的。

**20. 如何配置 Glide 使用 OkHttp 作为网络层？为什么要这么做？**
**答：**
*   **如何配置：** 添加依赖 `com.github.bumptech.glide:okhttp3-integration:4.x.x` 即可。Glide 会自动使用 `AppGlideModule` 的注册机制来替换默认的 `HttpUrlConnection`。
*   **为什么：**
    *   **更好的性能：** OkHttp 支持 SPDY、连接池、GZIP 压缩和响应缓存等现代网络特性。
    *   **统一网络栈：** 如果你的应用本身就在使用 OkHttp，这样可以统一网络行为（如 Cookie、Header 管理、拦截器等）。
    *   **更方便的调试：** 可以使用 OkHttp 的日志拦截器。

**21. 如何自定义一个 Glide 的模块 (`AppGlideModule`)？**
**答：**
1.  创建一个类并添加 `@GlideModule` 注解。
2.  继承 `AppGlideModule` 类。
3.  重写 `applyOptions` 方法来进行全局配置（如缓存大小、默认转换等）。
4.  重写 `registerComponents` 方法来注册自定义组件（如 ModelLoader）。

```java
@GlideModule
public class MyAppGlideModule extends AppGlideModule {
    @Override
    public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
        // 例如：设置内存缓存大小为 20MB
        builder.setMemoryCache(new LruResourceCache(20 * 1024 * 1024));
    }

    @Override
    public void registerComponents(@NonNull Context context, @NonNull Glide glide, @NonNull Registry registry) {
        // 例如：注册自定义的 ModelLoader
        // registry.append(Photo.class, InputStream.class, new CustomModelLoader.Factory());
    }
}
```
编译后，就可以使用 `GlideApp.with(context)...` 来调用生成的 API。

**22. 遇到过 Glide 的 OOM 问题吗？如何分析和解决？**
**答：**
*   **原因：** 通常是加载了超大尺寸的图片，或者同时加载大量图片，超过了内存缓存和 Java 堆的限制。
*   **解决方案：**
    1.  **使用 `override()`:** 明确指定图片加载的尺寸，避免加载原图。` .override(300, 200)`
    2.  **使用 `downsample` (Downsampling)：** Glide 会自动根据 ImageView 的大小对图片进行下采样，这是其默认行为，非常有效。确保你的 ImageView 有明确的大小（非 `wrap_content`）能帮助 Glide 更好地计算采样率。
    3.  **调整缓存策略：** 在特定场景下，适当减少内存缓存大小（通过 `AppGlideModule`）。
    4.  **使用合适的格式：** 考虑使用 `WebP` 等更高效的图片格式。
    5.  **排查内存泄漏：** 使用 Profiler 工具确认是否有因生命周期管理不当导致的内存泄漏。

