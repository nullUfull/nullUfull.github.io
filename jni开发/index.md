# JNI开发


# 常用

## GetIntArrayElements & ReleaseIntArrayElements

GetIntArrayElements是JNI提供的避免数组拷贝,直接在native代码中操作Java数组的高效方法。正确使用可以优化传递数组参数的性能。使用后需要调用ReleaseIntArrayElements释放本地数组,否则可能会内存泄漏。Get和Release需要配对使用。

## 在JNI层打印日志

1. 使用Android日志API

可以使用Android日志API,如__android_log_write, __android_log_print等在JNI层直接打印日志到Android系统日志中。

示例:

```c++
#include <android/log.h>

__android_log_print(ANDROID_LOG_DEBUG, "JNITag", "Hello from JNI!");
```

2. 调用Java层日志方法

可以通过JNIEnv调用Java层定义的日志方法打印日志。需要获取Java方法ID并调用。

示例:

```c++
jclass loggerCls = env->FindClass("com/example/Logger");
jmethodID logMethod = env->GetMethodID(loggerCls, "d", "(Ljava/lang/String;)V");

env->CallVoidMethod(loggerObj, logMethod, env->NewStringUTF("Hello from JNI!")); 
```

3. 输出到stdout/stderr

可以使用printf/std::cout直接输出到标准输出/错误,通过日志捕获工具获取。

示例:

```c++
std::cout << "Hello from JNI!" << std::endl;
```

4. 写入本地文件

可以在JNI层直接打开文件并写入日志信息。

总结下,常见的JNI日志方法是使用Android日志API,或者调用Java层日志方法打印。也可以直接输出到标准流或文件。
