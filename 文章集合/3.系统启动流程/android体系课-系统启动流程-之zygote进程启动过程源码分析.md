## 前言
> 本文主要记录笔者对Android系统启动流程的一个的一个过程了解

笔者刚开始学习Android的时候也和大部分同学一样，只会使用一些应用层面的知识，对于一些比较常见的开源框架如<mark>RxJava</mark>，<mark>OkHttp</mark>，<mark>Retrofit</mark>，以及后来谷歌推出的<mark>协程</mark>等，都只在使用层面，对于他们<mark>内部原理</mark>，基本没有去了解觉得够用就可以了，又比如Activity，Service等四大组件的使用原理，系统开机过程，Launcher启动过程等知之甚少，知其然而不知其所以然，结果就是出现某些问题，不知道从哪里找原因，只能依赖万能的百度，但是百度看多了，你会发现自己还是什么都不会，过一段时间可能就忘记了

- 为了能更加深刻的理解这些机制，笔者决定自己深入Framework层面去考察下这些代码的机制：

***讲解时间线如下：***

- 1.[android体系课-系统启动流程-之zygote进程启动过程源码分析](https://juejin.cn/post/7112465356991496200/)

- 2.[android体系课-系统启动流程-之SystemServer启动过程源码分析](https://juejin.cn/post/7112663893205319716/)

- 3.android体系课-系统启动流程-之startActivity启动流程源码分析
- 4.android体系课-系统启动流程-之android系统启动面试题

对于源码解析部分，可能大家会有部分疑惑，我们列出部分：

- 1.zygoye是什么？有什么作用 

    答：zygote英文翻译是“孵化”。

在了解zygote之前首先我们需要知道我们进程是如何创建的？

	我们安卓设备内核使用的是linux内核，
	开机先是启动一个init进程，在init进程中会创建一个zygote进程，
	zygote进程会启动java虚拟机并注册java-native 的jni方法，通过jni调用到java层的ZygoteInit，在这个类里面会注册一个ServerSocket用于设备创建应用进程的需要。
	当启动一个应用需要创建一个进程时，会调用到这个ServerSocket的创建进程流程，主要是通过fork的方式创建进程，这样子进程就拥有了父进程所有的资源，
	fork会返回两次，当返回的pid=0时说明现在是处于子进程，继续处理子进程流程就可以了。如果是SystemServer进程，则会启动各类服务，包括AMS,WMS等系统服务。

**按上面所讲，zygote就是作为创建子进程的一个孵化器。**‘

- 2.SystemServer是什么？有什么作用？它与zygote的关系是什么？

答：SystemServer是一个由zygotefork出来的子进程，该进程主要作用是创建创建<mark>binder</mark>线程池和启动android运行需要的诸多系统服务，比如：<mark>AMS</mark>，<mark>PMS</mark>，<mark>WMS</mark>等并启动系统Launcher
	
- 3.ActivityManagerService是什么？什么时候初始化的？有什么作用？

答：ActivityManagerService，简称AMS，服务端对象，负责统一调度系统中四大组件的生命周期。ActivityManagerService进行初始化的时机很明确，就是在SystemServer进程开启的时候，就会初始化ActivityManagerService

    **APP和AMS其实是通过binder进行通讯的。**
    
    binder机制简述。
> 		binder其实是进程间通信的一种方式，在linux进程中有socket，管道，共享内存，消息队列等方式，
> 		而由于binder对性能消耗低和安全性方面的考虑，android大量的使用了binder通讯 ，可以说binder是android的基石。

	binder模型简述：

> 		首先我们binder是通过CS架构进行设计的：Client和Service
> 		先搞懂三个概念：client，server和serverManger
> 		server通过binder通讯将自己注册到serverManager中，client通过binder获取serverManager中注册的server，
> 		其实是获取一个handle，通过这个handle就可以在bidner驱动中找找到server进程和需要处理的方法地址等信息。
> 		从而间接实现了进程间通讯，这个是binder的一个简要概述，详细的需要去了解binder驱动实现原理。

- 4.Launcher是什么？什么时候启动的？
	
        答：Launcher也是一个应用app，是在SystemServer启动的时候，启动的一个应用
	

5.Instrumentation是什么？和ActivityThread是什么关系？

	答：Instrumentation是一个沟通Activity和AMS以及ActivityThread和Activity中间桥梁工具类
	例如：Activity.startActivity
			->mInstrumentation.execStartActivity
			->ActivityManagerService.startActivity->ActivityThread.thread.scheduleLaunchActivity
			->ActivityThread.handleLaunchActivity
			->mInstrumentation.callActivityOnCreate


**针对以上问题：笔者会以源码的方式呈现**

我们先来看下Android设备启动过程

- init进程 0号进程

**作用**：

1.创建和挂载启动所需的文件系统：tmpfs,devpts,proc,sysfs,selinuxfs等

2.初始化和启动属性服务

3.设置子进程(zygote进程)异常退出处理函数。

	在UNIX/linux中，父进程调用fork函数创建子进程，在子进程在异常退出后，如果父进程不知道子进程已经退出，其会在系统进程表中保留进程信息(进程号，退出状态和运行时间等)，
	这个子进程就变成僵尸进程，系统进程表是一项有限资源，如果系统进程被僵尸进程耗尽就会无法创建新的进程

4.解析init.rc文件,并启动孵化zygote进程（Android初始化语言）



5.启动孵化zygote进程，这个是本篇文章的分析重点，从源码角度分析了Android zygote进程启动过程

**app_main.cpp**


```java
	int main(){
		runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
			AndroidRuntime.cpp:start:
			//启动java虚拟机
			startVm(&mJavaVM, &env, zygote) != 0；
			//注册android启动需要的jni方法
			startReg(env) < 0；
			//startClass="com.android.internal.os.ZygoteInit" startMeth=main方法
			env->CallStaticVoidMethod(startClass, startMeth, strArray); 
			//即调用ZygoteInit的main方法
			ZygoteInit.java:main{	
				//创建一个Server端的socket，socketName=“zygote” 用于接收应用的请求创建应用进程fork方式
				zygoteServer.registerServerSocket(socketName);
					ZygoteServer.java:registerServerSocket
				//预加载系统类和系统资源
				preload(bootTimingsTraceLog){
					preloadClasses();
					preloadResources();
					preloadOpenGL();
					preloadSharedLibraries();
					preloadTextResources();
					WebViewFactory.prepareWebViewInZygote();
				}
				//启动SystemServer进程
				startSystemServer(abiList, socketName, zygoteServer){
					//fork system server进程
					pid = Zygote.forkSystemServer(parsedArgs.uid, parsedArgs.gid...);
					//pid=0,说明运行在子进程中：SystemServer进程中
					if (pid == 0) {
						//关闭SystemServer进程中的socket，因为是通过fork创建，子进程也会有socket服务，需要关闭子进程的socket通道
						zygoteServer.closeServerSocket();
						//处理SystemServer进程
						handleSystemServerProcess(parsedArgs){
							//创建PathClassLoader
							cl = createPathClassLoader(...);
							//注释2
							ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl){
								//RuntimeInit初始化操作
								RuntimeInit.commonInit();
									RuntimeInit.java:commonInit
									//设置主线程未捕获异常时回调的Handler,可以在app中替换
									Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler());
								//对应AndroidRunTime：com_android_internal_os_ZygoteInit_nativeZygoteInit:
								ZygoteInit.nativeZygoteInit(){
									//gCurRuntime = AppRuntime在app_main.cpp中初始化
									gCurRuntime->onZygoteInit(); 
										sp<ProcessState> proc = ProcessState::self();
										//启动线程池，内部会循环获取其他应用发过来的数据，根据binder机制
										proc->startThreadPool();
								}
								RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader){
								
									invokeStaticMain(args.startClass, args.startArgs, classLoader){
										//前面传入的是"com.android.server.SystemServer"，加载类SystemServer
										cl = Class.forName(className, true, classLoader); 
										//反射获取SystemServer的main方法
										m = cl.getMethod("main", new Class[] { String[].class });
										//可以再注释1出捕获，最终进入SystemServer的main方法
										throw new Zygote.MethodAndArgsCaller(m, argv);
									}
								}
							}
						}			
					}
				}
				//这个还是在app_main进程中等待AMS创建新的应用程序进程 注释3
				zygoteServer.runSelectLoop(abiList){
					while (true) {
						StructPollfd[] pollFds = new StructPollfd[fds.size()];
						for (int i = 0; i < pollFds.length; ++i) {
							pollFds[i] = new StructPollfd();
							pollFds[i].fd = fds.get(i);
							pollFds[i].events = (short) POLLIN;
						}
						try {
							//poll函数使用pollfd类型的结构来监控一组文件句柄
							//https://blog.csdn.net/sijinxiaotongxue/article/details/81988411
							Os.poll(pollFds, -1);
						} catch (ErrnoException ex) {
							throw new RuntimeException("poll failed", ex);
						}
						for (int i = pollFds.length - 1; i >= 0; --i) {
							if ((pollFds[i].revents & POLLIN) == 0) {
								continue;
							}
							if (i == 0) {i=0为connect阶段
								ZygoteConnection newPeer = acceptCommandPeer(abiList);
								peers.add(newPeer);
								fds.add(newPeer.getFileDesciptor());
							} else {
								//不为0则说明AMS向孵化进程发出一个创建进程的请求，调用fork方式创建进程
								boolean done = peers.get(i).runOnce(this){
									acceptCommandPeer.runOnce(this){
										//读客户端发来的数据
										args = readArgumentList();
										调用fork创建新的进程		
										pid = Zygote.forkAndSpecialize(...) 
									}
								}		
								//创建成功，移除ZygoteConnection
								if (done) {
									peers.remove(i);
									fds.remove(i);
								}
							}
						}
					}
					//0为子进程
					if (pid == 0) { 
						//关闭子进程的socket连接，子进程不需要创建socket
						zygoteServer.closeServerSocket();
						//处理子进程
						handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
							//和注释2一样内部为初始化和创建子进程的线程池和启动子进程
							ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion,parsedArgs.remainingArgs, null /* classLoader */);
								
					}
				}
				catch (Zygote.MethodAndArgsCaller caller) {
					//子进程入口 注释1，zygote进程还是在注释3处等待
					caller.run();
				} 
				
			}
	}
```
zygote进程总结：

- 1.启动java虚拟机
- 2.注册android启动需要的jni方法
- 3.调用com.android.internal.os.ZygoteInit的main方法
	- 3.1:创建一个Server端的socket，socketName=“zygote” 用于接收应用的请求创建应用进程：使用的fork方式
	- 3.2:预加载系统类和系统资源,如果进程是使用fork方式创建的，则这个预加载的类和资源也可以被子进程所用
	- 3.2:使用fork方式创建SystemServer进程，最终会在3.3处接收到socket请求并创建SystemServer进程。并在异常后调用caller.run()运行SystemServer的main方法;
		SystemServer进程中：
		- 3.2.1：创建PathClassLoader
		- 3.2.2：设置主线程未捕获异常时回调的Handler,可以在app中替换
		- 3.2.3：启动binder线程池
		- 3.2.4：使用抛出异常加反射的方式，启动SystemServer的main方法
	- 3.3:在app_main进程中等待AMS创建新的应用程序进程

**下一篇我们会讲解关于SystemServer启动过程源码分析**