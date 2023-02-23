> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
### 1.两种启动方式的区别：
    - 生命周期：
    start： onCreate->onStartCommand->onDestroy
    bind: onCreate->onBind->onUnbind->onDestroy
    
    -使用场景：
    start：一般启动的服务是独立运行的，不依赖调用者，调用者也不关心服务启动成功或者失败
    bind：服务是依赖客户端调用的，服务为客户端提供接口服务，通过ipc机制通讯实现远程调用

### 2.service的启动流程

    A进程调用`startService`或者`bindService`，通过`AMS`的代理类调用到`AMS`中的启动服务的方法，再调用`realStartService`之前会判断进程是否启动，
    如果应用进程未启动，则先去请求孵化`zygote`进程创建进程服务B，创建成功后，发送请求给`AMS`，`AMS`收到应用创建成功的请求，重新调用`realStartService方法`
    最终回调到应用进程的`ApplicationThread`的创建服务`application`并启动服务生命周期的过程

`下面我们从源码层面来分析下服务启动过程：`

`我们这里使用更经典的bindService`

```java
Activity.bindService
	ContextWrapper.bindService
		ContextImpl.bindService
			bindServiceCommon(service, conn, flags..)
				sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
					sd = new ServiceDispatcher(c, context, handler, flags);{创建ServiceDispatcher服务分发，其实是对ServiceConnection的包装
						mIServiceConnection = new InnerConnection(this);{
							mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);包装为弱引用
							
							public void connected(ComponentName name, IBinder service, boolean dead)连接上后会调用这个方法
								throws RemoteException {
								LoadedApk.ServiceDispatcher sd = mDispatcher.get();
								if (sd != null) {
									sd.connected(name, service, dead);
										doConnected(name, service, dead);
											mConnection.onServiceConnected(name, service);调用ServiceConnect的onServiceConnected方法
								}
							}
						}
						mConnection = conn;
						mContext = context;
						mActivityThread = activityThread;
						mLocation = new ServiceConnectionLeaked(null);
						mLocation.fillInStackTrace();
						mFlags = flags;
					}
				IBinder token = getActivityToken()获取ActivityToken
				ActivityManager.getService().bindService(...)调用AMS的bindService操作
					mServices.bindServiceLocked(caller, token, service,...)
						final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);根据caller获取进程记录
						activity = ActivityRecord.isInStackLocked(token);判断ActivityRecord是否在TaskRecord
						final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;判断是否是系统进程
						ServiceLookupResult res = retrieveServiceLocked(service, resolvedType...)获取ServiceRecord信息
							ServiceMap smap = getServiceMapLocked(userId);
								ServiceMap smap = mServiceMap.get(callingUser);获取当前应用的ServiceMap
							final ComponentName comp = service.getComponent();
							r = smap.mServicesByName.get(comp);获取当前应用的ServiceRecord看下是否之前有注册过
							if (r == null) {
								r = new ServiceRecord(mAm, ss, name, filter, sInfo, callingFromFg, res);
								smap.mServicesByName.put(name, r);将ServiceRecord放入ServiceMap中，下次getService就可以直接获取不需要重建 = 在AMS中注册了该Service
								smap.mServicesByIntent.put(filter, r);
							}
							return new ServiceLookupResult(r, null);里面包含了该服务的ServiceRecord信息
						realStartServiceLocked(r, app, execInFg);
							app.thread.scheduleCreateService(r, r.serviceInfo,..)	调用ActivityThread的thread启动service
								handleCreateService((CreateServiceData)msg.obj);
									ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
									context.setOuterContext(service);

									Application app = packageInfo.makeApplication(false, mInstrumentation);创建Application
									service.attach(context, this, data.info.name, data.token, app,
											ActivityManager.getService());
									service.onCreate();调用服务的onCreate方法
									mServices.put(data.token, service);
							
							requestServiceBindingsLocked(r, execInFg);
								r.app.thread.scheduleBindService(r, i.intent.getIntent()..)调用ActivityThread的thread的onBind
									handleBindService((BindServiceData)msg.obj);
										IBinder binder = s.onBind(data.intent);调用服务的onBind
										ActivityManager.getService().publishService();通知应用服务bind成功了，并把绑定对象返回BinderProxy
											mServices.publishServiceLocked((ServiceRecord)token, intent, service);
												c.conn.connected(r.name, service, false);c.conn = mIServiceConnection 这里调用的是InnerConnection中的connected方法
													
											
							updateServiceClientActivitiesLocked(app, null, true);
```
### 3.Service与Activity怎么实现通信
`通过binder机制通讯:`
> Activity在服务绑定成功后，会返回一个服务端的代理类，通过这个代理类，内部实现ipc功能，实现远程调用；
> 	具体是AIDL通讯过程：

		InterfaceXXX xxx = InterfaceXXX.Stub.asInterface(obj)这个obj = BinderProxy(BpBinder(handle))
		xxx在使用过程中会调用transact方法，将需要发送的数据放到Parcel中，实际调用的是BinderProxy = transact
		最后会调用到BpBinder的BpBinder->transact方法，IPCThreadState::self()->transact,最终调用到驱动中，驱动通过传入的handle找到对应的binder_ref，通过binder_ref找到binder_node，
		通过binder_node找到binder_proc，并将数据放到binder_proc下面的线程池里面的某个线程的todo链表中，并唤醒线程，服务端在接收到数据之后，解析数据，根据code调用对应的服务实例的方法

