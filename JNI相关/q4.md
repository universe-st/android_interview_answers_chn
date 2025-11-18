# 请阐述动态绑定和静态绑定的原理。

在JNI（Java Native Interface）中，**动态绑定**和**静态绑定**是Java层的native方法与本地代码（如C/C++函数）建立关联的两种机制，核心区别在于关联建立的时机和方式。


### 一、静态绑定（Static Binding）
静态绑定是**编译期**通过固定的命名规则自动建立Java native方法与本地函数的关联，无需手动注册。


#### 原理：
1. **命名规范约束**：  
   本地函数的名称必须严格遵循JNI规定的格式，以便编译器在链接时能自动匹配Java的native方法。格式为：  
   ```c
   Java_<包名>_<类名>_<方法名>[_<参数签名>]
   ```  
   - 包名中的`.`替换为`_`（例如`com.example.Demo`对应`com_example_Demo`）；  
   - 若方法有重载（同方法名不同参数），需在方法名后追加参数签名（通过`javah`工具自动生成），以区分不同重载版本。  

2. **编译期匹配**：  
   当Java代码中声明`native`方法后，通过`javah`工具（或IDE自动处理）生成包含该方法声明的头文件（`.h`），头文件中会按照上述命名规范定义本地函数的原型。开发者只需实现该头文件中声明的函数，编译时编译器会通过函数名自动将Java的native方法与本地函数绑定。  


#### 特点：
- **时机**：编译期（链接阶段）完成绑定；  
- **优点**：无需手动编写注册逻辑，实现简单；  
- **缺点**：函数名冗长（尤其包名较长或方法重载时），维护困难；若方法名或参数修改，需同步修改本地函数名，易出错。  


### 二、动态绑定（Dynamic Binding）
动态绑定是**运行期**通过JNI提供的`RegisterNatives`函数手动注册Java native方法与本地函数的映射关系，无需依赖命名规则。


#### 原理：
1. **定义映射关系**：  
   在本地代码中，通过`JNINativeMethod`结构体数组定义Java方法与本地函数的映射，结构体包含三个字段：  
   - `const char* name`：Java层native方法的名称（字符串）；  
   - `const char* signature`：Java方法的参数和返回值签名（通过`javap -s`命令获取）；  
   - `void* fnPtr`：本地函数的指针（指向实现逻辑的函数）。  

2. **注册触发**：  
   本地库加载时（通过`System.loadLibrary`），JVM会自动调用本地库中的`JNI_OnLoad`函数（若存在）。在`JNI_OnLoad`中，通过`RegisterNatives`函数将上述映射关系注册到JVM，完成Java方法与本地函数的关联。  

   示例流程：  
   ```c
   // 1. 定义本地函数
   JNIEXPORT void JNICALL nativeFunc(JNIEnv* env, jobject obj) { ... }

   // 2. 定义映射关系
   static const JNINativeMethod methods[] = {
       {"javaNativeMethod", "()V", (void*)nativeFunc} // Java方法名、签名、本地函数指针
   };

   // 3. 在JNI_OnLoad中注册
   JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
       JNIEnv* env;
       if ((*vm)->GetEnv(vm, (void**)&env, JNI_VERSION_1_6) != JNI_OK) {
           return JNI_ERR;
       }
       // 注册到指定Java类（如com.example.Demo）
       jclass cls = (*env)->FindClass(env, "com/example/Demo");
       (*env)->RegisterNatives(env, cls, methods, sizeof(methods)/sizeof(methods[0]));
       return JNI_VERSION_1_6;
   }
   ```  


#### 特点：
- **时机**：运行期（本地库加载时）完成绑定；  
- **优点**：本地函数名可自定义（无需遵循冗长规则），灵活度高；方法修改时只需调整映射关系，维护方便；  
- **缺点**：需手动编写注册逻辑（如`JNINativeMethod`数组和`JNI_OnLoad`），易因签名错误或遗漏注册导致绑定失败。  


### 总结
- **静态绑定**：依赖编译期命名规则自动关联，简单但不灵活；  
- **动态绑定**：依赖运行期`RegisterNatives`手动注册，灵活但需额外代码。  

实际开发中，动态绑定更常用（尤其复杂项目），因其可避免命名冗余并简化维护。