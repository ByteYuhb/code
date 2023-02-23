---
highlight: agate
---
## 协程的基础使用：
### 协程的定义：
> 1.协程通过将复杂性放入库来简化异步编程。程序的逻辑可以在协程中顺序地表达，而底层库会为我们解决其异步性。该库可以将用户代码的相关部分包装为回调、订阅相关事件、在不同线程（甚至不同机器！）上调度执行，而代码则保持如同顺序执行一样简单。
>
> 2.协程是一种并发设计模式，您可以在Android平台上使用它来简化异步执行的代码

### 协程生命周期：

```java
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


### 协程的三种创建方式及返回值解析
**runBlocking**：阻塞创建协程的线程，并运行协程作用域中的job任务

**launch**：不阻塞创建线程，返回一个Job，且协程没有被挂起，任务执行中可以被异步取消

**async**：不阻塞创建线程，且协程没有被挂起，返回一个DeferredCoroutine，此时协程状态为Active，

## 协程的核心知识点：
1.**协程调度器CoroutlineDispatcher**：指定协程在哪个线程上运行

2.**协程上下文CoroutlineContext**：包含一个线程内部所需的所有上下文信息

3.**协程启动模式CoroutlineStart**：指定了协程的启动方式：如default或者lazy等

4.**协程作用域CoroutlineScope**：指定了协程的作用域

5.**挂起函数以及suspend关键字的使用**：挂起函数会将协程挂起，但不阻塞原有执行线程

***接下来***
*这篇文章我们来讲解下协程中的异常处理机制
笔者会由浅入深，深度解析协程的异常处理

**关于协程的其他系列文章，请根据需求自行阅读***

[Android体系课之--Kotlin协程篇-协程入门 -协程基础用法（一）](https://juejin.cn/post/7109853967957377038)

[Android体系课之--Kotlin协程进阶篇-协程中关键知识点梳理（二）](https://juejin.cn/post/7110011090443960327)

[Android体系课之--Kotlin协程进阶篇-协程的异常处理机制（三）](https://juejin.cn/post/7111594612761837604/)

Android体系课之--Kotlin协程进阶篇-协程加Retrofit创建一个MVVM模式的网络请求框架（四）

## 简介：
> 只要是代码就一定会出现异常，如果不能正确的处理好异常这块任务，可能会导致很多稀奇古怪的问题。
> 异常处理是协程的一个关键机制，因为其涉及的内容比较深，笔者单独启动一篇来介绍它

讲解异常处理之前我们先来看下**suspend**关键字的意义：

suspend：用于修饰一个需要挂起的函数

**挂起谁**：协程代码块

**挂起定义：**
在不阻塞当前协程代码块的运行线程的基础上，去执行耗时异步代码，原线程该干嘛干嘛去..

**恢复定义：**
原来被挂起的协程代码块，在异步任务执行完毕后，恢复执行

**协程中的代码块是以什么形式存在？**
> 每个suspend方法编译器都会将它编译为一个**Continuation对象**，每个Continuation对象都提供了协程恢复方法**resumeWith**和**invokeSuspend**协程代码块运行方法
> Continuation接口：

```java
interface Continuation<in T> {
  public val context: CoroutineContext
  public fun resumeWith(value: Result<T>)
}
```
- ***context***：当前协程的上下文
- ***resumeWith***:用于恢复当前协程，value是子协程提供的返回值，可能是异常
下面我们来举个例子查看的挂起函数的调用流程：当然这个suspend函数需要在一个协程代码块中执行或者在另外一个suspend函数中执行

```java
suspend fun test2(){
	println("ddd:${Thread.currentThread().id}")//1
	val x = withContext(Dispatchers.IO){
		println("ccc:${Thread.currentThread().id}")
		"1234"
	}
	println("eee:${Thread.currentThread().id} res:$x")//2
}
```



```java
执行结果：
ddd:11
ccc:12
eee:11 res:1234

