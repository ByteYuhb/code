> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
> Handler系列：

> [Android体系课--Handler—按方法进行源码解析](https://juejin.cn/post/7114552946412027911/)

> [Android体系课--Handler-Handler面试题]()

### Handler面试题
##### 1.面试官：说说Handler基本使用原理：
`Handler`：负责发送和处理

`Message`：消息对象，类似于一个`链表`的节点

`MessageQueue`：`消息队列`，Handler将`Message`发送给`MessageQueue`，里面维护了一个`链表结构`，节点为`message`

`Looper`：用于从`MessageQueue`的链表中取出`messag`e，最后调用Handler的`handleMessage`处理`message`

> 总结：Handler发送消息时调用MessageQueue的enqueueMessage插入一条信息到MessageQueue,Looper不断轮询调用

##### 2.面试官：子线程中能不能直接new一个Handler,为什么主线程可以？
我：可以，子线程通过调用`Looper.looper`和`looper.loop`开启一个循环，在子线程中可以调用`new Handler(looper.myLooper)`,创建一个子线程handler来处理消息，
主线程在进程启动调用`ActivityThread`的时候已经调用了`Looper.looper`和`looper.loop`开启循环

##### 3面试官：Handler导致的内存泄露原因及其解决方案
我：

**原因**：
    
    message在延迟处理期间，进程关闭了Activity,Activity退出之后，因为msg中会持有handler的引用，handler是属于Activity的内部类，内部类会持有外部类的引用。
    		handler持有Activity的引用，导致Activity内存泄露。
**解决方式**:
将`handler`定义为`静态内部类`，在内部持有`Activity`的`弱引用`。并及时`移除`所有消息：

```java
伪代码：
		private static class SafeHandler extends Handler {

			private WeakReference<HandlerActivity> ref;

			public SafeHandler(HandlerActivity activity) {
				this.ref = new WeakReference(activity);
			}

			@Override
			public void handleMessage(final Message msg) {
				HandlerActivity activity = ref.get();
				if (activity != null) {
					activity.handleMessage(msg);
				}
			}
		}
		在Activity中：
		@Override
		protected void onDestroy() {
		  safeHandler.removeCallbacksAndMessages(null);
		  super.onDestroy();
		}
```
##### 4.一个线程可以有几个Handler,几个Looper,几个MessageQueue对象
> N个Handler，1个Looper，1个MessageQueue
> Looper是在线程创建的时候调用Looper.prepare,创建后使用一个ThreadLocalMap来存储，这个类类似于HashMap，使用的key为当前线程currentThread，
> 获取的时候调用的是get(thread)，thread唯一，所以一个线程只有一个looper，因为MessageQueue是在Looper对象内部，Looper只有一个，所以MessageQueue只有一个

##### 5.Message对象创建的方式有哪些 & 区别
1.new Message()

2.Message.obtain()

3.handle.obtainMessage

其中3最后也是调用的2来获取:

1会在内存中创建消息对象，

而2,3会先从`Message`类中的消息池中获取消息，如果消息池为空，则创建一个new `Message`，如果不为空则从消息池中`取出头部数据`，让第二个节点作为新链表（`sPool` ）的头节点

```java
public static Message obtain() {
        synchronized (sPoolSync) {
                if (sPool != null) {
                        Message m = sPool;
                        sPool = m.next;
                        m.next = null;
                        m.flags = 0; // clear in-use flag
                        sPoolSize--;
                        return m;
                }
        }
        return new Message();
}
```
##### 6.Handler 有哪些发送消息的方法

```java
sendMessage(Message msg)
sendMessageDelayed(Message msg, long uptimeMillis)
post(Runnable r)
postDelayed(Runnable r, long uptimeMillis)
sendMessageAtTime(Message msg,long when)
```
##### 7.Handler的post与sendMessage的区别和应用场景
1.源码发送过程

```java
1.sendMessage-sendMessageAtTime-enqueueMessage。
2.post->sendMessage-getPostMessage-sendMessageAtTime-enqueueMessage 
```
> getPostMessage会先生成一个Messgae，并且把runnable赋值给message的callback

2.Looper->dispatchMessage处理时

```java
public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
                handleCallback(msg);
        } else {
                if (mCallback != null) {
                        if (mCallback.handleMessage(msg)) {
                        return;
                }
        }
                handleMessage(msg);
        }
}
```
> `post`：`dispatchMessage`方法中直接执行`post`中的`runnable`方法。`sendMessage`：调用`handler`的`handleMessage`方法

3.使用场景
> post一般用于单个场景 比如单一的倒计时弹框功能 

> sendMessage的回调需要去实现handleMessage Message则做为参数 用于多判断条件的场景。
>
> `post`方法和`handleMessage`方法的不同在于，区别就是调用`pos`t方法的消息是在post传递的`Runnable`对象的`run`方
> 法中处理，而调用`sendMessage`方法需要重写`handleMessage`方法或者给`handler`设置`callback`，在`callback`的
> `handleMessage`中处理并返回`true` 

##### 8.handler postDealy后消息队列有什么变化，假设先 postDelay 10s, 再postDelay 1s, 怎么处理这2条消息?

> handler post `10s` `消息A`传入后，会和当前时间做个`加法`，再去和`链表头结点延迟时间`做对比，如果现在时间更`超前`，则将消息插入到`链表头部`

> 下次调用handle post `1s``消息B`后，和链表头的`消息A`做对比，发现`B`更超前，则将B放到`链表头节点`，Looper`依次取出B和A处理`
>
> 如果`第二次` post的`消息B`是10s，则对比后，发现`A`更超前，则让`B和A的前一个节点的延迟时间做对比`，依次类推，`插入到比A更延迟的消息节点前面`

##### 9.MessageQueue是什么数据结构？
是一个`单链表`结构，根据延迟信息插入，插入方式为`头插法`，取消息是按头到尾的顺序

##### 10.Handler怎么做到的一个线程对应一个Looper，如何保证只有一个MessageQueue ThreadLocal在Handler机制中的作用

```java
设计的初衷是为了解决多线程编程中的资源共享问题，
	synchronized采取的是“以时间换空间”的策略，本质上是对关键资源上锁，让大家排队操作。
	而ThreadLocal采取的是“以空间换时间”的思路， 它一个线程内部的数据存储类，通过它可以在制定的线程中存储数
	据，数据存储以后，只有在指定线程中可以获取到存储的数据， 对于其他线程就获取不到数据，可以保证本线程任
	何时间操纵的都是同一个对象。比如对于Handler，它要获取当前线程的Looper,很显然Looper的作用域就是线程，
	并且不同线程具有不同的Looper。 ThreadLocal本质是操作线程中ThreadLocalMap来实现本地线程变量的存储的
	ThreadLocalMap是采用数组的方式来存储数据，其中key(弱引用)指向当前ThreadLocal对象，value为设的值 通过
	ThreadLocal计算出Hash key，通过这个哈 ThreadLocal对象，value为设的值
```
##### 11.HandlerThread是什么 & 好处 &原理 & 使用场景
`HandlerThread`本质上是一个线程类，它继承了`Thread`； 

`HandlerThread`有自己的内部`Looper`对象，通过`Looper.loop()`进行looper循环；

通过获取`HandlerThread`的`looper`对象传递给Handler对象，然后在`handleMessage`()方法中执行异步任务；

**优势**:
	
    1.将loop运行在子线程中处理,减轻了主线程的压力,使主线程更流畅,有自己的消息队列,不会干扰UI线程
    2.串行执行,开启一个线程起到多个线程的作用
    劣势:
    1.由于每一个任务队列逐步执行,一旦队列耗时过长,消息延时
    2.对于IO等操作,线程等待,不能并发
    我们可以使用HandlerThread处理本地IO读写操作（数据库，文件），因为本地IO操作大多数的耗时属于毫秒级别，对于单线程 + 异步队列的形式 不会产生较大的阻塞

##### 12.IdleHandler及其使用场景
**介绍:**

> `Handler`机制提供的一种，在`Looper`事件处理循环的过程中，当出现空闲的时候，允许我们执行一些`ide`的任务。
> 		通过调用`MessageQueue`的`addIdeHandle`r和`removeIdeHandler`的方法，将事件放到一个`list`中，
> 		在事件循环过程中，如果出现：
> 		1.`MessageQueue 为空`，没有 Message； 
> 		2.`MessageQueue` 中最近待处理的`Message`，是一个`延迟消息`（when>currentTime），需要`滞后执行`
> 		则会`循环遍历list`中的任务执行，执行成功后将消息空闲`状态置为true`

**使用场景：**
- 1.Activity启动优化：`onCreate，onStart，onResume`中`耗时较短`但`非必要`的代码可以放到`IdleHandler`中执行，`减少启动时间`

- 2.想要在一个`View`绘制完成之后添加其他依赖于这个`View的View`，当然这个用`View#post()`也能实现，区别就是前者会在`消息队列空闲时执行`,`优化页面的启动`,较复杂的`view`填充 填充里面的数据界面view绘制之前的话，就会出现以上的效果了，`view先是白`的，再出现. app的进程其实是`ActivityThread`,`performResumeActivity`先回调`onResume` ， 之后 执行`view`绘制的`measure, layout, draw`,也就是说`onResume`的方法是在绘制之前，在`onResume`中做一些耗时操作都会影响启动时间, 把在`onResume`以及其之前的调用的但非必须的事件（如某些界面View的绘制）`挪出来找一个时机`（即绘制完成以后）去调用即可。

##### 13：消息屏障，同步屏障机制what
View的绘制流程其实就是在View绘制流程启动前，给消息池发送一个`msg.target为空`的消息，然后给View的绘制任务的`msg设置为异步消息`，下次在Handler取消息的过程中`优先判断消息池的msg.target是不是空`，如果是，则去消息池中取出第一个`异步消息`执行。执行前先把`同步屏障消息移除`。这就是消息同步屏障机制

##### 14.子线程能不能更新UI，为什么Android系统不建议子线程访问UI
> 线程只更新在自己线程中创建的view
>
> 在`android`中`子线程可以有好多个`，但是如果每个线程都可以对ui进行访问，我们的界面可能就会变得`混乱不堪`，这样`多个线程操作`同一资源就会造成`线程安全`问题，当然，需要解决线程安全问题的时候，我们第一想到的可能就是`加锁`，但是加锁会`降低运行效率`，所以`android出于性能的考虑`，`并没有使用加锁来进行ui操作的控制`。

​                 