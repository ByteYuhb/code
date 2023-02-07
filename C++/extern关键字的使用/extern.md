🔥 **Hi，我是小余。** **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [[小余的自习室](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)] ，在成功的路上不迷路！**

## 前言

extern 是C/C++语言中**表明全局变量或者函数作用范围（可见性）的关键字**，编译器收到extern通知，则其声明的变量或者函数可以在本模块或者其他模块使用。

> 对于函数而言，由于函数的声明如“extern int method();”与函数定义“int method(){}”可以很清晰的区分开来，为了简便起见，可以把extern关键字省略，于是有了我们常见的函数声明方式“int method();”，然而对于变量并非如此，变量的定义格式如“int i;”，声明格式为“extern int i;”，如果省略extern关键字，就会造成混乱，故不允许省略。

## 一般用法

### 在本模块中使用：

```c++
extern int a;
extern int b;
int maxbb(int l,int r) {
	return l > r ? l : r;
}
int main() {
	cout << maxbb(a, b) << endl;
}

int a = 10;
int b = 20;
```

当前模块的main函数在a和b之前定义，所以main函数对a和b是没有访问权限的，可以在main之前定义

```c++
extern int a;
extern int b;
```

这样就可以正常访问到a和b了。

### 跨模块中
在模块1 _extern.cpp中定义：

```c++
extern int _a = 10;
extern int _b = 20;

int maxAB(int a,int b) {
	return a > b ? a : b;
}
```

在模块2 main.cpp中定义：

```c++
extern int _a;
extern int _b;
int maxAB(int a,int b);
int main() {
	cout << "a:" << _a << " b:" << _b << endl;
	cout << maxAB(100,200) << endl;
}

结果：
a:10 b:20
200
```

看到使用extern关键字使用到了外部模块_extern.cpp中的全局变量以及函数。

**标准定义使用extern关键字的步骤为**：

- 1.定义一个.h文件用来声明需要提供外部访问的变量或者函数。

```c++
module1.h
extern int _a;
extern int _b;
int maxAB(int a, int b);
```

- 2.定义一个.cpp文件来初始化全局变量或者函数的实现

  ```c++
  module1.cpp
  
  #include "module1.h"
  
  int _a = 100;
  int _b = 200;
  
  int maxAB(int x, int y) {
  	return x > y ? x : y;
  }
  ```

  

- 3.在需要使用到的地方使用extern关键字修饰。

  ```c++
  //main.cpp
  extern int _a;
  extern int _b;
  int maxAB(int a,int b);
  int main() {
  cout << "a:" << _a << " b:" << _b << endl;
  cout << maxAB(100,200) << endl;
  }
  
  运行结果：
  a:100 b:200
  200
  ```

  

如果我们把module1.h中如下定义：

```c++
extern int _a = 100；
extern int _b = 200;
```

**这样肯定会报错的，因为extern int _a是变量声明，而extern int _a = 100则是变量声明和定义。**
**因为module1.cpp中是将"module1.h" include到cpp中的，如果在.h中声明和定义即使用extern int _a = 100方式，则会引起重复定义，而extern int _a是变量声明，并非定义，所有不会重复**。

## extern 使用过程中的一些注意事项

这里引用掘友出的题：

>数组通过外部声明为指针时，数组和指针是不能互换使用的；那么请思考一下，在 A 文件中定义数组 char a[100]；在 B 文件中声明为指针：extern char *a；此时访问 a[i]，会发生什么；

先说结果，会引起 **segmentation fault** 报错；

这里涉及到了数组与指针的区别
### 数组与指针的区别

数组变量和枚举常量一样都属于符号常量。注意，不是数组变量这个符号的值是那块内存的首地址，
而是数组变量这个符号本身代表了首地址，它就是这个地址值。这就是数组变量属于符号常量的意义所在。

**由于数组变量是一个符号常量，所以其可以看做是右值，而指针作为变量，只能看作为左值。**
**右值永远不等于左值，所以将指针赋予数组常量是不合法的。**

例如：char a[] 中的 a 是常量，是一个地址，char *a 中 a 是一个变量，一个可以存放地址的变量。