运行多次也可能出现：
ddd:11
ccc:12
eee:13 res:1234
```

下面我们来分析下这块代码
**我们将其分析3块：**



```java
代码块1：
println("ddd:${Thread.currentThread().id}")//1
```

```java
代码块2：
println("eee:${Thread.currentThread().id} res:$x")//2
```


```java
代码块3：
println("ccc:${Thread.currentThread().id}")
"1234"
```


- **withContext是一个挂起函数**

按照之前的理解我们来猜测下运行过程：

**使用一个线程我们id为11：运行完代码块1后，运行挂起函数withContext，withContext方法会有个返回值：如果是CoroutineSingletons.COROUTINE_SUSPENDED，则表示当前协程被挂起，挂起后，原执行代码块1的线程11被释放，这个时候线程11该干嘛干嘛去
待withContext内部代码块3 被执行完毕后，withContext内部会调用一个恢复函数resumeWith(value)恢复原线程11或其他线程继续去执行代码块2的任务，
并将withContext结果1234以参数形式返回。**

> 我们使用Tools->Kotlin->show Kotlin Bytecode->Decompile反编译得到test2的java代码来证实下我们的处理过程



```java
public final Object test2(@NotNull Continuation $completion) {
  ...
  Object var7 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
  ...
  switch(((<undefinedtype>)$continuation).label) {
  case 0:
	//代码块1：默认第一次执行为lable=0
	 var10 = (new StringBuilder()).append("ddd:");
	 var10001 = Thread.currentThread();
	 Intrinsics.checkExpressionValueIsNotNull(var10001, "Thread.currentThread()");
	 x = var10.append(var10001.getId()).toString();
	 boolean var3 = false;
	 System.out.println(x);
	 
	 //这里将withContext中的代码块封装到一个Function2对象中
	 CoroutineContext var11 = (CoroutineContext)Dispatchers.getIO();
	 Function2 var12 = (Function2)(new Function2((Continuation)null) {
		private CoroutineScope p$;
		int label;
		
		//协程块执行入口
		@Nullable
		public final Object invokeSuspend(@NotNull Object $result) {
		   Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
		   switch(this.label) {
		   case 0:
			  //代码块3
			  ResultKt.throwOnFailure($result);
			  CoroutineScope $this$withContext = this.p$;
			  StringBuilder var10000 = (new StringBuilder()).append("ccc:");
			  Thread var10001 = Thread.currentThread();
			  Intrinsics.checkExpressionValueIsNotNull(var10001, "Thread.currentThread()");
			  String var3 = var10000.append(var10001.getId()).toString();
			  boolean var4 = false;
			  System.out.println(var3);
			  return "1234";//返回1234
		   default:
			  throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
		   }
		}
		//这里创建了当前协程的Continuation：用于恢复当前协程的执行
		@NotNull
		public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
		   Intrinsics.checkParameterIsNotNull(completion, "completion");
		   Function2 var3 = new <anonymous constructor>(completion);
		   var3.p$ = (CoroutineScope)value;
		   return var3;
		}
		...
	 });
	 ((<undefinedtype>)$continuation).label = 1;//将lable置为1
	 var10000 = BuildersKt.withContext(var11, var12, (Continuation)$continuation);//调用withContext方法，传入了前面创建的var12对象，内部会使用对应的调度器执行代码块3中的任务
	 if (var10000 == var7) {
		return var7;//函数被挂起，直接返回挂起状态
	 }
	 break;
  case 1://恢复协程先执行这里，获取响应如果是异常会抛出异常，并将value放在var10000属性中
	 CoroutlineScope var8 = (CoroutlineScope)((<undefinedtype>)$continuation).L$0;
	 ResultKt.throwOnFailure($result);
	 var10000 = $result;
	 break;
  default:
	 throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
  }
 
 //这里执行代码块2的任务
  x = (String)var10000;//var10000为lable=1中的result
  var10 = (new StringBuilder()).append("eee:");
  var10001 = Thread.currentThread();
  Intrinsics.checkExpressionValueIsNotNull(var10001, "Thread.currentThread()");
  String var9 = var10.append(var10001.getId()).append(" res:").append(x).toString();
  boolean var4 = false;
  System.out.println(var9);
  return Unit.INSTANCE;
}