### 4.IntentService是什么,IntentService原理，应用场景及其与Service的区别
- 1.**介绍**：
主要实现在服务中使用子线程处理`intent`请求，如一些`耗时操作`,`后台下载任务`或者`静默上传`等场景
- 2.**使用方式**：
    - 1.创建一个类继承`IntentService`
    
        ```java
        public void MyIntentService extends IntentService{}
        ```
    - 2.实现父类`IntentService`的`onHandleIntent`方法，可以处理客户端传入的`intent`请求
    
        ```java
        public void onHandleIntent(intent){}
        ```
    - 3.客户端调用`startService`方法传入Intent
- 3.原理源码解析
`IntentService.java`

```java
@Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
		//1.创建HandlerThread
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        //2.启动这个HandlerThread
		3.thread.start();
		//获取HandlerThread线程的looper
        4.mServiceLooper = thread.getLooper();
		5.新建一个Handler并传入之前创建的HandlerThread对象的looper
        //mServiceHandler = new ServiceHandler(mServiceLooper);
    }
	
	private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
	
	@Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
	
	@Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
	
	@Override
    public void onDestroy() {
        mServiceLooper.quit();
    }
```
`通过上面的源码分析可知`
> - 1.`IntentService`是`Service`的子类
> - 2.在`onCreate`方法中默认开启了一个`HandleThread`，使用这个工作线程一一处理所以启动请求，请求处理完毕后自动停止服务。
> - 3.只要实现`onHandleIntent`方法，因为是子线程，可以在这个方法内部做一些耗时的任务。
> - 4.可以启动多次`IntentService`，每次请求都会以队列的方式在`onHandleIntent`方法中执行，所有请求执行完毕之后，才会终止服务

### 5.Service 的 onStartCommand 方法有几种返回值?各代表什么意思?
- `START_NOT_STICKY`
在执行完 `onStartCommand` 后,服务被异常 `kill` 掉,系统不会自动重启该服务
- `START_STICKY`
	重传 `Intent`。使用这个返回值时,如果在执行完 `onStartCommand` 后,服务被异 常 `kill` 掉,系统会自动重启该服务 ，并
	且`onStartCommand`方法会执行,`onStartCommand`方法中的`intent`值为`null`。适用于`媒体播放器`或类似服务
- `START_REDELIVER_INTEN`
	使用这个返回值时,服务被异 常 `kill` 掉,系统会自动重启该服务,并将 `Intent` 的值传入。
	适用于`主动执行应该立即恢复的作业（例如下载文件）的服务`。

### 6.进程保活的方式有哪几种？ adj 提高进程保留优先级方式
##### 1.进程划分：

     1.前台进程：处于前台的进程，用户正在使用的程序，一般不会被杀死，前台类别：
    
            1.1：Activity处于resume状态的进程
    	1.2：某个进程持有一个Service，且该Service正在与用户进程交互的Activity进程绑定
    	1.3：某个进程持有一个Service，并且该Service调用startForeground()方法使之位于前台运行
    	1.4：某个进程持有一个Service，并且该Service正在执行它的某个生命周期回调方法，比如onCreate()、 onStart()或onDestroy()
    	1.5：某个进程持有一个BroadcastReceiver，并且该BroadcastReceiver正在执行其onReceive()方法。
     
     2.可见进程
         2.1: 最顶层Activity处于onPause状态的进程
         2.2：拥有绑定到可见（或前台）Activity 的 Service
     3.服务进程
         3.1：在后台运行的服务进程
     4.后台进程进程是根据什么来杀进程的呢？

出于性能和用户体验考虑，app在退出到后台时，不会直接杀死进程，而是将其缓存起来，
		当缓存到一定值的时候就会出现内存不足的情况时，最后会根据内存阈值来杀死对应的进程。
		可以根据：3.1：泛指看不见的一切后台进程
     5.空进程
         5.1：只是一个壳，不包含任何组件，唯一作用是当系统有需要创建进程时，可以直接从空进程转过来。
      
##### 2.内存阈值
`进程是根据什么来杀进程的呢？`

出于性能和用户体验考虑，`app`在退出到后台时，不会直接杀死进程，而是将其缓存起来，当缓存到一定值的时候就会出现`内存不足`的情况时，最后会根据`内存阈值`来杀死对应的进程。

`可以使用`：下面的命令来查看某个手机的内存阈值

```java
cat /sys/module/lowmemorykiller/parameters/minfree
结果：18432,23040,27648,32256,36864,46080
```
> 注意这些数字的单位是page. 1 page = 4 kb.上面的六个数字对应的就是(MB): 72,90,108,126,144,180
> 	如数180代表内存低于180M时会清除优先级最低的空进程。

##### 3.adj优先级值
`获取adj`

