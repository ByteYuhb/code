***本文正在参加[「金石计划 . 瓜分6万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")***
> 🔥 **Hi，我是小余。本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 为什么要学习屏幕刷新知识？

很多同学觉得屏幕刷新绘制知识点对他们开发不重要，没必要学习这些东西，这部分同学可能平时维护的是一些中小型项目或者应用是安装在特定设备上，只要求写写主界面，做一些简单的网络请求，业务交互相关知识，对性能这块要求不是很高，确实涉及不到太多屏幕刷新这块知识。

![又不是不能用.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7a59d925f7148eaac15e220f2b86cf8~tplv-k3u1fbpfcp-watermark.image?)

但对一些大中型项目来说可能就不一样了：**他们涉及业务较多，设备种类较多，往往一个app内部集成了十几个子业务甚至上百个，这对应用性能要求就更加严格了，app的体验也会间接导致用户的留存问题**。

所以学习屏幕绘制这类理论性较强的知识也是非常有必要的。

> 如果你想进阶成为高级开发：屏幕绘制这块知识也是一个绕不过去的坎。
## 1.带着问题出发
前面一篇[卡顿优化](https://juejin.cn/post/7161757546875715615#heading-4)的文章我们说过，**主流屏幕刷新频率是每秒60次**（高的有90,120等），也就是**16.6ms刷新一次屏幕**，
如果我们在主线程做一些耗时的操作，最直观的现象就是屏幕卡顿，其实就是发生了丢帧。

**由此抛出几个问题：**
- 1.16.6ms是什么意思，每次16.6ms都会调用一个绘制流程么？
- 2.为什么在主线程做一些耗时操作会出现卡顿？丢帧？
- 3.丢帧是个什么意思，是字面上的直接丢弃当前帧还是延后显示当前帧？
- 4.双缓冲是什么？有了双缓存就不会出现丢帧了么？三缓冲呢？
- 5.了解Vsync么？它的作用是什么？
- 6.画面撕裂是怎么造成的？
- 7.编舞者是个什么东西？

带着这些问题我们出发吧。

## 2.Android屏幕刷新前置知识
### CPU/GPU：
- **CPU**：`中央处理器`，主要用于计算数据，在Android中主要用于三大绘制流程中Surface的计算过程，起着生产者的作用
- **GPU**：`图像处理器`，主要用于游戏画面的渲染，在Android中主要用于将CPU计算好的Surface数据合成后放到buffer中，让显示器进行读取，起着消费者的作用。

如下图：
	
![屏幕刷新过程.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcd6c36fb3fb47c7b10444e541b74dfc~tplv-k3u1fbpfcp-watermark.image?)
其中GPU在架构中是以`SurfaceFlinger`服务的形式工作
	
### SurfaceFlinger：
SurfaceFlinger作用是接受多个来源的图形显示数据Surface，合成后发送到显示设备。

比如我们的主界面中：**可能会有statusBar，侧滑菜单，主界面，这些View都是独立Surface渲染和更新，最后提交给SF后，SF根据Zorder，透明度，大小，位置等参数，合成为一个数据buffer，传递HWComposer或者OpenGL处理，最终给显示器**。

![cpu和gpu.webp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/636dd527800349a886eff1696ca49f03~tplv-k3u1fbpfcp-watermark.image?)
	
### 逐行扫描
屏幕在刷新buffer的时候，并不是一次性扫描完成，而是**从左到右，从上到下的一个读取过程**，顺序显示一屏的每个像素点，你为什么看不到，因为太快了嘛，按60HZ的屏幕刷新率来算，这个过程只有16.66666...ms。
	
### 帧、帧率（数）、屏幕刷新率：
在视频领域中，帧就代表一张图片。玩过短视频剪辑的朋友应该对此很了解。

图中为放大后的**一帧图片**。	

![帧截图.jpg](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d1c7b8b46e34e00bea385123d7d1703~tplv-k3u1fbpfcp-watermark.image?)

而帧率和屏幕刷新率确是两个不同的概念：
- **帧率**：表示**GPU在1s中内可以渲染多少帧到buffer中**，单位是fps，这里要理解的是**帧率是一个动态的**，比如我们平时说的60fps，只是1s内最多可以渲染60帧，假如我们屏幕是静止的，则GPU此时就没有任何操作，帧率就为0.
- **屏幕刷新率**：屏幕在**1s内去buffer中取数据的次数**，单位为HZ，常见屏幕刷新率为60HZ。和帧率不一样，屏幕刷新率是一个固定值和硬件参数有关。
		
### 画面撕裂：
画面撕裂简单说就是显示器把多个帧显示在同一个画面中。如图：

![屏幕撕裂.webp](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/31ec877ae8e24dda88798c81fe71be14~tplv-k3u1fbpfcp-watermark.image?)

**画面撕裂的原因**：我们知道屏幕刷新率是固定的，假设为60HZ，正常情况下当我们的GPU的帧率也是60fps的时候，GPU绘制完一帧，屏幕刷新一帧，这样是不会出问题的，但是随着GPU显卡性能的提升，GPU的帧率超过60fps后，就会出现画面撕裂的情况，实际**在帧率较低的时候也会出现撕裂的情况。**

**所以其本质是帧率和屏幕刷新率的不一致导致的撕裂**。
	
那可能大家要说了，等屏幕一帧刷新完成后，再将新的一帧存到buffer中不就可以了，那你要知道，**早期的4.0之前设备是只有一个buffer，且其并没有buffer同步的概念，屏幕读取buffer中的数据时，GPU是不知道的，屏幕读取的同时，GPU也在写入，导致buffer被覆盖，出现同一画面使用的是不同帧的数据**。

那既然是因为使用同一个Buffer引起的画面撕裂，**使用两个buffer不就可以了**？
### 双缓冲
前面我们说到画面撕裂是由于单buffer引起的，在4.1之前，使用了双缓冲来解决画面撕裂。

- GPU写入的缓存为：`Back Buffer`
- 屏幕刷新使用的缓存为：`Frame Buffer`

如下图：
	
![双buffer.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db8b95c61c4a4275a8c67a798a503dcc~tplv-k3u1fbpfcp-watermark.image?)
**因为使用双buffer，屏幕刷新时，frame buffer不会发生变化，通过交换buffer来实现帧数据切换，那什么时候交换buffer呢？**
	
这就要引入Vsync的概念了。
	
### VSync（垂直同步）
我们知道如果一个屏幕在刷新的过程中，是不能交换buffer的，只有等屏幕刷新完成后以后才可以考虑buffer的交换.

**那具体什么时候交换呢**？`当设备屏幕刷新完毕后到下一帧刷新前，因为没有屏幕刷新，所以这段时间就是缓存交换的最佳时间`。

此时硬件屏幕会发出一个脉冲信号，告知GPU和CPU可以交换了，这个就是Vsync信号。
	
有了双缓冲和VSync是不是就都ok了？虽然上面方式可以解决屏幕撕裂的问题，但是还是会出现一些其他问题。
### Jank
双缓冲buffer交换还有个**前提就是GPU已经准备好了back buffer的数**据，如果在Vsync到来时back buffer并没有准备好，就不会进行缓存的交换，屏幕显示的还是前一帧画面，这种情况就是Jank。
	
有了上面的基础我们再来聊聊Android屏幕刷新机制的演变过程	
## 3.屏幕刷新机制的演变过程	
Android屏幕刷新机制演变过程按buffer的个数可以分为3个阶段：
- 1.`单buffer时代`
- 2.`双buffer时代`
- 3.`三buffer时代`
	
### 1.单buffer时代
GPU和显示器共用一块buffer，会引起画面撕裂。

### 2.双buffer时代

#### 2.1：在引入VSync前（Drawing without VSync）

![drawing without vsync.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6abe7cd8a7f54552ae974a1600b793c2~tplv-k3u1fbpfcp-watermark.image?)
- **CPU**：表示CPU绘制的时间段
- **GPU**：表示GPU合成back buffer的时间段
- **Display**:显示器读取frame buffer的时间段
		

按时间顺序：
- 1.Display显示第0帧画面，而CPU和GPU正在合成第1帧，且在Display显示下一帧之前完成了。
- 2.由于GPU在Display第一个VSync来之前完成了back buffer的填充，此时交换back buffer和frame buffer，屏幕进行刷新，可以正常显示第一帧数据。
- 3.再来看第2个VSync，第二个VSync到来之时，GPU并没有及时的填充back buffer，这个时候不能交互buffer，屏幕刷新的还是第1帧的画面。就说这里发生了“jank”
- 4.在第3个VSync信号到来时，第2帧的数据已经写入back buffer，第3帧的数据GPU还没写入，所以这个时候交互buffer显示的是第2帧的数据
- 5.同理，在第4个VSync时，第3帧数据已经处理完毕，交换buffer后显示的是第2帧的数据
		

这里发生`jank`的原因是：**在第2帧CPU处理数据的时候太晚了，GPU没有及时将数据写入到buffer中，导致jank的发生。**
		
如果可以把CPU绘制流程提前到每个VSync信号来的时候进行CPU的绘制，那是不是就可以让CPU的计算以及GPU的合成写入buffer的操作有完整的16.6ms。
		
#### 2.1：在引入VSync后（Drawing with VSync）
为了进一步优化性能，谷歌在4.1之后对屏幕绘制与刷新过程引入了**Project Butter**（`黄油工程`），系统在收到VSync信号之后，马上进行CPU的绘制以及GPU的buffer写入。
这样就可以让cpu和gpu有个完整的16.6ms处理过程。最大限度的减少jank的发生。
		
![drawing withvsync.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a3666b07fc3468389319ecade2b817b~tplv-k3u1fbpfcp-watermark.image?)

引入VSync后，新的问题又出现了：如下图：

![with问题.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/842b2c2ab4b1417a9f237a6ad16cd090~tplv-k3u1fbpfcp-watermark.image?)
		
**由于主线程做了一些相对复杂耗时逻辑，导致CPU和GPU的处理时间超过16.6ms，由于此时back buffer写入的是B帧数据，在交换buffer前不能被覆盖，而frame buffer被Display用来做刷新用，所以在B帧写入back buffer完成到下一个VSync信号到来之前两个buffer都被占用了，CPU无法继续绘制，这段时间就会被空着，
于是又出现了三缓存。**
### 3.三buffer时代
为了进一步优化用户体验，Google在双buffer的基础上又**增加了第三个buffer**（Graphic Buffer），
如图：
		
![三buffer.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a16ce3c316a4e5bacc0b6ff611c9f01~tplv-k3u1fbpfcp-watermark.image?)
**按时间顺序**：
- 1.第一个jank是无法避免的，因为**第一个B帧处理超时，A帧肯定是会重复的**。
- 2.在第一个VSync信号来时，虽然back buffer以及frame buffer都被占用了，CPU此时会启用第三个Graphic Buffer，避免了CPU的空闲状态。
		

**这里可以最大限度避免2中CPU空闲的情况，记住只是最大限度，没有说一定能避免。**
		
那又有人要说了，那就再多开几个不就可以了，是的，buffer越多jank越少，但是你得**考虑性价**比：
**3 buffer已经可以最大限度的避免jank的发生了，再多的buffer起到的作用就微乎其微，反而因为buffer的数量太多，浪费更多内存，得不偿失。**
buffer收益比：


> 不过假想下哪天由于硬件的改进，3 buffer已经满足不了的时候，谷歌又会加4 buffer，5 buffer..这都是可能的事情。


## 4.Choreographer
前面我们分析“双buffer时代“说过：**谷歌在4.1之后对屏幕绘制与刷新过程引入了Project Butter（黄油工程），系统在收到VSync信号之后，马上进行CPU的绘制以及GPU的buffer写入**。这样就可以让cpu和gpu有个完整的16.6ms处理过程。最大限度的减少jank的发生。
	
`那么在Android源码层是如何实现的呢？`
	
那就是这小节要讲解的Choreographer，译为编舞者，多么唯美的词，看来写这个源码的开发者也是个很优雅的绅士。
	
**Choreographer在屏幕绘制中的作用：**
- 1.注册VSync信号回调
- 2.接收SurfaceFlinger服务回调的onSync事件，SurfaceFlinger服务在接收到硬件发出的定时中断信号VSync后，将信号传递给App，这里App的接收者就是1中注册的回调。
		一般SurfaceFlinger服务接收到的中断信号VSync和App收到的VSync回调是有个offsets的。
	

**下面就来看下Choreographer在源码层的工作原理：**

**首先声明当前使用的是8.0的源码**

### 1.源码入口scheduleTraversals：
我们知道一个View在添加到窗口中时，绘制流程会调用到ViewRootImpl的setView()方法，setView方法会调用requestLayout()方法请求绘制，requestLayout方法中会调用scheduleTraversals()方法，**那就从scheduleTraversals开始吧。**
	

```java
void scheduleTraversals() {
		//这个字段保证该View已经绘制过，不会重复绘制
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
			//添加了一个同步屏障，保证异步绘制消息优先执行
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
			//post一个绘制的Runnable任务，即View的layout/measure/draw流程以及后续的GPU合成流程。
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
	
	final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
	
	final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
	
    void doTraversal() {
		//已经绘制过，才会走到内部
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
			//移除同步屏障
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }
			//开始执行performTraversals
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```

scheduleTraversals方法主要做了下面事情：	
- 1.先检查mTraversalScheduled是否已经绘制过，没有绘制过继续走下面流程，并将mTraversalScheduled标志置为true，防止重复绘制
- 2.调用当前Handler的Looper的MessageQueue的postSyncBarrier，设置一个**同步屏障**。
- 3.使用mChoreographer发送一个Choreographer.CALLBACK_TRAVERSAL的任务。
	

进入mChoreographer.postCallback方法里面看看：

### 2.任务提交postCallback
postCallback最终会调用到postCallbackDelayedInternal方法.

```java
private void postCallbackDelayedInternal(int callbackType,..) {
         ...
         synchronized (mLock) {
    final long now = SystemClock.uptimeMillis();
    final long dueTime = now + delayMillis;
    //将action封装到一个CallBackRecord中并放到mCallbackQueues的callbackType索引处
    mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

    if (dueTime <= now) {
        //表示delayMillis=0 立即执行
        scheduleFrameLocked(now);
    } else {
        //发送一个异步延迟任务msg.what = MSG_DO_SCHEDULE_CALLBACK,action = mTraversalRunnable，
        //这里经过Handler后最终也会执行到scheduleFrameLocked
        Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
        msg.arg1 = callbackType;
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, dueTime);
    }
}
}
```
postCallbackDelayedInternal做了下面这些事情：

- 1.将action封装到一个CallBackRecord中并放到mCallbackQueues的callbackType索引处
- 2.如果是立即执行的消息，则直接调用scheduleFrameLocked
- 3.如果是延迟消息，则发送一个MSG_DO_SCHEDULE_CALLBACK的msg

我们来看下MSG_DO_SCHEDULE_CALLBACK的Handler逻辑：

这里mHandler = FrameHandler类对象
	
```java
private final class FrameHandler extends Handler {
public FrameHandler(Looper looper) {
    super(looper);
}

@Override
public void handleMessage(Message msg) {
    switch (msg.what) {
        case MSG_DO_FRAME:
            doFrame(System.nanoTime(), 0);
            break;
        case MSG_DO_SCHEDULE_VSYNC:
            doScheduleVsync();
            break;
        case MSG_DO_SCHEDULE_CALLBACK:
            doScheduleCallback(msg.arg1);
            break;
    }
}
```

MSG_DO_SCHEDULE_CALLBACK的消息类型会走到doScheduleCallback，msg.arg1 = callbackType，

进入doScheduleCallback方法：

```java
void doScheduleCallback(int callbackType) {
    synchronized (mLock) {
        if (!mFrameScheduled) {
            final long now = SystemClock.uptimeMillis();
            if (mCallbackQueues[callbackType].hasDueCallbacksLocked(now)) {
                scheduleFrameLocked(now);
            }
        }
    }
}
```

判断mCallbackQueues[callbackType]是否有需要任务。有任务则执行scheduleFrameLocked
**	
可以看到postCallbackDelayedInternal最终都是执行scheduleFrameLocked方法**
	
直接看scheduleFrameLocked方法

### 3.回调注册scheduleVsync
```java
private void scheduleFrameLocked(long now) {
    //检测mFrameScheduled是否已经为true
    if (!mFrameScheduled) {
            mFrameScheduled = true;
            //是否开启了VSYNC，4.1之后默认开启，直接看这里即可
            if (USE_VSYNC) {
                    if (DEBUG_FRAMES) {
                            Log.d(TAG, "Scheduling next frame on vsync.");
                    }

                    // If running on the Looper thread, then schedule the vsync immediately,
                    // otherwise post a message to schedule the vsync from the UI thread
                    // as soon as possible.
                    //如果是在主线程上，则直接调用scheduleVsyncLocked
                    if (isRunningOnLooperThreadLocked()) {
                            scheduleVsyncLocked();
                    } else {
                            //其他线程则需要使用Handler将任务执行在主线程中。
                            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                            msg.setAsynchronous(true);
                            mHandler.sendMessageAtFrontOfQueue(msg);
                    }
            } else {
                    final long nextFrameTime = Math.max(
                                    mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                    if (DEBUG_FRAMES) {
                            Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                    }
                    Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
    }
}
```

- 1.检测mFrameScheduled是否已经为true
- 2.如果开启了VSYNC则调用scheduleVsyncLocked方法，没有开启则发送一个MSG_DO_FRAME的msg给mHandler。Android 4.1之后默认开启了VSYNC，所以直接看USE_VSYNC流程即可
- 3.如果是运行在当前线程的上，当前线程是UI线程。则直接调用scheduleVsyncLocked方法。
- 4.如果没有运行在主线程上， 则发送一个MSG_DO_SCHEDULE_VSYNC的msg给mHandler。根据FrameHandler的源码可以看出最终也是走到scheduleVsyncLocked方法
	

看scheduleVsyncLocked方法：

```java
private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
}
    这个mDisplayEventReceiver是什么时候赋值的呢。
    我们来看Choreographer的构造方法：
    private Choreographer(Looper looper, int vsyncSource) {
    //设置looper
    mLooper = looper;
    //创建mHandler实例
    mHandler = new FrameHandler(looper);
    //4.1默认开启，所以直接使用的是FrameDisplayEventReceiver类对象
    mDisplayEventReceiver = USE_VSYNC
            ? new FrameDisplayEventReceiver(looper, vsyncSource)
            : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;

    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

    //初始化一个CallbackQueue数组对象
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
}
```
回到scheduleVsyncLocked方法：调用了mDisplayEventReceiver.scheduleVsync()，mDisplayEventReceiver是FrameDisplayEventReceiver类对象
	
进入FrameDisplayEventReceiver类scheduleVsync方法中,子类未实现，在其父类DisplayEventReceiver中实现

```java
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                + "receiver has already been disposed.");
    } else {
        nativeScheduleVsync(mReceiverPtr);
    }
}

    public DisplayEventReceiver(Looper looper, int vsyncSource) {
    if (looper == null) {
        throw new IllegalArgumentException("looper must not be null");
    }

    mMessageQueue = looper.getQueue();
    //这个方法会将当前对象this = mDisplayEventReceiver以及mMessageQueue，vsyncSource传递给native层对象，并返回native层对象的地址值mReceiverPtr
    mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue,
            vsyncSource);

    mCloseGuard.open("dispose");
}
```

scheduleVsync中调用的是nativeScheduleVsync方法进行注册，**注册的是一个mReceiverPtr**，**这是一个native层的对象地址，这个地址是在DisplayEventReceiver构造方法中初始化，调用nativeInit方法返回的**，**nativeInit方法传入一个this，这个this就是前面的mDisplayEventReceiver对象**，所以重新回到前面的mDisplayEventReceiver讲解，
	
	
```java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {
    private boolean mHavePendingVsync;
    private long mTimestampNanos;
    private int mFrame;

    public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
        super(looper, vsyncSource);
    }

    @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
                    ...           
        mFrame = frame;
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }
}
```

**FrameDisplayEventReceiver类实现了onVsync方法，这个方法就是native层在接收到VSync信号后回调的方法。onVsync方法直接发送一个异步消息，执行的任务是自己的run方法**。

run中执行doFrame()，进入doFrame看看：

### 4.图帧绘制doFrame
```java
void doFrame(long frameTimeNanos, int frame) {
    ...
    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

        mFrameInfo.markInputHandlingStart();
                    //处理输入事件
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

        mFrameInfo.markAnimationsStart();
                    //处理动画事件
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

        mFrameInfo.markPerformTraversalsStart();
                    //处理CALLBACK_TRAVERSAL，三大绘制流程
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
                    //处理CALLBACK_COMMIT事件
        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    ...
}
```

doFrame中：
- **1.执行输入事件**
- **2.处理动画事件**
- **3.处理CALLBACK_TRAVERSAL，三大绘制流程，其实就是前面的mTraversalRunnable事件**
- **4.处理CALLBACK_COMMIT提交帧事件**
	

进入doCallbacks方法：

```java
void doCallbacks(int callbackType, long frameTimeNanos) {
    for (CallbackRecord c = callbacks; c != null; c = c.next) {
            if (DEBUG_FRAMES) {
                    Log.d(TAG, "RunCallback: type=" + callbackType
                                    + ", action=" + c.action + ", token=" + c.token
                                    + ", latencyMillis=" + (SystemClock.uptimeMillis() - c.dueTime));
            }
            c.run(frameTimeNanos);
    }
}
```
循环执行callbacks中的记录：

```java
private static final class CallbackRecord {
    public CallbackRecord next;
    public long dueTime;
    public Object action; // Runnable or FrameCallback
    public Object token;

    public void run(long frameTimeNanos) {
        if (token == FRAME_CALLBACK_TOKEN) {
            ((FrameCallback)action).doFrame(frameTimeNanos);
        } else {
            ((Runnable)action).run();
        }
    }
}
```


​	
CallbackRecord的run方法在token = null的情况下执行的是action的run方法
这里再看

```java
mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
```
**传入的token确实为null，且action = mTraversalRunnable**，

**`这样整个处理流程就闭环了。`**
	
**这里token = FRAME_CALLBACK_TOKEN是在什么情况下呢？**
### 5.帧率计算postFrameCallback
在Choreographer的postFrameCallback方法中：

```java
public void postFrameCallback(FrameCallback callback) {
        postFrameCallbackDelayed(callback, 0);
}

public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    if (callback == null) {
        throw new IllegalArgumentException("callback must not be null");
    }

    postCallbackDelayedInternal(CALLBACK_ANIMATION,
            callback, FRAME_CALLBACK_TOKEN, delayMillis);
}
```


最终也是执行到postCallbackDelayedInternal方法，不同之处在于，其传入的token是FRAME_CALLBACK_TOKEN

那么这个方法有什么作用呢？**计算丢帧**。

我们使用下面的方法对丢丢帧30次以上在logcat中打印一个日志。
	

- 1.创建一个FrameCallback子类

```java
public class YourFrameCallback implements Choreographer.FrameCallback {

  private static final String TAG = "FPS_TEST";
  private long mLastFrameTimeNanos = 0;
  private long mFrameIntervalNanos;

  public YourFrameCallback(long lastFrameTimeNanos) {
      mLastFrameTimeNanos = lastFrameTimeNanos;
      mFrameIntervalNanos = (long)(1000000000 / 60.0);
  }

  @Override
  public void doFrame(long frameTimeNanos) {

      //初始化时间
      if (mLastFrameTimeNanos == 0) {
          mLastFrameTimeNanos = frameTimeNanos;
      }
      final long jitterNanos = frameTimeNanos - mLastFrameTimeNanos;
      if (jitterNanos >= mFrameIntervalNanos) {
          final long skippedFrames = jitterNanos / mFrameIntervalNanos;
          if(skippedFrames>30){
              //丢帧30以上logcat打印日志
              Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                      + "The application may be doing too much work on its main thread.");
          }
      }
      mLastFrameTimeNanos=frameTimeNanos;
      //因为每次doFrame都会消费掉，需要重新注册下一帧回调
      Choreographer.getInstance().postFrameCallback(this);
  }
}
```
- 2.在Application启动的时候：调用Choreographer的postFrameCallback方法，并传入一个FrameCallback

```java
Choreographer.getInstance().postFrameCallback(new YourFrameCallback(System.nanoTime()));
```


**对这小节总结**：
- 1.在Choreographer的构造函数中会创建一个FrameDisplayEventReceiver类对象，这个对象实现了onVSync方法，用于VSync信号回调。
- 2.FrameDisplayEventReceiver这个对象的父类构造方法中会调用nativeInit方法将当前FrameDisplayEventReceiver对象传递给native层，native层返回一个地址mReceiverPtr给上层。
- 3.主线程在scheduleVsync方法中调用nativeScheduleVsync，并传入2中返回的mReceiverPtr，这样就在native层就正式注册了一个FrameDisplayEventReceiver对象。
- 4.native层在GPU的驱使下会定时回调FrameDisplayEventReceiver的onVSync方法，从而实现了：在VSync信号到来时，立即执行doFrame方法
- 5.doFrame方法中会执行输入事件，动画事件，layout/measure/draw流程并提交数据给GPU。这样就闭环了
	
	

绘制流程图如下：来自《[Android 之 Choreographer 详细分析](https://www.jianshu.com/p/86d00bbdaf60)》
	
![Android 之 Choreographer 详细分析.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b1576c3cab964dfda06881f3ca2acd82~tplv-k3u1fbpfcp-watermark.image?)
	
## 5.Handler同步屏障机制
核心思想：**在主线程Looper获取msg的时候，让异步消息优先执行，同步消息滞后**。

这里先上一张原理图：
	
![同步机制.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/778afe9baca44fc0aee2977912bbc3f4~tplv-k3u1fbpfcp-watermark.image?)
我们依次对图中几个点进行源码讲解：
- 1.同步屏障的创建
- 2.异步消息的优先执行
	
### 1.同步屏障的创建
使用的是:MessageQueue#postSyncBarrier()

```java
/**
 * 同步屏障就是一个同步消息，只不过这个消息的target为null
 */
private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            // 从消息池中获取Message
            final Message msg = Message.obtain();
            msg.markInUse();
            // 初始化Message对象的时候，并没有给Message.target赋值，
            // 因此Message.target==null
            msg.when = when;
            msg.arg1 = token;

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                // 这里的when是要加入的Message的时间
                // 这里遍历是找到Message要加入的位置
                while (p != null && p.when <= when) {
                        // 如果开启同步屏障的时间（假设记为T）T不为0，且当前的同步
                        // 消息里有时间小于T，则prev也不为null
                        prev = p;
                        p = p.next;
                }
            }
            // 根据prev是否为null，将msg按照时间顺序插入到消息队列的合适位置
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
}
```

注释中有说明了，**这里的屏障消息就是一个target为null的消息，因为其不需要执行任务消息，只是用来设置一堵墙**.

再根据时间戳将其插入到合适的位置。
	
### 2.异步消息的优先执行
移步到：MessageQueue的next方法：

```java
Message next() {
    ...

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    // 1.如果nextPollTimeoutMillis=-1，一直阻塞不会超时
    // 2.如果nextPollTimeoutMillis=0，不会阻塞，立即返回
    // 3.如果nextPollTimeoutMillis>0，最长阻塞nextPollTimeoutMillis毫秒（超时）
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // 获取系统开机到现在的时间戳
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            // 取出target==null的消息
            // 如果target==null，那么它就是屏障，需要循环遍历，
            // 一直往后找到第一个异步消息，即msg.isAsynchronous()为true
            // 这个target==null的消息，不会被取出处理，一直会存在
            // 每次处理异步消息的时候，都会从头开始轮询
            // 都需要经历从msg.target开始的遍历
            if (msg != null && msg.target == null) {
                    // 使用一个do..while循环
                    // 轮询消息队列里的消息，这里使用do..while循环的原因
                    // 是因为do..while循环中取出的这第一个消息是target==null的消息
                    // 这个消息是同步屏障的标志消息
                    // 接下去进行遍历循环取出Message.isAsynchronous()为true的消息
                    // isAsynchronous()为true就是异步消息
                    do {
                            prevMsg = msg;
                            msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                    // 如果有消息需要处理，先判断时间有没有到，如果没有到的话设置阻塞时间
                    if (now < msg.when) {
                            // 计算出离执行时间还有多久赋值给nextPollTimeoutMillis
                            // 表示nativePollOnce方法要等待nextPollTimeoutMillis时长后返回
                            nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                            // 获取到消息
                            mBlocked = false;
                            // 链表操作，获取msg并且删除该节点
                            if (prevMsg != null) {
                                    prevMsg.next = msg.next;
                            } else {
                                    mMessages = msg.next;
                            }
                            msg.next = null;
                            if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                            msg.markInUse();
                            // 返回拿到的消息
                            return msg;
                    }
            } else {
                    // 没有消息，nextPollTimeoutMillis复位
                    nextPollTimeoutMillis = -1;
            }

                ...
        }
    }
}
```

**优先判断是否有同步屏障存在，然后取同步屏障后面的异步消息进行处理**。就达到了优先执行异步消息的目的。
	
好了，关于同步屏障机制就讲到这里。

有了上面这些基础讲解。下面对一开始的问题进行一个总结归纳。
	

## 6.问题总结

### 1.16.6ms是什么意思，每次16.6ms都会调用一个绘制流程么？
16.6ms是指刷新频率为是60HZ，1s需要执行60次，平均每次16.6ms。也可以理解为VSync的一个周期是16.6ms。
并非每次16.6ms都会执行三大绘制流程，屏幕静止状态，CPU并不会执行绘制流程

### 2.画面撕裂是怎么造成的？
画面撕裂是早期使用的是一个buffer进行屏幕的刷新读取和GPU的写入操作，且不存在同步锁的情况下，新数据覆盖旧的数据导致一张画面显示多帧的场景
### 3.为什么在主线程耗时，布局层级太多，会出现卡顿？丢帧？
主线程耗时，布局层级太多会影响CPU的计算和GPU的合成过程，超过VSync信号后，需要等下一个VSync信号来才能进行buffer交换，发生丢帧现象

### 4.丢帧是个什么意思，是字面上的直接丢弃当前帧还是延后显示当前帧？
丢帧是指在第一个VSync信号来之前并没有合成好back buffer数据，无法交换buffer，屏幕刷新的还是上一帧数据，这就是丢帧。
下一个VSync信号来之后，back buffer绘制好后，再交换buffer，所以其不会丢弃，而是延后显示，直观感受就是卡顿。

### 5.双缓冲是什么？有了双缓存就万事大吉了么？三缓冲呢？
双缓冲是指使用两个buffer进行数据的缓存，一个用于GPU的合成，一个用于屏幕的刷新，互不干扰，防止出现画面撕裂的场景
有了双缓存还是会出现丢帧的现象和CPU的空闲等问题。
三缓冲就是在双缓冲上再增加一个Graphic Buffer，避免CPU长时间空闲。
### 6.了解Vsync么？它的作用是什么？
VSync（垂直同步）有两个作用：
1.提醒GPU进行buffer的交换
2.提醒CPU立即进入屏幕绘制过程，别闲着啦。

### 7.编舞者是个什么东西？
编舞者Choreographer是4.1以后引入的：
作用：用来提醒CPU在VSync信号来时立即对View进行绘制，防止出现CPU空闲状态。


好了，就讲到这里吧，上面文章涉及了：**UI绘制流程，Handler同步异步消息机制，屏幕刷新流程，卡顿优化**等知识点	

如果你能把本文涉及的知识点都吸收，对后期framework层的学习是很有帮助的。

**如果你喜欢我的文章，请帮忙点赞，关注，这是对我的巨大鼓励！**

**欢迎关注我的公众号**：`小余的自习室`


**参考**

[数据的流动——计算机是如何显示一个像素的](https://zhuanlan.zhihu.com/p/32136704)

[聊聊Android屏幕刷新机制](https://jishuin.proginn.com/p/763bfbd577a7)

[Android垂直同步信号VSync的产生及传播结构详解](https://blog.csdn.net/houliang120/article/details/50908098)