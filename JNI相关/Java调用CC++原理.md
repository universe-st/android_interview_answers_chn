# Java是如何调用到C/C++的函数的？

在Android开发中，Java调用C/C++函数主要通过**JNI（Java Native Interface，Java本地接口）** 实现。JNI是Java平台提供的一套编程接口，用于连接Java代码与本地代码（如C/C++），它允许Java虚拟机（JVM）调用本地代码，同时也支持本地代码调用Java代码。


### 一、Java调用C/C++的基本流程
要让Java调用C/C++函数，需遵循以下步骤，这些步骤体现了JNI的核心交互逻辑：

1. **在Java类中声明native方法**  
   用`native`关键字标记需要调用本地代码的方法（仅声明，不实现），告知JVM：该方法的实现由本地代码提供。  
   示例：
   ```java
   public class JNIDemo {
       // 声明native方法，由C/C++实现
       public native String getHelloFromC();
       
       static {
           // 加载包含本地方法实现的动态库（.so文件）
           System.loadLibrary("native-lib"); 
       }
   }
   ```


2. **生成JNI头文件**  
   通过`javac`编译Java类，再用`javah`工具生成对应的JNI头文件（`.h`），头文件中包含了本地方法的函数声明（遵循JNI规范的命名规则）。  
   命令示例：
   ```bash
   # 编译Java类
   javac JNIDemo.java
   # 生成JNI头文件（包名为com.example则输出com_example_JNIDemo.h）
   javah -jni JNIDemo
   ```

   生成的头文件中，函数声明格式为：  
   ```c
   JNIEXPORT jstring JNICALL Java_com_example_JNIDemo_getHelloFromC
   (JNIEnv *, jobject);
   ```  
   命名规则：`Java_包名_类名_方法名`（包名中的`.`替换为`_`），确保JVM能通过方法名找到对应的本地函数。


3. **用C/C++实现头文件中的函数**  
   新建C/C++源文件（如`native-lib.cpp`），实现头文件中声明的函数，通过JNI提供的接口与Java层交互（如操作Java对象、调用Java方法等）。  
   示例实现：
   ```cpp
   #include "com_example_JNIDemo.h"
   
   // 实现native方法
   JNIEXPORT jstring JNICALL Java_com_example_JNIDemo_getHelloFromC
   (JNIEnv *env, jobject thiz) {
       // 通过JNIEnv的NewStringUTF方法创建Java字符串并返回
       return env->NewStringUTF("Hello from C++");
   }
   ```


4. **编译C/C++代码为动态库**  
   通过NDK（Android Native Development Kit）将C/C++代码编译为Android支持的动态链接库（`.so`文件），并放置在`app/src/main/jniLibs/`目录下（或通过CMakeLists.txt配置编译路径）。


5. **Java层调用native方法**  
   在Java中创建`JNIDemo`实例，直接调用`getHelloFromC()`方法，JVM会自动加载`.so`库并执行对应的C/C++函数。  


### 二、JNI的核心原理
JNI的本质是**JVM与本地代码之间的“桥梁”**，它通过一套标准接口实现了Java与C/C++的跨语言交互，核心原理可从以下几个方面理解：


#### 1. **JNI桥接层：连接Java与本地代码的中间层**
JNI定义了一套统一的接口规范（包含函数、数据类型、引用类型等），作为Java层与本地层的“翻译官”：  
- 对Java层：通过`native`方法声明“需要调用本地代码”，通过`System.loadLibrary()`加载动态库。  
- 对本地层：提供`JNIEnv`（JNI环境）接口，封装了操作Java对象、调用Java方法、处理异常等能力。  
- JVM负责管理桥接过程：当Java调用`native`方法时，JVM会暂停当前线程，查找对应的本地函数（通过方法名或注册映射），执行本地代码后返回结果，恢复线程执行。


#### 2. **数据类型映射：解决跨语言类型差异**
Java与C/C++的数据类型不同（如Java的`int`是32位带符号整数，C的`int`可能因平台不同而变化），JNI定义了一套**中间类型**作为桥梁：  