```

> **通过对上面反编译代码的分析。我们基本上对挂起函数的任务调度有了一个大概的了解，那么可能大家会有疑问了，执行入口invokeSuspend和恢复出口resumeWith是在哪里执行的呢??**

我们到源码里面去看看：
注意：方法内部块以{}标注,省略了参数

```java
withContext{
1：
	block.startCoroutineCancellable(coroutine, coroutine){
			//这里的createCoroutineUnintercepted会调用Function2中的create方法;创建一个当前挂起函数的Continuation接口，用于恢复当前协程
			createCoroutineUnintercepted(receiver, completion).intercepted().resumeCancellableWith(Result.success(Unit), onCancellation){
				//继续看resumeCancellableWith
				is DispatchedContinuation -> resumeCancellableWith(result, onCancellation){
					//这个dispatcher如果没有设置调度器则为默认调度器：DefaultDispatcher
					dispatcher.dispatch(context, this){
						coroutineScheduler.dispatch(block){
							val task = createTask(block, taskContext)//这里创建一个task将block传入
							worker.start(){//调用线程池执行异步任务
								override fun run() = runWorker()
								private fun runWorker() {
									val task = findTask(mayHaveLocalTasks)
									executeTask(task){
										runSafely(task){
											task.run(){task=DispatchedContinuation：执行父类DispatchedTask中的run方法
												if (exception != null) {//异常处理
													continuation.resumeWithException(exception)
												} else {//非异常处理，我们看这里
													continuation.resume(getSuccessfulResult(state)){
														resumeWith(Result.success(value)){
															 while (true) {这里有个死循环
																val outcome: Result<Any?> =
																try {
																	val outcome = invokeSuspend(param)//看到这里执行了invokeSuspend，这就是前面最早分析的是对协程内部代码块的封装接口，
																	if (outcome === COROUTINE_SUSPENDED) return//如果是挂起，关注点1
																	Result.success(outcome)//结果不是挂起，则返回一个Result.success包含了结果：关注点2
																} catch (exception: Throwable) {
																	Result.failure(exception)//结果异常，则返回一个Result.failure
																}
																completion.resumeWith(outcome)//看到这里调用了completion的resumeWith恢复当前被挂起的协程：恢复操作其实最后也是调用一个invokeSuspend取执行代码块只是这个时候lable=1了
																{
																	val state = makeCompletingOnce(result.toState())这个result=1234
																	afterResume{
																		uCont.intercepted().resumeCancellableWith(recoverResult(state, uCont)){//看到这里又调用resumeCancellableWith，这个前面分析过最终也是走到invokeSuspend执行剩余代码
																			...
																			...
																		}
																	}
														
																}
															 }
														}
													}
												}
											}
										}
									}
								} 				
							}
						}
					}
				
				}
			
			}	
	}
}
```


> 通过对withContext源代码的分析可知，协程其实就是将代码块以挂起函数为分界线分为多个任务，挂起函数可能会被挂起可能不会。

> 其实对withContext源码的分析也是对整个协程 体系的分析过程。
> 每个挂起函数都是一个状态机，状态机包括多个lable标签，每个标签代表一个代码块逻辑

**了解了挂起函数，下面来讲解协程的异常处理机制就比较容易了**
## 协程的异常处理机制：
***协程是如何处理异常的呢：***

我们用个例子来看：

```java
GlobalScope.launch(Dispatchers.IO) {
	val scope = CoroutlineScope()
	println("协程1执行前")
	GlobalScope.launch(Dispatchers.IO){
		println("协程2执行前")
		throw NullPointerException("test exception!!")
		println("协程2执后")
	}
	println("协程1执行后")
}
运行结果：
协程1执行前
协程1执行后
协程2执行前
Exception in thread "DefaultDispatcher-worker-3 @coroutine#2" java.lang.NullPointerException: test exception!!
	at com.allinpay.testkotlin.ExampleUnitTest$coroutline$1$1.invokeSuspend(ExampleUnitTest.kt:27)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:106)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:571)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:738)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:678)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:665)
