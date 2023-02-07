> 🔥 **Hi，我是小余。** **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [[小余的自习室](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)] ，在成功的路上不迷路！**

## 前言

作为一个Android开发者，可能你觉得我是不是跑错场了，Android开发又用不到C++的知识。。

额，如果你这么觉得，只能说明你还是一个Android基础开发者，C++在高级领域，如性能优化，NDK，音视频，framework，ART虚拟机等都使用的它，所以学习C++对我们Android开发其实非常必要。

**本篇是重学C++系列的第一篇，希望文章对你有启发**。

## 目录



## 1.char类型以及char*类型的变量初始化问题
```
### 案例：

void charBug() 
{
	char c1 = 'yes';//截断，取最后一个字符:'s'
	char c2 = "yes";//报错：const char*类型的值不能用于初始化char类型的实体
	char c3 = &c1;//报错： char*类型的值不能用于初始化char类型的实体
	

	const char* cs1 = '/'; //报错：char 类型的值不能用于初始化const char*类型的实体
	const char* cs2 = "/";//正确取值为：'/''\0'
	const char* cs3 = &c1;//正确取到c1的地址值

}
```

上面代码显示了：

- 1.字符串作为等号右边数值，则给左边变量赋值的是该字符串常量的地址。
- 2.const char*类型或者char*类型的值（如一个字符串或者字符的地址），不能赋值给一个char类型的变量。
- 3.char类型的值作为左值不能直接赋值给一个char*或者const char*类型的变量。

**那在C++中怎么去规避这些操作呢？使用string**

```
string s1(1, 'yes');//s
string s2(3, 'yes');//sss

string s3("yes");//yes
string s4("/");// /
string s5(1,'/');// /
```

**C++中使用string规避了这些因为char*和char赋值一起的编译器错误。**

##  2.C语言数组作为参数退化问题
### 案例：
```
double average(int arr[]) {
	double result = 0.0;
	int len = sizeof(arr) / sizeof(arr[0]);
	for (int i = 0; i < len; i++) {
		result += arr[i];
	}
	return result/len;
}
int main()
{
   cout << "Hello World!\n";
   //charBug();
   int arr[] = { 1,2,3,4,5,6,7,8,9,10 };
   cout<<average(arr)<<endl;

}
打印结果为：1 
```

说明在函数内部的是arr其实是一个指针了，更好命名应为p_arr，一个指针的sizeof为4个字节，而arr[0]也是指针所以len=1。C语言设计者考虑的是不能将一个大的容器传递到一个函数内部，而**只传递容器的地址**，用于节省空间。

**下面来看C++是如何优化这个问题的。**
使用STL中的容器来计算：

```
double average2(vector<int>& vec) {
	double result = 0.0;

	vector<int>::iterator it = vec.begin();
	for (; it != vec.end(); ++it) {
		result += *it;
	}
	return result / vec.size();

}
vector<int> vec{ 1,2,3,4,5,6,7,8,9,10 };
cout << average2(vec) << endl;

打印结果：5.5
```

输出了正确的平均结果

##  3.C语言移位问题
### 案例1：
```
void bitMove() {
	char c1 = 0x63;//0110 0011
	c1 = c1 << 4;//右移4位
	printf("0x%02x\n",c1); //推算：0011 0000 ->0x30

	c1 = 0x63;
	c1 = c1 >> 4;//左移4位
	printf("0x%02x\n", c1);//推算：0000 0110 ->0x06
	
	char c2 = 0x93;//1001 0011
	c2 = c2 << 4;//右移4位
	printf("0x%02x\n", c2); //推算：0011 0000 ->0x30
	
	c2 = 0x93;
	c2 = c2 >> 4;//左移4位
	printf("0x%02x\n", c2);//推算：0000 0110 ->0x09?

}

结果：
0x30 0x06 0x30 0xfffffff9
```

看到最后一个值为0xfffffff9，推算因为0x06.
这是由于算术位移和逻辑位移的区别导致的。
#### 逻辑位移
对于无符号数，左移时对低位补0，右移对高位也是补0.

#### 算术位移
对于无符号数的正数因为三码一致，则和逻辑位移的补齐方式一致，对于有符号数的负数，则左移时对低位补0，右移时则对高位补1.所以就发生前面的0x93在左移的过程中使用的是高位补1，得到的就是0xfffffff9形式，因为负数使用的是算术位移。

**那C语言中如何规避这种问题的发生了？考虑将有符号数转为无符号数即可：char -> unsigned char**

