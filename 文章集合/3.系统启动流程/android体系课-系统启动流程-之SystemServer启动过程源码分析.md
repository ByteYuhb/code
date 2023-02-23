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
	
- 5.Instrumentation是什么？和ActivityThread是什么关系？

	答：Instrumentation是一个沟通Activity和AMS以及ActivityThread和Activity中间桥梁工具类
	例如：Activity.startActivity
			->mInstrumentation.execStartActivity
			->ActivityManagerService.startActivity->ActivityThread.thread.scheduleLaunchActivity
			->ActivityThread.handleLaunchActivity
			->mInstrumentation.callActivityOnCreate


**针对以上问题：笔者会以源码的方式呈现**

前面一篇文章我们已经介绍过了Android系统启动-zygote进程详解：

[android体系课-系统启动流程-之zygote进程启动过程源码分析](https://juejin.cn/post/7112465356991496200/)

## 下面我们来分析下SystemServer的启动过程。

介绍SystemServer前我们先来分析下系统启动的几个关键字：

- init.cpp
> 		linux系统启动第一个进程：

> 		1.mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");挂载临时文件系统

> 		2.selinux_initialize(true);启动selinux 主要作用就是最大限度地减小系统中服务进程可访问的资源（最小权限原则）

> 		3.InitKernelLogging(argv);初始化内核日志

> 		4.property_init();对属性服务进行初始化

> 		5.signal_handler_init();设置信号处理函数，如果子进程（Zygote）进程异常退出，会调用该函数设置的信号函数进行处理

> 		6.start_property_service();启动属性服务

> 		7.解析init.rc文件


- init.rc

> 		由android初始化语言编写的脚本文件
> 		主要作用是启动Zygote进程：
> 		启动方式：fork子进程，会启动app_main.cpp的main函数

- app_main.cpp
> 		根据参数判断启动哪个进程：
```java
case strcmp(arg, "--zygote") == 0：
runtime.start("com.android.internal.os.ZygoteInit", args, zygote);

case strcmp(arg, "--start-system-server") == 0:
runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
```
> 开机启动阶段参数中含有--zygote
> 启动的是ZygoteInit进程，进入AndroidRuntime。

- AndroidRuntime.cpp

	- 1.startVm(&mJavaVM, &env, zygote)启动java虚拟机
	- 2.startReg(env)注册jni方法：
		register_jni_procs(gRegJNI, NELEM(gRegJNI), env)
	- 3.env->CallStaticVoidMethod(startClass, startMeth, strArray);调用java方法：ZygoteInit的main方法，进入java代码
> 		startClass = com.android.internal.os.ZygoteInit

- ZygoteInit.java
	- 1.zygoteServer.registerServerSocket(socketName);
        
            注册一个ServerSocket,文件名：ANDROID_SOCKET_zygote
        
        ```java
        mServerSocket = new LocalServerSocket(fd);
                                impl = new LocalSocketImpl(fd);
                                impl.listen(LISTEN_BACKLOG);
                                        Os.listen(fd, backlog);
        ```
		
	- 2.startSystemServer 
        
            启动SystemServer进程,这里是SystemServer核心处理代码。
        
```java
//fork出SystemServer进程 fork操作会返回两次，一次是父进程一次是子进程
		pid = Zygote.forkSystemServer() 
		//pid=0说明运行在子线程SystemServer进程
		if(pid ==0){	
			zygoteServer.closeServerSocket();
			//处理SystemServer进程
			handleSystemServerProcess(parsedArgs);
				ZygoteInit.zygoteInit();
					//RuntimeInit初始化
					RuntimeInit.commonInit();
					//创建binder线程池
					ZygoteInit.nativeZygoteInit();
					RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
						invokeStaticMain(args.startClass, args.startArgs, classLoader);
							throw new Zygote.MethodAndArgsCaller(m, argv);
							//会跳转到ZygoteInit的main方法中，class: SystemServer method: main
							catch (Zygote.MethodAndArgsCaller caller) { 
								caller.run();
									main(){
										// Ensure binder calls into the system always run at foreground priority.
										//这个就是为什么在服务保活的时候，在需要保活的服务里面启动另外一个服务可以让该保活的服务在前台，提高其优先级
										BinderInternal.disableBackgroundScheduling(true);
										//设置系统最大的binder驱动线程 sMaxBinderThreads = 31个
										BinderInternal.setMaxThreads(sMaxBinderThreads);
										//创建SystemServer应用的looper
										Looper.prepareMainLooper();
										//加载SystemServer需要的so库
										System.loadLibrary("android_servers");
										//创建SystemServer应用的context
										Looper.prepareMainLooper();
										// Initialize the system context.
										createSystemContext();
											//创建ActivityThread，应用进程方法入口
											ActivityThread activityThread = ActivityThread.systemMain();
												ActivityThread thread = new ActivityThread();
												thread.attach(true);
												if(!isSystem){			
													//从ServiceManager中获取 ActivityManagerService
													final IActivityManager mgr = ActivityManager.getService();  
													//将ActivityThread中的ApplicationThread发给AMS，此后AMS就可以使用该mAppThread调用ActivityThread中的方法。		
													mgr.attachApplication(mAppThread); 
												}else{
													ContextImpl context = ContextImpl.createAppContext(this, getSystemContext().mPackageInfo);
													mInitialApplication = context.mPackageInfo.makeApplication(true, null);
													mInitialApplication.onCreate();
												}
											//创建进程的上下文context
											mSystemContext = activityThread.getSystemContext();
												ContextImpl.createSystemContext(this);
													//创建LoadedApk
													LoadedApk packageInfo = new LoadedApk(mainThread);
													//创建context
													ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0, null);
													//设置context的resources
													context.setResources(packageInfo.getResources());
											//设置系统Theme：com.android.internal.R.style.Theme_DeviceDefault_System
											mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
										//注册过程有addService和startService，前者是将服务注册到ServiceManager中后者是将服务注册到本地SystemServerManager中。
										LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
										//启动引导服务
										startBootstrapServices();
										//启动核心服务
										startCoreServices();
										//启动其他服务
										startOtherServices();
											//启动Launcher
											mActivityManagerService.systemReady(() -> {}
											AMS：systemReady：
														mStackSupervisor.resumeFocusedStackTopActivityLocked();
											ActivityStackSupervisor.java:resumeFocusedStackTopActivityLocked:
															resumeFocusedStackTopActivityLocked(null, null, null);
																//开始的时候r=ActivityRecord为null
																if (r == null || r.state != RESUMED) {
																	mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
											ActivityStack.java:resumeTopActivityUncheckedLocked:
																		result = resumeTopActivityInnerLocked(prev, options);
																			return isOnHomeDisplay() && mStackSupervisor.resumeHomeStackTask(prev, "prevFinished");
											ActivityStackSupervisor.java:resumeHomeStackTask
																				return mService.startHomeActivityLocked(mCurrentUser, myReason);
											AMS:startHomeActivityLocked
																					Intent intent = getHomeIntent();//获取Launcher Intent
																					mActivityStarter.startHomeActivityLocked(intent, aInfo, myReason);
											ActivityStarter.java:startHomeActivityLocked：
																						mSupervisor.moveHomeStackTaskToTop(reason);Returns true if the focus activity was adjusted to the home stack top activity
																						startActivityLocked
																							startActivity//这里启动了Launcher的MainActivity
										//可看出SystemServer进程也使用了Looper消息处理机制
										Looper.loop();
									}									
							} 
		}
	//Zygote进程会继续执行下面的语句。	
	3.zygoteServer.runSelectLoop(abiList);
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
					//返回的事件掩码被设置为POLLIN说明可读，如果不是POLLIN则继续看下一个Fd是否为POLLIN；
					if ((pollFds[i].revents & POLLIN) == 0) {
						continue;
					}
					//此处的pollFds->revents为POLLIN说明可读，i=0说明有客户端做连接操作，服务端则调用accept操作，这样就创建了连接。把连接上的fd放入需要监听的fds中，继续while监听。
					if (i == 0) {
						//新建一个ZygoteConnection
						ZygoteConnection newPeer = acceptCommandPeer(abiList);
							//获取sokcet客户端传过来的数据并封装到ZygoteConnection中。
							LocalSocket localSocket = mServerSocket.accept();
							createNewConnection(localSocket, abiList);
								new ZygoteConnection(socket, abiList);
						peers.add(newPeer);
						fds.add(newPeer.getFileDesciptor());
						
					} else { i!=0
						//不为0则说明签名i=0放入的fd句柄有事件，调用fork方式创建进程
						boolean done = peers.get(i).runOnce(this);
							//读参数
							args = readArgumentList();
							//fork出子进程
							pid = Zygote.forkAndSpecialize(args)；
							if (pid == 0) {
								// pid=0说明在子进程中
								zygoteServer.closeServerSocket();			
								//处理子进程	
								handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);		
									ZygoteInit.zygoteInit();
										//RuntimeInit初始化
										RuntimeInit.commonInit();
										//创建binder线程池
										ZygoteInit.nativeZygoteInit();
										RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
											invokeStaticMain(args.startClass, args.startArgs, classLoader);
												throw new Zygote.MethodAndArgsCaller(m, argv);
												会跳转到ZygoteInit的main方法中，如果是子进程则为ActivityThread类的main方法，或者是SystemServer进程的main方法
												catch (Zygote.MethodAndArgsCaller caller) {
													caller.run();
												} 
								return true;
							} else {
								// in parent...pid of < 0 means failure	
								//处理父进程
								return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
							}
						if (done) {
							//移除连接
							peers.remove(i);
							fds.remove(i);
						}
					}
				}
			}
		PS：这里的为什么是i=0为连接操作呢，因为pollFds的第一个元素的fd字段为socket监听的fd文件。
		而i>0的情况为i=0获取到连接信息时fds.add(newPeer.getFileDesciptor());添加的fd，索引肯定大于0，
		这里巧妙的用了索引位置的关系，节省了一些代码

	3.zygoteServer.closeServerSocket(); 一般只有zygote进程退出的时候才会走到这里，也就是关机的时候才会，前面while循环会一直等待AMS发起创建fork进程的操作	
```


## 总结下SystemServer进程：

- 1.SystemServer创建方式：fork
- 2.SystemServer进程做了哪些事情：
	- 2.1：创建binder线程池
	- 2.2：调用BinderInternal.disableBackgroundScheduling(true)：可以确保需要保活的服务里面启动另外一个服务可以让该保活的服务在前台，提高其优先级
	- 2.3：设置系统最大的binder驱动线程
	- 2.4：加载运行SystemServer需要的so库
	- 2.5：创建SystemServer应用的上下文context
	- 2.6：启动引导服务，启动核心服务，启动其他服务等系统服务
	- 2.7：在启动其他服务中，启动launcher
	- 2.8：SystemServer也是要了Looper机制，只是这个是系统的应用
	

关于SystemServer的运行就讲到这里。
下一篇我们会讲解关于startActivity源码方面的机制：**根Activity启动和普通Activity启动方式**