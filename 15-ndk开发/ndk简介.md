## 1.ndk简介

ndk全称Native Developer Kits，Android NDK也是Android SDK的一个扩展集，用来扩展SDK的功能。
**NDK打通了Java和C/C++之间的开发障碍，让Android开发者也可以使用C/C++语言开发APP**。

众所周知：

> Java是在C/C++之上的语言，语言金字塔越往上对开发者就更加贴近，也就是更容易开发，但是性能相对也就越低。越往下对开发人员的要求也就越高，但是实现后的产品性能也越高，因为可以自己控制内存等模块的使用，而不是让Java虚拟机自行处理。

**NDK的使用场景**一般在：

- 1.**为了提升这些模块的性能**，对图形，视频，音频等计算密集型应用，将复杂模块计算封装在.so或者.a文件中处理。
- 2.使用的是C/C++进行编写的**第三方库移植**。如ffmppeg，OpenGl等。
- 3.某些情况下为了**提高数据安全性**，也会封装so来实现。毕竟使用纯Java开发的app是有很多逆向工具可以破解的。

NDK开发并不一定适合所有程序员，光要会用Java又要懂C/C++这一点就已经淘汰很多了，更别说还得理解CPU架构相关知识。大部分Android开发只需要使用Java代码以及sdk提供的api就可以了，同时这也是中级和高级开发的分水岭。

> **所以对于一些希望进阶高阶开发的程序员来说，NDK是必须理解且经常练习深化的一个技能。**

## 2.目录

![NDK](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/NDK.png)!](F:\联迪生活文档\笔记\code\15-ndk开发\NDK.png)

## 3.NDK架构分层

我们知道使用NDK开发最终目标是为了将C/C++代码编译生成.so动态库或者.a静态库文件，并将库文件提供给Java代码调用。
**所以按架构来分可以分为以下三层**：

- **1.构建层**
- **2.Java层**
- **3.native层**

### 3.1：构建层：

要得到目标的so文件，需要有个构建环境以及过程，**将这个过程和环境称为构建层**。

构建层需要将C/C++代码编译为动态库so，那么这个编译的过程就需要一个构建工具，构建工具按照开发者指定的规则方式来构建库文件，类似apk的Gradle构建过程。

在讲解NDK构建工具之前，我们先来了解一些关于CPU架构的知识点：**Android abi**

#### Android abi

ABI即Application Binary Interface，定义了二进制接口交互规则，以适应不同的CPU，**一个ABI对应一种类型的CPU**。

Android目前支持以下7种ABI：

- 1.**armeabi**：第5代和6代的ARM处理器，早期手机用的比较多。
- 2.**armeabi-v7a**:第7代及以上的 ARM 处理器。
- 3.**arm64-v8a**:第8代，64位ARM处理器
- 4.**x86**:一般用在平板，模拟器。
- 5.**x86_64**:64位平板。

大多数CPU都支持多种ABI，但是为了获得最佳性能，最好使用CPU的主要ABI。**如同时存在多个ABI（比如so文件），会安装最优ABI，其他的不会安装**。