```java
adb shell ps|grep com.android.yuhb.test
adb shell cat /proc/21375/oom_adj

> 每个等级的进程又有对应的优先级，使用adj值来表示，进程回收机制就是根据这个adj值来进行的
前台进程adj值最低，代表进程优先级最高，
空进程adj值越高，最容易被kill
对于相等优先级的进程：使用的内存越多越容易被杀死
```
##### 4.进程保留方案：
> 通过上面的分析，进程保活其实就是提高adj进程优先级
- `方案1`:启动一个像素的`Activity`保活
`具体`：
> 监听屏幕亮屏和暗屏广播，当接收到暗屏时，启动一个透明的一个像素的Activity，在亮屏时，关闭这个Activity
> Demo地址：https://github.com/ByteYuhb/onePxDemo
- `方案2`：调用`Service`的`startForeGround`
`具体`:

1.`api<18` :调用`startForeground`(NOTIFICATION_ID, new Notification());
2.`api>=18`:启用一个通知并拉起另外一个服务，另外服务启用通知，并关闭通知，关闭服务，也可以`提高`进程的`优先级`
`伪代码如下：`

```java
2.1调用startForeground(NOTIFICATION_ID, builder.build());
			  2.2并启动一个服务InnerService，在InnerService中startForeground(NOTIFICATION_ID, builder.build())，这个id要和需要保活的服务通知id一致
			  2.3InnerService调用stopForeground(true)和manager.cancel(NOTIFICATION_ID);stopSelf();取消服务
			  
			    Notification.Builder builder = new Notification.Builder(this);
				builder.setSmallIcon(R.mipmap.ic_launcher);
				startForeground(NOTIFICATION_ID, builder.build());
				startService(new Intent(this, InnerService.class));
			
				class InnerService
					startForeground(NOTIFICATION_ID, builder.build()); 通需要保活的NOTIFICATION_ID
					stopForeground(true);
					NotificationManager manager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
					manager.cancel(NOTIFICATION_ID);
					stopSelf();
```
- `方案3`：相互唤醒，很多互联网应用都使用的这种方式
`A-B：A死了B收到信息唤醒A `

`B-A：B死了A收到信息唤醒B`
- `方案4`：`JobSheduler`，可以根据间隔时间启动服务
- `方案5`：粘性服务&与系统服务捆绑
  根据`onStartService`的返回值:

    **START_STICKY**
    如果系统在onStartCommand返回后被销毁，系统将会重新创建服务并依次调用onCreate和onStartCommand
    **START_NOT_STICKY**
    如果系统在onStartCommand返回后被销毁，如果返回该值，则在执行完onStartCommand方法后如果Service被杀掉系统将不会重启该服务。
    **START_REDELIVER_INTENT**
    START_STICKY的兼容版本，不同的是其不保证服务被杀后一定能重启
  
- `方案6`：使用`NotificationListenerService`监听通知栏信息,监听到信息就重新拉起需要启动的服务
> 只要手机收到了通知，NotificationListenerService都能监听到,即使用户把进程杀死，也能重启，所以说要是把这个服务放到我们的进程之中
> `使用方式：`

```java
1.创建一个继承NotificationListenerService的类
    public class LiveService extends NotificationListenerService {}
2.在manfest中注册服务并添加权限BIND_NOTIFICATION_LISTENER_SERVICE

<service android:name=".service.notificationlistenerservice.LiveService"
    android:label="@string/app_name"
    android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE"
    >
    <intent-filter>
        <action android:name="android.service.notification.NotificationListenerService" />
    </intent-filter>
</service>
3.早LiveService中重写下面这三个方法
    onNotificationPosted(StatusBarNotification sbn) ：当有新通知到来时会回调；比如监听红包等信息
    onNotificationRemoved(StatusBarNotification sbn) ：当有通知移除时会回调；
    onListenerConnected() ：当 NotificationListenerService 是可用的并且和通知管理器连接成功时回调。
4.取消通知方法：
    public void cancelNotification(StatusBarNotification sbn) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                cancelNotification(sbn.getKey());
        } else {
                cancelNotification(sbn.getPackageName(), sbn.getTag(), sbn.getId());
        }
    }
5.检测通知监听服务是否被授权
    public boolean isNotificationListenerEnabled(Context context) {
        Set<String> packageNames = NotificationManagerCompat.getEnabledListenerPackages(this);
        if (packageNames.contains(context.getPackageName())) {
                return true;
        }
        return false;
    }
6.打开通知监听设置页面
    public void openNotificationListenSettings() {
            try {
                    Intent intent;
                    if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.LOLLIPOP_MR1) {
                            intent = new Intent(Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS);
                    } else {
                            intent = new Intent("android.settings.ACTION_NOTIFICATION_LISTENER_SETTINGS");
                    }
                    startActivity(intent);
            } catch (Exception e) {
                    e.printStackTrace();
            }
    }
```

### 总结：
关于Service我们上面介绍了其`使用方式`,`IntentService`,`Service 的 onStartCommand 方法有几种返回值`,`进程优先级以及保活的几种方式`，后续会推出更多`Android`相关知识，喜欢的就留个脚印吧。