```

> 抛出了自定义的 test exception异常

根据异常的调用栈，可以看出任务调用过程：
> 雇主找了个包工头CoroutineScheduler，包工头找了个工人Worker，雇主让Worker去干DispatchedTask的活，Worker就开始以runSafely的方式干活，并最终在执行invokeSuspend时出现了异常，然后扔给雇主。。。
> 从源码来看看：之前我们也分析过挂起函数的任务调度，其实协程的任务调度也是一样的：
> 我们直接看resumeWith方法：

```java
public final override fun resumeWith(result: Result<Any?>) {
	
	while (true) {
		
		with(current) {
			val completion = completion!! // fail fast when trying to resume continuation without completion
			val outcome: Result<Any?> =
				try {
					val outcome = invokeSuspend(param)//关注点1
					if (outcome === COROUTINE_SUSPENDED) return
					Result.success(outcome)
				} catch (exception: Throwable) {
					Result.failure(exception)//关注点2
				}
			releaseIntercepted() // this state machine instance is terminating
			if (completion is BaseContinuationImpl) {
				// unrolling recursion via loop
				current = completion
				param = outcome
			} else {
				// top-level completion reached -- invoke and return
				completion.resumeWith(outcome)//关注点3
				return
			}
		}
	}
}
```


1. 关注点1中invokeSuspend方法会执行协程代码逻辑，在运行到异常时，
2. 会在关注点2处catch住异常，并给outcome赋值为Result.failure(exception)
3. 在关注点3处运行completion.resumeWith(outcome)恢复原协程的执行，并传入一个outcome = Result.failure(exception)
**我们继续分析completion.resumeWith协程恢复做了什么？**

```java
resumeWith{
	val state = makeCompletingOnce(result.toState()){
		val finalState = tryMakeCompleting(state, proposedUpdate){
			tryMakeCompletingSlowPath(state, proposedUpdate){
				finalizeFinishingState(finishing, proposedUpdate){
					if (finalException != null) {
						val handled = cancelParent(finalException) || handleJobException(finalException){ 
							//cancelParent:如果需要父类处理协程的情况，未设置的父类异常捕获的情况下：返回false
							//handleJobException执行异常逻辑。我们进去看看
							handleJobException{
								handleCoroutineException(context, exception){
									// Invoke an exception handler from the context if present
									try {
										//协程上下文中设置了CoroutineExceptionHandler，则调用协程的CoroutineExceptionHandler
										context[CoroutineExceptionHandler]?.let {
											it.handleException(context, exception)
											return
										}
									} catch (t: Throwable) {
										//CoroutineExceptionHandler异常处理中出现异常，则调用handleCoroutineExceptionImpl处理
										handleCoroutineExceptionImpl(context, handlerException(exception, t))
										return
									}
									//如果没有设置异常处理器，则使用全局异常处理
									// If a handler is not present in the context or an exception was thrown, fallback to the global handler
									handleCoroutineExceptionImpl(context, exception){
										//设置了外置处理器：AndroidExceptionPreHandler
										// use additional extension handlers
										for (handler in handlers) {
											try {
												handler.handleException(context, exception)
											} catch (t: Throwable) {
												// Use thread's handler if custom handler failed to handle exception
												val currentThread = Thread.currentThread()
												currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, handlerException(exception, t))
											}
										}
							
										// use thread's handler 使用当前线程的Handler处理器
										val currentThread = Thread.currentThread()
										//使用当前线程未捕获异常处理器处理异常
										currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, exception)

									}
								}
							}
						}	
					}
				}
			}
		}
	}

}
```

**从上面分析**
可以看出：

异常处理逻辑：
1.如果父协程设置了异常处理器，则会将异常传递给父协程处理器去处理，且会终止其他子协程的运行

2.如果父协程没有设置异常处理器，则先看当前协程有没有设置异常处理器CoroutineExceptionHandler

	2.1:如果设置了则直接使用当前异常处理器处理。不会继续传递到下面逻辑中
	2.2:如果没有设置异常处理器会将异常传递给异常预处理器AndroidExceptionPreHandler，并且使用当前线程的uncaughtExceptionHandler处理异常，如果没有设置uncaughtExceptionHandler则会报异常并崩溃

> 注：AndroidExceptionPreHandler是在kotlinx-coroutines-android.kotlin_module内部设置


![AndroidExceptionPreHandler.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b9c0c62ea3264469bd41bbed4eca461e~tplv-k3u1fbpfcp-watermark.image?)

**我们再来看个例子来证实下我们上面分析的逻辑：**

```java
fun goTest(){
	val e1 = CoroutineExceptionHandler { coroutineContext, throwable ->
		println("父协程接收到异常：${throwable.message}")
	}
	GlobalScope.launch(e1+Dispatchers.IO) {
		println("协程1执行前:$coroutineContext")
		launch{
			println("协程2执行前:$coroutineContext")
			throw NullPointerException("test exception!!")
			println("协程2执后")
		}
		println("协程1执行中途")
		launch{
			println("协程3执行前")
			Thread.sleep(3000)
			println("协程3执行后")
		}
		println("协程1执行最后")
	}
	Thread.sleep(500000)
}
运行结果：
协程1执行前:[com.allinpay.testkotlin.CoroutlineScope$goTest$$inlined$CoroutineExceptionHandler$1@7bd1a618, CoroutineId(1), "coroutine#1":StandaloneCoroutine{Active}@1d7e8bd0, Dispatchers.IO]
协程1执行中途
协程2执行前:[com.allinpay.testkotlin.CoroutlineScope$goTest$$inlined$CoroutineExceptionHandler$1@7bd1a618, CoroutineId(2), "coroutine#2":StandaloneCoroutine{Active}@691ca59a, Dispatchers.IO]
协程1执行最后
父协程接收到异常：test exception!!
```


**可以看到协程2异常后，将异常处理传递给了父协程的异常处理器e1并且终止了协程3的运行，证实了1的说法**

> 那是不是说异常一定会传递给父协程呢？
> 我们再来看个例子：

```java
val e1 = CoroutineExceptionHandler { coroutineContext, throwable ->
		println("父协程接收到异常：${throwable.message}")
	}
	GlobalScope.launch(e1+Dispatchers.IO) {
		println("协程1执行前:$coroutineContext")
		supervisorScope {
			launch{
				println("协程2执行前:$coroutineContext")
				throw NullPointerException("test exception!!")
				println("协程2执后")
			}
			println("协程1执行中途")
			launch{
				println("协程3执行前")
				Thread.sleep(3000)
				println("协程3执行后")
			}
		}
		launch{
			println("协程4执行前")
			Thread.sleep(3000)
			println("协程4执行后")
		}

		println("协程1执行最后")
	}
	Thread.sleep(500000)
}
运行结果：
协程1执行前:[com.allinpay.testkotlin.CoroutlineScope$goTest$$inlined$CoroutineExceptionHandler$1@6f9b3d3f, CoroutineId(1), "coroutine#1":StandaloneCoroutine{Active}@487031f2, Dispatchers.IO]
协程1执行中途
协程2执行前:[com.allinpay.testkotlin.CoroutlineScope$goTest$$inlined$CoroutineExceptionHandler$1@6f9b3d3f, CoroutineId(2), "coroutine#2":StandaloneCoroutine{Active}@4e8e72ec, Dispatchers.IO]
父协程接收到异常：test exception!!
协程3执行前
协程3执行后
协程1执行最后
协程4执行前
协程4执行后
```


1.看到打印出异常信息后，子协程最后还是执行了，说明这次没有将异常传递给父协程，而是子协程使用父协程继承下来的异常处理器处理了
2.其他子协程没有被终止，说明使用supervisorScope修饰的主从作用域内的异常不会传递给父协程。


从上面例子可以看出：

- 1.协程作用域中的异常会传递给父协程的异常处理器处理，并且终止其他子协程

![图2协同作用域异常处理.awebp](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07bec8ce5f854cef9bb0d94edcd907ae~tplv-k3u1fbpfcp-watermark.image?)

- 2.主从作用域中的异常不会传递的父协程处理，自己内部消化，且父协程的其他子协程不会被取消

![图3主从作用域异常处理.awebp](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b9e471d3682405085a102c62a846232~tplv-k3u1fbpfcp-watermark.image?)



## 总结：
**最后我们来总结下协程异常处理机制**：

- 对于协同作用域：
1.如果父协程设置了异常处理器，则会将异常传递给父协程处理器去处理，且会终止其他子协程的运行

2.如果父协程没有设置异常处理器，则先看当前协程有没有设置异常处理器CoroutineExceptionHandler，

> 2.1. 	如果设置了则直接使用当前异常处理器处理。不会继续传递到下面逻辑中
>
> 2.2. 	如果没有设置异常处理器会将异常传递给异常预处理器AndroidExceptionPreHandler，并且使用当前线程的uncaughtExceptionHandler处理异常并退出应用

- 对于主从作用域：
> 异常自己内部消化，且不会影响其他子协程的处理

**关于协程的其他系列文章，请根据需求自行阅读***

[Android体系课之--Kotlin协程篇-协程入门 -协程基础用法（一）](https://juejin.cn/post/7109853967957377038)

[Android体系课之--Kotlin协程进阶篇-协程中关键知识点梳理（二）](https://juejin.cn/post/7110011090443960327)

[Android体系课之--Kotlin协程进阶篇-协程的异常处理机制（三）](https://juejin.cn/post/7111594612761837604/)

Android体系课之--Kotlin协程进阶篇-协程加Retrofit创建一个MVVM模式的网络请求框架（四）