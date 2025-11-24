# 如何捕获Native层的崩溃？

在Android中捕获Native层崩溃有多种方案，下面我会详细介绍几种主流的实现方式：

## 1. Google Breakpad方案

### 1.1 集成Breakpad

**添加依赖：**
```gradle
// build.gradle
dependencies {
    implementation 'com.google.firebase:firebase-crashlytics-ndk:18.3.2'
    // 或者直接使用Breakpad
    // implementation project(':breakpad')
}
```

**或者手动集成Breakpad源码：**
```bash
# 下载Breakpad源码
git clone https://github.com/google/breakpad.git
```

### 1.2 初始化Breakpad

```java
public class NativeCrashHandler {
    private static final String TAG = "NativeCrashHandler";
    
    static {
        System.loadLibrary("native-crash-handler");
    }
    
    public static void init(Context context, String crashDumpPath) {
        // 设置崩溃文件存储路径
        File crashDir = new File(crashDumpPath);
        if (!crashDir.exists()) {
            crashDir.mkdirs();
        }
        
        // 初始化Breakpad
        setupNativeCrashHandler(crashDumpPath);
        
        // 启动监控线程，定期检查崩溃文件
        startCrashMonitor();
    }
    
    private native static void setupNativeCrashHandler(String crashDumpPath);
    
    private static void startCrashMonitor() {
        new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    Thread.sleep(5000); // 每5秒检查一次
                    checkAndUploadCrashDumps();
                } catch (InterruptedException e) {
                    break;
                }
            }
        }, "CrashMonitor").start();
    }
}
```

### 1.3 Native层实现

```cpp
// native-crash-handler.cpp
#include <jni.h>
#include <android/log.h>
#include <client/linux/handler/exception_handler.h>
#include <client/linux/handler/minidump_descriptor.h>

#define LOG_TAG "NativeCrashHandler"
#define ALOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)
#define ALOGW(...) __android_log_print(ANDROID_LOG_WARN, LOG_TAG, __VA_ARGS__)
#define ALOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)

namespace {
    
google_breakpad::ExceptionHandler* exception_handler = nullptr;

// 崩溃回调函数
bool OnMinidumpGenerated(const google_breakpad::MinidumpDescriptor& descriptor,
                        void* context,
                        bool succeeded) {
    ALOGI("Native crash detected! Minidump: %s", descriptor.path());
    
    // 这里可以添加额外的处理逻辑
    // 比如记录日志、通知Java层等
    
    return succeeded;
}

}  // namespace

#ifdef __cplusplus
extern "C" {
#endif

JNIEXPORT void JNICALL
Java_com_example_NativeCrashHandler_setupNativeCrashHandler(JNIEnv* env,
                                                           jclass clazz,
                                                           jstring dump_path) {
    if (exception_handler != nullptr) {
        ALOGW("Native crash handler already initialized");
        return;
    }
    
    const char* path = env->GetStringUTFChars(dump_path, nullptr);
    if (path == nullptr) {
        ALOGE("Failed to get dump path");
        return;
    }
    
    // 创建Breakpad异常处理器
    google_breakpad::MinidumpDescriptor descriptor(path);
    exception_handler = new google_breakpad::ExceptionHandler(
        descriptor,
        nullptr,  // Filter callback
        OnMinidumpGenerated,
        nullptr,  // callback context
        true,     // install handlers
        -1        // server socket FD (unused)
    );
    
    env->ReleaseStringUTFChars(dump_path, path);
    
    ALOGI("Native crash handler initialized successfully");
}

// 测试Native崩溃的函数
JNIEXPORT void JNICALL
Java_com_example_NativeCrashHandler_causeNativeCrash(JNIEnv* env,
                                                    jclass clazz) {
    ALOGI("Causing native crash for testing...");
    
    // 触发段错误
    volatile int* ptr = nullptr;
    *ptr = 1;
}

#ifdef __cplusplus
}
#endif
```

### 1.4 CMake配置

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.18.1)

