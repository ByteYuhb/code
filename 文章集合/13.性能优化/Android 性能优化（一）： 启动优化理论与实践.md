> 🔥 **Hi，我是小余。本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
本文章总结了目前市面上常见的一些启动优化常用手段，**开发和面试必备哦**。

首先要做应用启动优化，**你得对应用启动流程有个整体甚至细化的了解**。
## 1.应用启动全路径分析
应用启动过程整体分为两大阶段：**Application启动 阶段**、**Activity 启动阶段**
### Application启动阶段
- 1.点击桌面应用图标这个时候会触发`Launcher` app的item事件，AMS首先会判断当前应用进程`ProcessRecord`是否存在，不存在，则会请求`zygote`进程去创建对应的`app进程`，app进程由zygote孵化出来后，首先会执行`ActivityThread`的`main`方法，**这里可以看成是单个进程的入口方法**，和java中的main方法一样。
- 2.在main方法中，会**创建消息循环和主线程Handler**，接着会调用`AMS`的`attachApplication`并传入当前应用的binder对象，用于AMS和当前应用进程交互。
- 3.AMS的attachApplication中会创建一个ProcessRecord用于记录当前进程状态。并回调`thread.bindApplication`的方法，将当前进程状态信息传递给应用进程。
- 4.应用进程在受到AMS的bindApplication方法后，将进程信息存储在一个`AppBindData`对象中，并使用Handler传递，最终会执行到`ActivityThread`的`handleBindApplication`。**这个时间点就可以看做是应用进程启动开始时间**
- 5.在`handleBindApplication`中调用 `data.info.makeApplication`()，这个方法内部会创建应用context且使用反射创建Application，并依次调用Application的`attach`，attach方法会调用`attachBaseContext`方法，**这也是应用的最早加载时机**。
- 6.在handleBindApplication中`makeApplication`创建application后，会继续执行`installContentProviders`，这个方法内部执行`installProvider`，installProvider方法中会使用反射创建进程中的`ContentProvider`，也就是清单中的ContentProvider。并回调ContentProvider对象的`attachInfo`，在attachInfo中会将清单中的属性赋值给当前ContentProvider对象。并调用ContentProvider的`onCreate`方法**。很多三方sdk在这里进行一些初始化操作，可能导致启动耗时的不可控，需要按具体case优化。**
- 7.最后执行application的`onCreate`方法。**这里是很多三方库和业务初始化之处，也是启动优化最主要优化点，可通过异步，按需，预加载等方式进行优化。**

**看Application启动图**：
![Activity启动流程1.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e901d88c19344dc78ed25a16fceb9b38~tplv-k3u1fbpfcp-watermark.image?)

这里我们提取应用可触及点：**按时间顺序**
- 1.**Application的`attachBaseContext`方法**
- 2.**ContentProvider的`attachInfo以及onCreate`方法**
- 3.**Application的`onCreate`方法**


### Activity启动阶段
- 1.回到Application步骤分析3：AMS回调`thread.bindApplication`的方法后，**在bindApplication的方法执行完成后**，会继续回调`mStackSupervisor.attachApplicationLocked(app)`这个方法中：获取当前进程的第一个非LauncherActivity，然后调用`realStartActivityLocked`去**启动根Activity**。
- 2.然后就是**创建Activity，并执行Activity生命周期**，**OnCreate方法是首屏业务优化的主要场景也是开启并发的主要时机，其中会执行**`setContentView`，这里会触发`DecorView` 的 `install`，去**解析xml数据，并转换为View**。这里是一个耗时操作。**可采用异步 Inflate 配合 X2C（编译期将 xml 布局转代码）并提升相应异步线程优先级的方法综合优化**	
- 3.最后就是View的渲染操作，包括View的**三大measure，layout，draw**。`可尝试从层级，布局，渲染上取得优化收益`。

**看Activity启动完整图：**

![Activity启动流程.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a1efc75d20447ceae3323abdb400d30~tplv-k3u1fbpfcp-watermark.image?)

同样我们提取应用Activity可触及点：按时间顺序
- 1.setContentView
- 2.view的三大步骤以及渲染过程  

总结下我们应用可优化路径。
![启动优化全路径.awebp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2628941351564ee49b972f82f4358289~tplv-k3u1fbpfcp-watermark.image?)
这里经过对应用启动全流程分析，提取出了应用的一些可触及点，这些可触及点就是我们启动优化过程**中应用可优化部分**。
## 2.应用启动耗时归因
启动耗时主要是由以下几方面组成：
- 1.`CPU time`：指不合理的占用CPU时间片。 
    举例：
    - 1.**一个低效的遍历算法**，可通过使用更加高效的算法如空间换时间的方式。
    - 2.**类加载过程**：可通过类重排优化
    - 3.**反射**：反射也是一个相对耗时过程
- 2.`CPU Scheduler`：主线程获取不到足够的Cpu时间片，这种情况比较少见，毕竟主线程优先级也比较高。有个特殊情况：就是渲染，渲染是需要 RenderThread 提交 GPU 的渲染命令，而 RenderThread 并没有主线程那么高的优先级，因此比较容易受 CPU 的负载的影响，导致渲染耗时
		这种情况还是需要考虑降低Cpu负载进行优化或者提升RenderThread的优先级
- 3.`IO Wait`：读取资源和文件，类加载等过程产生的IO耗时问题。
    - **对于资源读取：可以使用预加载，资源重排，资源异步加载的方案进行优化**
    - **对于类加载：使用预加载，类重排的方案。**
- 4.`Lock Wait`: 也是主要针对主线程，指其处于等锁状态。
    一般方案：
  - **方案1：加快业务锁执行过程**
  - **方案2：移除主线程的锁**。
- 5.`binder多进程`：进程间通讯也是一个耗时过程，非必要条件下尽量少使用多进程

