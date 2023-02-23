---
theme: channing-cyan
---
## 前言
### 协程的基础使用：
#### 协程的定义：
1.协程通过将复杂性放入库来简化异步编程。程序的逻辑可以在协程中顺序地表达，而底层库会为我们解决其异步性。该库可以将用户代码的相关部分包装为回调、订阅相关事件、在不同线程（甚至不同机器！）上调度执行，而代码则保持如同顺序执行一样简单。

2.协程是一种并发设计模式，您可以在Android平台上使用它来简化异步执行的代码

#### 协程生命周期：

```js
+-----+ start  +--------+ complete   +-------------+  finish  +-----------+
| New | -----> | Active | ---------> | Completing  | -------> | Completed |
+-----+        +--------+            +-------------+          +-----------+
                 |  cancel / fail       |
                 |     +----------------+
                 |     |
                 V     V
             +------------+                           finish  +-----------+
             | Cancelling | --------------------------------> | Cancelled |
             +------------+                                   +-----------+
```

#### 协程的三种创建方式及返回值解析
##### runBlocking：
阻塞创建协程的线程，并运行协程作用域中的job任务
##### launch：
不阻塞创建线程，返回一个Job，且协程没有被挂起，任务执行中可以被异步取消
##### async：
不阻塞创建线程，且协程没有被挂起，返回一个DeferredCoroutine，此时协程状态为Active，

### 协程的核心知识点：
- 1.<mark>CoroutlineDispatcher</mark>：协程调度器指定协程在哪个线程上运行
- 2.<mark>CoroutlineContext</mark>：协程上下文包含一个线程内部所需的所有上下文信息
- 3.<mark>CoroutlineStart</mark>：协程启动模式指定了协程的启动方式：如default或者lazy等
- 4.<mark>CoroutlineScope</mark>：协程作用域指定了协程的作用域：包括协同作用域，主从作用域
- 5.<mark>挂起函数</mark>以及<mark>suspend</mark>关键字的使用：挂起函数会将协程挂起，但不阻塞原有执行线程

### 协程的异常处理机制：
- **对于协同作用域：**
> 1.如果父协程设置了异常处理器，则会将异常传递给父协程处理器去处理，且会终止其他子协程的运行
>
> 2.如果父协程没有设置异常处理器，则先看当前协程有没有设置异常处理器CoroutineExceptionHandler，
> 	如果设置了则直接使用当前异常处理器处理。不会继续传递到下面逻辑中
> 	如果没有设置异常处理器会将异常传递给异常预处理器AndroidExceptionPreHandler，并且使用当前线程的uncaughtExceptionHandler处理异常并退出应用
- **对于主从作用域**：
> 异常自己内部消化，且不会影响其他子协程的处理

### suspend关键字详解
协程其实就是将代码块以挂起函数为分界线分为多个任务，
每个挂起函数都是一个状态机，状态机包括多个lable标签，每个标签代表一个代码块逻辑