project("native-crash-handler")

# 添加Breakpad源码
add_subdirectory(breakpad)

# 创建Native库
add_library(
    native-crash-handler
    SHARED
    native-crash-handler.cpp
)

# 包含头文件
include_directories(breakpad/src breakpad/src/common/android/include)

# 链接Breakpad库
target_link_libraries(
    native-crash-handler
    breakpad_client
    log
)
```

## 2. 信号处理方案

如果不使用Breakpad，也可以直接使用Linux信号处理机制：

### 2.1 信号处理器实现

```cpp
#include <signal.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unwind.h>
#include <dlfcn.h>
#include <cxxabi.h>

#define STACK_TRACE_MAX_DEPTH 32
#define CRASH_LOG_PATH "/data/data/com.example.app/files/crash_log.txt"

// 栈帧结构
struct StackTrace {
    void** stack;
    int depth;
};

// 获取栈回溯
_Unwind_Reason_Code unwind_callback(struct _Unwind_Context* context, void* arg) {
    StackTrace* stack_trace = static_cast<StackTrace*>(arg);
    
    if (stack_trace->depth >= STACK_TRACE_MAX_DEPTH) {
        return _URC_END_OF_STACK;
    }
    
    void* ip = (void*)_Unwind_GetIP(context);
    if (ip) {
        stack_trace->stack[stack_trace->depth] = ip;
        stack_trace->depth++;
    }
    
    return _URC_NO_REASON;
}

// 信号处理函数
void signal_handler(int sig, siginfo_t* info, void* context) {
    // 记录崩溃信息
    int fd = open(CRASH_LOG_PATH, O_CREAT | O_WRONLY | O_APPEND, 0644);
    if (fd == -1) {
        return;
    }
    
    char buffer[1024];
    int len;
    
    // 写入信号信息
    len = snprintf(buffer, sizeof(buffer), 
                  "=== Native Crash Report ===\n"
                  "Signal: %d (%s)\n"
                  "Time: %ld\n"
                  "PID: %d\n"
                  "TID: %d\n",
                  sig, strsignal(sig),
                  time(nullptr),
                  getpid(), gettid());
    write(fd, buffer, len);
    
    // 获取栈回溯
    void* stack[STACK_TRACE_MAX_DEPTH];
    StackTrace stack_trace = {stack, 0};
    _Unwind_Backtrace(unwind_callback, &stack_trace);
    
    // 写入栈回溯
    write(fd, "Stack trace:\n", 13);
    for (int i = 0; i < stack_trace.depth; i++) {
        Dl_info dl_info;
        if (dladdr(stack_trace.stack[i], &dl_info)) {
            const char* symbol_name = dl_info.dli_sname;
            if (symbol_name) {
                int status;
                char* demangled = abi::__cxa_demangle(symbol_name, nullptr, nullptr, &status);
                if (demangled) {
                    symbol_name = demangled;
                }
                
                len = snprintf(buffer, sizeof(buffer), "#%02d: %s (%s)\n",
                              i, symbol_name, dl_info.dli_fname);
                write(fd, buffer, len);
                
                if (demangled) {
                    free(demangled);
                }
            }
        }
    }
    
    close(fd);
    
    // 重新抛出信号给默认处理器
    signal(sig, SIG_DFL);
    raise(sig);
}

// 安装信号处理器
void install_signal_handlers() {
    struct sigaction sa;
    sa.sa_sigaction = signal_handler;
    sa.sa_flags = SA_SIGINFO | SA_ONSTACK;
    sigemptyset(&sa.sa_mask);
    
    // 注册需要捕获的信号
    sigaction(SIGSEGV, &sa, nullptr);  // 段错误
    sigaction(SIGABRT, &sa, nullptr);  // 中止信号
    sigaction(SIGBUS, &sa, nullptr);   // 总线错误
    sigaction(SIGFPE, &sa, nullptr);   // 算术异常
    sigaction(SIGILL, &sa, nullptr);   // 非法指令
    sigaction(SIGTRAP, &sa, nullptr);  // 跟踪陷阱
}
```

## 3. 完整的Native Crash收集框架

### 3.1 Java层管理类

```java
public class NativeCrashManager {
    private static final String TAG = "NativeCrashManager";
    private Context mContext;
    private CrashUploader mUploader;
    