![](https://pic3.zhimg.com/80/v2-947da210f0ee54880431b58761616e3e_720w.webp)

注意：**64位设备支持使用32位的so库，但是以32位模式运行时，会丢失64位一些性能特性（ART, WebView, Media, etc）**

有了abi知识的铺垫，下面我们再来讲解下关于NDK中构建工具的使用

### 构建工具
常规的NDK构建工具有两种：

- 1.**ndk-build**： 
- 2.**Cmake**

#### 1.ndk-build：

**ndk-build其实就是一个脚本**。早期的NDK开发一直都是使用这种模式，

运行ndk-build相当于运行一下命令：

```
$GNUMAKE -f <ndk>/build/core/build-local.mk
```

`$GNUMAKE` 指向 GNU Make 3.81 或更高版本，`<ndk>` 则指向 NDK 安装目录

使用ndk-build需要配合两个mk文件：**Android.mk**和**Application.mk**。

下面我们来了解**ndk-build构建模型**：



![EshyU](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/EshyU.png)



上图中画出了一个完整的so库的生成以及使用过程：可以大致分为三个步骤：

- **1.使用JNI编译带native的Java文件，生成对应的.h头文件。**

- **2.带上1中生成的头文件，C/C++文件以及其他三方库.a或者.so，一起编译并链接为so动态库。**

- 3.**Java代码通过JNI调用到so库中的函数。**

根据上面的so库的生成过程，我们可以提取出以下元素：

C/C++的`src文件`，.`h头文件`，`链接库.a/.so`，`编译的库名称name`，`编译类型静态库/动态库`，`abi类型（armeabi，armeabi-v7a..）`。

**以上相关元素的配置都会在Android.mk和Application.mk中体现出来**

##### Android.mk

Android.mk文件更像是一个传统的makefile文件，其定义源代码路径，头文件路径，链接器的路径来定位库，模块名，构建类型等。

语法：

- **LOCAL_PATH :=$(call my-dir)**

  call my-dir表示的意思是调用构建系统提供的my-dir函数，获取当前文件所在的文件系统目录。每个Android.mk文件都必须在开始时调用这个函数获取LOCAL_PATH。

- **include $(CLEAR_VARS)**

  这里表示清除当前系统的各种变量，可以简单理解为进行一个初始化的操作。

- **LOCAL_SRC_FILES := xx.c xxx.c**

  用来指定当前使用的C/C++源文件。注意这里不需要添加头文件，编译系统会自动给我们查找。

- **LOCAL_LDLIBS    := -llog**

  表示指定需要使用到的第三方库文件，静态或者动态都可以。使用 `-l` 前缀传递特定系统库的名称。例如，以上示例指示链接器生成在加载时链接到 `/system/lib/liblog.so` 的模块：

- **TARGET_PLATFORM :=  android-3**

  指定当前需要编译的目标Android版本号

- **LOCAL_MODULE    :=  xxname**

  指定当前库的名称，生成的so文件会自动在name前面添加lib前缀，如指定的LOCAL_MODULE 为log，则获取到的so库文件的liblog。

- **include $(BUILD_SHARED_LIBRARY)**

  指定当前库是静态库还是动态库或者其他类型库。包括下面几种：

  - `BUILD_STATIC_LIBRARY`: 构建静态库
  - `PREBUILT_STATIC_LIBRARY`: 对已有的静态库进行包装，使其成为一个模块。
  - `BUILD_SHARED_LIBRARY`：构建动态库、
  - `PREBUILT_SHARED_LIBRARY`: 对已有的静态库进行包装，使其成为一个模块。
  - `BUILD_EXECUTABLE`: 构建可执行文件。

这里列出了关于Android.mk的常用语法，其他语法可以参考[官网](https://developer.android.google.cn/ndk/guides/android_mk?hl=zh-cn)。

##### Application.mk

其定义了Android app的相关属性。如：`Android Sdk版本`，`调试或者发布模式`，`目标平台ABI`，`标准C/C++库`等

- **APP_ABI := XXX**：指定目标平台ABI，可以选填的有 x86 、X86_64 、armeabi-v8a、armeabi-v7a、all 等，如若选择 all 则会构建构建出所有平台的 so，如果不填写该项，那么将默认构建为 armeabi 平台下的库。

  您也可以指定多个值，方法是将它们放在同一行上，中间用空格分隔。

  ```makefile
  APP_ABI := armeabi-v7a arm64-v8a x86
  ```

- **APP_STL := gnustl_static**：NDK 构建系统提供了由 Android 系统给出的最小 C++ 运行时库 （system/lib/libstdc++.so）的 C++ 头文件。

- **APP_CPPFLAGS :=-std=gnu++11 -fexceptions，**：指定编译过程的 flag ,可以在该选项中开启 exception rtti 等特性，但是为了效率考虑，最好关闭 rtti。

- **APP_PLATFORM :=android-21**：指定创建的动态库的平台

  其他字段可以参考[官网](https://developer.android.google.cn/ndk/guides/application_mk?hl=zh-cn)

关于ndk-build就讲到这里了，毕竟这种方式以及很少用了。下面我们来讲解另外一种编译方式。

#### 2.Cmake

##### Cmake简介

在AS2.2之后，工具中增加了Cmake的支持。

cmake和unix make工具不一样，其并不是一个编译系统，而是一个编译系统的生成器，简单理解就是，**他是用来生成makefile文件的**，而前面讲解的Android.mk其实就是一个makefile类文件，cmake使用一个CmakeLists.txt的配置文件来生成对应的makefile文件。

使用Cmake构建过程如下图：

![cmakebuild](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/cmakebuild.png)

可以看到Cmake构建so的过程其实包括两步：

步骤1：**使用Cmake生成编译的makefiles文件**

步骤2：**使用Make工具对步骤1中的makefiles文件进行编译为库或者可执行文件。**

**那使用Cmake优势在哪里呢**？相信了解Gradle构建的都知道，为什么现在的apk构建过程会这么快，就是因为其在编译apk之前会生成一个任务依赖树，因此在多核状态下，任务可以在异步状态下执行，所以apk构建过程会非常快。而我们的Cmake也是类似，其在生成makefile过程中会自动分析源代码，创建一个组件之间依赖的关系树，这样就可以大大缩减在make编译阶段的时间。

这里再发一张编译系统的流程图：

![buildsystem](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/buildsystem.jpg)

从图中我们也可以看到可执行文件在被生成之前，是有很多依赖任务的，这些任务使用Cmake创建一张任务依赖树，可以大大降低编译时间。这也是为什么谷歌在使用了Cmake编译之后，对ndk-build的支持就大大降低了。

> 所以在2.2以后有两种方式来编译C/C++代码、谷歌为了兼容一些旧项目，扔保留了ndk-build的方式。

如果非必须，不推荐使用ndk-build来构建，因为这样构建源码后，是**无法使用方法跳转、方法提示等功能**的！如果要改代码，就等于文本编辑器写代码。相反 CMake 是支持这些的，因此更有助于提高开发效率，且使用CMake最大优点就是可以**动态调试C/C++代码**，和VS中调试一样。是不是很cool、。

讲解了这么多，下面我们重点来看Cmake的语法：

##### Cmake基本语法：

Cmake提供了很多对编译进行配置的语法，这里只提取平时用的比较频繁的几个：

**cmake_minimum_required**：要求的Cmake最低版本

```makefile
cmake_minimum_required(VERSION 3.10.2)
```

**file(<opr> <filename> <out-var> [...])**：文件操作，参数opr：表示文件类型如读文件还是写文件或者文件是Sting类型还是二进制类型等

```makefile
Reading
  file(READ <filename> <out-var> [...])
  file(STRINGS <filename> <out-var> [...])
  file(<HASH> <filename> <out-var>)
  file(TIMESTAMP <filename> <out-var> [...])
  file(GET_RUNTIME_DEPENDENCIES [...])

Writing
  file({WRITE | APPEND} <filename> <content>...)
  file({TOUCH | TOUCH_NOCREATE} [<file>...])
  file(GENERATE OUTPUT <output-file> [...])
  file(CONFIGURE OUTPUT <output-file> CONTENT <content> [...])
...
```

**find_file**:查找文件

```makefile
find_file(myfile
        ${CMAKE_CURRENT_SOURCE_DIR}"/include/SerialPort.h)
```

**add_library**：添加项目so库，就是你需要创建的目标库声明处

```makefile
add_library( # Sets the name of the library.
        dy-register-lib

        # Sets the library as a shared library.
        SHARED
    
        # Provides a relative path to your source file(s).//这里可以使用多个文件
        dy_register.cpp people/people.cpp)

```

**find_library**：查找第三方库，这个在某个库需要关联其他库进行连接的时候就需要查找到库

```makefile
find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )
```

**target_link_libraries**：将第三方库链接到目标库中，使用find_library找到第三方库之后，使用target_link_libraries某个库就可以在链接时找到第三方库。

```makefile
target_link_libraries( # Specifies the target library.
        dy-register-lib
        # Links the target library to the log library
        # included in the NDK.
        ${log-lib} )
```

**link_libraries**：将第三方库链接到之后声明的所有目标库中，和target_link_libraries不同之处，这里不需要声明关联的库名称，而是之后声明的所有库文件

```makefile
link_libraries(${log-lib} //目标库名称)
```



**target_include_directories**：目标库关联的include头文件夹。

```makefile
target_include_directories(log-lib //目标库名称
        PUBLIC //库类型
        ${CMAKE_CURRENT_SOURCE_DIR}/include //目标include文件夹路径)
```

**include_directories**：所有目标库关联的include头文件夹。可以添加多个路径

```makefile
include_directories(
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

**set**：设置变量，如使用某个变量存储路径等。

```makefile
set(SRC_PATH ${CMAKE_CURRENT_SOURCE_DIR}/include/SerialPort.h)
```

**aux_source_directory**：查找在一个文件夹下所有文件，并写入一个List变量下面，之后对这个文件夹下的src文件的所有操作都可以使用这个变量来处理，就不需要一个个文件导入了，方便和安全很多。

```makefile
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src _SRC_FILES)
add_library( # Sets the name of the library.
        dy-register-lib

        # Sets the library as a shared library.
        SHARED
    
        # Provides a relative path to your source file(s).//这里可以使用多个文件
        ${_SRC_FILES})
```

  **messages**：打印日志，如需要打印某个变量的值，可以使用这个方式：

```makefile
General messages
  message([<mode>] "message text" ...)
  mode：FATAL_ERROR SEND_ERROR WARNING，DEBUG等值。用法和logcat类似

Reporting checks
  message(<checkState> "message text" ...)
  checkState：CHECK_START  CHECK_PASS  CHECK_FAIL等值
Configure Log
  message(CONFIGURE_LOG <text>...)
```

  好了，关于Cmake语法就讲到这里，需要看更多命令的可以参考[官方文档](https://cmake.org/cmake/help/latest/manual/cmake-commands.7.html)

  ##### Cmake构建项目配置

使用Cmake进行构建需要在build.gradle配置文件中声明externalNativeBuild

```groovy
android {
    defaultConfig {
        externalNativeBuild {
            // For ndk-build, instead use the ndkBuild block.
            cmake {
            	//声明当前Cmake项目使用的Android abi
            	abiFilters "armeabi-v7a"
               	//提供给Cmake的参数信息 可选
                arguments "-DANDROID_ARM_NEON=TRUE", "-DANDROID_TOOLCHAIN=clang"

     			//提供给C编译器的一个标志 可选
                cFlags "-D__STDC_FORMAT_MACROS"

               //提供给C++编译器的一个标志 可选
                cppFlags "-fexceptions", "-frtti","-std=c++11"

               	//指定哪个so库会被打包到apk中去 可选
                targets "libexample-one", "my-executible-demo"
            }
        }
    }
    externalNativeBuild {
        cmake {
            path "src/main/cpp/CMakeLists.txt" //声明cmake配置文件路径
            version "3.10.2" //声明cmake版本
        }

    }
}
```

### 3.2：Java层

#### 怎么选择正确的so？

通常情况下，我们在编译so的时候就需要确定自己设备类型，根据**设备类型选择对应abiFilters**。

由于不同CPU指令的向前兼容性，假设我们只有arm7代处理器，那么只需要选择armeabi-v7a即可，如果既有7代也有7代之前的，可以同时选择armeabi和armeabi-v7a，设备会自动选择使用正确版本，同理对于32位还是64位处理器也是一样的道理。模拟器一般使用x86的，所以如果该so也需要运行在模拟器上需要加上x86的abi。

注意：使用as编译后的so会自动打包到apk中，如果需要提供给第三方使用，可以到`build/intermediates/cmake/debug or release` 目录中copy出来。

第三方库一般直接放在main/jniLibs文件夹下，也有放在默认libs目录下的，但是必须在build.gradle中声明jni库的目录：

```java
sourceSets {
    main {
        jniLibs.srcDirs = ['jniLibs']
    }
}
```

#### Java层如何调用so文件中的函数？

对于Android上层代码来说，在将包正确导入到项目中后，只需要一行代码就可以完成动态库的加载过程。

```java
System.load("/data/local/tmp/libnative_lib.so"); 
System.loadLibrary("native_lib");
```

以上两个方法用于加载动态，区别如下：

- 1.**加载路径不同**：load是加载so的完整路径，而loadLibrary是加载so的名称，然后加上前缀lib和后缀.so去默认目录下查找。

- 2.**自动加载库的依赖库的不同**：load不会自动加载依赖库；而loadLibrary会自动加载依赖库。

[动态库加载](http://gityuan.com/2017/03/26/load_library/)过程调用栈如下：

```java
System.loadLibrary()
  Runtime.loadLibrary()
    Runtime.doLoad()
      Runtime_nativeLoad()
          LoadNativeLibrary()
              dlopen()
              dlsym()
              JNI_OnLoad()
```

loadLibrary()和load()都用于加载动态库，loadLibrary()可以方便自动加载依赖库，load()可以方便地指定具体路径的动态库。对于loadLibrary()会将将xxx动态库的名字转换为libxxx.so，再从/data/app/[packagename]-1/lib/arm64，/vendor/lib64，/system/lib64等路径中查询对应的动态库。无论哪种方式，最终都会调用到LoadNativeLibrary()方法，该方法主要操作：

- **1.通过dlopen打开动态库文件**

- **2.通过dlsym找到JNI_OnLoad符号所对应的方法地址**

- **3.通过JNI_OnLoad去注册对应的jni方法**

### 3.3：Native层

说到native层就要讲到JNI了。

#### 什么是JNI

JNI（全名Java Native Interface）Java native接口，其可以让一个运行在Java虚拟机中的Java代码被调用或者调用native层的用C/C++编写的基于本机硬件和操作系统的程序。简单理解为就是一个连接Java层和Native层的桥梁。

开发者可以在native层通过JNI调用到Java层的代码，也可以在Java层声明native方法的调用入口。

#### JNI注册方式

JNI有静态注册和动态注册两种注册方式：

##### 静态注册

**步骤1**.在Java中声明native方法

```java
package com.android.myapplication;
...
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        TextView tv = findViewById(R.id.sample_text);
        tv.setText(stringFromJNI());
    }
    private native String stringFromJNI();
    static {
        System.loadLibrary("native-lib");
    }

}
```

**步骤2**.在native层新建一个C/C++文件,并创建对应的方法，

```c++
#include <jni.h>
#include <string>

extern "C" JNIEXPORT jstring JNICALL
Java_com_android_myapplication_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```

注意C/C++文件声明的方法名规则为：**Java+包名+方法名**，千万别搞错了，可能会报找不到对应函数的错。

建议使用AS快捷键自动生成函数名。

![image-20230217232932334](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/image-20230217232932334.png)

##### 动态注册

动态注册其实就是使用到了前面分析的so加载原理：**在最后一步的JNI_OnLoad中注册对应的jni方法**

这样在类加载的过程中就可以自动注册native函数。

```java
#include <jni.h>
#include <string>
#include <assert.h>

extern "C" JNIEXPORT jstring JNICALL
Java_com_android_myapplication_MainActivity_stringFromJNI1(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
#define JNI_CLASS_NAME "com/android/myapplication/MainActivity" //java路径
static JNINativeMethod gMethods[] = {
        {"stringFromJNI","()Ljava/lang/String;",(void *)Java_com_android_myapplication_MainActivity_stringFromJNI1},

};
int register_dynamic_Methods(JNIEnv *env){
    std::string s = JNI_CLASS_NAME;
    const char* className = s.c_str();
    jclass clazz = env->FindClass(className);
    if(clazz == NULL){
        return JNI_FALSE;
    }
    //注册JNI方法
    if(env->RegisterNatives(clazz,gMethods,sizeof(gMethods)/sizeof(gMethods[0]))<0){
        return JNI_FALSE;
    }
    return JNI_TRUE;
}
//类加载时会调用到这里
jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env = NULL;
    if(vm->GetEnv(reinterpret_cast<void **>(&env), JNI_VERSION_1_6) != JNI_OK){
        return JNI_ERR;
    }
    assert(env != NULL);

    if(!register_dynamic_Methods(env)){
        return JNI_ERR;
    }
    return JNI_VERSION_1_6;
}

