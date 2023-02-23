

## 前言

> RxJava已经出现很多个年头了，但是依然被很多公司使用，如果现在还对RxJava了解的不够透彻， 可以看这个系列对它的分析：相信看完后你对它会有个更全面的认识。 这个系列主要从下面几个方面来讲解： **RxJava基本操作符使用** **RxJava响应式编程是如何实现的** **RxJava的背压机制及Flowable是如何实现背压的** **RxJava的线程切换原理** 关于RxJava的其他系列文章，可以点击下方链接


[面试官：RxJava背压机制有了解么？](https://juejin.cn/post/7108013418614898702/)

[面试官：RxJava是如何做到响应式编程的？](https://juejin.cn/post/7108159256590974983/)

[使用rxjava创建一个rxbus事件处理框架](https://juejin.cn/post/7108174644238090248/)

[Rxjava操作符详解--看看你还记得多少](https://juejin.cn/post/7107992677932597255/)


## 简介：

说道rxjava，大家并不陌生，对它的操作符应该也都可以信手拈来。 也都知道rxjava采用的是观察者模式的响应式编程，也可以说是Steam的编程模式 **Steam的编程模式**： 我们在使用IO流的过程中经常使用这种方式读取流和写入流：

```
FileInputStream fileInputStream = new FileInputStream(new File("fileDir/filepath"));
DataInputStream dis = new DataInputStream(fileInputStream);
BufferedInputStream bis = new BufferedInputStream(dis);
```

-   上面就是一种Stream数据流模式的处理方式，使用的是*装饰器模式*，每次装饰后，如果数据流流到某个点都会做一些额外处理 将处理后的数据流继续分发到下一级，这就是流模式的处理方法

-   在同步方式下，数据依次向下流动，每个数据流执行完毕后在执行下个数据流，是有顺序的执行，这种模式的缺点就是无法执行并发状态下的数据流

-   那么就需要使用到*异步方式*： 异步的方式，可以大大提高我们处理数据流的执行效率，同一时间会有多个数据流进行处理，**效率提高的同时会引起一些其他的问题**， 比如*数据处理的不同步*，又比如*当上游事件发送的速度快于下游处理的速度*，这个时候我们就说出现了*背压*。

    > 早期rxjava对这种情况的处理是使用一个无大小限制的队列将积压的事件存储起来，这种情况有个问题是，如果数据流太多一直得不到处理或者处理了一半出现异常退出 那么就会出现OOM的情况 下面我们使用一个例子来看下这个情况：

```
PublishProcessor<Integer> so = PublishProcessor.create();
so.observeOn(Schedulers.computation()).subscribe(v->compute(v),Throwable::printStackTrace);
int count = 100;
for (int i = 0; i < count; i++){
    so.onNext(i);
}
```

```
打印结果：
computing : 0
computing : 1
computing : 2
computing : 3
...
...
computing : 96
computing : 97
computing : 98
computing : 99
```

*我们将count设置为1000*

```
打印结果：
computing : 0
computing : 1
io.reactivex.exceptions.MissingBackpressureException: Could not emit value due to lack of requests
    at io.reactivex.processors.PublishProcessor$PublishSubscription.onNext(PublishProcessor.java:315)
```

**可以看出上面是使用异步的方式，同时发送100个和1000个事件，前者正常而后者报了*MissingBackpressureException*异常** 这就是因为我们的PublishProcessor默认最大支持存储128个并发数，如果超过这个数就会报异常。

```

**我们在每次onNext后延迟一秒来看看：**
PublishProcessor<Integer> so = PublishProcessor.create();
so.observeOn(Schedulers.computation()).subscribe(v->compute(v),Throwable::printStackTrace);
int count = 100;
for (int i = 0; i < count; i++){
    so.onNext(i);
    sleep(1);//秒
}
打印结果：
computing : 0
computing : 1
computing : 2
computing : 3
...
...
computing : 96
computing : 97
computing : 98
computing : 99
看出在延迟一秒后，都正常打印了，说明确实是积压数超过最大值128导致
```

从上面例子我们知道，我们的积压数MAX在背压处理过程中起着关键作用 我们尝试将这个值改大：

使用so.onBackpressureBuffer(1001).observeOn..改变大小

打印结果：

`computing : 0 computing : 1 computing : 2 ... ... computing : 997 computing : 998 computing : 999`

-   可以看到在1001这个值内都是正常打印的 超过这个值后： 报：io.reactivex.exceptions.MissingBackpressureException: Buffer is full 缓存已满

*既然这个值这么重要那么我们来从源码分析下这个值*：

## 源码分析

老规矩我们先把**源码分层**： 1.创建PublishProcessor 2.调用observerOn创建一个任务执行切换线程的观察者 3.执行任务

#### **1.创建PublishProcessor**

```

    PublishProcessor.create()
    public static <T> PublishProcessor<T> create() {
        return new PublishProcessor<T>();//直接返回一个PublishProcessor
    }
    //构造方法：
    PublishProcessor() {
        subscribers = new AtomicReference<PublishSubscription<T>[]>(EMPTY);
    }
    //这里创建了一个PublishSubscription[]类型的AtomicReference格式的对象：默认值EMPTY = new PublishSubscription[0]，这个默认值很关键，后面会对这个值进行判断，看是否有改写
```

#### **2.调用observerOn创建一个任务执行切换线程的观察者**

通过之前我们对观察者订阅操作的分析， 订阅回调到最上游的：PublishProcessor的subscribeActual操作:

```

    protected void subscribeActual(Subscriber<? super T> t) {
        PublishSubscription<T> ps = new PublishSubscription<T>(t, this);
        t.onSubscribe(ps);
        ...
    }
```

subscribeActual中会调用t.onSubscribe(ps)订阅dispose方法：t是下游ObservableObserveOn传过来的FlowableObserveOn **FlowableObserveOn.java**

```
public void onSubscribe(Subscription s) {
    if (SubscriptionHelper.validate(this.s, s)) {
        this.s = s;

        if (s instanceof QueueSubscription) { 
            @SuppressWarnings("unchecked")
            QueueSubscription<T> f = (QueueSubscription<T>) s;

            int m = f.requestFusion(ANY | BOUNDARY);

            if (m == SYNC) {//1
                sourceMode = SYNC;
                queue = f;
                done = true;

                actual.onSubscribe(this);
                return;
            } else
            if (m == ASYNC) {
                sourceMode = ASYNC;
                queue = f;

                actual.onSubscribe(this);

                s.request(prefetch);//2

                return;
            }
        }

        queue = new SpscArrayQueue<T>(prefetch);

        actual.onSubscribe(this);

        s.request(prefetch);//2
    }
}
```

在1处这里是对异步操作进行判断，如果是actual.onSubscribe(this); 如果是异步操作需要调用2处的s.request操作,这里的s = PublishSubscription[0]对象

**PublishSubscription.java**

```
public void request(long n) {
        if (SubscriptionHelper.validate(n)) {
            BackpressureHelper.addCancel(this, n);
        }
    }
    BackpressureHelper.java
    public static long addCancel(AtomicLong requested, long n) {
        ...
            long u = addCap(r, n);
            if (requested.compareAndSet(r, u)) {
                return r;
            }
        ..
    }
```

看到在这里对requested的值进行了更改

        通过以上步骤可以看出可以修改FlowableObserveOn中的prefetch值可以改变大小，这个就是缓存大小

```
那是不是说改的越大越好呢，当我们把缓存改的太大会发生什么呢？
首先缓存太大会有内存溢出甚至OOM的风险，设置的值最好根据自己的需求来设置
```

*这里我总结了几个比较常见的背压优化方法：*

## 背压优化方案：

#### 1.使用改变缓存大小的方式

1.1：observeOn(Schedulers.computation(),false,1000)，第三个参数1000即时默认大小 根据FlowableObserveOn的构造方法：

```

            public FlowableObserveOn(
                    Flowable<T> source,
                    Scheduler scheduler,
                    boolean delayError,
                    int prefetch) {
                super(source);
                this.scheduler = scheduler;
                this.delayError = delayError;
                this.prefetch = prefetch;
            }
```

可知可以从外部传入值即可：observeOn(Schedulers.computation(),false,1000)，第三个参数1000即时默认大小

1.2：也可以通过在observerOn前面调用onBackpressureBuffer 这个方法在onSubscribe方法中： BackpressureBufferSubscriber.java public void onSubscribe(Subscription s) { if (SubscriptionHelper.validate(this.s, s)) { this.s = s; actual.onSubscribe(this); s.request(Long.MAX_VALUE); }\
} 这里设置了s.request(Long.MAX_VALUE);将缓存设置为最大值

##

#### 2.使用策略模式对背压进行处理

```
        onBackpressureBuffer(long capacity, Action onOverflow, BackpressureOverflowStrategy overflowStrategy) {
        overflowStrategy：可以选择
        ERROR：直接报错
        DROP_LATEST：丢弃最新的数据，只存储旧的数据，使用场景如对数据精度要求不高的情况，旧的数据也可以正常使用
        DROP_OLDEST：丢弃旧的数据，存储新的数据，使用场景如定位情况，可以把旧的丢弃，存储新的定位数据就可以
```

#### 3.其他方式

```
        onBackpressureDrop(Consumer<? super T> onDrop) ：来不及处理就丢弃，不缓存任何数据
        onBackpressureLatest()：会缓存一个数据，当正在执行某个任务的时候有新的数据过来，会把它缓存起来，如果又有新的数据过来，那就把之前的替换掉，缓存里面的总是最新的
```

#### 这里我们再来看下Flowable中的背压机制

我们根据调试进入Flowable源码看看： 在FlowableObserveOn.java中： 方法request会在订阅的时候调用：*t.request(Long.MAX_VALUE);*

```
public final void request(long n) {
        if (SubscriptionHelper.validate(n)) {
            BackpressureHelper.add(requested, n);
            trySchedule();
        }


}
```

内部调用了*BackpressureHelper.add(requested, n)*：

```
public static long add(AtomicLong requested, long n) {
        for (;;) {
            long r = requested.get();
            if (r == Long.MAX_VALUE) {
                return Long.MAX_VALUE;
            }
            long u = addCap(r, n);
            if (requested.compareAndSet(r, u)) {
                return r;
            }
        }
    }
```

> 这里调用了requested.compareAndSet(r, u)将缓存设置为了Long.MAX_VALUE，所以说Flowable是支持背压的 Flowable对不同的操作符有不同给的背压处理方式，可以自己去阅读源码看看，我这只是对一个抛砖引玉的效果。

## 总结

以上就是背压机制的一些内容，以及我们介绍了 Flowable 中的几个背压相关的方法。实际上，RxJava 的官方文档也有说明—— Flowable 适用于数据量比较大的情景，因为它的一些创建方法本身就使用了背压机制。 这部分方法我们就不再一一进行说明，因为，它们的方法签名和 Observable 基本一致，只是多了一层背压机制。

其他系列文章

[面试官：RxJava背压机制有了解么？](https://juejin.cn/post/7108013418614898702/)

[面试官：RxJava是如何做到响应式编程的？](https://juejin.cn/post/7108159256590974983/)

[[使用rxjava创建一个rxbus事件处理框架](https://juejin.cn/post/7108174644238090248/)
[Rxjava操作符详解--看看你还记得多少](https://juejin.cn/post/7107992677932597255/)