这里使用View的构建和渲染过程为例来讲解下应用启动耗时归因步骤：

![启动耗时分析.awebp](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/76f8a23c72624502878133aca02619bd~tplv-k3u1fbpfcp-watermark.image?)
**UI构建阶段**：

- 1.读取xml文件信息，存在IO Wait.
- 2.根据TAG去解析xml中的节点，存在循环嵌套递归.
- 3.根据class的name使用反射创建view实例最后生成View树，存在反射
  

**数据绑定阶段**：
- 1.对数据进行请求，解析，适配，这部分涉及到网络IO Wait，如使用JSON解析成Data Class对象，像Retrofit的使用过程，还会涉及到反射以及循环嵌套的数据类等，会增加CPU time
- 更新UI


**View的显示**：
- 1.最常见的就是View的绘制三大步骤measure,layout,draw。这三大步骤会存在xml布局文件的文件树遍历，存在一定Cpu Time耗时。
- 2.使用RenderThread线程将1中绘制好的数据提供给GPU去渲染，这里涉及到进程间通讯。

## 3.应用启动耗时现状分析
使用Profile 工具对应用启动过程打点分析：
工具：**TraceView、Systrace、Android Profiler ，抖音Rhea**
### Systrace
用来记录当前应用的系统以及应用(使用Trace类打点)的各阶段耗时信息包括绘制信息。

使用方式：
```java
Trace.beginSection("MyApp.onCreate_1");
alt(200);
Trace.endSection();
```

![systrace截图.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e79a4bb837cb4f8d969bda3e655f193b~tplv-k3u1fbpfcp-watermark.image?)

### TraceView
用来记录当前当前应用的方法耗时路径，可以选取开始和结束位置，只在Debug状态下有效，需要使用Debug类进行打点记录。
### Android Profiler
studio自带的性能分析利器。不仅可以分析当前应用的CPU使用率，还可以记录当前应用的内存使用方式。
可以直接替代TraceView记录方法耗时信息。

使用方式：

```java
Debug.startMethodTracing();
back(100);
alt(200);
Debug.stopMethodTracing();
```

![androidprofile.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/23bb3df6fd644f0f9f6d5c64dc55467c~tplv-k3u1fbpfcp-watermark.image?)

### Rhea
字节自研的新一代全能型性能分析工具，功能强大，且性能损耗低。缺点就是还不支持windows获取html文件。
地址：https://juejin.cn/post/7052625610295738382

![打点工具对比.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a4a8289cf164c9dae90e18c5a01cc0c~tplv-k3u1fbpfcp-watermark.image?)
## 4.应用启动常见耗时优化
优化过程主要分为：主线程直接优化、后台线程间接优化、全局优化
### 4.1：主线程直接优化
按应用启动生命周期优化方案
#### 4.1.1：4.x机型存在MutilDex问题，
4.x低版本机型中 Dalvik 虚拟机只能执行经过优化后的 odex 文件，而4.x设备为了加快安装时间，对于分包多dex的情况下，
安装时只会优化第一个dex文件为odex，这就导致其他子包需要在Application的attachBaseContext方法中调用MutilDex.install来优化剩余的dex文件成为odex。
这是个相当耗时的过程。

