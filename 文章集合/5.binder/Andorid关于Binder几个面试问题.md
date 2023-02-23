> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
### 1.简单介绍下`binder`

`binder`是一种进程间通讯的机制

进程间通讯需要了解`用户空间`和`内核空间`

每个进程拥有自己的`独立虚拟机`，系统为他们分配的地址空间都是`互相隔离`的。
如两个进程需要进行通讯，则需要使用到`内核空间`做载体，内核空间是所有进程`共享`的一块内存区域。
而用户空间切到内核空间需要使用到系统api `ioctl`进行通讯。内核获取用户的数据需要使用`copy_from_user`，内核将数据发送给其他进程需要使用`copy_to_user`，这两个方法是有`性能开销`的，对于`socket`就是使用的这种模式，为了减少这部分的开销，内核提供了`binder`，`binder`只需要一次拷贝就可以实现进程通讯.

主要是使用`mmap`的原理：

在`内核空间`和`用户空间`都开辟一块`虚拟内存`区域同时指向一块`物理地址`，这样内核需要传递数据给用户空间时，只需要将数据`拷贝`到对应的`虚拟内存地址`中，用户可以通过虚拟内存`映射`关系，获取到内核中的数据，实现了`一次拷贝`通讯。

`binder`架构上面使用的是C/S架构：

![CS架构.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fdf0126a1f234b898048069fc2cafae9~tplv-k3u1fbpfcp-watermark.image?)

    binder中有三要素：
    客户端，服务端和ServiceManager
`binder`整体过程：

1.注册服务
2.获取服务
3.使用服务

![binder注册使用过程.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c3dfed4adf6402790a848e0dee3ac01~tplv-k3u1fbpfcp-watermark.image?)

### 2.Binder的定向制导，如何找到目标Binder，唤起进程或者线程?
**数据结构流程：**

	1.server注册过程 
	      1.server传入一个flat_binder_object给内核态。内核根据这个flat_binder_object创建binder_node节点，为每个进程服务，内部有个binder_proc.proc = server进程
	      2.serviceManager在内核态创建binder_ref引用这个binder_node，内部有一项desc = 1,2,3..，在用户态会创建一个服务链表{name ="server name",handle = "server handle"}
	2.client获取服务过程
	      3.client向sm查询服务，传递name
	      4.sm返回handle给驱动程序
	      5.驱动程序在sm的binder_ref_desc红黑树中根据handle找到binder_ref，再根据binder_ref.node找到binder_node,最后给client创建新的binder_ref指向这个binder_node，他的desc从1开始binder_ref{desc=1,node = binder_node},驱动返回desc给client，即handle总结：sm中的handle顺序是根据服务注册顺序显示，返回给client中的handle是根据服务获取的顺序显示的
	3.client使用handle过程
	      6.：驱动里面根据handle找到找到binder_ref，根据binder_ref找到binder_node，根据binder_node找到进程server

注：

```java
flat_binder_object{
    type:是binder实体还是引用，只有需要注册的服务可以传binder实体，其他只能传handle引用
    flag(联合体)
    binder(实体：处理函数)/handle(引用：服务的引用)：
    cookie
}
```

**数据传输过程(进程切换)：**

![整体过程.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/60c8c0f3204146ab8f70c758b00b2fd5~tplv-k3u1fbpfcp-watermark.image?)

**数据如何复制：**

![数据复制.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d1a56cb12df4f248f978f9671b81ebd~tplv-k3u1fbpfcp-watermark.image?)

### 3.Binder中的红黑树，为什么会有两棵binder_ref红黑树
- `refs_by_desc`主要是通过`desc`来查找对应的`binder_ref`
- `refs_by_node`主要是通过`node`来查找对应的`binder_ref`
> 查找方式不一样

### 4.Binder一次拷贝原理

传统的数据拷贝方式如`socket`：


```js
用户空间---->内核空间：`copy_from_user `
内核空间---->用户空间：`copy_to_user`
```

而binder使用`mmap`机制

        在内核空间和用户空间中间使用物理地址开辟了一个映射关系
    内核空间调用copy_from_user会直接将数据拷贝到内核空间并反馈到映射后的物理地址上，
    由于用户空间和物理地址也有个映射关系，用户空间可以直接通过映射的虚拟地址指针访问到写入物理地址的数据。
    这就是binder一次拷贝的原理

 ### 5.Binder传输数据的大小限制?
 对于`内核`可以传输的是`4M`，但是`应用层`限制在`1M-8K`范围内，这就是在进程间传输过大的数据会导致`崩溃`的原因

 ### 6.系统服务与bindService等启动的服务的区别
 `系统服务`需要将服务注册到`ServiceManager`，使用的时候需要通过服务名称去ServiceManger中获取服务的引用，

 而`bindService`等启动的服务是将服务注册到AMS中的`ServiceMap`中，所有的服务的`生命周期`都由`AMS`控制。启动服务的进程如果需要使用IPC通讯，都是和获取`AMS`的代理类进行通讯，`AMS`也是在`SystemServer`启动的时候一个注册到`ServiceManager`的系统服务。

### 7.Binder多线程
`binder`线程池`默认`提供了`15`个线程进行处理进程间`并发`事件，如果服务端线程`不够`用，则驱动会发出一个`信号`，应用层收到这个信号调用`Register_Thread`，这样驱动层就可以使用这个`新建`出来的`子线程`进行数据的处理

### 8.Android APP进程天生支持Binder通信的原理是什么?
> Android APP进程都是由Zygote进程孵化出来的。
> **常见场景**：

点击桌面`icon`启动`APP`，或者`startActivity`启动一个新进程里面的`Activity`，最终都会由`AMS`去调用`Process.start()`方法去向`Zygote`进程发送请求，让`Zygote`去`fork`一个新进程，`Zygote`收到请求后会调用`Zygote.forkAndSpecialize()`来`fork`出新进程,之后会通过`RuntimeInit.nativeZygoteInit`来初始化`Andriod` APP运行需要的一些环境，而`binder`线程就是在这个时候新建启动的

```js
virtual void onZygoteInit()
{
    sp proc = ProcessState::self();
    //启动新binder线程loop
    proc->startThreadPool();
}
```

### 9.同一个线程的请求必定是顺序执行，即使是异步请求(oneway)
一般而言，`Client`同步阻塞请求`Service`，直到`Service`提供完服务后才返回，不过，也有`特殊`的，比如请求用`ONE_WAY`方式，这种场景一般主要是用来`通知`，至于通知`被谁消费，是否被消费压根不会关心`。
	拿`ContentService`服务为例子，它是一个`全局的通知中心`，负责转发通知，而且，一般是群发，由于在转发的时候，`ContentService`被看做`Client`，如果这个时候采用普通的同步阻塞势必会造成通知的延时发送送，所以这里的`Client`采用了`oneway`，异步。
        

​	