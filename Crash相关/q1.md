# 如果让你自己实现一个Crash收集框架，你会如何实现？

要实现一个Android平台的Crash收集框架，我会从以下几个方面进行设计：

## 1. 整体架构设计

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  异常捕获层      │ -> │  数据处理层       │ -> │  数据上报层      │
│  Crash捕获      │    │  信息收集与处理   │    │  存储与上报      │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

## 2. 核心实现代码

### 2.1 异常捕获器

```java
public class CrashCollector implements Thread.UncaughtExceptionHandler {
    private static final String TAG = "CrashCollector";
    private Thread.UncaughtExceptionHandler mDefaultHandler;
    private static CrashCollector INSTANCE;
    private Context mContext;
    private CrashConfig mConfig;
    
    private CrashCollector() {}
    
    public static CrashCollector getInstance() {
        if (INSTANCE == null) {
            synchronized (CrashCollector.class) {
                if (INSTANCE == null) {
                    INSTANCE = new CrashCollector();
                }
            }
        }
        return INSTANCE;
    }
    
    public void init(Context context, CrashConfig config) {
        mContext = context.getApplicationContext();
        mConfig = config;
        // 保存系统默认的异常处理器
        mDefaultHandler = Thread.getDefaultUncaughtExceptionHandler();
        // 设置当前处理器为默认
        Thread.setDefaultUncaughtExceptionHandler(this);
    }
    
    @Override
    public void uncaughtException(Thread thread, Throwable ex) {
        if (!handleException(thread, ex) && mDefaultHandler != null) {
            // 交给系统默认处理器处理
            mDefaultHandler.uncaughtException(thread, ex);
        } else {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                Log.e(TAG, "Error : ", e);
            }
            // 退出程序
            android.os.Process.killProcess(android.os.Process.myPid());
            System.exit(1);
        }
    }
    
    private boolean handleException(Thread thread, Throwable ex) {
        if (ex == null) {
            return false;
        }
        
        // 收集崩溃信息
        CrashInfo crashInfo = collectCrashInfo(thread, ex);
        
        // 保存到本地文件
        saveCrashToFile(crashInfo);
        
        // 异步上报
        if (mConfig.isAutoUpload()) {
            uploadCrashAsync(crashInfo);
        }
        
        return true;
    }
}
```

### 2.2 崩溃信息收集

```java
public class CrashInfo {
    private String crashTime;
    private String appVersion;
    private String osVersion;
    private String deviceModel;
    private String brand;
    private String cpuArch;
    private long totalMemory;
    private long availableMemory;
    private String threadName;
    private String stackTrace;
    private String logcat;
    private Map<String, String> customInfo;
    
    // 构造方法和getter/setter
}

private CrashInfo collectCrashInfo(Thread thread, Throwable ex) {
    CrashInfo crashInfo = new CrashInfo();
    
    // 基本信息
    crashInfo.setCrashTime(getCurrentTime());
    crashInfo.setThreadName(thread.getName());
    crashInfo.setStackTrace(Log.getStackTraceString(ex));
    
    // 应用信息
    try {
        PackageManager pm = mContext.getPackageManager();
        PackageInfo pi = pm.getPackageInfo(mContext.getPackageName(), 0);
        crashInfo.setAppVersion(pi.versionName + "_" + pi.versionCode);
    } catch (Exception e) {
        crashInfo.setAppVersion("unknown");
    }
    
    // 设备信息
    crashInfo.setOsVersion(Build.VERSION.RELEASE + "_" + Build.VERSION.SDK_INT);
    crashInfo.setDeviceModel(Build.MODEL);
    crashInfo.setBrand(Build.BRAND);
    crashInfo.setCpuArch(Build.CPU_ABI);
    
    // 内存信息
    ActivityManager.MemoryInfo memoryInfo = new ActivityManager.MemoryInfo();
    ActivityManager activityManager = (ActivityManager) 
        mContext.getSystemService(Context.ACTIVITY_SERVICE);
    activityManager.getMemoryInfo(memoryInfo);
    crashInfo.setTotalMemory(memoryInfo.totalMem);
    crashInfo.setAvailableMemory(memoryInfo.availMem);
    
    // 自定义信息
    crashInfo.setCustomInfo(collectCustomInfo());
    
    // 收集Logcat
    if (mConfig.isCaptureLogcat()) {
        crashInfo.setLogcat(captureLogcat());
    }
    
    return crashInfo;
}
```

### 2.3 本地存储管理

