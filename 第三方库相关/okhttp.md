### **Android OkHttp 面试问题与答案大全**

#### **一、 基础概念与核心组件**

**1. OkHttp是什么？它有哪些主要优势？**
**答：** OkHttp是一个高效的HTTP客户端，由Square公司开发。它的主要优势包括：
*   **连接池**：支持HTTP/2和多路复用，减少请求延迟。
*   **透明GZIP压缩**：自动压缩请求体和解压响应体，节省带宽。
*   **响应缓存**：可配置的缓存机制，避免重复网络请求。
*   **拦截器链**：强大的拦截器机制，可以方便地处理日志、重试、认证等任务。
*   **自动重连**：在连接失败时自动重试，提高请求成功率。

**2. 简述一次OkHttp同步请求和异步请求的执行流程。**
**答：**
*   **同步请求**：`client.newCall(request).execute()`
    1.  调用`execute()`方法，当前线程（通常是子线程）会阻塞。
    2.  OkHttp通过拦截器链处理请求，发送到网络并获取响应。
    3.  直接返回`Response`对象。
*   **异步请求**：`client.newCall(request).enqueue(Callback callback)`
    1.  调用`enqueue()`方法，将请求放入队列，立即返回，不阻塞当前线程（通常是主线程）。
    2.  OkHttp内部的线程池会在后台线程中执行网络请求。
    3.  请求完成后，将响应或失败回调到`Callback`中，回调执行在子线程。

**3. OkHttpClient对象为什么建议共享实例而不是每次创建新的？**
**答：** 因为每个`OkHttpClient`实例都管理着自己的一套资源池（如连接池、线程池、缓存等）。共享实例可以：
*   **复用连接**：避免为每个请求进行TCP/TLS握手，极大提升性能。
*   **共享线程池**：减少线程创建和销毁的开销。
*   **共享缓存**：所有请求可以访问统一的缓存数据。
*   **节省资源**：避免重复创建相同的配置对象。

**4. 如何构建一个HTTP请求（Request）？关键可以配置哪些部分？**
**答：** 使用`Request.Builder`构建。
```java
Request request = new Request.Builder()
    .url("https://api.example.com/data")
    .header("User-Agent", "My-App") // 添加Header
    .addHeader("Accept", "application/json")
    .get() // 指定方法，如.get(), .post(RequestBody body), .put(...)等
    .build();
```
关键配置：URL、方法（GET/POST/PUT/DELETE等）、请求头（Header）、请求体（RequestBody，用于POST/PUT等）。

**5. 请解释OkHttp中的Call对象是什么？**
**答：** `Call`是一个接口，代表一个已准备好执行的请求。它是对单个请求/响应交互的建模。你可以通过它来执行（同步`execute()`或异步`enqueue()`）请求，也可以取消请求（`cancel()`）。一个`Call`实例只能执行一次。

---

#### **二、 拦截器（Interceptors）与网络栈**

**6. 什么是OkHttp的拦截器（Interceptor）？它的作用是什么？**
**答：** 拦截器是OkHttp提供的一种强大机制，可以在发出请求和获取响应的过程中进行拦截和处理。它形成了一个链式结构，每个拦截器可以：
*   修改请求（如添加通用Header）。
*   检查响应（如处理认证失败）。
*   重试请求。
*   记录日志。
*   编解码（如加密/解密）。

**7. 应用拦截器（Application Interceptor）和网络拦截器（Network Interceptor）有什么区别？**
**答：** 这是面试高频考点。
| 特性 | 应用拦截器 | 网络拦截器 |
| :--- | :--- | :--- |
| **位置** | 在重定向和重试之前最先被调用，最后获得响应。 | 在非重定向、重试等“核心”网络操作之后被调用。 |
| **调用次数** | **总是只调用一次**，即使响应来自缓存。 | **可能被调用多次**，如果发生了重定向或重试。 |
| **访问连接** | 无法访问承载请求的`Connection`。 | 可以访问`Connection`，了解使用的协议（HTTP/1.1, HTTP/2）。 |
| **使用场景** | 添加全局Header、日志记录、请求/响应体转换。 | 添加与网络相关的Header（如`Accept-Encoding`）、监控网络数据。 |

**8. 如何添加一个应用拦截器和一个网络拦截器？**
**答：**
```java
OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new HttpLoggingInterceptor()) // 添加应用拦截器
    .addNetworkInterceptor(new StethoInterceptor()) // 添加网络拦截器
    .build();
```

**9. 请自己实现一个简单的日志拦截器（打印请求和响应信息）。**
**答：**
```java
public class SimpleLoggingInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();

        long startTime = System.nanoTime();
        Log.d("OKHTTP", String.format("Sending request %s on %s%n%s",
                request.url(), chain.connection(), request.headers()));

        Response response = chain.proceed(request); // 关键：将请求传递给下一个拦截器

        long endTime = System.nanoTime();
        Log.d("OKHTTP", String.format("Received response for %s in %.1fms%n%s",
                response.request().url(), (endTime - startTime) / 1e6d, response.headers()));

        return response;
    }
}
```