**优化方案**：[抖音BoostMultiDex优化实践,Android低版本上APP首次启动时间减少80%](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247485522&idx=1&sn=cddfb1c64642b53ee51ca00ce3c696ca&scene=21#wechat_redirect)

**优化步骤**：
- 1.首先从 APK 中解压获取原始的非首个 dex 文件的字节码
- 2.调用 **Dalvik_dalvik_system_DexFile_openDexFile_bytearray**，逐个传入之前从 APK 获取的 DEX 字节码，完成 DEX 加载，得到合法的 DexFile 对象；
- 3.将 DexFile 都添加到 APP 的 PathClassLoader 的 DexPathList 里
- 4.延后异步对非首个 dex 进行 odex 优化。

`原理`：**在启动阶段绕过dex转odex，直接让Dalvik 虚拟机加载未经优化的dex文件，然后在后台将dex优化为odex文件。需要对java虚拟机有一定的认知**。
#### 4.1.2：ContentProvider优化
前面在启动流程分析中我们说到：Application的`attachBaseContext`方法执行完后，会执行`installProvider`方法，并最终执行当前应用清单中声明的所有`ContentProvider`的`onCreate`方法。
这里会有一些第三方库的初始化会放到这里面。
**如果是我们自己的`ContentProvider`可以通过逻辑优化来降低耗时，如果是第三方库的初始化，则可以考虑使用下面的方式进行优化。**

我们以`FileProvider`优化为例：
> `FileProvider` 是 `Android7.0` 引入的用于进行文件访问权限控制的组件：

`7.0前`我们访问`uri`方式直接通过：

```java
Uri.fromFile(new File(filePath));
```
`7.0之后`我们使用:

```java
FileProvider.getUriForFile(context, context.getPackageName() + ".fileprovider",new File(filePath));
```
`FileProvider`继承`ContentProvider`，按前面分析我们来看他的`attachInfo`方法：

```java
public void attachInfo(@NonNull Context context, @NonNull ProviderInfo info) {
				super.attachInfo(context, info);

    // Check our security attributes
    if (info.exported) {
            throw new SecurityException("Provider must not be exported");
    }
    if (!info.grantUriPermissions) {
            throw new SecurityException("Provider must grant uri permissions");
    }

    mStrategy = getPathStrategy(context, info.authority.split(";")[0]);
}
进入getPathStrategy：
private static PathStrategy getPathStrategy(Context context, String authority) {
    PathStrategy strat;
    synchronized (sCache) {
            strat = sCache.get(authority);
            if (strat == null) {
                    try {
                            strat = parsePathStrategy(context, authority);
                    } catch (IOException e) {
                            throw new IllegalArgumentException(
                                            "Failed to parse " + META_DATA_FILE_PROVIDER_PATHS + " meta-data", e);
                    } catch (XmlPullParserException e) {
                            throw new IllegalArgumentException(
                                            "Failed to parse " + META_DATA_FILE_PROVIDER_PATHS + " meta-data", e);
                    }
                    sCache.put(authority, strat);
            }
    }
    return strat;
}
```
大部分耗时方法就集中在`parsePathStrategy`方法中，因为要**去解析xml文件路径和文件读写操作，会有一定IO和CPU时间片的消耗**.

![FileProvider耗时分析.awebp](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8866f9e2a0f4999be7f446268dd1254~tplv-k3u1fbpfcp-watermark.image?)
为了优化这部分，我们可以使用`字节码插桩`的方式：
- **1.在attachInfo执行getPathStrategy前，插入info.grantUriPermissions为false，并在外部捕获这个异常，这样就会抛出SecurityException而不会执行下面的getPathStrategy方法。**
- 2.**对 FileProvider 的 query、getType、openFile 等方法进行插桩，在调用原方法之前首先进行 getPathStrategy 的初始化，完成初始化之后再调用原始实现。**

#### 4.1.3：Application 的 onCreate 阶段优化
Application 的 onCreate 其实是整个**app优化的大头**，大部分的第三方初始化工作以及自身业务相关的启动任务都会集中在这里面。

**任务重构**：任务重构其实就是对任务的一个删减和重排的操作
我们基于以下原则：

    - 1.Application 中的任务应当是全局核心任务，就是一定要这个时候执行的
    - 2.Application 创建时应当尽量减少网络请求操作，网络请求会调用IO线程执行下载，会占用较多CPU时间片信息。
    - 3.Application 创建时不允许有强业务相关的任务
    - 4.Application 创建时尽量减少有 Json 解析处理和 IO 操作的工作

通过上面几个原则，可以将应用中大部分启动任务删除或者分到异步任务中。
最后任务主要分为：基础库初始化任务，功能配置任务和全局配置任务
- **基础库初始化任务**：主要是对网络库，日志库等基础库进行初始化配置，我们最终目标是在启动阶段删除这些任务。首先这些任务会有些耗时，删除了又可能会影响功能的稳定性。

    **主要优化方式**：对任务进行原子化改造，对于需要向sdk中注入context,callback等各类参数的实现，改为按需调用。
    - 1.对自己写的代码，如果需要设置context，callback等，可以在需要的时候，判断参数是否存在，然后去应用层获取，最后保存在内存中，方便下次使用
    - 2.对第三方sdk中的context，callback需求，可以对三方sdk再封装的方式，然后使用1中的方式进行处理，达到按需获取。    
- **功能配置任务**：主要是对一些全局相关的业务功能的前置配置，例如对首页业务数据缓存的预加载等，移除它们会造成业务有损。

    这里可以使用业务降级或者业务打散的方式进行处理，如一个网络请求请求包含一些基础数据，还包含一些图片数据，可以将这个网络请求分成两个接口，优先去获取
						基础数据，再去获取图片数据，图片数据又可以分为前台可见图片和后台不可见图片，这些都可以排列先后顺序进行处理，尽量让首页优先展示。
	
- **全局配置任务**：主要是对于全局 UI 配置，文件路径的处理操作，它们占比少，耗时少，是首页创建的前置任务，优化方面可以暂不处理

**任务排布**：任务重构好后就是对任务的排布：其核心就是处理好任务的前后依赖问题，这就要求开发者对业务有较强的理解，由于每个应用的情况都不一致，这里就不举例了。

**这里提几个关于Application的onCreate阶段启动任务的优化方案：**
##### 1.线程优化基础方案：
- 1.严禁使用new Thread的方式创建对象，这种方式创建的对象，如果线程一直没处理完或者处理缓慢，最直观的感受就是界面卡顿。推荐使用线程池的方式处理
- 2.提供基础线程池供各个业务使用，不要让各个业务维护各自的线程池，防止线程过多，占用过多CPU时间。
- 3.根据任务类型选择合适的异步方式：优先级低，长时间执行，HandlerThread；定时执行耗时任务，线程池
- 4.创建线程必须命名，以方便定位线程归属，在运行期 Thread.currentThread().setName 修改名字。
- 5.关键异步任务监控，注意异步不等于不耗时，建议使用AOP的方式来做监控。
- 6.重视优先级设置（根据任务具体情况），Process.setThreadPriority() 可以设置多次。
##### 2.线程收敛
由于项目需求越来越多，各业务层，sdk层都会使用到多个线程，为了避免线程膨胀，过多的线程抢占CPU，甚至会导致主线程卡顿，影响用户体验。需要进行线程收敛，那么收敛第一步就是要定位线程归属。
- 1.**线程锁定**：找Hook点：构造函数或者特定方法，如Thread的构造函数。
这里我们直接使用维数的 [epic](https://github.com/tiann/epic) 对Thread进行Hook。在`attachBaseContext`中调用`DexposedBridge.hookAllConstructors`方法即可，如下所示：

```java
DexposedBridge.hookAllConstructors(Thread.class, new XC_MethodHook() { 
    @Override protected void afterHookedMethod（MethodHookParam param）throws Throwable {                         
        super.afterHookedMethod(param); 
        Thread thread = (Thread) param.thisObject; 
        LogUtils.i("stack " + Log.getStackTraceString(new Throwable());
    }
);
```
从log找到线程创建信息，根据堆栈信息跟相关业务方沟通解决方案

- 2.**线程收敛**：根据线程创建堆栈考量合理性，使用统一线程库
那么如何统一线程库？

**统一线程库时区分任务类型**：
- **IO密集型任务**：IO密集型任务不消耗CPU，核心池可以很大。常见的IO密集型任务如文件读取、写入，网络请求等等。
- **CPU密集型任务**：核心池大小和CPU核心数相关。常见的CPU密集型任务如比较复杂的计算操作，此时需要使用大量的CPU计算单元。

统一线程库代码实现：

```java
public class DispatcherExecutor {

    /**
     * CPU 密集型任务的线程池
     */
    private static ThreadPoolExecutor sCPUThreadPoolExecutor;

    /**
     * IO 密集型任务的线程池
     */
    private static ExecutorService sIOThreadPoolExecutor;

    /**
     * 当前设备可以使用的 CPU 核数
     */
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();

    /**
     * 线程池核心线程数，其数量在2 ~ 5这个区域内
     */
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 5));

    /**
     * 线程池线程数的最大值：这里指定为了核心线程数的大小
     */
    private static final int MAXIMUM_POOL_SIZE = CORE_POOL_SIZE;

    /**
    * 线程池中空闲线程等待工作的超时时间，当线程池中
    * 线程数量大于corePoolSize（核心线程数量）或
    * 设置了allowCoreThreadTimeOut（是否允许空闲核心线程超时）时，
    * 线程会根据keepAliveTime的值进行活性检查，一旦超时便销毁线程。
    * 否则，线程会永远等待新的工作。
    */
    private static final int KEEP_ALIVE_SECONDS = 5;

    /**
    * 创建一个基于链表节点的阻塞队列
    */
    private static final BlockingQueue<Runnable> S_POOL_WORK_QUEUE = new LinkedBlockingQueue<>();

    /**
     * 用于创建线程的线程工厂
     */
    private static final DefaultThreadFactory S_THREAD_FACTORY = new DefaultThreadFactory();

    /**
     * 线程池执行耗时任务时发生异常所需要做的拒绝执行处理
     * 注意：一般不会执行到这里
     */
    private static final RejectedExecutionHandler S_HANDLER = new RejectedExecutionHandler() {
        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                Executors.newCachedThreadPool().execute(r);
        }
    };

    /**
     * 获取CPU线程池
     *
     * @return CPU线程池
     */
    public static ThreadPoolExecutor getCPUExecutor() {
        return sCPUThreadPoolExecutor;
    }

    /**
     * 获取IO线程池
     *
     * @return IO线程池
     */
    public static ExecutorService getIOExecutor() {
        return sIOThreadPoolExecutor;
    }

    /**
     * 实现一个默认的线程工厂
     */
    private static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger POOL_NUMBER = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                            Thread.currentThread().getThreadGroup();
            namePrefix = "TaskDispatcherPool-" +
                            POOL_NUMBER.getAndIncrement() +
                            "-Thread-";
        }

        @Override
        public Thread newThread(Runnable r) {
            // 每一个新创建的线程都会分配到线程组group当中
            Thread t = new Thread(group, r,
                            namePrefix + threadNumber.getAndIncrement(),
                            0);
            if (t.isDaemon()) {
                    // 非守护线程
                    t.setDaemon(false);
            }
            // 设置线程优先级
            if (t.getPriority() != Thread.NORM_PRIORITY) {
                    t.setPriority(Thread.NORM_PRIORITY);
            }
            return t;
        }
    }

    static {
        sCPUThreadPoolExecutor = new ThreadPoolExecutor(
                        CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                        S_POOL_WORK_QUEUE, S_THREAD_FACTORY, S_HANDLER);
        // 设置是否允许空闲核心线程超时时，线程会根据keepAliveTime的值进行活性检查，一旦超时便销毁线程。否则，线程会永远等待新的工作。
        sCPUThreadPoolExecutor.allowCoreThreadTimeOut(true);
        // IO密集型任务线程池直接采用CachedThreadPool来实现，
        // 它最多可以分配Integer.MAX_VALUE个非核心线程用来执行任务
        sIOThreadPoolExecutor = Executors.newCachedThreadPool(S_THREAD_FACTORY);
    }

}
```
线程库使用方式：

```java
// 如果当前执行的任务是CPU密集型任务，则从基础线程池组件
// DispatcherExecutor中获取到用于执行 CPU 密集型任务的线程池
DispatcherExecutor.getCPUExecutor().execute(YourRunable());

// 如果当前执行的任务是IO密集型任务，则从基础线程池组件
// DispatcherExecutor中获取到用于执行 IO 密集型任务的线程池
DispatcherExecutor.getIOExecutor().execute(YourRunable());
```
##### 3.异步启动器：
`启动器核心思想`：**充分利用CPU多核，自动梳理任务顺序**

![启动器.awebp](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd308c810bd1409ab0b5677dc8d0e8f0~tplv-k3u1fbpfcp-watermark.image?)
`启动器流程`：
启动器的主题流程主要分为**主线程与并发**两个区域块。而 `head task`与`tail task` 仅仅是用于处理启动前/启动后的一些通用任务，例如我们可以在head task中做一些获取通用信息的操作，在tail task可以做一些log输出、数据上报等操作。
- 1、**任务Task化**，每个任务都对应一个Task。
- 2、根据任务属性以及前后依赖关系将任务排列为一个**有向无环图**。
- 3、根据有向无环图的任务先后顺序**依次执行Task**。

**[异步初始化基础代码](https://github.com/zeshaoaaa/AppStarter)**：

```java
// 1.初始化任务分发器
TaskDispatcher.init(this)
// 2.添加并开始执行初始化任务
val dispatcher = TaskDispatcher.createInstance()
//3.给启动器添加任务
dispatcher
    .addTask(InitAMapTask())    // 高德 SDK 初始化任务
    .addTask(InitStethoTask())  // Stetho 初始化任务
    .addTask(InitWeexTask())    // weex 初始化任务
    .addTask(InitBuglyTask())   // Bugly 初始化任务
    .addTask(InitFrescoTask())  // Frescode 初始化任务
    .addTask(InitJPushTask())   // 极光推送 SDK 初始化任务
    .addTask(InitUmengTask())   // 友盟 SDK 初始化任务
    .addTask(GetDeviceIdTask()) // 获取设备 ID 初始化任务
    .addTask(DelayInitTaskA())  // 延迟初始化任务 A
    .addTask(DelayInitTaskB())  // 延迟初始化任务 B
    .start()
//4.等待需要wait的任务执行完毕才继续向下执行
dispatcher.await()
```
**注释1**：init

```java
fun init(context: Context) {
	Companion.context = context
	sHasInit = true
	isMainProcess = Utils.isMainProcess(Companion.context)
}
```
只是初始化了一些启动器需要的基础参数

**注释2**：createInstance

```java
/**
 * 注意：每次获取的都是新对象
 */
@JvmStatic
fun createInstance(): TaskDispatcher {
	if (!sHasInit) {
		throw RuntimeException("must call TaskDispatcher.init first")
	}
	return TaskDispatcher()
}
```
**注释3**：这里给启动器添加了一系列的Task，包括异步和非异步的

这里我们来分析下Task如何异步并发执行的？
分析需求：
- 1.可以区分主线程任务和异步任务
- 2.需要有前后依赖关系，就是说一个任务可能需要等待其他任务执行完后才会执行当前任务。

**1.定义一个任务接口**

```java
// 任务接口
interface ITask {

    /**
     * Task主任务执行完成之后需要执行的任务
     */
    fun getTailRunnable(): Runnable?

    fun setTaskCallBack(callBack: TaskCallBack)

    fun needCall(): Boolean

    /**
     * 优先级的范围，可根据Task重要程度及工作量指定；之后根据实际情况决定是否有必要放更大
     */
    @IntRange(
        from = Process.THREAD_PRIORITY_FOREGROUND.toLong(),
        to = Process.THREAD_PRIORITY_LOWEST.toLong()
    )
    fun priority(): Int

    fun run()

    /**
     * Task执行所在的线程池，可指定，一般默认
     */
    fun runOn(): Executor?

    /**
     * 依赖关系
     */
    fun dependsOn(): List<Class<out Task?>?>?

    /**
     * 异步线程执行的Task是否需要在被调用await的时候等待，默认不需要
     */
    fun needWait(): Boolean

    /**
     * 是否在主线程执行
     */
    fun runOnMainThread(): Boolean

    /**
     * 只是在主进程执行
     */
    fun onlyInMainProcess(): Boolean

}
```
说明：
- 1.接口使用`runOnMainThread`表示是否是主线程任务还是异步线程任务。

- 2.接口使用`dependsOn`来增加依赖关系。

```java
/**
 * 需要在getDeviceId之后执行
 */
class InitJPushTask : Task() {
    override fun dependsOn(): List<Class<out Task?>>? {
        val task: MutableList<Class<out Task?>> = ArrayList()
        task.add(GetDeviceIdTask::class.java)
        return task
    }

    override fun run() {
        JPushInterface.init(mContext)
        val app = mContext as MyApplication
        JPushInterface.setAlias(mContext, 0, app.deviceId)
    }
}
```
在启动一个任务之前会优先判断依赖任务是否执行完毕，如果没有执行会等待依赖任务执行完毕，然后再去执行当前Task，
执行完当前Task后会清除和当前Task相关的状态。

```java
 override fun run() {
		//给应用打点监控每个任务的耗时
        Trace.beginSection(mTask.javaClass.simpleName)
		//设置线程优先级
        Process.setThreadPriority(mTask.priority())
        //等待依赖任务执行完毕，内部会一直await住
        mTask.waitToSatisfy()
        // 执行Task
        mTask.isRunning = true
        mTask.run()
        // 执行Task的尾部任务
        val tailRunnable = mTask.getTailRunnable()
        tailRunnable?.run()
		
		//执行完毕后标记该任务已完成并将依赖他的任务的countdown -1；
        if (!mTask.needCall() || !mTask.runOnMainThread()) {
            printTaskLog(startTime, waitTime)
            TaskStat.markTaskDone()
            mTask.isFinished = true
            mTaskDispatcher.satisfyChildren(mTask)
            mTaskDispatcher.markTaskDone(mTask)
            DispatcherLog.i(mTask.javaClass.simpleName + " finish")
        }
        Trace.endSection()

    }
    
    看satisfyChildren和markTaskDone
	 /**
     * 通知Children一个前置任务已完成
     */
    fun satisfyChildren(launchTask: Task) {
        val arrayList = mDependedHashMap[launchTask.javaClass]
        if (arrayList != null && arrayList.size > 0) {
            for (task in arrayList) {
                task.satisfy()
            }
        }
    }
    //改变当前task在架构中给的状态：如添加到执行完成队列，删除在mNeedWaitTasks的task，将等待mCountDownLatch，mNeedWaitCount计数器值减1.
    fun markTaskDone(task: Task) {
    if (needWait(task)) {
        mFinishedTasks.add(task.javaClass)
        mNeedWaitTasks.remove(task)
        mCountDownLatch!!.countDown()
        mNeedWaitCount.getAndDecrement()
    }
 }
	
```
这里有个`needWait`是应用在某个时刻需要等待住任务执行完成才能继续向下，这个时刻就是一开始的**注释4**处`dispatcher.await`

```java
@UiThread
fun await() {
    try {
        ...
        if (mNeedWaitCount.get() > 0) {
            if (mCountDownLatch == null) {
                throw RuntimeException("You have to call start() before call await()")
            }
            // 等待 10 秒
            mCountDownLatch?.await(WAIT_TIME.toLong(), TimeUnit.MILLISECONDS)
        }
    } catch (e: InterruptedException) {
    }
}

     /**
 * 需要等待的任务数
 */
private val mNeedWaitCount = AtomicInteger() //
```
**其实就是根据mNeedWaitCount的count值来判断是否需要等待，大于0说明还有任务需要等待。
调用mCountDownLatch的await进行等待。mCountDownLatch的初始值就是mNeedWaitCount的个数，
每次执行完一个任务后会执行markTaskDone方法，并将mNeedWaitCount和mCountDownLatch值都减1，
这样在规定超时时间内任务执行完毕就可以继续向后执行，没有执行完毕超时后也会继续向后执行**。

> 可以看到异步启动器的核心是一定要了解任务的依赖关系，对任务属性的需要有一定深刻了解。

这里还有个注意点：**就是任务的拓扑排序生成一个有向无环图**

在`Dispatcher`的`start`方法中：

```java
mAllTasks = TaskSortUtil.getSortResult(mAllTasks, mClsAllTasks)
```
在`getSortResult`中:

```java
 /**
 * 任务的有向无环图的拓扑排序
 */
@Synchronized
fun getSortResult(
    originTasks: List<Task>,
    clsLaunchTasks: List<Class<out Task>>
): MutableList<Task> {
    val makeTime = System.currentTimeMillis()
    val dependSet: MutableSet<Int> = ArraySet()
    val graph = Graph(originTasks.size)
    for (i in originTasks.indices) {
        val task = originTasks[i]
        val list = task.dependsOn()
        if (task.isSend || list == null || list.isEmpty()) {
            continue
        }
        for (cls in task.dependsOn()!!) {
            val indexOfDepend = getIndexOfTask(originTasks, clsLaunchTasks, cls)
            check(indexOfDepend >= 0) {
                task.javaClass.simpleName +
                        " depends on " + cls.simpleName + " can not be found in task list "
            }
            dependSet.add(indexOfDepend)
            graph.addEdge(indexOfDepend, i)
        }
    }
    val indexList: List<Int> = graph.topologicalSort()
    val newTasksAll = getResultTasks(originTasks, dependSet, indexList)
    DispatcherLog.i("task analyse cost makeTime " + (System.currentTimeMillis() - makeTime))
    printAllTaskName(newTasksAll)
    return newTasksAll
}
	
```
##### 4.延迟初始化：
**核心思想**：利用**IdleHandler**特性，在**CPU空闲时**执行，对延迟任务进行分批初始化。延迟启动器可以加载一些非即时性的任务，如界面上的某些UI更新
代码讲解：

```java
/**
 * 延迟初始化分发器
 */
public class DelayInitDispatcher {

    private Queue<Task> mDelayTasks = new LinkedList<>();

    private MessageQueue.IdleHandler mIdleHandler = new     MessageQueue.IdleHandler() {
        @Override
        public boolean queueIdle() {
            // 分批执行的好处在于每一个task占用主线程的时间相对
            // 来说很短暂，并且此时CPU是空闲的，这些能更有效地避免UI卡顿
            if(mDelayTasks.size()>0){
                Task task = mDelayTasks.poll();
                new DispatchRunnable(task).run();
            }
            return !mDelayTasks.isEmpty();
        }
    };

    public DelayInitDispatcher addTask(Task task){
        mDelayTasks.add(task);
        return this;
    }

    public void start(){
        Looper.myQueue().addIdleHandler(mIdleHandler);
    }

}

```
代码很简单，就是将需要启动的延迟任务使用addTask的方式加入即可，这样就可以达到在Cpu空闲时段去执行这些任务。
使用的话可以在`SplashActivity`的`onWindowFocusChanged`进行任务添加。

```java
@Override
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    GlobalHandler.getInstance().getHandler().post((Runnable) () -> {
        if (hasFocus) {
            DelayInitDispatcher delayInitDispatcher = new DelayInitDispatcher();
            delayInitDispatcher.addTask(new InitOtherTask())
                    .start();
        }
    });
}

```
####  4.1.4：Activity阶段优化
这里举两个优化例子
##### 1.Splash 与 Main 合并
SplashActivity主要承载着广告、活动等开屏相关逻辑。一般启动流程为：
- 1.进入 SplashActivity，在 SplashActivity 中判断当前是否有待展示的开屏；
- 2.如果有待展示的开屏则展示开屏，等待开屏展示结束再跳转到 MainActivity，如果没有开屏则直接跳转到 MainActivity。

![两个Activity合并图.awebp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/149659dcaa6245c08761e0df63e51eb3~tplv-k3u1fbpfcp-watermark.image?)
合并后收益：
- 1.合并前需要经过两次Activity的启动，合并后只有一次，减少一次启动时间
- 2.利用读取开屏信息的时间，做一些与Activity强关联的并发任务，比如异步View预加载等。

我们来看抖音app是进行合并的,我们在合并过程中也可以按这个思路来分析。
抖音在合并两者的过程中，有两个问题：
- **1.合并后如何解决外部通过 Activity 名称跳转的问题；**
- 2.**如何解决 LaunchMode 与多实例的问题。**

第 1 个问题比较容易解决，我们可以**通过 activity-alias+targetActivity 将 SplashActivity 指向 MainActivity 解决**。

![Activityalia.awebp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0e2cb5113f042a88cda5acbe1fe8850~tplv-k3u1fbpfcp-watermark.image?)
接下来我们来看一下第二个问题。

**launchMode 问题**：

在 Splash 与 Main 合并之前，SplashActivity 与 MainActivity 的 LaunchMode 分别是 **standard** 和 **sinngletask**。这种情况下我们能够确保 MainActivity 只有一个 实例，并且在我们从应用 home 出去再次进入时，能够重新回到之前的页面。

将 SplashActivity 与 MainActivity 合并以后，我们的 launcher Activity 变成了 MainActivity，**如果继续使用 singletask 这个 launchMode，当我们从二级页面 home 出去再次点击 icon 进入时，我们将无法回到二级页面，而会回到 Main 页面**，因此合并后 MainActivity 的 launch mode 将不再能够使用 singletask。经过调研，我们最终选择了使用 singletop 作为我们的 launchMode。
        
![合并方法.awebp](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed8ea14d091c45489a12fa5e61f441c5~tplv-k3u1fbpfcp-watermark.image?)
**多实例问题**：

- 1、**内部启动多实例的问题**

使用 singletop 虽然能够解决 home 出去再次进入无法回到之前页面的问题，但是随之而来的是 MainActivity 多实例的问题。在抖音的逻辑中存在一些与 MainActivity 生命周期强关联的逻辑，如果 MainActivity 存在多个实例，这部分逻辑将会受到影响，同时多个 MainActivity 的实现，也会导致我们不必要的资源开销，与预期是不符的，因此我们希望能够解决这个问题。

针对这个问题我们的解决方案是，**对于应用内所有启动 MainActivity 的 Intent 增加 FLAG_ACTIVITY_NEW_TASK 与 FLAG_ACTIVITY_CLEAR_TOP 的 flag，以实现类似于 singletask 的 clear top 的特性**

经过分析，我们发现在这部分系统上，即使通过 activity-alias+targetActivity 方式将 SplashActivity 指向了 MainActivity，但是在 AMS 侧它仍然认为启动的是 SplashActivity，后续再启动 MainActivity 时会认为之前是不存在 MainActivity 的，因此会再次启动一个 MainActivity。

**针对这个问题我们的解决方案是，修改启动 MainActivity Intent 的 Component 信息，将其改从 MainActivity 改为 SplashActivity，这样我们就彻底解决了内部启动 MainActivity 导致的多实例的问题。**

![合并后.awebp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2891c57ed4e147d7999f0608d2f56827~tplv-k3u1fbpfcp-watermark.image?)
为了尽可能少的侵入业务，同时也防止后续迭代再出现内部启动导致 MainActivity 问题，我们对 Context startActivity 的调用进行了插桩。对于启动 MainActivity 的调用，在完成向 Intent 中添加 flag 和替换 Component 信息后再调用原有实现。之所以选择插桩方式实现，是因为抖音的代码结构比较复杂，存在多个基类 Activity，且部分基类 Activity 无法直接修改到代码。对于没有这方面问题的业务，可以通过重写基类 Activtity 及 Application 的 startActivity 方法的方式实现。

- 2、**外部启动多实例问题**

以上解决 MainActivity 多实例的方案，是建立在启动 Activity 之前去修改待启动 Activity 的 Intent 的方式实现的，**这种方式对于应用外部启动 MainActivity 导致的 MainActivity 多实例的问题显然是无法解决的**。那么针对外部启动 MainActivity 导致的多实例问题，我们是否有其他解决方案呢？

我们先回到解决 MainActivity 多实例问题的出发点。之所以要**避免 MainActivity 多实例**，是为了防止同时出现多个 MainActivity 对象，出现不符合预期的 MainActivity 生命周期的执行。因此只要确保不会同时出现多个 MainActivity 对象，一样可以解决 MainActivity 多实例问题。

要**避免同时出现多个 MainActivity 对象，我们首先需要知道当前是否已经存在 MainActivity 对象，解决这个问题的思路比较简单，我们可以去监听 Activity 的生命周期**，在 MainActivity 的 onCreate 和 onDestroy 中分别去增加减少 MainActivity 的实例数。如果 MainActivity 实例数为 0 则认为当前不存在 MainActivity 对象。

解决了 MainActivity 对象数统计的问题，接下来我们就**需要让 MainActivity 同时存在的对象数永远保持在 1 个以下**。要解决这个问题我们需要回顾一下 Activity 的启动流程，启动一个 Activity 首先会经过 AMS，AMS 会再调用到 Activity 所在的进程，在 Activity 所在的进程会经过主线程的 Handler post 到主线程，然后通过 Instrumentation 去创建 Activity 对象，以及执行后续的生命周期。对于外部启动 MainActivity ，我们能够控制的是从 AMS 回到进程之后的部分，这里可以选择以 **Instrumentation 的 newActivity 作为入口。**

![合并外部调用.awebp](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/690ad895cf4c42b5a25b01941351d9b9~tplv-k3u1fbpfcp-watermark.image?)
具体来说我们的优化方案如下：
- 1.继承 Instrumentation 实现一个自定义的 Instrumentaion 类，以代理转发方式重写里面的所有方法；
- 2.反射获取 ActivityThread 中 Instrumentaion 对象，并以其为参数创建一个自定义的 Instrumentaion 对象，通过反射方式用自定义的 Instrumentaion 对象替换 ActivityThread 原有的 Instrumentaion；
- 3.在自定义 Instrumentaion 类的 newActivity 方法中，进行判断当前待创建的 Activity 是否为 MainActivity，如果不是 MainActivity 或者当前不存在 MainActivity 对象，则调用原有实现，否则替换其 className 参数将其指向一个空的 Activity，以创建一个空的 Activity；
- 4.在这个空的 Activity 的 onCreate 中 finish 掉自己，同时通过一个添加了 FLAG_ACTIVITY_NEW_TASK 和 FLAG_ACTIVITY_CLEAR_TOP flag 的 Intent 去启动一下 SplashActivity。

> 需要注意的是我们这里 hook Instrumentaion 的实现方案，在高版本的 Android 系统上我们也可以以 **AppComponentFactory instantiateActivity 的方式替换。**

##### 2.xml2code
创建View有两种方式：1.**手动调用View的构造函数创建** 2.**通过xml文件配置**
这里优化主要讲的是第二种xml文件配置，一般流程：
- 1.将 xml 文件解析到内存中 XmlResourceParser 的 IO 过程；
- 2.根据 XmlResourceParser 的 Tag name 获取 Class 的 Java 反射过程；
- 3.创建 View 实例，最终生成 View 树。

![x2c.awebp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f4d4483cdf1c4973ae1ee76aaebbf3f0~tplv-k3u1fbpfcp-watermark.image?)
**传统优化方式**：
- 1.优化xml布局层级，使用ConstraintLayout平滑布局非嵌套布局
- 2.使用ViewStub实现按需加载

这里还有一种就是**异步加载xml布局**的方式：

我们知道xml布局最终是在measure阶段会被使用到。那我们从应用启动阶段开始就可以使用异步加载xml布局到内存中，然后在measure阶段从内存中取出内存中的code。

理想很美好，这个过程中你可能会遇到下面几个问题：

**1.LayoutParams 的问题**

我们知道LayoutParams参数在inflate阶段会根据root属性进行设置，如果root为空那么LayoutParams就会为空，这个时候被添加到View中就会使用默认值，那么自己设置的那些View的属性就会丢失。
	这显然不符合预期情况，解决这个问题，需要在进行预加载时候 new 一个相应类型的 root，以实现对待 inflate view 属性的正确解析


```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    // 省略其他逻辑
    if (root != null) {
        // Create layout params that match root, if supplied
        params = root.generateLayoutParams(attrs);
        if (!attachToRoot) {
            // Set the layout params for temp if we are not
            // attaching. (If we are, we use addView, below)
            root.setLayoutParams(params);
        }
    }
}

public void addView(View child, int index) {
    LayoutParams params = child.getLayoutParams();
    if (params == null) {
        params = generateDefaultLayoutParams();
        if (params == null) {
            throw new IllegalArgumentException("generateDefaultLayoutParams() cannot return null");
        }
    }
    addView(child, index, params);
}
```
**2.inflate 线程优先级的问题**

后台解析线程可能会优先级比较低，无法及时实现View的解析，这个时候需要适当提高解析线程的优先级

**3.Handler问题**

某些自定义View在创建View的时候会初始化一些Handler，这个时候需要指定Handler为主线程Looper

**4.动画问题**

动画在 start 时会校验是否是 UI 线程主线程，这种情况我们需要去修改业务代码，将相关逻辑移动到后续真正添加到 View tree 时。

**5.context问题**

由于在Application阶段进行异步xml解析出的View只能拿到Application的context，如一些Dialog 显示等场景就需要Activity的context，
	那么需要在add到view tree之前将context替换为Activity的context。
    
这里再讲解一些全局角度的优化方案：
### 4.2：全局优化
#### 1.GC抑制
大家都知道GC的过程会有CPU从应用层到内核层的一个转变，这个过程：阻塞 Java 程序的执行，占用 CPU 资源，用设计者说的话：世界都停止了。
所以在启动阶段对Gc的优化可以大大改善界面的流畅度和启动速度。
gc触发条件之一就是内存达到了某个阈值，所以我们的方案就是减少内存的申请和占用，所以解决gc问题就需要改造我们的代码，尽量减少代码的执行量和对象的创建特别是一些高内存图片。
还可以使用的是使用一些特殊手段达到对某些Gc类型的抑制和可控。

参考这篇文章：支付宝客户端架构解析：Android 客户端启动速度优化之「垃圾回收」

注意：这篇文章只对Dalvik 做了讲解，对ART并没有讲解，笔者在网上也没找到类似文章，如果你知道，欢迎推荐。
#### 2.类重排
类重排的实现通过 ReDex 的 Interdex 调整类在 Dex 中的排列顺序。Interdex 优化不需要去分析类引用，它只需要调整 Dex 中类的顺序，把启动时需要加载的类按顺序放到主 dex 里，这个工作我们完全可以在编译过程中实现，而且这个优化可以提升启动速度，优化效果从 facebook 公布的数据来看也比较可观，性价比高。具体实现可以参考 Redex 初探与 Interdex：Andorid 冷启动优化

#### 3.类加载优化
主要是从加载类的复用和校验过程角度去优化 
#### 4.资源重排
利用 Linux 的 IO 读取策略，PageCache 和 ReadAhead 机制，按照读取顺序重新排列，减少磁盘 IO 次数 。具体操作可以参考支付宝 App 构建优化解析：通过安装包重排布优化 Android 端启动性能 这篇文章

## 5.应用启动耗时防劣化。
启动优化还有一个核心就是防劣化过程。对于一些大厂：从代码提交阶段到线下测试阶段，再到灰度发布阶段，再到线上版本发布阶段都有一套防劣化机制，
而对于一些中小型项目就只能在开发过程中注意咯，你觉得呢？


**参考资料**
- [抖音 Android 性能优化系列：启动优化实践](https://juejin.cn/post/7080065015197204511#heading-4)
- [抖音BoostMultiDex优化实践：Android低版本上APP首次启动时间减少80%（一）](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247485522&idx=1&sn=cddfb1c64642b53ee51ca00ce3c696ca&scene=21#wechat_redirect)
- [深入探索Android启动速度优化（上）](https://juejin.cn/post/6844904093786308622#heading-133)
- [老大难的GC原理及调优，这下全说清楚了](https://juejin.cn/post/6844903953004494856)
- [支付宝客户端架构解析：Android 客户端启动速度优化之「垃圾回收」](https://juejin.cn/post/6844903705028853767#heading-3)
- [抖音 Android 性能优化系列：启动优化之理论和工具篇](https://juejin.cn/post/7058080006022856735)
- [心遇 Android 启动优化实践：将启动时间降低 50%](https://juejin.cn/post/7138596300672958471)
- [Android App 启动优化全记录](https://juejin.cn/post/7109655666683281415)
- [抖音 Android 性能优化系列：新一代全能型性能分析工具 Rhea](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247487720&idx=1&sn=4c1dc31b6201e420dcac49250b81055f&chksm=e9d0db0adea7521c9337aacc22877d4e6f33a774499c650f3bfb12ad23ffdc0b8ab6ef40a9a5&scene=21#wechat_redirect)