### extern 声明全局变量的内部实现

**被extern修饰的全局变量，在编译期不会分配空间，而是在链接的时候通过索引去别的文件中查找索引对应的地址**。假设文件中声明了一个：

```c++
extern char a[];
```

这是一个外部变量的声明，声明了一个外部的字符数组，编译器看到这玩意时不会立即给a分配空间，而是等链接器进行寻址，编译器会将所有关于a的引用化为一个不包含类型的标号，编译完成后会得到一个目标中间产物a.o，但是此时a.o中关于a还是一个无类型标号，链接器连接的时候发现这个标号，会去其他
中间产物中查找和这个标号对应的地址，找到之后替换这个标号。最后链接为一个可执行的文件。

```c++
extern char * a;
```

这也是一个外部变量的声明，它声明了一个字符指针。编译以及链接过程和前面字符数组过程类似，只是此时链接器在寻找符号地址的时候，找到的是前面声明的
extern char a[] 字符数组，这里就有问题了**：由于在这个文件中声明的 a 是一个指针变量而不是数组，链接器的行为实际上是把指针 a 自身的地址定位到了另一个 .cpp 文件中定义的数组首地址上，**
**而不是我们所希望的把数组的首地址赋予指针 a**。（这很容易理解：指针变量也需要占用空间，如果说把数组的首地址赋给了指针 a，那么指针 a 本身在哪里存放呢？）
这就是症结所在了。所以此例中指针 a 的内容实际上变成了数组 a 首地址开始的 4 字节表示的地址

上述加粗部分的可以理解为，链接器认为 a 变量本身的内存位置是数组的首地址，但其实 a 的位置是其他位置，其内容才是数组首地址。

这里着重要理解的是：**指针的地址以及指针的内容的区别，指针本身也存在地址，链接器将数组的首地址赋予了指针本身，这样肯定是不行的**。

举个例子，定义 char a[] = "abcd"，则外部变量 extern char a[] 的地址是 0x12345678 (数组的起始地址)，
而 extern char *a 是重新定义了一个指针变量 a，其地址可能是 0x87654321，因此直接使用 extern char *a 是错误的。

通过上述分析，我们得到的最重要的结论是：**使用 extern 修饰的变量在链接的时候只找寻同名的标号，不检查类型，所以才会导致编译通过，运行时出错。**

## extern "C"
**extern "C"的真实目的是实现类C和C++的混合编程**。在C++源文件中的语句前面加上extern "C"，表明它按照类C的编译和连接规约来编译和连接，而不是C++的编译的连接规约。这样在类C的代码中就可以调用C++的函数or变量等。（注：我在这里所说的类C，代表的是跟C语言的编译和连接方式一致的所有语言）

### C和C++互相调用
前面我们说了extern “C”是为了实现C和C++混编，接下来就来讲解下C与C++如何相互调用，在讲解相互调用之前，我们先来**了解C和C++编译和链接过程的差异**。

#### C++的编译和链接
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

#### C的编译和连接
C语言中并没有重载和类这些特性，故不会像C++一样将log(int i)编译为_log_int,而是直接编译为_log函数，当C++去调用C中的log(int i)方法时，会找不到_log_int方法，此时extern “C”的作用就体现出来了。

下面来看下C和C++是如何互相调用的。

#### C++中调用C的代码
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

#### C中调用C++的代码

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

- 1.如果直接在.c文件中include “cppHeader.h”是会报错的，因为cppHeader.h中包含了extern “C”，而将cppHeader.h包含进来，会直接展开cppHeader.h内容，而extern “C”在C语言中是不支持的，所以会报错。
- 2.在.c文件中不加extern void _log_i(int i)也会报错

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
    
    

## 总结
本文主要讲解了关于extern的三个知识点：

- 1.extern的基础用法：本模块以及跨模块的使用
- 2.extern的在使用过程中的一些注意点，主要通过数组和指针的区别来讲解。
- 3.extern “C”在C++中的用法以及原理：讲解了关于C和C++互相调用以及内部实现机制。


好了，关于C++/C中的extern讲解就到这里，我是[小余](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)，欢迎点赞加关注，我们下期见。









 