> 关于协程的系列文章：

 [Android体系课之--Kotlin协程篇-协程入门 -协程基础用法（一）](https://juejin.cn/post/7109853967957377038)

[Android体系课之--Kotlin协程进阶篇-协程中关键知识点梳理（二）](https://juejin.cn/post/7110011090443960327)

[# Android体系课之--Kotlin协程进阶篇-协程的异常处理机制以及suspend关键字（三）](https://juejin.cn/post/7111594612761837604)

[ Android体系课之--Kotlin协程进阶篇-协程在Android组件中的使用（四）](https://juejin.cn/post/7111900946732417061/)

 Android体系课之--Kotlin协程进阶篇-协程加Retrofit创建一个MVVM模式的网络请求框架（五）
## 简介：
本篇文章主要讲解Kotlin协程在Android中的基础使用
我们先引入相关扩展组件库

```java
implementation "androidx.activity:activity-ktx:1.2.2"
implementation "androidx.fragment:fragment-ktx:1.3.3"
```


- 在第一章我们介绍过协程的三种基本使用方式runBlocking和GlobalScope的launch和async来创建协程
- 对于runBlocking会阻塞原线程的执行，在Android中适用场景较少，而使用GlobalScope创建的协程因为是属于全局协程，如果使用不当可能会导致内存泄露。

**举个例子**：

```java
fun testGlobal(){
	GlobalScope.launch {
		launch {
			throw NullPointerException("test exception!!")
		}
		val r = withContext(Dispatchers.IO){
			//...
		}
		textView.setText(r)
	}
}
```
上面的例子中如果是在Activity中调用会出现哪些问题：
- 1.父协程没有设置异常处理器，导致应用崩溃
- 2.在子线程中更新UI，报异常
- 3.使用GlobalScope创建的协程，在Activity退出后，如果没有取消会导致内存泄露，因为GlobalScope是属于全局应用

针对上面例子，**如果我们需要避免这些问题该如何做呢**？
- 1.父协程设置一个异常处理器或者使用主从作用域SuperVisorJob
- 2.父协程使用Dispatchers.Main处理
- 3.将创建的协程使用全局变量保存，并在Activity调用onDestroy方法时需要取消协程

代码如下：

```java
var job:Job?=null
fun testGlobal(){
	val e1 = CoroutineExceptionHandler { coroutineContext, throwable ->
		println("${throwable.message}")
	}
	job = GlobalScope.launch(e1+Dispatchers.Main) {
		launch {
			throw NullPointerException("test exception!!")
		}
		val r = withContext(Dispatchers.IO){
			//...
		}
		textView.setText(r)
	}
}
override fun onDestroy() {
	super.onDestroy()
	job?.cancel("")
}
```

**可以看到处理起来非常麻烦，那有没有可以代替的对象呢？<mark>MainScope</mark>**

```java
val mainScope = MainScope()
fun testMainScope(){
	mainScope.launch {
		launch {
			//...
		}
		val x = withContext(Dispatchers.IO){
			//...
		}
		textView.test = x
	}
}
override 
fun onDestroy() {
	super.onDestroy()
	mainScope.cancel("")
}
```

来看看MainScope内部：

```java
public fun MainScope(): CoroutineScope = ContextScope(SupervisorJob() + Dispatchers.Main)
```


> 内部其实也是使用<mark>SupervisorJob() + Dispatchers.Main</mark>处理，只是做了一个封装
> 而且还需要在onDestroy中调用协程的cancel
>
- 那有没有更方便的替代对象呢？<mark>LifeCycleScope</mark>

首先需要添加依赖：

```java
implementation "androidx.lifecycle:lifecycle-runtime-ktx:2.3.1"
```

**用法如下：**

```java
fun testLifeCycleScope(){
	lifecycleScope.launch {
		launch {
			//...
		}
		val x = withContext(Dispatchers.IO){
			//...
		}
		textView.test = x
	}
}
```


> 在需要的地方调用lifecycleScope.launch即可创建一个具有监听Activity生命周期的协程。
> 这样就可以做到在Activity被销毁的时候，自动取消协程，防止内存泄露

进入源码看看是怎么实现的：

```java
public val LifecycleOwner.lifecycleScope: LifecycleCoroutineScope
    get() = lifecycle.coroutineScope

public val Lifecycle.coroutineScope: LifecycleCoroutineScope
    get() {
        while (true) {
            val existing = mInternalScopeRef.get() as LifecycleCoroutineScopeImpl?
            if (existing != null) {
                return existing
            }
            val newScope = LifecycleCoroutineScopeImpl(
                this,
                SupervisorJob() + Dispatchers.Main.immediate
            )
            if (mInternalScopeRef.compareAndSet(null, newScope)) {
                newScope.register()
                return newScope
            }
        }
    }
```

> **这里创建了一个<mark>LifecycleCoroutineScopeImpl</mark>对象：**
> 使用的是<mark>SupervisorJob() + Dispatchers.Main</mark>，并将这个LifecycleCoroutineScopeImpl注册到Activity的生命周期中

```java
internal class LifecycleCoroutineScopeImpl(
    override val lifecycle: Lifecycle,
    override val coroutineContext: CoroutineContext
) : LifecycleCoroutineScope(), LifecycleEventObserver {
    init {
        
        if (lifecycle.currentState == Lifecycle.State.DESTROYED) {
            coroutineContext.cancel()
        }
    }

    fun register() {
        launch(Dispatchers.Main.immediate) {
            if (lifecycle.currentState >= Lifecycle.State.INITIALIZED) {
                lifecycle.addObserver(this@LifecycleCoroutineScopeImpl)
            } else {
                coroutineContext.cancel()
            }
        }
    }

    override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
        if (lifecycle.currentState <= Lifecycle.State.DESTROYED) {
            lifecycle.removeObserver(this)
            coroutineContext.cancel()
        }
    }
}
```

- 上面源码可以看出当Activity回调onDestroy方法的时候，会自动解除观察者关系和取消协程


```java
public abstract class LifecycleCoroutineScope internal constructor() : CoroutineScope {
    internal abstract val lifecycle: Lifecycle

    public fun launchWhenCreated(block: suspend CoroutineScope.() -> Unit): Job = launch {
        lifecycle.whenCreated(block)
    }

    public fun launchWhenStarted(block: suspend CoroutineScope.() -> Unit): Job = launch {
        lifecycle.whenStarted(block)
    }

    public fun launchWhenResumed(block: suspend CoroutineScope.() -> Unit): Job = launch {
        lifecycle.whenResumed(block)
    }
}
```

内部实现了
- <mark>launchWhenCreated</mark>:可以在Activity调用onCreate方法后启动协程 
- <mark>launchWhenStarted</mark> ：可以在Activity调用onStart方法后启动协程
- <mark>launchWhenResumed</mark>：可以在Activity调用onResume方法后启动协程


如下实现：

```java
init {
	lifecycleScope.launchWhenResumed {
		println("创建了协程")
	}
}
override fun onResume() {
	super.onResume()
	println("onResume")
}
运行结果：
	onResume
	创建了协程
```

- **在onResume后在执行协程操作**

这个时候看到比一开始已经简单很多了。

我们再来尝试添加异常处理器

```java
lifecycleScope.launch(e1){
//...
}
lifecycleScope.launch(e2){
//...
}
lifecycleScope.launch(e3){
//...
}
```

- 每次执行协程都需要添加异常处理器，那有没方法可以一劳永逸呢？当然有？看下面方法

**我们创建一个全局的GlobalCoroutineExceptionHandler**

```java
class GlobalCoroutineExceptionHandler(val error:Int?=-1,val errorMsg:String?="",val report:Boolean?=false) :CoroutineExceptionHandler{
    override val key: CoroutineContext.Key<*>
        get() = CoroutineExceptionHandler

    override fun handleException(context: CoroutineContext, exception: Throwable) {
        println("接收到异常")
        println("$error:${exception.stackTrace.toString()}")
    }

}
```

- 然后在BaseActivity中使用AppCompatActivity的扩展函数处理

```java
inline fun AppCompatActivity.requestMain(errorCode:Int = -1, errMsg:String = "默认主线程错误", report:Boolean = false,
                noinline block:suspend LifecycleCoroutineScope.() ->Unit){
	lifecycleScope.launch(GlobalCoroutineExceptionHandler(errorCode,errMsg,report)) {
		block.invoke(lifecycleScope)
	}
}
```

在Activity中只需要调用：

```java
requestMain{
	//协程代码
}
```

- 就不需要额外添加异常处理器了
- 同理可以添加requestIO，可以让协程在IO线程上执行异步耗时任务

```js
inline fun AppCompatActivity.requestIO(errorCode:Int = -1, errMsg:String = "默认IO线程错误", report:Boolean = false,
                                             noinline block:suspend LifecycleCoroutineScope.() ->Unit){
   val x =  lifecycleScope.launch(Dispatchers.IO+GlobalCoroutineExceptionHandler(errorCode,errMsg,report)) {
		block.invoke(lifecycleScope)
	}
}

requestIO{
	//协程代码
}

```

> 上面只是针对Activity
>
> 如果是Fragment需要创建一套Fragment的扩展函数

- 以上解析的是Activity或者Fragment中使用协程，如果需要在ViewModel中使用协程如何使用呢？
1. 首先添加ViewModel协程扩展库：

```java
implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:2.3.1"
```

> 与Activity和Fragment不同的是，在ViewModel我们使用的不是lifecycleScope，而是使用viewModelScope，使用viewModelScope，使用viewModelScope。重要的事情说三遍。



```java
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) {
            return scope
        }
        return setTagIfAbsent(
            JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
        )
    }
internal class CloseableCoroutineScope(context: CoroutineContext) : Closeable, CoroutineScope {
    override val coroutineContext: CoroutineContext = context

    override fun close() {
        coroutineContext.cancel()
    }
}
```


- viewModelScope使用的SupervisorJob() + Dispatchers.Main上下文
- 注册了ViewModel的生命周期在ViewModel被销毁的时候做取消操作

```java
final void clear() {
    mCleared = true;
    if (mBagOfTags != null) {
        synchronized (mBagOfTags) {
            for (Object value : mBagOfTags.values()) {
                closeWithRuntimeException(value);
            }
        }
    }
    onCleared();
}


<T> T setTagIfAbsent(String key, T newValue) {
    T previous;
    synchronized (mBagOfTags) {
        previous = (T) mBagOfTags.get(key);
        if (previous == null) {
            mBagOfTags.put(key, newValue);
        }
    }
    T result = previous == null ? newValue : previous;
    if (mCleared) {
        closeWithRuntimeException(result);
    }
    return result;
}

private static void closeWithRuntimeException(Object obj) {
    if (obj instanceof Closeable) {
        try {
            ((Closeable) obj).close();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

- 同理也可以为ViewModelScope创建一套ViewModel的扩展函数，方便每个ViewModel使用


```java
inline fun ViewModel.requestMain(
        errCode: Int = -1, errMsg: String = "", report: Boolean = false,
        noinline block: suspend CoroutineScope.() -> Unit) {
    viewModelScope.launch(GlobalCoroutineExceptionHandler(errCode, errMsg, report)) {
        block.invoke(this)
    }
}

inline fun ViewModel.requestIO(
        errCode: Int = -1, errMsg: String = "", report: Boolean = false,
        noinline block: suspend CoroutineScope.() -> Unit) {
    viewModelScope.launch(Dispatchers.IO + GlobalCoroutineExceptionHandler(errCode, errMsg, report)) {
        block.invoke(this)
    }
}
```


> 以上我们讲解的是Activity，Fragment和ViewModel下面的协程处理，那如果是其他环境下如Service，Dialog，popuWindow等呢？
> 那就得用回我们前面讲解的supervisorJob+Dispatchers.Main的模式，创建一个主从作用域的scope，并记得在dialog等不需要的时候关闭协程

## 总结：

- Android中对Activity，Fragment可以使用lifecycleScope创建一个协程，
- 对ViewModel可以使用ViewModelScope创建协程，
> 上面说的协程创建方式都注册了组件的生命周期，不需要单独调用协程的取消操作。
- 对其他组件，可以使用单独的supervisorJob+Dispatchers.Main的方式创建协程，并在组件被销毁前取消协程。

以上就是笔者对协程在Android组件中使用的理解。

***有理解错误之处，欢迎私信我***

***关注我带你了解更多Android体系课知识***

> 关于协程的系列文章：

 [Android体系课之--Kotlin协程篇-协程入门 -协程基础用法（一）](https://juejin.cn/post/7109853967957377038)

[Android体系课之--Kotlin协程进阶篇-协程中关键知识点梳理（二）](https://juejin.cn/post/7110011090443960327)

[# Android体系课之--Kotlin协程进阶篇-协程的异常处理机制以及suspend关键字（三）](https://juejin.cn/post/7111594612761837604)

[ Android体系课之--Kotlin协程进阶篇-协程在Android组件中的使用（四）](https://juejin.cn/post/7111900946732417061/)

 Android体系课之--Kotlin协程进阶篇-协程加Retrofit创建一个MVVM模式的网络请求框架（五）