```

核心方法：**RegisterNatives**，jni注册native方法。

动态注册和静态注册最终都可以将native方法注册到虚拟机中，推荐使用动态注册，更不容易写错，静态注册每次增加一个新的方法都需要查看原函数类的包名。

#### JNI基础语法

##### 1.Java类型以及数据结构

- **1.基本数据类型**

  Java的数据类型可以直接与C/C++的基本类型映射，因此Java的基本类型对开发人员是透明的。

  | Java类型 | Native类型 | 描述             |
  | -------- | ---------- | ---------------- |
  | boolean  | jboolean   | unsigned 8 bits  |
  | byte     | jbyte      | signed 8 bits    |
  | char     | jchar      | unsigned 16 bits |
  | short    | jshort     | signed 16 bits   |
  | int      | jint       | signed 32 bits   |
  | long     | jlong      | signed 64 bits   |
  | float    | jfloat     | 32 bits          |
  | double   | jdouble    | 64 bits          |
  | void     | void       |                  |

  为了方便使用下面宏定义来表示true和false

  ```c++
  #define JNI_FALSE  0
  #define JNI_TRUE   1
  ```

- 2.**引用类型**

  JNI中提供了一系列的引用类型，这些引用类型和Java中的类型是一一对应的。

  - jobject

    - `jclass` (`java.lang.Class` objects)
    - `jstring` (`java.lang.String` objects)
    - jarray(arrays)
      - `jobjectArray` (object arrays)
      - `jbooleanArray` (`boolean` arrays)
      - `jbyteArray` (`byte` arrays)
      - `jcharArray` (`char` arrays)
      - `jshortArray` (`short` arrays)
      - `jintArray` (`int` arrays)
      - `jlongArray` (`long` arrays)
      - `jfloatArray` (`float` arrays)
      - `jdoubleArray` (`double` arrays)
    - `jthrowable` (`java.lang.Throwable` objects)
  
  注意在C中JNI引用类型是以别名的方式定义的：例如
  
  ```c
  typedef jobject jclass;
  ```
  
  而在C++中JNI引用一般都是以对象指针的方式定义：如下：
  
  ```c
  class _jobject {};
  class _jclass : public _jobject {};
  // ...
  typedef _jobject *jobject;
  typedef _jclass *jclass;
  ```
  
  

-  3.**属性（Field）和方法（Method）的ID**

  属性和方法的ID其实是一个C结构体类型的指针：

  ```c
  struct _jfieldID;              /* opaque structure */
  typedef struct _jfieldID *jfieldID;   /* field IDs */
  
  struct _jmethodID;              /* opaque structure */
  typedef struct _jmethodID *jmethodID; /* method IDs */
  ```

  **_jfieldID**：表示**Java层的一个类的属性类型**，是一个结构体，而jfieldID是结构体的一个指针类型。native层可以使用jni对这个属性进行赋值操作。

  **_jmethodID**：表示**Java层的某个类的方法类型**，也是一个结构体，而jmethodID是结构体的一个指针类型。

  

- **4.签名**（Signatures）

  JNI使用的是Java虚拟机的签名描述方式：

  | Type Signature            | Java Type                                       |
  | ------------------------- | ----------------------------------------------- |
  | Z                         | boolean                                         |
  | B                         | byte                                            |
  | C                         | char                                            |
  | S                         | short                                           |
  | I                         | int                                             |
  | J                         | long                                            |
  | F                         | float                                           |
  | D                         | double                                          |
  | L fully-qualified-class ; | 类的权限定描述符：如String -> Ljava.lang.String |
  | [ type                    | type[] ：属性描述                               |
  | ( arg-types ) ret-type    | method type：方法描述                           |

  例如：Java代码：

  ```java
  long f (int n, String s, int[] arr);
  ```

  那么该方法的签名如下：

  ```c
  (I;Ljava/lang/String;[I)J
  ```

##### 2..JavaVM 和 JNIEnv

- **定义**：

  - **JavaVm**

    虚拟机在JNI层的代表，一个进程只有一个JavaVM，所有的线程共用一个JavaVM。
  
  
    - **JNIEnv**
  
      JNIEnv代表Java调用native层的环境，一个封装了几乎所有的JNI方法的指针。
  
      其**只在创建它的线程有效，不能跨线程传递**，不同的线程的JNIEnv彼此独立。
  
      native 环境中创建的线程，如果需要访问JNI，必须调用AttachCurrentThread 进行关联，然后使用DetachCurrentThread 解除关联。
  
      ```c++
      JavaVM *jvm; /* already set */
       f()
       {
           JNIEnv *env;
           (*jvm)->AttachCurrentThread(jvm, (void **)&env, NULL);
           ... /* use env */
       }
      ```
  


值得注意的是：**JNIENV在C语言和C++中调用方式是有区别的：**

```
C风格：(*env)->NewStringUTF(env, “Hellow World!”);
C++风格：env->NewStringUTF(“Hellow World!”);
```

注：**C++风格其实就是对C风格的再次封装,下次碰到这个问题就不要想不通啦、**


##### 3.JNI相关函数

前面我们说过JNI方法一般都是使用JNIEnv去调用，而JNIEnv又是一个指针，所以JNI中有哪些函数，**只需要找到JNIEnv的实现体就可以了**。找到JNIEnv的实现：

**在jni.h中:**

```c++
#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;
typedef _JavaVM JavaVM;
#else
typedef const struct JNINativeInterface* JNIEnv;
typedef const struct JNIInvokeInterface* JavaVM;
#endif
```

可以看到如果是C文件中，则**JNIEnv是JNINativeInterface结构体的一个指针**。

在C++文件中是对JNIEnv起的一个别名，定位到`_JNIEnv`是在哪里定义的。

```c++
struct _JNIEnv {
    /* do not rename this; it does not seem to be entirely opaque */
    const struct JNINativeInterface* functions;

#if defined(__cplusplus)

    jint GetVersion()
    { return functions->GetVersion(this); }

    jclass DefineClass(const char *name, jobject loader, const jbyte* buf,
        jsize bufLen)
    { return functions->DefineClass(this, name, loader, buf, bufLen); }

    jclass FindClass(const char* name)
    { return functions->FindClass(this, name); }
    ...
    ...

```

可以看到`_JNIEnv`中定义了一个JNINativeInterface指针functions，然后对JNIEnv的所有操作都是使用这个functions指针，相当于`_JNIEnv`只是一个代理。而在C语言中直接使用的是JNINativeInterface指针，**这也是为什么JNIEnv在C和C++调用方式不一致的原因**。

**_JNIEnv方法整理如下：**

```c++
struct JNINativeInterface {
    /*获取当前JNI版本信息:*/
    jint        (*GetVersion)(JNIEnv *);
	/*
	定义一个类：类是从某个字节数组吧buf中读取出来的
	原型：jclass DefineClass(JNIEnv *env, const char *name, jobject loader,
const jbyte *buf, jsize bufLen);
	*/
    jclass      (*DefineClass)(JNIEnv*, const char*, jobject, const jbyte*,
   	                     jsize);
    /*
    找到某个类：
    函数原型：
    jclass FindClass(JNIEnv *env, const char *name);
    参数name：为类的全限定名
    如String类："java/lang/String"
    如java.lang.Object[] ： "[Ljava/lang/Object;"
    */
    jclass      (*FindClass)(JNIEnv*, const char*);
    
    /*
    获取当前类的父类：
    通常在使用FindClass获取到类之后，再调用这个函数
    */
    jclass      (*GetSuperclass)(JNIEnv*, jclass);
    /*
    函数原型：
    jboolean IsAssignableFrom(JNIEnv *env, jclass clazz1,jclass clazz2);
	定义某个类clazz1是否可以安全的强制转换为另外一个类clazz2
    */
    jboolean    (*IsAssignableFrom)(JNIEnv*, jclass, jclass);
	/*检测是否发生了异常*/
    jboolean    (*ExceptionCheck)(JNIEnv*);
    /*检测是否发生了异常,并返回异常*/
    jthrowable  (*ExceptionOccurred)(JNIEnv*);
    /*打印出异常描述栈*/
    void        (*ExceptionDescribe)(JNIEnv*);
    /*清除异常*/
    void        (*ExceptionClear)(JNIEnv*);
    /* 抛出一个异常 成功返回0，失败返回其他值*/
    jint        (*Throw)(JNIEnv*, jthrowable);
    /* 创建一个新的Exception，并制定message，然后抛出*/
    jint        (*ThrowNew)(JNIEnv *, jclass, const char *);
   	/*抛出一个FatalError*/
    void        (*FatalError)(JNIEnv*, const char*);
	/*创建一个全局的引用,需要在不使用的时候调用DeleteGlobalRef解除全局引用*/
    jobject     (*NewGlobalRef)(JNIEnv*, jobject);
    /*删除全局引用*/
    void        (*DeleteGlobalRef)(JNIEnv*, jobject);
    /*删除局部引用*/
    void        (*DeleteLocalRef)(JNIEnv*, jobject);
    /*是否是同一个Object*/
    jboolean    (*IsSameObject)(JNIEnv*, jobject, jobject);
	/*创建一个局部引用*/
    jobject     (*NewLocalRef)(JNIEnv*, jobject);
	/*在不调用构造函数的情况下，给jclass创建一个Java对象，注意该方法不能用在数组的情况*/
    jobject     (*AllocObject)(JNIEnv*, jclass);
    
    /*创建一个Object，对于jmethodID参数必须使用GetMethodID获取到构造函数（with <init> as the method name and void (V) as the return type）*/
    jobject     (*NewObject)(JNIEnv*, jclass, jmethodID, ...);
    jobject     (*NewObjectV)(JNIEnv*, jclass, jmethodID, va_list);
    jobject     (*NewObjectA)(JNIEnv*, jclass, jmethodID, const jvalue*);
	
    /*获取到当前对象的class类型*/
    jclass      (*GetObjectClass)(JNIEnv*, jobject);
    /*某个对象是否是某个类的实现对象，和Java中instanceof类似*/
    jboolean    (*IsInstanceOf)(JNIEnv*, jobject, jclass);
    /*获取某个类的方法类型id，非静态方法
    原型：jfieldID GetFieldID(JNIEnv *env, jclass clazz,const char *name, const char *sig);
    clazz：类权限定名
    name：为方法名
    sig：为方法签名描述
    */
    jmethodID   (*GetMethodID)(JNIEnv*, jclass, const char*, const char*);
	
    /*调用某个对象的方法
    jobject：对象
    jmethodID：对象的方法
    返回值：jobject
    */
    jobject     (*CallObjectMethod)(JNIEnv*, jobject, jmethodID, ...);
    jobject     (*CallObjectMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jobject     (*CallObjectMethodA)(JNIEnv*, jobject, jmethodID, const jvalue*);
     /*调用某个对象的方法
    jobject：对象
    jmethodID：对象的方法
    返回值：jboolean
    同理后面的CallByteMethod，CallCharMethodV，CallIntMethod只是返回值不一样而已。
    */
    jboolean    (*CallBooleanMethod)(JNIEnv*, jobject, jmethodID, ...);
    jboolean    (*CallBooleanMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jboolean    (*CallBooleanMethodA)(JNIEnv*, jobject, jmethodID, const jvalue*);
    jbyte       (*CallByteMethod)(JNIEnv*, jobject, jmethodID, ...);
    jbyte       (*CallByteMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jbyte       (*CallByteMethodA)(JNIEnv*, jobject, jmethodID, const jvalue*);
    jchar       (*CallCharMethod)(JNIEnv*, jobject, jmethodID, ...);
    jchar       (*CallCharMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jchar       (*CallCharMethodA)(JNIEnv*, jobject, jmethodID, const jvalue*);
    jshort      (*CallShortMethod)(JNIEnv*, jobject, jmethodID, ...);
    jshort      (*CallShortMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jshort      (*CallShortMethodA)(JNIEnv*, jobject, jmethodID, const jvalue*);
    jint        (*CallIntMethod)(JNIEnv*, jobject, jmethodID, ...);
    jint        (*CallIntMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jint        (*CallIntMethodA)(JNIEnv*, jobject, jmethodID, const jvalue*);
    jlong       (*CallLongMethod)(JNIEnv*, jobject, jmethodID, ...);
    jlong       (*CallLongMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jlong       (*CallLongMethodA)(JNIEnv*, jobject, jmethodID, const jvalue*);
    jfloat      (*CallFloatMethod)(JNIEnv*, jobject, jmethodID, ...);
    jfloat      (*CallFloatMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jfloat      (*CallFloatMethodA)(JNIEnv*, jobject, jmethodID, const jvalue*);
    jdouble     (*CallDoubleMethod)(JNIEnv*, jobject, jmethodID, ...);
    jdouble     (*CallDoubleMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jdouble     (*CallDoubleMethodA)(JNIEnv*, jobject, jmethodID, const jvalue*);
    void        (*CallVoidMethod)(JNIEnv*, jobject, jmethodID, ...);
    void        (*CallVoidMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    void        (*CallVoidMethodA)(JNIEnv*, jobject, jmethodID, const jvalue*);

    
	/*	
	返回一个类的非静态属性id
	原型：jfieldID GetFieldID(JNIEnv *env, jclass clazz,
const char *name, const char *sig);
	参数name：属性的名字
	sig：属性的签名
	*/
    jfieldID    (*GetFieldID)(JNIEnv*, jclass, const char*, const char*);
	
    /*
    获取当前类的某个属性值    
    同理：对于后面的GetShortField，GetBooleanField，GetByteField等只是属性的类型不一样。
	在使用GetFieldID得到jfieldID属性id后，就可以使用Get<type>Field获取属性值。
    */
    jobject     (*GetObjectField)(JNIEnv*, jobject, jfieldID);
    jboolean    (*GetBooleanField)(JNIEnv*, jobject, jfieldID);
    jbyte       (*GetByteField)(JNIEnv*, jobject, jfieldID);
    jchar       (*GetCharField)(JNIEnv*, jobject, jfieldID);
    jshort      (*GetShortField)(JNIEnv*, jobject, jfieldID);
    jint        (*GetIntField)(JNIEnv*, jobject, jfieldID);
    jlong       (*GetLongField)(JNIEnv*, jobject, jfieldID);
    jfloat      (*GetFloatField)(JNIEnv*, jobject, jfieldID);
    jdouble     (*GetDoubleField)(JNIEnv*, jobject, jfieldID);
	
    /*
    设置当前类的某个属性值    
    同理：对于后面的BooleanField，SetByteField，SetShortField等只是属性的类型不一样。
	在使用GetFieldID得到jfieldID属性id后，就可以使用Set<type>Field设置对应属性值。
    */
    void        (*SetObjectField)(JNIEnv*, jobject, jfieldID, jobject);
    void        (*SetBooleanField)(JNIEnv*, jobject, jfieldID, jboolean);
    void        (*SetByteField)(JNIEnv*, jobject, jfieldID, jbyte);
    void        (*SetCharField)(JNIEnv*, jobject, jfieldID, jchar);
    void        (*SetShortField)(JNIEnv*, jobject, jfieldID, jshort);
    void        (*SetIntField)(JNIEnv*, jobject, jfieldID, jint);
    void        (*SetLongField)(JNIEnv*, jobject, jfieldID, jlong);
    void        (*SetFloatField)(JNIEnv*, jobject, jfieldID, jfloat);
    void        (*SetDoubleField)(JNIEnv*, jobject, jfieldID, jdouble);
	
    /*
    获取某个类的静态方法id
    */
    jmethodID   (*GetStaticMethodID)(JNIEnv*, jclass, const char*, const char*);
	
    /*
    调用某个类的静态方法
    同理：后面的CallStaticBooleanMethod，CallStaticByteMethod等方法只是返回类型不一样而已。
    */
    jobject     (*CallStaticObjectMethod)(JNIEnv*, jclass, jmethodID, ...);
    jobject     (*CallStaticObjectMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jobject     (*CallStaticObjectMethodA)(JNIEnv*, jclass, jmethodID, const jvalue*);
    jboolean    (*CallStaticBooleanMethod)(JNIEnv*, jclass, jmethodID, ...);
    jboolean    (*CallStaticBooleanMethodV)(JNIEnv*, jclass, jmethodID,
                        va_list);
    jboolean    (*CallStaticBooleanMethodA)(JNIEnv*, jclass, jmethodID, const jvalue*);
    jbyte       (*CallStaticByteMethod)(JNIEnv*, jclass, jmethodID, ...);
    jbyte       (*CallStaticByteMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jbyte       (*CallStaticByteMethodA)(JNIEnv*, jclass, jmethodID, const jvalue*);
    jchar       (*CallStaticCharMethod)(JNIEnv*, jclass, jmethodID, ...);
    jchar       (*CallStaticCharMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jchar       (*CallStaticCharMethodA)(JNIEnv*, jclass, jmethodID, const jvalue*);
    jshort      (*CallStaticShortMethod)(JNIEnv*, jclass, jmethodID, ...);
    jshort      (*CallStaticShortMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jshort      (*CallStaticShortMethodA)(JNIEnv*, jclass, jmethodID, const jvalue*);
    jint        (*CallStaticIntMethod)(JNIEnv*, jclass, jmethodID, ...);
    jint        (*CallStaticIntMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jint        (*CallStaticIntMethodA)(JNIEnv*, jclass, jmethodID, const jvalue*);
    jlong       (*CallStaticLongMethod)(JNIEnv*, jclass, jmethodID, ...);
    jlong       (*CallStaticLongMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jlong       (*CallStaticLongMethodA)(JNIEnv*, jclass, jmethodID, const jvalue*);
    jfloat      (*CallStaticFloatMethod)(JNIEnv*, jclass, jmethodID, ...);
    jfloat      (*CallStaticFloatMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jfloat      (*CallStaticFloatMethodA)(JNIEnv*, jclass, jmethodID, const jvalue*);
    jdouble     (*CallStaticDoubleMethod)(JNIEnv*, jclass, jmethodID, ...);
    jdouble     (*CallStaticDoubleMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jdouble     (*CallStaticDoubleMethodA)(JNIEnv*, jclass, jmethodID, const jvalue*);
    void        (*CallStaticVoidMethod)(JNIEnv*, jclass, jmethodID, ...);
    void        (*CallStaticVoidMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    void        (*CallStaticVoidMethodA)(JNIEnv*, jclass, jmethodID, const jvalue*);
	
    //获取静态属性的id
    jfieldID    (*GetStaticFieldID)(JNIEnv*, jclass, const char*,
                        const char*);
	/*
	获取某个类的静态属性的值：
	同理：GetStaticBooleanField，GetStaticByteField等后续函数都只是属性的类型不一样而已
	*/
    jobject     (*GetStaticObjectField)(JNIEnv*, jclass, jfieldID);
    jboolean    (*GetStaticBooleanField)(JNIEnv*, jclass, jfieldID);
    jbyte       (*GetStaticByteField)(JNIEnv*, jclass, jfieldID);
    jchar       (*GetStaticCharField)(JNIEnv*, jclass, jfieldID);
    jshort      (*GetStaticShortField)(JNIEnv*, jclass, jfieldID);
    jint        (*GetStaticIntField)(JNIEnv*, jclass, jfieldID);
    jlong       (*GetStaticLongField)(JNIEnv*, jclass, jfieldID);
    jfloat      (*GetStaticFloatField)(JNIEnv*, jclass, jfieldID);
    jdouble     (*GetStaticDoubleField)(JNIEnv*, jclass, jfieldID);
	
    /*
    设置某个类的静态属性的值
    同理：SetStaticObjectField，SetStaticBooleanField只是设置的值属性类型不同罢了*/
    void        (*SetStaticObjectField)(JNIEnv*, jclass, jfieldID, jobject);
    void        (*SetStaticBooleanField)(JNIEnv*, jclass, jfieldID, jboolean);
    void        (*SetStaticByteField)(JNIEnv*, jclass, jfieldID, jbyte);
    void        (*SetStaticCharField)(JNIEnv*, jclass, jfieldID, jchar);
    void        (*SetStaticShortField)(JNIEnv*, jclass, jfieldID, jshort);
    void        (*SetStaticIntField)(JNIEnv*, jclass, jfieldID, jint);
    void        (*SetStaticLongField)(JNIEnv*, jclass, jfieldID, jlong);
    void        (*SetStaticFloatField)(JNIEnv*, jclass, jfieldID, jfloat);
    void        (*SetStaticDoubleField)(JNIEnv*, jclass, jfieldID, jdouble);
	
    /*
    从一段unicode字符串中创建一个String对象
    原型：jstring NewString(JNIEnv *env, const jchar *unicodeChars,jsize len);
	
	*/
    jstring     (*NewString)(JNIEnv*, const jchar*, jsize);
    /*获取String对象的字符串长度，字符串是默认的UNICODE*/
    jsize       (*GetStringLength)(JNIEnv*, jstring);
    /*
    将jstring转换为一个Unicode字符串数组的指针，在调用ReleaseStringChars之前，这个指针都是有效的
    原型：const jchar * GetStringChars(JNIEnv *env, jstring string,jboolean *isCopy);
    */
    const jchar* (*GetStringChars)(JNIEnv*, jstring, jboolean*);
    /*释放一个Unicode字符串数组的指针*/
    void        (*ReleaseStringChars)(JNIEnv*, jstring, const jchar*);
    
    /*创建一个string对象，使用的字符串是UTF-8类型*/
    jstring     (*NewStringUTF)(JNIEnv*, const char*);
    /*获取UTF-8类型的jstring对象的长度*/
    jsize       (*GetStringUTFLength)(JNIEnv*, jstring);
    /* JNI spec says this returns const jbyte*, but that's inconsistent */
	/*
	返回一个string类型的utf-8类型字符串的指针。生命周期是在调用ReleaseStringUTFChars之前。
	原型：const char * GetStringUTFChars(JNIEnv *env, jstring string,jboolean *isCopy);*/
    const char* (*GetStringUTFChars)(JNIEnv*, jstring, jboolean*);
    /*释放GetStringUTFChars获取到的指针*/
    void        (*ReleaseStringUTFChars)(JNIEnv*, jstring, const char*);
    
    /*获取一个数组对象的长度*/
    jsize       (*GetArrayLength)(JNIEnv*, jarray);
    /*创建一个Object类型的数组对象
    原型：jobjectArray NewObjectArray(JNIEnv *env, jsize length,jclass elementClass, jobject initialElement);
    elementClass：对象类型
    initialElement：对象初始化元素*/
    jobjectArray (*NewObjectArray)(JNIEnv*, jsize, jclass, jobject);
    /*获取某个数组对象索引上的元素，最后一个参数为索引位置*/
    jobject     (*GetObjectArrayElement)(JNIEnv*, jobjectArray, jsize);
    /*设置某个数组对象索引上的元素，倒数第二个参数为索引位置*/
    void        (*SetObjectArrayElement)(JNIEnv*, jobjectArray, jsize, jobject);
	/*创建一个Boolean类型的数组对象，长度为jsize*/
    jbooleanArray (*NewBooleanArray)(JNIEnv*, jsize);
	/*创建一个Byte类型的数组对象，长度为jsize*/
    jbyteArray    (*NewByteArray)(JNIEnv*, jsize);
    jcharArray    (*NewCharArray)(JNIEnv*, jsize);
    jshortArray   (*NewShortArray)(JNIEnv*, jsize);
    jintArray     (*NewIntArray)(JNIEnv*, jsize);
    jlongArray    (*NewLongArray)(JNIEnv*, jsize);
    jfloatArray   (*NewFloatArray)(JNIEnv*, jsize);
    jdoubleArray  (*NewDoubleArray)(JNIEnv*, jsize);
	
    /*获取Boolean数组对象的第一个对象的地址指针：注意和ReleaseBooleanArrayElements配合使用
    函数原型：NativeType *Get<PrimitiveType>ArrayElements(JNIEnv *env,ArrayType array, 		jboolean *isCopy);
    isCopy：当前返回的数组对象可能是Java数组的一个拷贝对象
    */
    jboolean*   (*GetBooleanArrayElements)(JNIEnv*, jbooleanArray, jboolean*);
    /*获取Byte数组对象的第一个对象的地址指针*/
    jbyte*      (*GetByteArrayElements)(JNIEnv*, jbyteArray, jboolean*);
    /*同上*/
    jchar*      (*GetCharArrayElements)(JNIEnv*, jcharArray, jboolean*);
    jshort*     (*GetShortArrayElements)(JNIEnv*, jshortArray, jboolean*);
    jint*       (*GetIntArrayElements)(JNIEnv*, jintArray, jboolean*);
    jlong*      (*GetLongArrayElements)(JNIEnv*, jlongArray, jboolean*);
    jfloat*     (*GetFloatArrayElements)(JNIEnv*, jfloatArray, jboolean*);
    jdouble*    (*GetDoubleArrayElements)(JNIEnv*, jdoubleArray, jboolean*);
	
    //是否数组对象内存
    void        (*ReleaseBooleanArrayElements)(JNIEnv*, jbooleanArray,
                        jboolean*, jint);
    void        (*ReleaseByteArrayElements)(JNIEnv*, jbyteArray,
                        jbyte*, jint);
    void        (*ReleaseCharArrayElements)(JNIEnv*, jcharArray,
                        jchar*, jint);
    void        (*ReleaseShortArrayElements)(JNIEnv*, jshortArray,
                        jshort*, jint);
    void        (*ReleaseIntArrayElements)(JNIEnv*, jintArray,
                        jint*, jint);
    void        (*ReleaseLongArrayElements)(JNIEnv*, jlongArray,
                        jlong*, jint);
    void        (*ReleaseFloatArrayElements)(JNIEnv*, jfloatArray,
                        jfloat*, jint);
    void        (*ReleaseDoubleArrayElements)(JNIEnv*, jdoubleArray,
                        jdouble*, jint);
	/*将一个数组区间的值拷贝到一个新的地址空间，然后返回这个地址空间的首地址，最后一个参数为接收首地址用
	函数原型：
	void Get<PrimitiveType>ArrayRegion(JNIEnv *env, ArrayType array,jsize start, jsize 		len, NativeType *buf);
	*/
    void        (*GetBooleanArrayRegion)(JNIEnv*, jbooleanArray,
                        jsize, jsize, jboolean*);
    void        (*GetByteArrayRegion)(JNIEnv*, jbyteArray,
                        jsize, jsize, jbyte*);
    void        (*GetCharArrayRegion)(JNIEnv*, jcharArray,
                        jsize, jsize, jchar*);
    void        (*GetShortArrayRegion)(JNIEnv*, jshortArray,
                        jsize, jsize, jshort*);
    void        (*GetIntArrayRegion)(JNIEnv*, jintArray,
                        jsize, jsize, jint*);
    void        (*GetLongArrayRegion)(JNIEnv*, jlongArray,
                        jsize, jsize, jlong*);
    void        (*GetFloatArrayRegion)(JNIEnv*, jfloatArray,
                        jsize, jsize, jfloat*);
    void        (*GetDoubleArrayRegion)(JNIEnv*, jdoubleArray,
                        jsize, jsize, jdouble*);

    /* spec shows these without const; some jni.h do, some don't */
    /*设置某个数组对象的区间的值*/
    void        (*SetBooleanArrayRegion)(JNIEnv*, jbooleanArray,
                        jsize, jsize, const jboolean*);
    void        (*SetByteArrayRegion)(JNIEnv*, jbyteArray,
                        jsize, jsize, const jbyte*);
    void        (*SetCharArrayRegion)(JNIEnv*, jcharArray,
                        jsize, jsize, const jchar*);
    void        (*SetShortArrayRegion)(JNIEnv*, jshortArray,
                        jsize, jsize, const jshort*);
    void        (*SetIntArrayRegion)(JNIEnv*, jintArray,
                        jsize, jsize, const jint*);
    void        (*SetLongArrayRegion)(JNIEnv*, jlongArray,
                        jsize, jsize, const jlong*);
    void        (*SetFloatArrayRegion)(JNIEnv*, jfloatArray,
                        jsize, jsize, const jfloat*);
    void        (*SetDoubleArrayRegion)(JNIEnv*, jdoubleArray,
                        jsize, jsize, const jdouble*);
	/*注册JNI函数*/
    jint        (*RegisterNatives)(JNIEnv*, jclass, const JNINativeMethod*,
                        jint);
    /*反注册JNI函数*/
    jint        (*UnregisterNatives)(JNIEnv*, jclass);
    /*加同步锁*/
    jint        (*MonitorEnter)(JNIEnv*, jobject);
    /*释放同步锁*/
    jint        (*MonitorExit)(JNIEnv*, jobject);
    /*获取Java虚拟机VM*/
    jint        (*GetJavaVM)(JNIEnv*, JavaVM**);
	
    /*获取uni-code字符串区间的值，并放入到最后一个参数首地址中*/
    void        (*GetStringRegion)(JNIEnv*, jstring, jsize, jsize, jchar*);
     /*获取utf-8字符串区间的值，并放入到最后一个参数首地址中*/
    void        (*GetStringUTFRegion)(JNIEnv*, jstring, jsize, jsize, char*);
	/*
	1.类似Get/Release<primitivetype>ArrayElements这两个对应函数，都是获取一个数组对象的地址，但是返回是void*，所以是范式编程，可以返回任何对象的首地址，而Get/Release<primitivetype>ArrayElements是指定类型的格式。
	2.在调用GetPrimitiveArrayCcritical之后，本机代码在调用ReleasePrimitiveArray Critical之前不应长时间运行。我们必须将这对函数中的代码视为在“关键区域”中运行。在关键区域中，本机代码不得调用其他JNI函数，或任何可能导致当前线程阻塞并等待另一个Java线程的系统调用。（例如，当前线程不能对另一个Java线程正在编写的流调用read。）*/
    void*       (*GetPrimitiveArrayCritical)(JNIEnv*, jarray, jboolean*);
    void        (*ReleasePrimitiveArrayCritical)(JNIEnv*, jarray, void*, jint);
	
    /*功能类似 Get/ReleaseStringChars，但是功能会有限制：在由Get/ReleaseStringCritical调用包围的代码段中，本机代码不能发出任意JNI调用，或导致当前线程阻塞
    函数原型：const jchar * GetStringCritical(JNIEnv *env, jstring string, jboolean *isCopy);
    */
    const jchar* (*GetStringCritical)(JNIEnv*, jstring, jboolean*);
    void        (*ReleaseStringCritical)(JNIEnv*, jstring, const jchar*);
	
    //创建一个弱全局引用
    jweak       (*NewWeakGlobalRef)(JNIEnv*, jobject);
    //删除一个弱全局引用
    void        (*DeleteWeakGlobalRef)(JNIEnv*, jweak);
	/*检查是否有挂起的异常exception*/
    jboolean    (*ExceptionCheck)(JNIEnv*);
	
    /*
    创建一个ByteBuffer对象，参数address为ByteBuffer对象首地址，且不为空，capacity为ByteBuffe的容量
    函数原型：jobject NewDirectByteBuffer(JNIEnv* env, void* address, jlong capacity);*/
    jobject     (*NewDirectByteBuffer)(JNIEnv*, void*, jlong);
    /*获取一个Buffer对象的首地址*/
    void*       (*GetDirectBufferAddress)(JNIEnv*, jobject);
    /*获取一个Buffer对象的Capacity容量*/
    jlong       (*GetDirectBufferCapacity)(JNIEnv*, jobject);

    /* added in JNI 1.6 */
    /*获取jobject对象的引用类型：
    可能为： a local, global or weak global reference等引用类型：
    如下：
    JNIInvalidRefType = 0,
	JNILocalRefType = 1,
	JNIGlobalRefType = 2,
	JNIWeakGlobalRefType = 3*/
    jobjectRefType (*GetObjectRefType)(JNIEnv*, jobject);
};

```

看到这里面方法还是挺多的，可以总结为下面几类：Class操作，异常Exception操作，对象字段以及方法操作，类的静态字段以及方法操作，字符串操作，锁操作等等。

明细：

- [Interface Function Table](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#interface_function_table)
- 版本信息
  - [GetVersion](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetVersion)
  - [Constants](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#Constants)
- Class 操作
  - [DefineClass](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#DefineClass)
  - [FindClass](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#FindClass)
  - [GetSuperclass](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetSuperclass)
  - [IsAssignableFrom](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#IsAssignableFrom)
- Exceptions
  - [Throw](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#Throw)
  - [ThrowNew](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#ThrowNew)
  - [ExceptionOccurred](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#ExceptionOccurred)
  - [ExceptionDescribe](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#ExceptionDescribe)
  - [ExceptionClear](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#ExceptionClear)
  - [FatalError](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#FatalError)
  - [ExceptionCheck](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#ExceptionCheck)
- 全局和本地引用
  - [Global References](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#global_references)
  - [NewGlobalRef](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#NewGlobalRef)
  - [DeleteGlobalRef](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#DeleteGlobalRef)
  - [Local References](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#local_references)
  - [DeleteLocalRef](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#DeleteLocalRef)
  - [EnsureLocalCapacity](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#EnsureLocalCapacity)
  - [PushLocalFrame](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#PushLocalFrame)
  - [PopLocalFrame](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#PopLocalFrame)
  - [NewLocalRef](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#NewLocalRef)
- 弱全局引用
  - [NewWeakGlobalRef](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#NewWeakGlobalRef)
  - [DeleteWeakGlobalRef](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#DeleteWeakGlobalRef)
- Object 操作
  - [AllocObject](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#AllocObject)
  - [NewObject, NewObjectA, NewObjectV](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#NewObject)
  - [GetObjectClass](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetObjectClass)
  - [GetObjectRefType](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetObjectRefType)
  - [IsInstanceOf](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#IsInstanceOf)
  - [IsSameObject](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#IsSameObject)
- 对象字段Field 操作（可访问）
  - [GetFieldID](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetFieldID)
  - [GetField Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#Get_type_Field_routines)
  - [SetField Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#Set_type_Field_routines)
- 对象方法Method操作
  - [GetMethodID](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetMethodID)
  - [CallMethod Routines, CallMethodA Routines, CallMethodV Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#Call_type_Method_routines)
  - [CallNonvirtualMethod Routines, CallNonvirtualMethodA Routines, CallNonvirtualMethodV Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#CallNonvirtual_type_Method_routines)
- 静态字段 操作
  - [GetStaticFieldID](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetStaticFieldID)
  - [GetStaticField Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetStatic_type_Field_routines)
  - [SetStaticField Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#SetStatic_type_Field_routines)
- 静态方法 操作
  - [GetStaticMethodID](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetStaticMethodID)
  - [CallStaticMethod Routines, CallStaticMethodA Routines, CallStaticMethodV Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#CallStatic_type_Method_routines)
- 字符串 操作
  - [NewString](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#NewString)
  - [GetStringLength](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetStringLength)
  - [GetStringChars](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetStringChars)
  - [ReleaseStringChars](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#ReleaseStringChars)
  - [NewStringUTF](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#NewStringUTF)
  - [GetStringUTFLength](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetStringUTFLength)
  - [GetStringUTFChars](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetStringUTFChars)
  - [ReleaseStringUTFChars](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#ReleaseStringUTFChars)
  - [GetStringRegion](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetStringRegion)
  - [GetStringUTFRegion](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetStringUTFRegion)
  - [GetStringCritical, ReleaseStringCritical](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetStringCritical_ReleaseStringCritical)
- 数组 操作
  - [GetArrayLength](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetArrayLength)
  - [NewObjectArray](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#NewObjectArray)
  - [GetObjectArrayElement](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetObjectArrayElement)
  - [SetObjectArrayElement](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#SetObjectArrayElement)
  - [NewArray Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#New_PrimitiveType_Array_routines)
  - [GetArrayElements Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#Get_PrimitiveType_ArrayElements_routines)
  - [ReleaseArrayElements Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#Release_PrimitiveType_ArrayElements_routines)
  - [GetArrayRegion Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#Get_PrimitiveType_ArrayRegion_routines)
  - [SetArrayRegion Routines](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#Set_PrimitiveType_ArrayRegion_routines)
  - [GetPrimitiveArrayCritical, ReleasePrimitiveArrayCritical](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetPrimitiveArrayCritical_ReleasePrimitiveArrayCritical)
- 注册和反注册native方法
  - [RegisterNatives](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#RegisterNatives)
  - [UnregisterNatives](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#UnregisterNatives)
- Monitor 操作
  - [MonitorEnter](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#MonitorEnter)
  - [MonitorExit](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#MonitorExit)
- NIO 支持
  - [NewDirectByteBuffer](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#NewDirectByteBuffer)
  - [GetDirectBufferAddress](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetDirectBufferAddress)
  - [GetDirectBufferCapacity](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetDirectBufferCapacity)
- 反射操作
  - [FromReflectedMethod](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#FromReflectedMethod)
  - [FromReflectedField](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#FromReflectedField)
  - [ToReflectedMethod](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#ToReflectedMethod)
  - [ToReflectedField](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#ToReflectedField)
- Java VM Interface
  - [GetJavaVM](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html#GetJavaVM)



####  JNI三种引用

前面在分析JNI函数的时候有涉及到JNI的三种引用**Global引用和Local引用以及weak引用**，下面我们来介绍下JNI中的几种引用。

- **Local引用**

  JNI中使用 `jobject`, `jclass`, and `jstring`等来标志一个Java对象，然而在JNI方法在使用的过程中会创建很多引用类型，如果使用过程中不注意就会导致内存泄露。

  Local引用其实就是Java中的局部引用，在声明这个局部变量的方法结束或者退出其作用域后就会被GC回收。

  局部引用可以直接使用：**NewLocalRef来创建**，虽然局部引用可以在跳出作用域后被回收，但是还是希望在不使用的时候调用DeleteLocalRef来手动回收掉。

-  **Global引用**

  **全局引用**，多个地方需要使用的时候就会创建一个全局的引用（NewGlobalRef方法创建），全局引用只有在显示调用DeleteGlobalRef的时候才会失效，不然会一直存在与内存中，这点一定要注意。

  下面是一段不合法的代码：

  ```c++
/* This code is illegal */
  static jclass cls = 0;
  static jfieldID fld;
  
  JNIEXPORT void JNICALL
  Java_FieldAccess_accessFields(JNIEnv *env, jobject obj)
  {
      ...
      if (cls == 0) {
          cls = (*env)->GetObjectClass(env, obj);
          if (cls == 0) {
              ... /* error */
          }
          fid = (*env)->GetStaticFieldID(env, cls, "si", "I");
      }
      /* access the member variable using cls and fid */
      ...
  }
  ```
  
  使用GetObjectClass方法创建的是一个局部变量，此时这个局部变量的有效作用域是这个方法，如果出了这个方法，在第二次调用Java_FieldAccess_accessFields的时候，发现cls是一个不合法的本地引用，这个时候就会报错，所以正确的方法就如下：**使用NewGlobalRef创建全局变量，而GetObjectClass使用一个本地变量存储。**

  ```c++
/* This code is correct. */
  static jclass cls = 0;
  static jfieldID fld;
  
  JNIEXPORT void JNICALL
  Java_FieldAccess_accessFields(JNIEnv *env, jobject obj)
  {
      ...
      if (cls == 0) {
          jclass cls1 = (*env)->GetObjectClass(env, obj);
          if (cls1 == 0) {
              ... /* error */
          }
          cls = (*env)->NewGlobalRef(env, cls1);
          if (cls == 0) {
              ... /* error */      
          }
          fid = (*env)->GetStaticFieldID(env, cls, "si", "I");
      }
      /* access the member variable using cls and fid */
      ...
  }
  ```
  
  这个方法告诉我们**局部变量出了作用域后就失效了，不能被第二次使用，而全局变量可以被二次使用**

  **但是注意在全局变量不需要使用后需要手动调用DeleteGlobalRef，防止内存泄露**。

- **Weak引用**

  弱引用可以使用全局声明的方式，区别在于：**弱引用在内存不足或者紧张的时候会自动回收掉，可能会出现短暂的内存泄露，但是不会出现内存溢出的情况**，建议不需要使用的时候手动调用DeleteWeakGlobalRef释放引用。

#### JNI异常处理

##### Java层异常：

**一般处理方式**：由Java层抛出然后在native层处理

```java
void updateName(String name) throws Exception {
    this.name = name;
    Log.d("HelloJni","你成功调用了HelloCallBack的方法：updateName");
    throw new Exception("dead");
}
```

之后操作就是native层异常操作了。

##### native层异常

**处理方式1**：native层自行处理

```c++
jboolean hasException = env->ExceptionCheck();
if(hasException == JNI_TRUE){
    //打印异常，同Java中的printExceptionStack;
    env->ExceptionDescribe();
    //清除当前异常
    env->ExceptionClear();
}
```

**处理方式2**：native层抛出给Java层处理：

```c++
/*检测是否有异常*/
jboolean hasException = env->ExceptionCheck();
if(hasException == JNI_TRUE){
    //打印异常，同Java中的printExceptionStack;
    env->ExceptionDescribe();
    //清除当前异常
    env->ExceptionClear();

    /*方式2：抛出异常给上面，让Java层去捕获*/
    jclass noFieldClass = env->FindClass("java/lang/Exception");
    std::string msg(_fieldName);
    std::string header = "找不到该字段";
    env->ThrowNew(noFieldClass,header.append(msg).c_str());
    env->ReleaseStringUTFChars(fieldName,_fieldName);

    return;
    
}

```

然后java文件中捕获异常：

```java
try{
    jni.callJavaField("com/android/hellojni/HelloCallBack","namesss");
}catch (Exception e){
    e.printStackTrace();
    return;
}
```



#### C和C++互相调用

在讲解C和C++相互调用之前，我们先来**了解C和C++编译和链接过程的差异**。

##### C++的编译和链接

**大家都知道C++是一个面向对象的编程方式，而面向对象最核心的特性就是重载**，函数重载给我们带来了很大便利性。假设定义如下函数重载方法：

```c++
void log(int i);
void log(char c);
void log(float f);
void log(char* c);
```

则在编译后：

```c++
_log_int
_log_char
_log_float
_log_string
```

**编译后的函数名通过带上参数的类型信息**，这样连接时根据参数就可以找到正确的重载方法。

C++中给的变量编译也是这样一个过程，如全局变量会编译为g_xx,类变量编译为c_xx.连接时也是按照这种机制去查找对应的变量的。

##### C的编译和连接

C语言中并没有重载和类这些特性，故不会像C++一样将log(int i)编译为_log_int,而是直接编译为_log函数，当C++去调用C中的log(int i)方法时，会找不到_log_int方法，此时extern “C”的作用就体现出来了。

下面来看下C和C++是如何互相调用的。

##### C++中调用C的代码

假设一个C的头文件cHeader.h中声明了一个函数_log(int i)，如果C++要调用它，则必须添加上extern关键字。代码如下：

```c++
//cHeader.h
#ifndef C_HEADER
#define C_HEADER
extern void _log(int i);
#endif // !C_HEADER
```

在对应的cHeader.c文件中实现_log方法：

```c++
//cHeader.c
#include "cHeader.h"
#include <stdio.h>

void _log(int i) {
	printf("cHeader %d\n", i);
}
```

在C++中引用cHeader中的_log方法：

```c++
//main.cpp
extern "C" {
	//void _log(int i);
	#include "cHeader.h"
}
int main() {
	_log(100);
}
```

linux执行上述文件的命令为：

- 1.首先执行gcc -c cHeader.c，会产生cHeader.o；
- 2.然后执行g++ -o C++ main.cpp cHeader.o
- 3.执行程序输出：Header 100

**注意**：
在main.cpp文件中可以不用包含函数声明的文件，即“extern "C"{#include"cHeader.h"}”，而直接改用extern "C" void _log(int i)的形式。那main.cpp是如何找到C中的_log函数，并调用的呢？

那是因为**首先通过gcc -c cHeader.c生成一个目标文件cHeader.o，然后我们通过执行g++ -o C++ main.cpp cHeader.o这个命令指明了需要链接的目标文件cHeader.o。**
**main.cpp中只需要声明哪些函数需要以C的形式调用，然后去目标文件中查找即可**。“.o”为目标文件。类似Windows中的obj文件。

##### C中调用C++的代码

C中调用C++中的代码和前面的有所不同，首先在cppHeader.h中声明一个_log_i方法。

```c++
#pragma once
extern "C" {
	void _log_i(int i);
}
```

在对应的cppHeader.cpp中实现该方法：

```c++
#include "cppHeader.h"
#include <stdio.h>

void _log_i(int i) {
	printf("cppHeader:%d\n", i);
}
```

定义一个cMain.c文件调用_log_i方法：

```c++
extern void _log_i(int i);
int main() {
	_log_i(120);
}
```

**注意点**：

- 1.**如果直接在.c文件中include “cppHeader.h”是会报错的，因为cppHeader.h中包含了extern “C”，而将cppHeader.h包含进来，会直接展开cppHeader.h内容，而extern “C”在C语言中是不支持的，所以会报错**。
- 2.在.c文件中不加extern void _log_i(int i)也会报错，因为会无法找到对应的函数。

**linux执行上述文件的命令为**：

- （1）首先执行命令：**g++ cppHeader.cpp -fpic -shared -g -o cppHeader.so**
  该命令是将cppHeader.cpp编译成动态连接库，其中编译参数的解释如下：

  - **-shared** 该选项指定生成动态连接库（让连接器生成T类型的导出符号表，有时候也生成弱连接W类型的导出符号），不用该标志外部程序无法连接。相当于一个可执行文件
  - **-fPIC**：表示编译为位置独立的代码，不用此选项的话编译后的代码是位置相关的所以动态载入时是通过代码拷贝的方式来满足不同进程的需要，而不能达到真正代码段共享的目的。
  - **-g**：为调试

- （2）然后再执行命令：**gcc cMain.c cppHeader.so -o cmain**
  该命令是编译cMain.c文件，同时链接cppHeader.so文件，然后产生cmain的可执行文件。

- （3）最后执行命令：**./cmain** 来执行该可执行程序

  ```c++
  结果：cppHeader:120
  ```


> 前面已经对NDK开发做了一个比较长的理论知识讲解，下面我们使用一个NDK开发案例来对上面理论进行实战.

## 4.NDK实战

今天带来的Demo主要实现下面几个功能：

- **1.native层调用Java层的类的字段和方法**
- **2.native层调用调用第三方so库的api**

下面进入正题：先实现第一个需求：

### 需求1：native层调用Java层的类的字段和方法

- 1.我们先创建一个Native C++的Android项目

  ![pro1](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/pro1.png)

  项目整体结构如下：

  ![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/pro2.png)

  

- 2.然后在cpp文件夹下创建两个文件夹：**src**和**include**，并且在src下创建hellojni.cpp文件，在include文件夹下创建一个hellojni.h文件，用来声明hellojni.cpp中的方法。

  ![pro3](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/pro3.png)

- 3.做了以上准备后我们可以开始编写配置文件CmkaeLists.txt了。

  - 使用add_library创建一个新的so库

    ```cmake
    add_library(
            hello-jni #库名
            SHARED #设置为so共享库，如果需要创建静态库.a文件，需要声明为STAIC
            src/hellojni.cpp #需要参与编译的src文件，如果有多个，则使用空格或者换行包括进来
    )
    ```

  - 使用target_link_libraries将loglib链接到hello-jni.so中，这样可以方便我们在native层打印logcat日志

    ```cmake
    target_link_libraries(hello-jni
    
            ${log-lib})
    ```

    上面两步骤就可以成功完成后就可以去编写c++代码了，是不是很简单

- 4.编写hellojni.cpp文件。

  因为要实现native层调用Java层字段和方法，所以这里定义了两个方法：callJavaField和callJavaMethod

  hellojni.cpp

  ```c++
  //
  // Created by Administrator on 2023/2/22.
  //
  #include "../include/hellojni.h"
  #include <jni.h>
  #include <string>
  #include <assert.h>
  #include <android/log.h>
  #include <string.h>
  
  static const char* TAG ="HelloJni";
  #define LOGD(fmt, args...) __android_log_print(ANDROID_LOG_DEBUG, TAG, fmt, ##args)
  #define LOGE(fmt, args...) __android_log_print(ANDROID_LOG_ERROR, TAG, fmt, ##args)
  
  using namespace std;
  
  
  #define JNI_CLASS_NAME "com/android/hellojni/HelloJni" //java路径
  
  void callJavaField(JNIEnv* env,jobject obj,jstring className,jstring fieldName){
      jboolean iscopy;
      const char* name = env->GetStringUTFChars(fieldName,&iscopy);
      LOGD("invoke method:%s",name);
      /*
       * 步骤1：定义类的全限定名：const char* str = "java/lang/String"
       * 步骤2：找到类的jclass： env->FindClass()
       * 步骤3：读取类的构造函数：env->GetMethodID(c,"<init>","()V");
       * 步骤4：根据构造函数创建一个Object对象：env->NewObject(c,constructMethod);
       * 步骤5：调用对象的字段和方法：
       * */
      //步骤1：定义类的全限定名
      const char* classNameStr = env->GetStringUTFChars(className,&iscopy);
      //步骤2：找到类的jclass
      jclass c = env->FindClass(classNameStr);
      //步骤3：读取类的构造函数
      jmethodID constructMethod = env->GetMethodID(c,"<init>","()V");
      //步骤4：根据构造函数创建一个Object对象
      jobject objCallBack = env->NewObject(c,constructMethod);
  
      //步骤5：调用对象的字段和方法，需要先获取类的字段id和方法id
      /*
       * 获取字段id
       * 参数1：class
       * 参数2：字段名称
       * 参数3：字段签名格式
       * */
      jboolean isCopy;
      const char* _fieldName = env->GetStringUTFChars(fieldName,&isCopy);
     
      /*
      * 此处如果传入一个找不到的字段会报错，如果不做异常处理，应用直接会崩溃，为了更好的知晓问题所在，需要		* 使用jni异常处理机制
       * 此处是native异常，有两种异常处理机制：
       * 方式1：native层处理
       * 方式2：抛出给Java层处理
       * */
      jfieldID field_Name = env->GetFieldID(c,_fieldName,"Ljava/lang/String;");
  
      /*方式1：native层处理*/
      /*检测是否有异常*/
      jboolean hasException = env->ExceptionCheck();
      if(hasException == JNI_TRUE){
          //打印异常，同Java中的printExceptionStack;
          env->ExceptionDescribe();
          //清除当前异常
          env->ExceptionClear();
           
          /*方式2：抛出异常给上面，让Java层去捕获*/
          jclass noFieldClass = env->FindClass("java/lang/Exception");
          std::string msg(_fieldName);
          std::string header = "找不到该字段";
          env->ThrowNew(noFieldClass,header.append(msg).c_str());
          env->ReleaseStringUTFChars(fieldName,_fieldName);
  
          return;
      }
      //没有异常去获取字段的值
      jstring fieldObj = static_cast<jstring>(env->GetObjectField(objCallBack, field_Name));
      const char* fieldC = env->GetStringUTFChars(fieldObj,&isCopy);
      LOGD("你成功获取了字段%s值:%s",_fieldName,fieldC);
      env->ReleaseStringUTFChars(fieldObj,fieldC);
  }
  jboolean callJavaMethod(JNIEnv* env,jobject obj1,jstring className,jstring methodName){
      /*
       * 1.找到类：FindClass
       * 2.创建一个对象
       * 3.获取这个类对应的方法id
       * 4.通过对象和方法id调用对应方法
       * 5.释放内存
       * */
      jboolean isCopy;
      const char* classNameStr = env->GetStringUTFChars(className,&isCopy);
      //1.找到类：FindClass
      jclass callbackClass = env->FindClass(classNameStr);
      //获取构造函数
      jmethodID constructMethod = env->GetMethodID(callbackClass,"<init>","()V");
  	//2.创建一个对象
      jobject objCallBack = env->NewObject(callbackClass,constructMethod);
  
      const char* _methodName = env->GetStringUTFChars(methodName,&isCopy);
      //3.获取这个类对应的方法id
      jmethodID _jmethodName = env->GetMethodID(callbackClass,_methodName,"(Ljava/lang/String;)V");
      const char *str = "123";
      /*切记JNI返回类型不能直接使用基础类型，而要用jni语法中定义的类型：如String需要转换为jstring
       * 不然会报错：JNI DETECTED ERROR IN APPLICATION: use of deleted global reference*/
      jstring result = env->NewStringUTF(str);
      //4.通过对象和方法id调用对应方法
      env->CallVoidMethod(objCallBack,_jmethodName,result);
  
      if(env->ExceptionCheck()){
          env->ExceptionDescribe();
          env->ExceptionClear();
      }
  	//释放字符串内存
      env->ReleaseStringUTFChars(methodName,_methodName);
      env->ReleaseStringUTFChars(className,classNameStr);
      return JNI_TRUE;
  }
  static JNINativeMethod gMethods[] = {
          {"callJavaField","(Ljava/lang/String;Ljava/lang/String;)V",(void *)callJavaField},
          {"callJavaMethod","(Ljava/lang/String;Ljava/lang/String;)Z",(void *)callJavaMethod},
  
  };
  int register_dynamic_Methods(JNIEnv *env){
      std::string s = JNI_CLASS_NAME;
      const char* className = s.c_str();
      jclass clazz = env->FindClass(className);
      if(clazz == NULL){
          return JNI_FALSE;
      }
      //注册JNI方法
      if(env->RegisterNatives(clazz,gMethods,sizeof(gMethods)/sizeof(gMethods[0]))<0){
          return JNI_FALSE;
      }
      return JNI_TRUE;
  }
  jint JNI_OnLoad(JavaVM *vm, void *reserved) {
      JNIEnv *env = NULL;
      if(vm->GetEnv(reinterpret_cast<void **>(&env), JNI_VERSION_1_6) != JNI_OK){
          return JNI_ERR;
      }
      assert(env != NULL);
  
      if(!register_dynamic_Methods(env)){
          return JNI_ERR;
      }
      return JNI_VERSION_1_6;
  }
  
  ```

  这里笔者使用了动态注册的方式，方法比较少的时候使用静态注册也是可以的，方法多了之后建议使用动态注册，且动态注册的代码都是有框架的，直接拿过来改下方法名即可。

- 5.编写Java层的调用代码

  此处要注意的是**调用的类的类名以及包名都要和c++文件中声明的一致，否则会报错**。

  HelloJni.java

  ```java
  package com.android.hellojni;
  
  public class HelloJni {
      static {
          System.loadLibrary("hello-jni");
      }
      public native void callJavaField(String className,String fieldName) ;
      public native boolean callJavaMethod(String className,String methodName) ;
  }
  ```

  

- 以上步骤都完成以后，就可以开始测试了。

  这里我们写了一个测试类：HelloCallBack.java

  ```java
  package com.android.hellojni;
  
  import android.util.Log;
  
  public class HelloCallBack {
      String name = "HelloCallBack";
  
      void updateName(String name){
          this.name = name;
          Log.d("HelloJni","你成功调用了HelloCallBack的方法：updateName");
      }
  }
  ```

  调用代码：

  ```java
  try{
      jni.callJavaField("com/android/hellojni/HelloCallBack","name");
      jni.callJavaMethod("com/android/hellojni/HelloCallBack","updateName");
  }catch (Exception e){
      e.printStackTrace();
      return;
  }
  ```

  测试结果：

  ```java
  D/HelloJni: 你成功获取了字段name值:HelloCallBack
  D/HelloJni: 你成功调用了HelloCallBack的方法：updateName
  ```

  此时如果我们写错了类的字段名：如jni.callJavaField("com/android/hellojni/HelloCallBack","namesss");	

  再运行下：此时

  ```
  //1
  W/System.err: java.lang.NoSuchFieldError: no "Ljava/lang/String;" field "namesss" in class "Lcom/android/hellojni/HelloCallBack;" or its superclasses
  W/System.err:     at com.android.hellojni.HelloJni.callJavaField(Native Method)
  W/System.err:     at com.android.hellojni.MainActivity$1.onClick(MainActivity.java:30)
  ...
  //2
  W/System.err: java.lang.Exception: 找不到该字段namesss
  W/System.err:     at com.android.hellojni.HelloJni.callJavaField(Native Method)
  W/System.err:     at com.android.hellojni.MainActivity$1.onClick(MainActivity.java:30)
  ...
  ```

  注释1的错误日志是我们在helloJni.cpp中使用了ExceptionCheck和ExceptionDescribe检测和打印出来的错误日志。

  注释2的醋五日志是我们在helloJni.cpp中使用了将异常抛出，并且Java层进行了捕获，所以可以打印在Java层打印出报错日志，所以应用也不会崩溃。

  

以上就是一个完整的JNI调用过程。代码已经放在github上，大家可以自行查阅。

下面我们再来实现另外一个功能，**对第三方库的使用**。

### 需求2：native层调用第三方so库的api	

方便点，我直接拿前面案例的hellojni.so来测试，但是为了实现三方调用还需要对文件进行改造

- 1.要实现三方so库调用，我们在hellojni.h中声明两个和hellojni.cpp中对应的方法：callJavaField和callJavaMethod，一般情况下这个头文件是第三方库一起提供的给外部调用的。

  **hellojni.h**



  ```c++
  #include <jni.h>
  #ifndef HELLO_JNI_HELLOJNI_H
  #define HELLO_JNI_HELLOJNI_H
  
  #ifdef __cplusplus
  extern "C" {
  #endif
      void callJavaField(JNIEnv* env,jobject obj,jstring className,jstring fieldName);
      jboolean callJavaMethod(JNIEnv* env,jobject obj1,jstring className,jstring methodName);
   //HELLO_JNI_HELLOJNI_H
  #ifdef __cplusplus
  };
  #endif
  
  #endif
  ```

  注意：**此处对引入的是函数使用了extern "C"对方法进行了包裹，目的就是为了当引用的是cpp文件，extern "C"修饰的函数可以让外部访问到。**



- 2.CMakeLists配置文件改造

  ```c++
  #添加调用库
  add_library( # Sets the name of the library.
               third-call-lib
               # Sets the library as a shared library.
               SHARED
               # Provides a relative path to your source file(s).
               thirdCall.cpp )
  
  #将第三方库添加进来，设置第三个参数为IMPORTED
  add_library(hellojni-lib
          SHARED
          IMPORTED)
  #设置第三方库的路径：IMPORTED_LOCATION
  set_target_properties(hellojni-lib PROPERTIES IMPORTED_LOCATION
                        
          `$`{CMAKE_SOURCE_DIR}/../jniLibs/`$`{ANDROID_ABI}/libhello-jni.so
                        
          )
  #设置调用库的include文件夹
  target_include_directories(
          third-call-lib
          PRIVATE
          ${CMAKE_SOURCE_DIR}/include
  )
  #将第三方库链接到调用库中
  target_link_libraries( 
          third-call-lib
          hellojni-lib)
  
  ```

  

- 3.编写thirdCall.cpp文件，在这内部调用第三方库。

  ```c++
  #include <jni.h>
  #include <string>
  //#include "include/hellojni.h"
  #include <hellojni.h>
  
  extern "C" JNIEXPORT jstring JNICALL
  Java_com_android_thirdsocall_MainActivity_callThirdSoMethod(
          JNIEnv* env,
          jobject obj,jstring className,jstring methodName) {
  
      std::string hello = "hello c++";
      callJavaMethod(env,obj,className,methodName);
  
      return env->NewStringUTF(hello.c_str());
  }
  ```

  这里需要将第三方头文件导入进来，如果CmakeLists文件中没有声明头文件，就只能老老实实的使用`#include "include/hellojni.h"` 这种方式导入了.

- 4.最后测试下：

  ```c++
  callThirdSoMethod("com/android/thirdsocall/HelloCallBack","updateName");
  ```

  结果：

  ```c++
  D/HelloJni: 你成功调用了HelloCallBack的方法：updateName
  ```

### 踩坑记录

笔者写这个demo的过程中也出现一些比较坑的事情：

**比如**：对于JNI方法来说，使用如下方法返回或者调用直接崩溃了，看了半天也不知道为啥？？

```
env->CallVoidMethod(objCallBack,_jmethodName,"123");
```

这段代码编译没问题，但是在运行的时候就报错了：

```
报错：JNI DETECTED ERROR IN APPLICATION: use of deleted global reference
```

最终定位到是最后一个参数需要使用jstring而不能直接使用字符串表示。下面这个方式就没啥问题了。。大家切记哦。

```
env->CallVoidMethod(objCallBack,_jmethodName,env->NewStringUTF("123"));
```

**又比如**：头文件导入一直报找不到头文件，最后发现是使用target_include_directories关联Include文件的文件使用了文件作为目录，而非文件夹，正确做法是关联文件夹，如下：

```c++
#设置调用库的include文件夹
target_include_directories(
        third-call-lib
        PRIVATE
        ${CMAKE_SOURCE_DIR}/include
)
```

项目已经发到github，有需要自己去下载看吧。

## 总结

本文章几乎涵盖关于NDK开发的所有层面，主要是建立在三个层面：**构建层，Java层以及Native层**。

对每一层都进行了较为详细的讲解，其中构建层主要负责构建配置so文件的配置，而真正做事情的是Native层。最后使用了一个Demo对前面理论进行了实践，真正做到活学活用。

最后还是要说一句：**纸上得来终觉浅，绝知此事要躬行，多练才能熟能生巧**。

我是[小余](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)，我们下期见。
