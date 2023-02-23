## 前言：

笔者在写这篇文章之前，也白嫖了很多关于Kotlin协程的文章： 这里笔者将他们分为三种： 1.讲的内容很*浅*，没几句可能就结束了，看完就索然无味了 2.讲的内容很*深*，看到一半就开始晕乎乎了，然后可能还是手机好玩。。 3.内容比较*适中*，读者可以在里面获取到一些协程的基本信息，关键内容可能就浅尝辄止了，很难获取到核心知识点

> 知识的学习就像谈恋爱，不能一上来就想和对方深入了解，也不能聊得太浅，影响后续发展，得讲究循序渐进。 接下来笔者会根据由浅入深的方式，阶段性来讲解Kotlin协程相关知识点。读者可以根据自己的需求，选择对应的阶段文章。

**笔者主要以下面几个时间线来讲解协程**

**关于协程的系列文章**

> [Android体系课之--Kotlin协程篇-协程入门 -协程基础用法（一）](https://juejin.cn/post/7109853967957377038)
>
> [Android体系课之--Kotlin协程进阶篇-协程中关键知识点梳理（二）](https://juejin.cn/post/7110011090443960327/)
>
> [Android体系课之--Kotlin协程进阶篇-协程的异常处理机制以及suspend关键字详解](https://juejin.cn/post/7111594612761837604) 
>
> Android体系课之--Kotlin协程进阶篇-协程加Retrofit创建一个MVVM模式的网络请求框架（四）

**本系列文使用的版本：**

```
// Kotlin
implementation "org.jetbrains.kotlin:kotlin-stdlib:1.4.32"

// 协程核心库
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:1.4.1"
// 协程Android支持库
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.4.1"
```

**上一篇文章我们对协程的基础用法做了一个总结，主要讲了几点我们来回顾下**

**协程定义**： 1.协程通过将复杂性放入库来简化异步编程。程序的逻辑可以在协程中顺序地表达，而底层库会为我们解决其异步性。该库可以将用户代码的相关部分包装为回调、订阅相关事件、在不同线程（甚至不同机器！）上调度执行，而代码则保持如同顺序执行一样简单。 2.协程是一种并发设计模式，您可以在Android平台上使用它来简化异步执行的代码

**协程生命周期**

```
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

**协程的几种创建方式及返回值解析** *runBlocking*：阻塞创建协程的线程，并运行协程作用域中的job任务 *launch*：不阻塞创建线程，返回一个Job，且协程没有被挂起，任务执行中可以被异步取消 *async*：不阻塞创建线程，且协程没有被挂起，返回一个DeferredCoroutine，此时协程状态为Active，

也还简单介绍了**协程作用域**，**协程调度器**，**挂起函数suspen**d等知识点，***下面我们来详细讲解下协程中的几点关键知识点***

**协程知识点清单：**

-   **1.协程调度器CoroutlineDispatcher**
-   **2.协程上下文CoroutlineContext**
-   **3.协程启动模式CoroutlineStart**
-   **4.协程作用域CoroutlineScope**
-   **5.挂起函数以及suspend关键字的使用**

## 下面我们依清单来讲解

#### 1.协程调度器CoroutlineDispatcher：

调度器其实就是指定我们的协程任务Job在哪个线程上执行：协程给我们内置了几个调度器：

-   *Dispatchers.Default*:默认调度器，CPU密集型任务调度器，适合处理后台计算,和rxjava中的Computed调度器一样
-   *Dispatchers.IO*：适合处理IO操作，如网络读取，文件读取等操作
-   *Dispatchers.Main*:主线程执行，UI线程，适合需要在UI线程上操作的，如更新UI等
-   *Dispatchers.Unconfined*:非受限调度器，又或者称为“无所谓”调度器，不要求协程执行在特定线程上

如果我们需要在实现一个功能：*`如在IO线程读取网络数据，在UI线程更新获取的UI数据，如何操作？`* 协程内部给我们定义了一个挂起函数：*withContext* 可以在协程内部切换线程

```
GlobalScope.launch(Dispatchers.Main) {
    val result = withContext(Dispatchers.IO) {
        //网络请求...
        "请求结果"
    }
    btn.text = result
}
```

#### 2.协程上下文CoroutlineContext

**它是一个包含了用户定义的一些各种不同元素的*Element*对象集合。** 其中主要元素是 **Job、** **协程调度器CoroutineDispatcher、** **还有包含协程异常CoroutineExceptionHandler**、 **拦截器ContinuationInterceptor、** **协程名CoroutineName等。** 这些数据都是和协程密切相关的，`每一个Element都一个唯一key`。

我们来看下他的**源码**：

```
public interface CoroutineContext {
 public operator fun <E : CoroutineContext.Element> get(key: Key<E>): E? //`1
public fun <R> fold(initial: R, operation: (R, CoroutineContext.Element) -> R): R //4

public operator fun plus(context: CoroutineContext): CoroutineContext =
    if (context === EmptyCoroutineContext) this else context.fold(this) { ...}//2

public fun minusKey(key: Key<*>): CoroutineContext//3

//注意这里，这个key很关键
public interface Key <E : CoroutineContext.Element>

 public interface Element : CoroutineContext {
    public val key: Key<*>

    public override operator fun <E : Element> get(key: Key<E>): E? =
        if (this.key == key) this as E else null

    public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
        operation(initial, this)

    public override fun minusKey(key: Key<*>): CoroutineContext =
        if (this.key == key) EmptyCoroutineContext else this
     }
}
```

-   1处可以看出CoroutineContext是一个键值对形式的结构，key为他的内部类Key，而value为他的内部类Element的实例
-   2处重写了CoroutineContext的+法操作符，这个操作符的重写类似于*C++*中对数学操作符的重写，*外部可以直接使用+法对CoroutineContext进行操作*，如相加
-   3处减法同2中加法原理
-   4处fold其实是一个对context进行轮询的操作

下面来看下上下文是如何创建的以及什么时候用到： 我们使用launch操作来看：

```
 public fun CoroutineScope.launch(
      context: CoroutineContext = EmptyCoroutineContext,
      start: CoroutineStart = CoroutineStart.DEFAULT,
      block: suspend CoroutineScope.() -> Unit
  ): Job {
      val newContext = newCoroutineContext(context)//1
      val coroutine = if (start.isLazy)
          LazyStandaloneCoroutine(newContext, block) else
          StandaloneCoroutine(newContext, active = true)
      coroutine.start(start, coroutine, block)
      return coroutine
  }
  //进入1处：
  public actual fun CoroutineScope.newCoroutineContext(context: CoroutineContext): CoroutineContext {
      val combined = coroutineContext + context //2
      val debug = if (DEBUG) combined + CoroutineId(COROUTINE_ID.incrementAndGet()) else combined 
      return if (combined !== Dispatchers.Default && combined[ContinuationInterceptor] == null)
          debug + Dispatchers.Default else debug
  }
  //在2处coroutineContext=EmptyCoroutineContext的实例，调用了+法操作符
  public operator fun plus(context: CoroutineContext): CoroutineContext =
          if (context === EmptyCoroutineContext) this else // fast path -- avoid lambda creation
              context.fold(this) { acc, element ->
                  val removed = acc.minusKey(element.key)
                  if (removed === EmptyCoroutineContext) element else {
                      // make sure interceptor is always last in the context (and thus is fast to get when present)
                      val interceptor = removed[ContinuationInterceptor]
                      if (interceptor == null) CombinedContext(removed, element) else {
                          val left = removed.minusKey(ContinuationInterceptor)
                          if (left === EmptyCoroutineContext) CombinedContext(element, interceptor) else
                              CombinedContext(CombinedContext(left, element), interceptor)
                      }
                  }
              }
```

根据加法操作符的实现：最后返回的是一个CombinedContext的实例对象，内部包含了n个CoroutineContext.Element的子集

上面提到的 **CoroutineId(协程ID)，CoroutineName(协程名称)，CoroutineDispatcher(协程调度器)， ContinuationInterceptor(协程拦截器)，CoroutineExceptionHandler(协程异常处理) 等都继承自CoroutineContext.Element** 也就是说这些操作都可以使用+，-法进行操作，放入到一个协程的上下文中，组合成一个协程的上下文内容 **加法**：如果有这个key则，将覆盖原来key中对应的值，如果没有，则追加进入 **减法**：如果对应的值的key存在于这个context中，则删除这个key对应的值

例子：

```
fun testCoroutineContext(){
    val coroutineContext1 = Job() + CoroutineName("这是第一个上下文")
    println("coroutineContext1$coroutineContext1")
    val  coroutineContext2 = coroutineContext1 + Dispatchers.Default + CoroutineName("这是第二个上下文")
    println("coroutineContext2$coroutineContext2")
    val coroutineContext3 = coroutineContext2 + Dispatchers.Main + CoroutineName("这是第三个上下文")
    println("coroutineContext3$coroutineContext3")
}
结果：
D/coroutineContext1: [JobImpl{Active}@21a6a21, CoroutineName(这是第一个上下文)]
D/coroutineContext2: [JobImpl{Active}@21a6a21, CoroutineName(这是第二个上下文), Dispatchers.Default]
D/coroutineContext3: [JobImpl{Active}@21a6a21, CoroutineName(这是第三个上下文), Dispatchers.Main]
```

可以看到第二个context覆盖了第一个context的CoroutineName，并增加了Dispatchers.Default 第三个覆盖了第二个context的CoroutineName和Dispatchers.Main调度器

**协程上下文讲解的比较详细点，是因为他贯穿了整个协程体系**

#### 3.协程启动模式CoroutlineStart

协程启动模式，顾名思义就是*何时执行协程任务* CoroutineStart启动模式是启动协程的时候，传入的第二个参数，这个参数有4中模式

-   *DEFAULT*：默认启动模式，这个模式下协程启动之后并不是立即执行，有可能在异步操作中被提前终止
-   *LAZY*：懒汉模式：启动协程的时候不会立即执行协程，等待我们主动调用Job的start，join或者await的时候才会执行协程代码
-   *ATOMIC*：和DEFAULT一样也是启动协程后立即执行任务，区别是在协程执行到第一个挂起点之前是不会响应Cancel操作，一定要在挂起后才有意义
-   *UNDISPATCHED* ：和ATOMIC一样，区别是UNDISPATCHED在执行到第一个挂起点之前，和启动协程的线程是同一个，只有在挂起后，协程的运行线程才是协程指定的线程

我们用一个例子来看：

```
fun testCoroutineStart(){
     val defaultJob = GlobalScope.launch{
         println("CoroutineStart.DEFAULT")
     }
     defaultJob.cancel()
     val lazyJob = GlobalScope.launch(start = CoroutineStart.LAZY) {
         println("CoroutineStart.LAZY")
     }
     val atomicJob = GlobalScope.launch(start = CoroutineStart.ATOMIC) {
        println("CoroutineStart.ATOMIC-挂起前")
        delay(300)
        println("CoroutineStart.ATOMIC-挂起后")
     }
    atomicJob.cancel()
    val unDispatched = GlobalScope.launch(start = CoroutineStart.UNDISPATCHED) {
        println("CoroutineStart.UNDISPATCHED-挂起前")
        delay(300)
        println("CoroutineStart.UNDISPATCHED-挂起后")
    }
    unDispatched.cancel()

    println("${Thread.sleep(2000)}:休眠2s后，启动lazyJob")
    lazyJob.start()
}
//结果：
CoroutineStart.DEFAULT
CoroutineStart.ATOMIC-挂起前
CoroutineStart.UNDISPATCHED-挂起前
kotlin.Unit:休眠2s后，启动lazyJob
CoroutineStart.LAZY
//也有可能
CoroutineStart.ATOMIC-挂起前
CoroutineStart.UNDISPATCHED-挂起前
kotlin.Unit:休眠2s后，启动lazyJob
CoroutineStart.LAZY
```

**分析这个例子**：

> 我们在启动default模式协程后，调用了cancel，这个时候如果协程执行没有获取到cpu资源，则会直接取消执行 在启动lazy模式的时候，延迟了2s再启动，可以发现，在延迟启动前是不会执行任务的，在调用了start把协程挂起的时候开始执行协程 在启动atomic的时候，我们在协程内部调用了delay将函数挂起，按模式要求，在挂起前是不会受cancel影响，只有在挂起后，job的cancel才有意义，所以不会打印出挂起后的状态

在启动unDispatched模式的时候，和atomic模式一样，在内部调用delay将函数挂起，发现和atomic结果一样 **那unDispatched特殊点在哪里呢**。 我们来看另外一个例子：

```
fun testUnDispatched(){
    val x = GlobalScope.launch(start = CoroutineStart.UNDISPATCHED){
        println("挂起前-执行协程的线程id:${Thread.currentThread().id}")
        delay(200)
        println("挂起后-执行协程的线程id:${Thread.currentThread().id}")

    }
    println("启动协程的线程id:${Thread.currentThread().id}")
    Thread.sleep(500)
}
```

**打印结果：** `挂起前-执行协程的线程id:1 启动协程的线程id:1 挂起后-执行协程的线程id:12`

可以看到：在*不指定协程运行的线程*的情况下，在调用挂起函数之前协程默认使用的是主线程和启动协程的线程一致 我们改下代码: *增加调度器：Dispatchers.Main*

```
val x = GlobalScope.launch(context = Dispatchers.Main,start = CoroutineStart.UNDISPATCHED){
    println("挂起前-执行协程的线程id:${Thread.currentThread().id}")
    delay(200)
    println("挂起后-执行协程的线程id:${Thread.currentThread().id}")

}
println("启动协程的线程id:${Thread.currentThread().id}")
Thread.sleep(500)
打印结果：
挂起前-执行协程的线程id:1
启动协程的线程id:1
挂起后-执行协程的线程id:1
```

可以看到指定线程为主线程后走的是和挂起前的线程一致都是在主线程上 根据上面例子总结CoroutineStart.UNDISPATCHED模式：

-   1.不管指定了调度器与否，挂起前的执行都在当前线程下
-   2.对于指定了调度器的情况，挂起后的执行都在指定的调度器的线程下
-   3.如果指定的调度器线程和当前线程一直，则挂起后也会在当前线程下执行

#### 4.协程作用域CoroutlineScope

协程作用域分为三种： **1.顶级作用域** 没有父协程的协程所在的作用域 **2.协同作用域** 子协程所在的作用域，当子协程出现异常，会将异常事件场地给父协程，如果父协程也被取消，则所以的子协程也会被取消 **3.主从作用域** 也是子协程所在的作用域，区别是子协程作用域出现异常不会，不会传递给父协程，所以也不会引起其他子协程被取消

PS：对于父协程作用域需要等待所以的子协程作用域完成的时候才会变为Completed状态

子协程会继承父协程的上下文，如果子协程的上下文在父协程中存在，则会覆盖父协程的上下文 来看个例子：

```
val x1 = GlobalScope.launch (context = Dispatchers.IO,start = CoroutineStart.ATOMIC){
 println("scope1:$coroutineContext")
 val x2 = launch(Dispatchers.Default+CoroutineName("第二个协程")) {
 println("scope2:$coroutineContext")
 }
 val x3 = launch(CoroutineName("第三个协程")) {
 println("scope3:$coroutineContext")
 }
}
结果：
scope1:[CoroutineId(1), "coroutine#1":StandaloneCoroutine{Active}@7525500, Dispatchers.IO]
scope2:[CoroutineName(第二个协程), CoroutineId(2), "第二个协程#2":StandaloneCoroutine{Active}@18668527, Dispatchers.Default]
scope3:[CoroutineName(第三个协程), CoroutineId(3), "第三个协程#3":StandaloneCoroutine{Active}@5cfdb4c8, Dispatchers.IO]
```

上面可以看出：

-   scope3继承额scope1的IO调度器，scope2因为设置了默认调度器，所以会覆盖scope1中的。
-   scope2和3设置了CoroutineName，会在1的基础上新增name的context

**前面提到协同作用域取消的时候，会传递给父协程，如果父协程取消，所有子协程也会被取消** **再来看个例子:**

```
val e1 = CoroutineExceptionHandler { coroutineContext, throwable ->
    println("exception:+$coroutineContext[CoroutineName] ${throwable.message}")
}
val x1 = GlobalScope.launch (Dispatchers.IO+e1,start = CoroutineStart.ATOMIC){
    println("scope1:$coroutineContext")
    val x2 = launch(Dispatchers.Default+CoroutineName("第二个协程")) {
        println("scope2:$coroutineContext")
        throw NullPointerException("testexception")
    }
    val x3 = launch(CoroutineName("第三个协程")) {
        println("scope3:$coroutineContext")
    }
}
打印结果：
scope1:[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@210f1e7a, CoroutineId(1), "coroutine#1":StandaloneCoroutine{Active}@4c0cd601, Dispatchers.IO]
scope2:[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@210f1e7a, CoroutineName(第二个协程), CoroutineId(2), "第二个协程#2":StandaloneCoroutine{Active}@1b89f68, Dispatchers.Default]
exception:+[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@210f1e7a, CoroutineId(1), "coroutine#1":StandaloneCoroutine{Cancelling}@4c0cd601, Dispatchers.IO][CoroutineName] testexception
```

-   可以看到父协程接收到了子协程的异常，且并没有打印出scope3

-   结论：**子协程将异常传递给父协程，并终止其他子协程**

-   对于主从作用域，子协程不会将异常传递给父协程：使用supervisorScope或者使用supervisorJob创建协程上下文 **我们来看个例子**

    ```
    val x1 = GlobalScope.launch (Dispatchers.IO+e1,start = CoroutineStart.ATOMIC){
        supervisorScope {
            println("scope1:$coroutineContext")
            val x2 = launch(Dispatchers.Default+CoroutineName("第二个协程")) {
                println("scope2:$coroutineContext")
                throw NullPointerException("testexception")
            }
            val x3 = launch(CoroutineName("第三个协程")) {
                println("scope3:$coroutineContext")
            }
        }
        val x3 = launch(CoroutineName("第四个协程")) {
            println("scope4:$coroutineContext")
        }
    }
    打印结果：
    scope1:[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@478f98c0, CoroutineId(1), "coroutine#1":SupervisorCoroutine{Active}@3b54e532, Dispatchers.IO]
    scope2:[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@478f98c0, CoroutineName(第二个协程), CoroutineId(2), "第二个协程#2":StandaloneCoroutine{Active}@1b89f68, Dispatchers.Default]
    exception:+[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@478f98c0, CoroutineName(第二个协程), CoroutineId(2), "第二个协程#2":StandaloneCoroutine{Cancelling}@1b89f68, Dispatchers.Default][CoroutineName] testexception
    scope3:[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@478f98c0, CoroutineName(第三个协程), CoroutineId(3), "第三个协程#3":StandaloneCoroutine{Active}@276104b, Dispatchers.IO]
    scope4:[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@478f98c0, CoroutineName(第四个协程), CoroutineId(4), "第四个协程#4":StandaloneCoroutine{Active}@4d7eff18, Dispatchers.IO]
    ```

**可以看到scope2异常的时候，但是并没有终止其他子协程，其他子协程还是正常执行了，且打印出了父协程的异常，那是不是就是说异常事件传递给了父协程了** 我们再来看个例子：

```

val e1 = CoroutineExceptionHandler { coroutineContext, throwable ->
 println("exception1:+$coroutineContext[CoroutineName] ${throwable.message}")
}
val e2 = CoroutineExceptionHandler { coroutineContext, throwable ->
 println("exception2:+$coroutineContext[CoroutineName] ${throwable.message}")
}
val x1 = GlobalScope.launch (Dispatchers.IO+e1,start = CoroutineStart.ATOMIC){
 supervisorScope {
 println("scope1:$coroutineContext")
 val x2 = launch(e2+Dispatchers.Default+CoroutineName("第二个协程")) {
 println("scope2:$coroutineContext")
 throw NullPointerException("testexception")
 }
 val x3 = launch(CoroutineName("第三个协程")) {
 println("scope3:$coroutineContext")
 }
 }
 val x3 = launch(CoroutineName("第四个协程")) {
 println("scope4:$coroutineContext")
 }

}
打印结果：
scope1:[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@54940182, CoroutineId(1), "coroutine#1":SupervisorCoroutine{Active}@17d164e8, Dispatchers.IO]
scope2:[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$2@29942327, CoroutineName(第二个协程), CoroutineId(2), "第二个协程#2":StandaloneCoroutine{Active}@2753c865, Dispatchers.Default]
exception2:+[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$2@29942327, CoroutineName(第二个协程), CoroutineId(2), "第二个协程#2":StandaloneCoroutine{Cancelling}@2753c865, Dispatchers.Default][CoroutineName] testexception
scope3:[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@54940182, CoroutineName(第三个协程), CoroutineId(3), "第三个协程#3":StandaloneCoroutine{Active}@74d71d98, Dispatchers.IO]
scope4:[com.allinpay.testkotlin.CoroutlineScope$Companion$testCoroutineStart$$inlined$CoroutineExceptionHandler$1@54940182, CoroutineName(第四个协程), CoroutineId(4), "第四个协程#4":StandaloneCoroutine{Active}@fa99c8a, Dispatchers.IO]
```

**设置了scope2的异常之后，并没有打印父协程中的e1异常，可以看出子协程并没有将异常事件传递给子协程，那为什么前面例子中会出现打印出父协程的异常呢？** 相信仔细看了前面内容的知道答案：**子协程scope2继承了父协程的e1异常，所以打印出了父协程的异常**

#### 5.挂起函数以及suspend关键字的使用

我们先给个例子：

```
val x5 = GlobalScope.launch(Dispatchers.Unconfined) {
    println("scope1-挂起前:${Thread.currentThread().id} $coroutineContext")
    val s = test1()
    println("s:$s")
    withContext(Dispatchers.IO){
        println("scope2:${Thread.currentThread().id} $coroutineContext")
        delay(1000)
    }
    println("scope1-挂起后:${Thread.currentThread().id}")
}
println("after")
打印结果：
scope1-挂起前:1 [CoroutineId(1), "coroutine#1":StandaloneCoroutine{Active}@3ab39c39, Dispatchers.Unconfined]
run test1:1
after
s:test1 return
scope2:12 [CoroutineId(1), "coroutine#1":DispatchedCoroutine{Active}@699bd409, Dispatchers.IO]
scope1-挂起后:14
```

**在一个协程内部调用了withContext的挂起函数，看协程挂起前后的结果可知，挂起前和挂起后的代码是走在两个线程里面， 就是说，调用挂起函数后，并没有阻塞挂起前协程的代码，而是继续运行着，而调用挂起函数，恢复协程后，启动了一个新的线程去执行了代码。 由上面分析可知，调用挂起函数后，只是暂停了协程的执行，并没有阻塞原有线程，待挂起函数运行结束后，启动一个新的线程或者原线程空闲状态使用原线程进行处理。 用哪个线程取决于使用的是哪个调度器和cpu的处理**

*对挂起函数做个总结*：

> 挂起函数是在不阻塞原有线程的情况下，将协程挂起，并在挂起函数结束回调resumeWith回复协程的时候， 使用原线程或者启用一个新的协程去处理，具体使用哪个取决于传入的调度器和cpu， 挂起函数一定只能在挂起函数或者协程内部使用，如果在协程外部使用且没有在挂起函数内部使用，则编译器会报错，因为挂起函数就是为了把协程挂起而做的， 使用bytecode编译可以发现挂起函数编译后会有个协程的参数，且这个参数不能为null

## 总结：

本篇文章对协程的几个关键知识点做了总结，看完后相信你对协程会有更深刻的理解 
*下一篇文章笔者会讲解一些关于协程中异常处理的机制* 
**关于协程的系列文章** 
> [Android体系课之--Kotlin协程篇-协程入门 -协程基础用法（一）](https://juejin.cn/post/7109853967957377038)
>
> [Android体系课之--Kotlin协程进阶篇-协程中关键知识点梳理（二）](https://juejin.cn/post/7110011090443960327/)
>
> [Android体系课之--Kotlin协程进阶篇-协程的异常处理机制以及suspend关键字详解](https://juejin.cn/post/7111594612761837604) 
>
> Android体系课之--Kotlin协程进阶篇-协程加Retrofit创建一个MVVM模式的网络请求框架（四）