| Java类型       | JNI中间类型 | C/C++对应类型（通常） |
|----------------|-------------|----------------------|
| `boolean`      | `jboolean`  | `unsigned char`      |
| `int`          | `jint`      | `int`                |
| `long`         | `jlong`     | `long long`          |
| `float`        | `jfloat`    | `float`              |
| `double`       | `jdouble`   | `double`             |
| `String`       | `jstring`   | （需通过`JNIEnv`操作） |
| 数组（如`int[]`） | `jintArray` | （需通过`JNIEnv`操作） |
| 对象（如`Object`） | `jobject`   | （需通过`JNIEnv`操作） |

例如，Java的`String`在JNI中对应`jstring`，本地代码不能直接操作`jstring`，必须通过`JNIEnv`的`GetStringUTFChars()`等方法转换为C字符串后使用。


#### 3. **方法绑定：JVM如何找到本地函数**
JVM通过两种方式将Java的`native`方法与本地函数绑定：  

- **静态绑定（默认）**：通过函数名匹配（即前文提到的`Java_包名_类名_方法名`规则）。编译时，`javah`生成的头文件已按此规则声明函数，本地代码实现时需严格遵循，JVM加载库时会自动根据方法名查找。  

- **动态绑定（RegisterNatives）**：通过`JNIEnv->RegisterNatives()`函数手动注册方法映射表（避免严格的命名规则）。例如：  
  ```cpp
  // 方法映射表：{Java方法名, 方法签名, 本地函数指针}
  static JNINativeMethod methods[] = {
      {"getHelloFromC", "()Ljava/lang/String;", (void*)getHelloFromC}
  };
  
  // 注册方法（通常在JVM加载库时调用）
  JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
      JNIEnv* env;
      if (vm->GetEnv((void**)&env, JNI_VERSION_1_6) != JNI_OK) {
          return JNI_ERR;
      }
      jclass clazz = env->FindClass("com/example/JNIDemo");
      env->RegisterNatives(clazz, methods, sizeof(methods)/sizeof(methods[0]));
      return JNI_VERSION_1_6;
  }
  ```  
  动态绑定更灵活，常用于方法名较长或需要动态修改映射的场景。


#### 4. **JNIEnv：本地代码操作Java的“工具集”**
`JNIEnv`是本地代码中最核心的结构体指针，它封装了大量函数，用于实现本地代码与Java层的交互，主要功能包括：  

- **操作Java对象**：创建对象（`NewObject`）、获取/设置字段（`GetFieldID`/`SetIntField`）等。  
- **调用Java方法**：通过方法ID（`GetMethodID`）调用实例方法，通过`GetStaticMethodID`调用静态方法。  
- **处理字符串和数组**：字符串转换（`NewStringUTF`/`GetStringUTFChars`）、数组操作（`GetIntArrayElements`/`ReleaseIntArrayElements`）等。  
- **异常处理**：抛出Java异常（`ThrowNew`）、检查异常（`ExceptionCheck`）等。  

`JNIEnv`是线程私有（每个线程有独立的`JNIEnv`实例），不能跨线程传递。


#### 5. **内存管理：避免跨层内存泄漏**
Java和C/C++的内存管理机制不同（Java有垃圾回收GC，C/C++需手动管理），JNI定义了严格的引用规则：  

- **本地引用（Local Reference）**：默认创建的引用（如`FindClass`返回的`jclass`、`NewObject`返回的`jobject`），仅在当前native方法调用期间有效，方法返回后会被JVM自动释放。  
- **全局引用（Global Reference）**：通过`NewGlobalRef`创建，生命周期由开发者控制，需手动调用`DeleteGlobalRef`释放，否则会导致内存泄漏。  
- **弱全局引用（Weak Global Reference）**：通过`NewWeakGlobalRef`创建，不影响GC回收对应的Java对象，需手动调用`DeleteWeakGlobalRef`释放。  


### 三、总结
JNI的核心价值是**打破Java与本地代码的语言壁垒**，其原理可概括为：  
1. 定义`native`方法声明Java对本地代码的调用需求；  
2. 通过统一的类型映射解决跨语言数据传递问题；  
3. 借助`JNIEnv`接口实现本地代码对Java层的操作；  
4. 通过静态或动态绑定让JVM找到对应的本地函数；  
5. 通过引用规则管理跨层内存，避免泄漏。  

在Android中，JNI常被用于性能敏感场景（如游戏引擎、音视频处理）、访问硬件接口（如传感器驱动）或复用现有C/C++库（如OpenCV、FFmpeg）。