```java
public class CrashStorageManager {
    private static final String CRASH_DIR = "crash";
    private Context mContext;
    
    public CrashStorageManager(Context context) {
        this.mContext = context;
    }
    
    public String saveCrashToFile(CrashInfo crashInfo) {
        String fileName = generateFileName(crashInfo);
        File crashFile = new File(getCrashDir(), fileName);
        
        try (FileOutputStream fos = new FileOutputStream(crashFile);
             ObjectOutputStream oos = new ObjectOutputStream(fos)) {
            
            oos.writeObject(crashInfo);
            return crashFile.getAbsolutePath();
            
        } catch (Exception e) {
            Log.e(TAG, "Save crash info failed", e);
            return null;
        }
    }
    
    public List<File> getPendingCrashFiles() {
        File crashDir = getCrashDir();
        if (!crashDir.exists()) {
            return new ArrayList<>();
        }
        
        File[] files = crashDir.listFiles((dir, name) -> 
            name.endsWith(".crash"));
        return files != null ? Arrays.asList(files) : new ArrayList<>();
    }
    
    public void deleteCrashFile(File file) {
        if (file.exists()) {
            file.delete();
        }
    }
    
    private File getCrashDir() {
        File crashDir = new File(mContext.getFilesDir(), CRASH_DIR);
        if (!crashDir.exists()) {
            crashDir.mkdirs();
        }
        return crashDir;
    }
    
    private String generateFileName(CrashInfo crashInfo) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd_HH-mm-ss", 
            Locale.getDefault());
        return "crash_" + sdf.format(new Date()) + ".crash";
    }
}
```

### 2.4 数据上报器

```java
public interface CrashUploader {
    void upload(CrashInfo crashInfo, UploadCallback callback);
    void upload(File crashFile, UploadCallback callback);
}

public class HttpCrashUploader implements CrashUploader {
    private OkHttpClient mHttpClient;
    private String mUploadUrl;
    
    public HttpCrashUploader(String uploadUrl) {
        this.mUploadUrl = uploadUrl;
        this.mHttpClient = new OkHttpClient.Builder()
            .connectTimeout(10, TimeUnit.SECONDS)
            .writeTimeout(10, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .build();
    }
    
    @Override
    public void upload(CrashInfo crashInfo, UploadCallback callback) {
        // 将CrashInfo转换为JSON
        String json = convertToJson(crashInfo);
        RequestBody body = RequestBody.create(
            MediaType.parse("application/json"), json);
            
        Request request = new Request.Builder()
            .url(mUploadUrl)
            .post(body)
            .build();
            
        mHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                callback.onFailure(e);
            }
            
            @Override
            public void onResponse(Call call, Response response) {
                if (response.isSuccessful()) {
                    callback.onSuccess();
                } else {
                    callback.onFailure(new IOException("Upload failed: " + 
                        response.code()));
                }
            }
        });
    }
}
```

### 2.5 配置类

```java
public class CrashConfig {
    private boolean isAutoUpload = true;
    private boolean isCaptureLogcat = true;
    private int maxLocalStorageCount = 50;
    private String uploadUrl;
    private CrashUploader customUploader;
    private Map<String, String> customFields;
    
    // Builder模式
    public static class Builder {
        private CrashConfig config = new CrashConfig();
        
        public Builder setAutoUpload(boolean autoUpload) {
            config.isAutoUpload = autoUpload;
            return this;
        }
        
        public Builder setCaptureLogcat(boolean captureLogcat) {
            config.isCaptureLogcat = captureLogcat;
            return this;
        }
        
        public Builder setUploadUrl(String uploadUrl) {
            config.uploadUrl = uploadUrl;
            return this;
        }
        
        public Builder setCustomUploader(CrashUploader uploader) {
            config.customUploader = uploader;
            return this;
        }
        
        public CrashConfig build() {
            return config;
        }
    }
}
```

## 3. 使用方式

```java
// 初始化
CrashCollector.getInstance().init(
    context,
    new CrashConfig.Builder()
        .setAutoUpload(true)
        .setUploadUrl("https://api.example.com/crash/upload")
        .setCaptureLogcat(true)
        .build()
);

// 添加自定义信息
CrashCollector.getInstance().putCustomData("userId", "12345");
CrashCollector.getInstance().putCustomData("pageName", "HomeActivity");
```

## 4. 高级特性考虑

### 4.1 ANR检测
```java
public class AnrDetector {
    private static final int ANR_TIMEOUT = 5000;
    private Handler mHandler = new Handler(Looper.getMainLooper());
    private boolean mAnrFlag = false;
    
    public void startWatch() {
        mHandler.postDelayed(mAnrRunnable, ANR_TIMEOUT);
    }
    
    private Runnable mAnrRunnable = new Runnable() {
        @Override
        public void run() {
            if (mAnrFlag) {
                // 发生ANR
                reportAnr();
            } else {
                mAnrFlag = true;
                mHandler.postDelayed(this, ANR_TIMEOUT);
            }
        }
    };
}
```

### 4.2 Native Crash捕获
通过集成Breakpad或Crashpad来捕获Native层的崩溃。

## 5. 优化考虑

1. **性能优化**：异步处理崩溃信息，避免阻塞UI线程
2. **存储优化**：限制本地存储的崩溃文件数量，定期清理
3. **网络优化**：在网络良好的情况下上报，支持重试机制
4. **安全考虑**：对敏感信息进行脱敏处理
5. **兼容性**：兼容不同Android版本和设备

这样的设计既保证了功能的完整性，又具备良好的扩展性和可维护性。