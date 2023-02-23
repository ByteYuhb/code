

## 1.定义
**首先void*中的void代表一个任意的数据类型，"星号"代表一个指针，所以其就是一个任意数据类型的指针。**

> 对于指定数据类型的指针如int* ，double*等，他们的sizeof都是4个字节，因为都是一个指针，只是指针指向的数据类型不一致。

C语言是一个强类型的语言，**那么他们之间有什么区别呢**？[前面一篇文章](https://juejin.cn/post/7185937424516644922)我们说过，**指针+1和-1是和指向数据类型有关的**。

> 假设指针值为0x00000001,指针类型为int类型，整数为n,则计算出来的结果为0x00000001+ n乘以4,这里的4是因为指针类型为int，如果是double，则为0x00000001+ n乘以8. 所以我们使用double类型指针+1，地址移动了8位，而用int指针+1，地址移动了4位，就是这个道理。
> 这说起来其实和数据内存对齐有关。你可以把这里的4和8理解为跳跃力。

那对于void*呢？**其实就是一个未指定跳跃力的指针**。

那void*的跳跃力又什么时候指定？在需要使用的时候指定就可以了，好处：**可以实现泛型编程，节省代码**。

## 2.void*使用场景

### 2.1：当函数传参时不确定数据类型时或者支持多种数据类型传递时。
代码如下：

```c++
void say(int type,void* pArgs) {
	switch (type) 
	{
		case 0:
		{
			double* d = (double*)pArgs;
			break;
		}	
		case 1:
		{
			int* i = (int*)pArgs;
			break;
		}		
	}
}


```

该函数使用一个type来表示当前参数void*的类型，内部通过type判断转换的类型。

### 2.2：函数返回值不需要考虑类型，只关心返回的大小。
如malloc函数:

原型：

```c++
void* malloc(size_t size)
```

代码使用：

```
int* a = nullptr;
double* b = nullptr;
b = (double*)malloc(sizeof(double));
a = (int*)malloc(sizeof(double));
```

可以看到malloc返回值类型为void*，其只返回分配内存的大小，不关心分配后的内存你是使用int还是double类型进行划分.

> 注意：函数外部在接收到void*格式的返回值时，需要强转为自己的数据类型才能使用。

## 3.void*使用中的注意点：
### 1.使用赋值运算符“=”时，void*只能作为左值不能作为右值。
```c++
void*作为一个未指定数据类型的指针，可以指向任何一个数据类型的指针，但是有数据类型的指针，不能指向一个void* 的指针。
```

代码如下：

```c++
int i = 5;
int* pi = &i;
void* pv = pi;
int* pi1 = pv;//编译错误，void*类型的指针不能初始化为指定类型的指针
```

这其实是可以理解的:

```c++
假设void*指定了一个非int类型的数据，如果此时赋值给int*类型的指针，则会发生严重的bug**。
```



### 2.void*类型必须强转为指定类型的数据才能使用。
void*在未指定类型的情况下，是不能直接使用的，**只有在转换为显示类型后才能使用**。**

代码如下：

```
int i = 5;
int* pi = &i;
void* pv = pi;
//cout << *pv << endl;//表达式必须是指向完整对象类型的指针
cout << *(int*)pv << endl;
```

代码中可以看出在未强转为显示类型前，使用void*会报表达式必须是指向完整对象类型的指针.

> 说明void*一定要强转后才能使用.

> 没有强转的void*是没有意义的。


那可能有同学要问了，假设我们并不知道当前void*数据类型，强转错误了会发生什么事。

```c++
int i = 5;
int* pi = &i;
void* pv = pi;

cout<<"(int*)pv:" << (int*)pv << endl;
cout<<"(double*)pv:" << (double*)pv << endl;
cout <<"*(int*)pv:" << *(int*)pv << endl;
cout <<"*(double*)pv:" << *(double*)pv << endl;

运行结果：
(int*)pv:0043F724
(double*)pv:0043F724
*(int*)pv:5
*(double*)pv:-9.25596e+61
```

此时可以看到虽然pv强转为double后，指针指向的地址和强转为int类型指针指向地址是一样的都是0x0043F724。但是取值后就发生了异常，因为double占用的是8个字节，所以取值的是后8位字节，所以取到的是一个错误的值，实际只有4个字节是有效的。

**我们在使用void*强转的时候一定要注意这点**、

### 3.C++中使用(void*)0表示空指针。
在C语言中空指针定义方式：

```c++
#define NULL ((void*)0)
```

在C++语言中：

```c++
#ifndef NULL
	#ifdef __cplusplus
		#define NULL 0
	#else 
		#define NULL ((void*)0)
	#endif
#endif
```

```c++
可以看到在C语言中NULL代表(void*)0，而在C++中NULL代表的是0，使用nullptr来表示(void*)0空指针，
所以在C++中推荐使用新标准的nullptr来初始化一个空指针。
```

### 4.当void*作为函数的参数类型或者返回值类型时，说明该函数可以接收或者返回任意类型的指针。
代码如下：

```c++
void* _say(void* pArgs) {
	return pArgs;
}
int  main()
{
	int _a = 5;
	float f = 10.8;
	int* _pi = &_a;
	float* pf = &f;

	cout << *(int*)_say(_pi) << endl;
	cout << *(float*)_say(pf) << endl;

}
运行结果：
5
10.8
```

代码中可以看出参数void* pArgs可以使用任意类型的实参，返回值也可以返回任意类型的指针，但是最终需要转换为具体类型才能使用。

void*在C++中的作用其实就是为了实现泛型编程，和Java中使用Object来表示是一样的，所以又称为通用指针和泛指针，不过**C++中大部分情况下会使用模板编程来实现泛型**。

上面_say函数代码可以使用下面模板函数代替：

```
T _say(T t) {
	return t;
}
```


区别在于：**模板编程不需要将强制转换为具体类型**，其使用方式如下：

```c++
int _a = 5;
float f = 10.8;
int* _pi = &_a;
float* pf = &f;

cout <<*_say(_pi) << endl;
cout << *_say(pf) << endl;
```

未强转也可以直接得出结果，这是因为模板编程会在编译器帮我们生成具体的函数。
调用了两次_say会分别实现：

```c++
float* _say(float* t) {
	return t;
}
int* _say(int* t) {
	return t;
}
```

所以运行的时候是调用了不同的函数，而使用void*的泛型只调用一个函数。

## 总结
- 1.void*是一个过渡型的指针状态，可以代表任意类型的指针，取值的时候需要转换为具体类型才能取值。其是处于数据类型顶端的状态：

  ![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/void.png)

- 2.void* 使用赋值运算符“=”赋值时，只能将具体类型赋值给void星，不能将void*赋值给具体类型。

- 3.void*一般作为参数或者返回值来实现泛型编程，但是C++中一般考虑使用模板编程来实现。