**10. 拦截器链（Chain）中的`proceed()`方法是做什么的？**
**答：** `proceed(Request request)`方法是拦截器链的核心。它代表将当前的请求（可以是被当前拦截器修改后的）传递给链中的下一个拦截器（或最终的网络服务器），并返回对应的响应。**每个拦截器都必须调用一次`chain.proceed()`**，否则请求将无法发出。

---

#### **三、 连接、缓存与性能优化**

**11. OkHttp的连接池（Connection Pool）是如何工作的？它有什么好处？**
**答：** OkHttp的连接池维护着多个打开的Socket连接，以便将来请求可以复用它们。
*   **工作方式**：当一个请求完成后，其连接不会立即关闭，而是被放入连接池中并设置一个存活时间（默认5分钟）。当有新的请求发出时，会优先从池中查找是否有相同地址（URL的主机和端口）的可复用连接。
*   **好处**：避免了昂贵的TCP和TLS握手过程，显著减少了请求延迟，尤其对HTTP/2的多路复用支持极好。

**12. 如何为OkHttp配置缓存？缓存是如何工作的？**
**答：**
```java
// 配置缓存目录和大小
int cacheSize = 10 * 1024 * 1024; // 10 MiB
File cacheDirectory = new File(context.getCacheDir(), "http-cache");
Cache cache = new Cache(cacheDirectory, cacheSize);

OkHttpClient client = new OkHttpClient.Builder()
    .cache(cache)
    .build();
```
**工作原理**：OkHttp根据HTTP响应的缓存头（如`Cache-Control`, `Expires`, `ETag`, `Last-Modified`）自动处理缓存。它可以返回缓存的响应、验证缓存是否过期（向服务器发送条件GET请求）或直接请求新的网络响应。

**13. 解释一下HTTP缓存中的强制缓存和协商（对比）缓存，OkHttp如何支持？**
**答：**
*   **强制缓存**：浏览器（或OkHttp）直接根据`Cache-Control`和`Expires`判断缓存是否过期，未过期则直接使用缓存，不发请求。由`Cache`拦截器处理。
*   **协商缓存**：缓存过期后，浏览器会向服务器发送请求（携带`If-Modified-Since`或`If-None-Match`等标识）。如果资源没变，服务器返回304，浏览器继续用缓存；如果变了，返回200和新数据。由网络拦截器层处理。
    OkHttp的缓存机制完全遵循HTTP协议，自动处理这些逻辑。

**14. 如何配置超时时间？有哪些类型的超时？**
**答：**
```java
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS) // 连接超时
    .writeTimeout(10, TimeUnit.SECONDS)    // 写入（发送请求体）超时
    .readTimeout(30, TimeUnit.SECONDS)     // 读取（获取响应体）超时
    .callTimeout(60, TimeUnit.SECONDS)     // 整个Call的超时（包含重试/重定向）
    .build();
```

---

#### **四、 高级特性与实际问题**

**15. 如何使用OkHttp处理POST请求，提交JSON数据或表单数据？**
**答：**
*   **提交JSON**：
    ```java
    MediaType JSON = MediaType.get("application/json; charset=utf-8");
    String jsonBody = "{\"username\": \"user\", \"password\": \"pass\"}";
    RequestBody body = RequestBody.create(jsonBody, JSON);

    Request request = new Request.Builder()
        .url(url)
        .post(body)
        .build();
    ```
*   **提交表单**：
    ```java
    RequestBody formBody = new FormBody.Builder()
        .add("username", "user")
        .add("password", "pass")
        .build();

    Request request = new Request.Builder()
        .url(url)
        .post(formBody)
        .build();
    ```

**16. 如何使用OkHttp进行文件上传？**
**答：** 使用`MultipartBody`。
```java
RequestBody fileBody = RequestBody.create(file, MediaType.get("image/png"));
MultipartBody multipartBody = new MultipartBody.Builder()
        .setType(MultipartBody.FORM)
        .addFormDataPart("title", "My Image")
        .addFormDataPart("image", "image.png", fileBody) // "image"是字段名
        .build();

Request request = new Request.Builder()
        .url(uploadUrl)
        .post(multipartBody)
        .build();
```

**17. OkHttp如何管理Cookie？**
**答：** OkHttp本身不直接管理Cookie，但可以通过实现`CookieJar`接口来集成Cookie持久化。
```java
OkHttpClient client = new OkHttpClient.Builder()
    .cookieJar(new CookieJar() {
        // 简单的内存Cookie存储（重启失效）
        private final HashMap<HttpUrl, List<Cookie>> cookieStore = new HashMap<>();

        @Override
        public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
            cookieStore.put(url, cookies);
        }

        @Override
        public List<Cookie> loadForRequest(HttpUrl url) {
            List<Cookie> cookies = cookieStore.get(url);
            return cookies != null ? cookies : new ArrayList<Cookie>();
        }
    })
    .build();
```
实践中，通常会使用`PersistentCookieJar`等第三方库来实现磁盘持久化。