这里还有个点为什么是0xfffffff9而不是0xf9，题中明明是使用的char类型啊，为啥会有4个字节啊，
原因就是%x要求的是无符号整形变量，如果传入的是char类型，则会有一个整数提升的过程，由于0xf9是一个负数char，提升的时候会对高位使用1填充，所以得到的就是0xfffffff9，而不是0xf9。

**修复方式也是将有符号数转为无符号数：char -> unsigned char，这样高位会使用0来填充。**

### 案例2：
```
void bitMove1() {
	unsigned char x = 0xFF;
	const unsigned char BACK_UP = (1 << 7);
	const unsigned char ADMIN = (1 << 8);
	//printf("0x%x\n", BACK_UP);
	//printf("0x%x\n", ADMIN);

	if (x & BACK_UP) {
		cout << "BACK_UP" << endl;
	}
	if (x & ADMIN) {
		cout << "ADMIN" << endl;
	}

}  
```

结果只打印除了BACK_UP，这是为啥呢？
我们把两个printf打开看看：
结果：一个0x80 一个0x00

**原因位移数不能大于位数，如果移动超过位数就是全部移出，导致归0.**

以上两个案例都是因无符号数位移引起的异常情况。

**那么在C++中如何规避的呢？使用bitset**

```
void bitMove2() {
	bitset<10> x = 0xFF;
	const bitset<10> BACK_UP = (1 << 7);
	const bitset<10> ADMIN = (1 << 8);
	printf("BACK_UP:0x%x\n", BACK_UP);
	cout<<"BACK_UP:binary:" << BACK_UP << endl;
	printf("ADMIN:0x%x\n", ADMIN);
	cout << "ADMIN:binary:" << ADMIN << endl;

	if ((x & BACK_UP) == BACK_UP) {
		cout << "BACK_UP" << endl;
	}
	if ((x & ADMIN) == ADMIN) {
		cout << "ADMIN" << endl;
	}

}
结果：
BACK_UP:0x80
BACK_UP:binary:0010000000
ADMIN:0x100
ADMIN:binary:0100000000
BACK_UP
```

可以看到bitset指定了一个位长度10的无符号数，所以在移动8位后，得到的是0x100，而非0.

##  4.C语言中的强制转换问题
### 案例1：

```
void cast1() {
	int arr[] = { 1,2,3,4 };
	cout <<"arr size:" << sizeof(arr) / sizeof(arr[0]) << endl;
	int thresholp = -1;
	if (sizeof(arr) / sizeof(arr[0]) > thresholp) {
		cout << "arr len is big than thresholp" << endl;
	}
	else
	{
		cout << "thresholp is big than arr len" << endl;
	}
	cout << (unsigned int)(-1) << endl;
}
结果：
arr size:4
thresholp is big than arr len
4294967295
```

结果-1居然比4大，对一些不了解底层细节的同学会比较懵逼。

**重点在sizeof(arr) / sizeof(arr[0]) > thresholp这个判断，涉及了一个强转转换的问题**

- 1.sizeof(arr) / sizeof(arr[0])返回的是一个unsigned int类型的值，thresholp是一个int类型的值。

在计算机的比较中，如果无符号类型和有符号类型值进行比较，则会将有符号类型的一边转换为无符号类型，然后再进行比较，这是一个隐式 的强转操作。

int类型的-1转换为无符号后的值为：4294967295，所以得到的结果是thresholp is big than arr len

- 2.在这个案例中，只需要使用一个int类型的值去存储sizeof(arr) / sizeof(arr[0])即可

  ```
  void cast1() {
  	int arr[] = { 1,2,3,4 };
  	cout <<"arr size:" << sizeof(arr) / sizeof(arr[0]) << endl;
  	int thresholp = -1;
  	int len = sizeof(arr) / sizeof(arr[0]);
  	if (len > thresholp) {
  		cout << "arr len is big than thresholp" << endl;
  	}
  	else
  	{
  		cout << "thresholp is big than arr len" << endl;
  	}
  }
  ```

  两个都是有符号类型，不会进行强转，所以可以得到正确的结果。

### 案例2：

```
void cast2() {
	double result = 0.0;
	int arr[] = { 10,20,30,40 };
	unsigned int len  = sizeof(arr) / sizeof(arr[0]);
	for (unsigned int i = 0; i < len; i++) {
		result += (1 / arr[i]);
	}
	cout << result << endl;
	return;
}
结果为0，
```