    public void init(Context context, CrashConfig config) {
        mContext = context.getApplicationContext();
        mUploader = config.getUploader();
        
        // 初始化Native崩溃处理
        String crashDir = getCrashDumpDir();
        NativeCrashHandler.init(mContext, crashDir);
        
        // 检查已有的崩溃文件
        checkPendingCrashDumps();
    }
    
    private String getCrashDumpDir() {
        File crashDir = new File(mContext.getFilesDir(), "native_crashes");
        if (!crashDir.exists()) {
            crashDir.mkdirs();
        }
        return crashDir.getAbsolutePath();
    }
    
    private void checkPendingCrashDumps() {
        File crashDir = new File(getCrashDumpDir());
        File[] dumpFiles = crashDir.listFiles((dir, name) -> 
            name.endsWith(".dmp"));
            
        if (dumpFiles != null) {
            for (File dumpFile : dumpFiles) {
                processCrashDump(dumpFile);
            }
        }
    }
    
    private void processCrashDump(File dumpFile) {
        // 解析minidump文件
        CrashInfo crashInfo = parseMinidump(dumpFile);
        
        // 添加上下文信息
        crashInfo.setAppVersion(getAppVersion());
        crashInfo.setDeviceInfo(getDeviceInfo());
        
        // 上报崩溃信息
        if (mUploader != null) {
            mUploader.upload(crashInfo, new UploadCallback() {
                @Override
                public void onSuccess() {
                    // 上传成功后删除文件
                    dumpFile.delete();
                }
                
                @Override
                public void onFailure(Throwable error) {
                    // 上传失败，保留文件下次重试
                    Log.e(TAG, "Upload crash dump failed", error);
                }
            });
        }
    }
    
    private CrashInfo parseMinidump(File dumpFile) {
        // 使用Breakpad工具解析minidump文件
        // 这里可以调用minidump_stackwalk工具
        CrashInfo info = new CrashInfo();
        info.setCrashType("Native");
        info.setDumpFilePath(dumpFile.getAbsolutePath());
        return info;
    }
}
```

### 3.2 符号文件生成

为了能够解析minidump文件，需要生成对应的符号文件：

```bash
# 使用Breakpad工具生成符号文件
./dump_syms app > app.sym

# 创建符号目录结构
mkdir -p symbols/app/xxx/
mv app.sym symbols/app/xxx/

# 使用minidump_stackwalk解析崩溃文件
./minidump_stackwalk crash.dmp symbols/ > crash_log.txt
```

### 3.3 自动化脚本

可以创建Gradle任务来自动化符号文件生成：

```gradle
// build.gradle
task generateNativeSymbols(type: Exec) {
    commandLine 'python', 'generate_symbols.py'
}

android {
    applicationVariants.all { variant ->
        variant.assemble.doLast {
            generateNativeSymbols.execute()
        }
    }
}
```

## 4. 注意事项

1. **权限问题**：确保应用有权限写入崩溃文件目录
2. **性能影响**：信号处理函数要尽可能简单，避免复杂操作
3. **异步安全**：在信号处理函数中只能使用异步信号安全的函数
4. **文件管理**：定期清理旧的崩溃文件，避免占用过多存储空间
5. **符号文件**：发布版本时要保存对应的符号文件，用于后续分析

## 5. 测试Native崩溃

```java
// 测试方法
public class NativeCrashTester {
    public static void testNativeCrash() {
        try {
            NativeCrashHandler.causeNativeCrash();
        } catch (Exception e) {
            Log.e("Test", "Expected native crash");
        }
    }
}
```

这种方案能够有效地捕获Native层崩溃，生成详细的崩溃报告，并集成到整体的Crash收集框架中。