**18. 如何处理HTTPS请求和证书校验？如何绕过证书验证（仅用于测试）？**
**答：**
*   **默认情况**：OkHttp使用系统信任的证书链（CA）进行验证。
*   **自定义证书**：可以配置自定义的`SSLSocketFactory`和`X509TrustManager`。
*   **绕过验证（危险！仅用于测试环境）**：
    ```java
    // 创建一个不进行证书验证的TrustManager
    final TrustManager[] trustAllCerts = new TrustManager[]{
        new X509TrustManager() {
            @Override
            public void checkClientTrusted(java.security.cert.X509Certificate[] chain, String authType) {}
            @Override
            public void checkServerTrusted(java.security.cert.X509Certificate[] chain, String authType) {}
            @Override
            public java.security.cert.X509Certificate[] getAcceptedIssuers() { return new X509Certificate[]{}; }
        }
    };

    // 安装全信任的SocketFactory
    SSLContext sslContext = SSLContext.getInstance("SSL");
    sslContext.init(null, trustAllCerts, new java.security.SecureRandom());
    OkHttpClient client = new OkHttpClient.Builder()
        .sslSocketFactory(sslContext.getSocketFactory(), (X509TrustManager)trustAllCerts[0])
        .hostnameVerifier((hostname, session) -> true) // 绕过主机名验证
        .build();
    ```

**19. 如何取消一个OkHttp请求？**
**答：** 调用`Call`对象的`cancel()`方法。
```java
Call call = client.newCall(request);
call.enqueue(new Callback() { ... });

// 在需要取消的时候（如Activity的onDestroy中）
call.cancel();
```
取消请求会尽力中断正在进行的网络操作，释放资源。

**20. 如果遇到`SSLHandshakeException`或`CertificateException`，可能是什么原因？**
**答：** 常见原因：
1.  服务器的证书不是由受信任的机构（CA）签发。
2.  服务器证书已过期或配置错误。
3.  设备日期和时间不正确。
4.  目标主机名与证书中的域名不匹配。

**21. 如何用OkHttp实现下载文件并显示进度？**
**答：** 通过自定义拦截器或包装`ResponseBody`，在读取响应体时计算已读取的字节数。
```java
// 包装ResponseBody
public class ProgressResponseBody extends ResponseBody {
    private final ResponseBody responseBody;
    private final ProgressListener listener;
    private BufferedSource bufferedSource;

    public ProgressResponseBody(ResponseBody responseBody, ProgressListener listener) {
        this.responseBody = responseBody;
        this.listener = listener;
    }

    @Override
    public BufferedSource source() {
        if (bufferedSource == null) {
            bufferedSource = Okio.buffer(new ProgressSource(responseBody.source()));
        }
        return bufferedSource;
    }

    private class ProgressSource extends ForwardingSource {
        long totalBytesRead = 0;
        ProgressSource(Source source) { super(source); }

        @Override
        public long read(Buffer sink, long byteCount) throws IOException {
            long bytesRead = super.read(sink, byteCount);
            totalBytesRead += (bytesRead != -1) ? bytesRead : 0;
            listener.update(totalBytesRead, responseBody.contentLength(), bytesRead == -1);
            return bytesRead;
        }
    }
}

// 在拦截器中替换ResponseBody
Response originalResponse = chain.proceed(chain.request());
return originalResponse.newBuilder()
        .body(new ProgressResponseBody(originalResponse.body(), progressListener))
        .build();
```

---

#### **五、 架构设计与对比**

**22. OkHttp和Retrofit是什么关系？**
**答：** Retrofit是一个RESTful风格的HTTP网络请求框架的封装，它的底层默认使用OkHttp进行实际的网络请求。Retrofit关注于通过接口和注解将HTTP API转换为Java接口调用，而OkHttp关注于底层高效的网络通信。两者是互补关系，通常结合使用。

**23. 与HttpURLConnection和Volley相比，OkHttp有什么优缺点？**
**答：**
*   **vs HttpURLConnection**：
    *   **优点**：API更友好、功能更强大（连接池、缓存、拦截器）、性能更好、社区活跃。
    *   **缺点**：需要引入第三方库。
*   **vs Volley**：
    *   **优点**：性能更优（尤其HTTP/2）、更底层更灵活、功能更全面（如完整的缓存策略、WebSocket）。
    *   **缺点**：Volley更适合数据量小、频繁通信的RPC场景，且自带图片加载等UI集成功能。OkHttp是更纯粹的HTTP客户端。

**24. OkHttp支持WebSocket吗？**
**答：** 支持。可以通过`newWebSocket()`方法创建WebSocket连接，并实现`WebSocketListener`来监听连接、消息接收和关闭等事件。

**25. 在Android中使用OkHttp需要注意哪些内存和线程问题？**
**答：**
*   **内存泄漏**：在Activity/Fragment销毁时，取消所有未完成的请求（`call.cancel()`），防止Callback持有外部类引用导致无法回收。
*   **回调线程**：OkHttp的回调默认在**后台线程**执行，如果要在其中更新UI，必须使用`Handler`或`runOnUiThread`切换到主线程。
*   **生命周期**：将OkHttpClient的管理与Application上下文绑定，而不是Activity。