**其实问题在于“1 / arr[i]”,分子和分母都为int类型的情况下，则得到的也会是一个int型的结果，也就说会丢失进度。**对于分子为1的情况，则得到的结果都会为0.
**如何改正呢？**只需要将(1 / arr[i])改为(1.0 / arr[i]),由于1.0 / arr[i]不会丢失精度，所以最后得到的是一个正确的浮点数。

通过上面的两个案例可以看出，**C语言中的一些隐式转换会变为一些隐藏的bug和陷阱，且不容器排查**。

**C++中也哪些解决方案？**
C++中可以**使用显示的转换方式**：static_cast,const_cast,dynamic_cast,reinterpret_cast

- **static_cast**；等价于隐式转换的一种运算符，用来表示显示的转换。
- **const_cast**：用来去除复合类型中的const和volatile属性
- **dynamic_cast**：父子类指针之间转换
- **reinterpret_cast**：指针类型之间转换，有一些风险

对案例2：

```
void cast2() {
	double result = 0.0;
	int arr[] = { 10,20,30,40 };
	unsigned int len  = sizeof(arr) / sizeof(arr[0]);
	for (unsigned int i = 0; i < len; i++) {
		result += (static_cast<double>(1))/ arr[i];
	}
	cout << result << endl;
	return;
}
结果：0.208333
```

正确结果

## 5.C语言整数溢出问题
### 案例1：

```
void intOverflow() {
	int i = 0x7ffffff0;//2147483632
	for (; i > 0; i++) {
		cout << "adding" << endl;
	}
	cout << "end ??" << endl;
}
```

第一眼觉得，i用于大于0，for循环不会结束。
实际结果：

```
adding
adding
adding
adding
adding
adding
adding
adding
adding
adding
adding
adding
adding
adding
adding
adding
end ??-2147483648
```

在执行了16次adding还是退出了，这是因为i是int类型的变量，有符号类型的整数：范围会在-2147483648~2147483647之间，数值添加到2147483647之间后，再加1会变为-2147483648
就**好像时钟走了一圈后，又从0点开始一样**。

### 案例2：

```
void intOverflow2() {
	int a = 200, b = 300, c = 400, d = 500;
	cout << a * b * c * d << endl;
}
结果：-884901888
```

这里也是发生了整数溢出导致结果变为负数。

**C++中如何解决这种大数溢出的问题？使用扩展库，如boost库https://www.boost.org。**

```
#include <boost/multiprecision/cpp_int.hpp>
using namespace boost::multiprecision;

void intOverflow2() {
	cpp_int a = 200, b = 300, c = 400, d = 500;
	cout << a * b * c * d << endl;
}
结果：12000000000
```



##  6.C语言字符串缺陷。
###  案例1：
```
void strTest() {
	char str1[] = "abcdef";
	cout << "str1_strlen:" << strlen(str1) << endl;
	cout <<"str1_sizeof:" << sizeof(str1) / sizeof(str1[0]) << endl;

	char str2[] = "abc\0def";
	cout << "str2_strlen:" << strlen(str2) << endl;
	cout << "str2_sizeof:" << sizeof(str2) / sizeof(str2[0]) << endl;

}
结果：
str1_strlen:6
str1_sizeof:7
str2_strlen:3
str2_sizeof:8
```

从上面的结果可以看出C语言中的**字符串其实是以\0结束**的，这就会限制很多应用场景，且运行效率低。

**C++中的解决方案**
**使用C++中的string类或者一些开源库解决方案如redis库的实现**：https://redis.com https://github.com/redis

> Redis 是一种开源（BSD 许可）内存数据结构存储，用作数据库、缓存、消息代理和流引擎。Redis 提供数据结构，
> 例如 字符串、散列、列表、集合、带范围查询的排序集合、位图、hyperloglogs、地理空间索引和流。
>
> 关于更多Redis的信息，可以自行查阅Redis官网。
>



## 小结

- 1.C语言是一种高级语言的的低级语言，小巧，高效，接近底层，但是细节和陷阱较多
- 2.C++完全继承了C语言的特性，但是提出了一系列更现代化和工程化的特性，包容性强，但是语言自身比较复杂。
- 3.本章针对的字符，字符串，指针，数组，整数等表示，类型转化和移位等操作的说明，讲解了C的设计和C++中响应的解决方案，帮助大家搭好C/C++基础。我是[小余](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)，我们下期见。
