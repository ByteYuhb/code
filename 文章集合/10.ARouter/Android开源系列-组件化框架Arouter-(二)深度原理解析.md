携手创作，共同成长！这是我参与「掘金日新计划 · 8 月更文挑战」的第7天，[点击查看活动详情](https://juejin.cn/post/7123120819437322247 "https://juejin.cn/post/7123120819437322247") >>
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
最近组里需要进行**组件化框架**的改造，用到了**ARouter**这个开源框架，为了更好的对项目进行改造，笔者花了一些时间去了解了下ARouter

`ARouter`是阿里巴巴团队在17年初发布的一款针对**组件化**模块之间**无接触通讯**的一个开源框架，经过多个版本的迭代，现在已经非常成熟了。

`ARouter`**主要作用**：**组件间通讯，组件解耦，路由跳转，涉及到我们常用的Activity，Provider，Fragment等多个场景的跳转**

接下来笔者会以几个阶段来对`Arouter`进行讲解：

> - [Android开源系列-组件化框架Arouter-(一)使用方式详解](https://juejin.cn/post/7134326382565261348)
> - [Android开源系列-组件化框架Arouter-(二)深度原理解析](https://juejin.cn/post/7134632360087126023)
> - Android开源系列-组件化框架Arouter-(三)APT技术详解
> - Android开源系列-组件化框架Arouter-(四)AGP插件详解

这篇文章我们来讲解下：**Arouter的基本原理**

## 1.ARouter认知
首先我们从命名来看:ARouter翻译过来就是`一个路由器`。

`官方定义`：
> 一个用于帮助 Android App 进行组件化改造的框架 —— 支持模块间的路由、通信、解耦

那么什么是路由呢？
简单理解就是：**一个公共平台转发系统**

## 工作方式：

![arouter原理.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d2e47eeaa2c4a9f83bb6fa48c6ac835~tplv-k3u1fbpfcp-watermark.image?)
- 1.**注册服务**：将我们需要对外暴露的页面或者服务注册到ARouter公共平台中
- 2.**调用服务**：调用ARouter的接口，传入地址和参数，ARouter解析传入的地址和参数转发到对应的服务中

通过ARouter形成了一个无接触解耦的调用过程

## 2.ARouter架构解析
我们来看下`ARouter`的源码架构：

![ARouter源码架构.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/814a072417a140679f71e75a8979bd0f~tplv-k3u1fbpfcp-watermark.image?)

- **app**:是ARouter提供的一个测试Demo
- **arouter-annotation**：这个lib模块中声明了很多注解信息和一些枚举类
- **arouter-api**：ARouter的核心api，转换过程的核心操作都在这个模块里面
- **arouter-compiler**：APT处理器，自动生成路由表的过程就是在这里面实现的
- **arouter-gradle-plugin**：这是一个编译期使用的Plugin插件，主要作用是用于编译器自动加载路由表，节省应用的启动时间。

## 3.原理讲解

这里我们不会一开始就大篇幅对源码进行讲解:
我们先来介绍ARouter中的几个重要概念：有了这几个概念，后面在去看源码就会轻松多了

### 前置基础概念：

#### 概念1：`PostCard`(明信片)
既然是明信片要将信件寄到目的人的手上就至少需要：收件人的姓名和地址，寄件人以及电话和地址等

ARouter就是使用`PostCard`这个类来存储**寄件人和收件人信息**的。

```java
public final class Postcard extends RouteMeta {
    // Base
    private Uri uri; //如果使用Uri方式发起luyou
    private Object tag;             // A tag prepare for some thing wrong. inner params, DO NOT USE!
    private Bundle mBundle;         // 需要传递的参数使用bundle存储
    private int flags = 0;         // 启动Activity的标志：如NEW_FALG
    private int timeout = 300;      // 路由超时
    private IProvider provider;     // 使用IProvider的方式跳转
    private boolean greenChannel;	//绿色通道，可以不经过拦截器
    private SerializationService serializationService; //序列化服务serializationService：需要传递Object自定义类型对象，就需要实现这个服务
    private Context context;        // May application or activity, check instance type before use it.
    private String action;			//Activity跳转的Action

    // Animation
    private Bundle optionsCompat;    // The transition animation of activity
    private int enterAnim = -1;
    private int exitAnim = -1;
	...
}

```

`PostCard`继承了`RouteMeta`：

```java
public class RouteMeta {
    private RouteType type;         // 路由类型：如Activity，Fragment，Provider等
    private Element rawType;        // 路由原始类型，在编译时用来判断
    private Class<?> destination;   // 目的Class对象
    private String path;            // 路由注册的path
    private String group;           // 路由注册的group分组
    private int priority = -1;      // 路由执行优先级，priority越低，优先级越高，这个一般在拦截器中使用
    private int extra;              // Extra data
    private Map<String, Integer> paramsType;  //  参数类型，例如activity中使用@Autowired的参数类型
    private String name; //路由名字，用于生成javadoc

    private Map<String, Autowired> injectConfig;  // 参数配置（对应paramsType）.

}
```
`RouteMeta`:主要存储的是一些目的对象的信息，这些对象是在路由注册的时候才会生成。


#### 概念2：`Interceptor`拦截器
了解`OkHttp`的都知道，其内部调用过程就是使用的拦截器模式，每个拦截器执行的对应的任务。

而`ARouter`中也是如此，所有的路由调用过程在到达目的地前都会先经过自定义的一系列拦截器，实现一些`AOP切面`编程。

```java
public interface IInterceptor extends IProvider {

    /**
     * The operation of this interceptor.
     *
     * @param postcard meta
     * @param callback cb
     */
    void process(Postcard postcard, InterceptorCallback callback);
}

```

`IInterceptor`是一个接口，继承了`IProvider`，所以其也是一个服务类型

只需要实现`process`方法就可以实现拦截操作。


#### 概念3：`greenChannel`:绿色通道
设置了**绿色通道**的跳转过程，可以不经过拦截器


#### 概念4：`Warehouse`：路由仓库
**Warehouse**意为仓库，用于存放被 `@Route、@Interceptor`注释的 路由相关的信息，也就是我们关注的`destination`等信息

举个例子：
> moduleB发起路由跳转到moduleA的activity，moduleB没有依赖moduleA，只是在moduleA的activity上增加了@Route注解。 由于进行activity跳转需要目标Activity的class对象来构建intent，所以必须有一个中间人，把路径"/test/activity"翻译成Activity的class对象，然后moduleB才能实现跳转。（因此在ARouter的使用中 moduleA、moduleB 都是需要依赖 arouter-api的）

这个中间人那就是`ARouter`了，而这个翻译工所作用到的词典就是`Warehouse`，它存着所有路由信息。

```java
class Warehouse {
    //所有IRouteGroup实现类的class对象，是在ARouter初始化中赋值，key是path第一级
    //（IRouteGroup实现类是编译时生成，代表一个组，即path第一级相同的所有路由，包括Activity和Provider服务）
    static Map<String, Class<? extends IRouteGroup>> groupsIndex = new HashMap<>(); 
    //所有路由元信息，是在completion中赋值，key是path
    //首次进行某个路由时就会加载整个group的路由，即IRouteGroup实现类中所有路由信息。包括Activity和Provider服务
    static Map<String, RouteMeta> routes = new HashMap<>();
    
    //所有服务provider实例，在completion中赋值，key是IProvider实现类的class
    static Map<Class, IProvider> providers = new HashMap<>();
    //所有provider服务的元信息(实现类的class对象)，是在ARouter初始化中赋值，key是IProvider实现类的全类名。
    //主要用于使用IProvider实现类的class发起的获取服务的路由，例如ARouter.getInstance().navigation(HelloService.class)
    static Map<String, RouteMeta> providersIndex = new HashMap<>();
    
    //所有拦截器实现类的class对象，是在ARouter初始化时收集到，key是优先级
    static Map<Integer, Class<? extends IInterceptor>> interceptorsIndex = new UniqueKeyTreeMap<>("...");
    //所有拦截器实例，是在ARouter初始化完成后立即创建
    static List<IInterceptor> interceptors = new ArrayList<>();
...
}
```

**Warehouse**存了哪些信息呢？

- **groupsIndex**：存储所有路由组元信息:

![1661149556811.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af7ee377eccd490180364b0fb6eac8fc~tplv-k3u1fbpfcp-watermark.image?)
```js
key：group的名称
value：路由组的模块类class类：
赋值时机：初始化的时候
```
- **routes**：存储所有路由元信息。切记和上面路由组分开，路由是单个路由，路由组是一批次路由

![routes.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe1a7c128f274544a635310abb5b760a~tplv-k3u1fbpfcp-watermark.image?)
```js
key:路由的path
value:路由元信息
赋值时机：LogisticsCenter.completion中赋值
备注：首次进行某个路由时就会加载整个group的路由，即IRouteGroup实现类中所有路由信息。包括Activity和Provider服务
```
- **providers**：存储所有服务provider实例。

```js
key:IProvider实现类的class
value:IProvider实例
赋值时机：在LogisticsCenter.completion中赋值
```
- **providersIndex**：存储所有provider服务元信息(实现类的class对象)。

![provider.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aae367d629b7473c879c6d66c8d35b66~tplv-k3u1fbpfcp-watermark.image?)
```js
key：IProvider实现类的全类名
value:provider服务元信息
赋值时机：ARouter初始化中赋值。
备注：用于使用IProvider实现类class发起的获取服务的路由，例如ARouter.getInstance().navigation(HelloService.class)
```

- **interceptorsIndex**：存储所有拦截器实现类class对象。

![Interceptor.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df87641330cf4e82a1a584bd8346213c~tplv-k3u1fbpfcp-watermark.image?)
```js
key：优先级
value:所有拦截器实现类class对象
赋值时机：是在ARouter初始化时收集到

```

- **interceptors**，所有拦截器实例。是在ARouter初始化完成后立即创建

其中**groupsIndex、providersIndex、interceptorsIndex**是ARouter初始化时就准备好的基础信息，为业务中随时发起路由操作（Activity跳转、服务获取、拦截器处理）做好准备。

#### 概念5：`APT`注解处理器
ARouter使用注解处理器，自动生成路由帮助类：
我们使用ARouter编译后，会在对应模块下自动生成以下类：
这些类的生成规则都是通过APT在编译器自动生成的，关于APT在ARouter中的使用方式，后面会单独拿一节出来讲解：
- Android开源系列-组件化框架Arouter-(三)APT技术详解


#### 概念6：`AGP`插件
ARouter使用了一个可选插件："com.alibaba:arouter-register:1.0.2"
使用这个插件可以在编译器在包中自动检测以及加载路由表信息，而不需要在运行启动阶段再使用包名去dex文件中加载，提高app启动效率
关于这块的，后面会在：
- Android开源系列-组件化框架Arouter-(四)AGP插件详解


有了以上几个概念做基础现在我们再到源码中去看看ARouter是如何跨模块运行起来的

### 源码分析：
首先我们来看路由过程：
- 步骤1：初始化ARouter

```js
ARouter.init(this)
```

- 步骤2：注册Activity路由

```js
@Route(path = "/test/activity1", name = "测试用 Activity")
public class Test1Activity extends BaseActivity {
    @Autowired
    int age = 10;
	protected void onCreate(Bundle savedInstanceState) {
		ARouter.getInstance().inject(this);
	}
}
```

- 步骤3：通过path启动对应的Activity

```js
ARouter.getInstance().build("/test/activity2").navigation();
```

下面我们分别来分析以上过程：

#### 步骤1分析：ARouter.init(this)

```java
/**
 * Init, it must be call before used router.
 */
public static void init(Application application) {
	if (!hasInit) {
		logger = _ARouter.logger;
		_ARouter.logger.info(Consts.TAG, "ARouter init start.");
		hasInit = _ARouter.init(application);

		if (hasInit) {
			_ARouter.afterInit();
		}

		_ARouter.logger.info(Consts.TAG, "ARouter init over.");
	}
}

```

调用了_ARouter同名init方法，进入看看

```java
protected static synchronized boolean init(Application application) {
	mContext = application;
	LogisticsCenter.init(mContext, executor);
	logger.info(Consts.TAG, "ARouter init success!");
	hasInit = true;
	mHandler = new Handler(Looper.getMainLooper());

	return true;
}
```

内部初始化了一些mContext，mHandler以及字段信息
最重要的是`LogisticsCenter.init`(mContext, executor)：这句
进入看看：

```java
/**
 * LogisticsCenter init, load all metas in memory. Demand initialization
 */
public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
	
	try {
		//使用AGP插件进行路由表的自动加载
		loadRouterMap();
		//如果registerByPlugin被设置为true，说明使用的是插件加载，直接跳过
		if (registerByPlugin) {
			logger.info(TAG, "Load router map by arouter-auto-register plugin.");
		} else {
			//如果是false，则调用下面步骤加载
			Set<String> routerMap;

			// 如果是debug模式或者是新版本的，则每次都会去加载routerMap，这会是一个耗时操作
			if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
				logger.info(TAG, "Run with debug mode or new install, rebuild router map.");
				// These class was generated by arouter-compiler.
				routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
				if (!routerMap.isEmpty()) {
					context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
				}

				PackageUtils.updateVersion(context);    // Save new version name when router map update finishes.
			} else {
				//如果是其他的情况，则直接去文件中读取。
				logger.info(TAG, "Load router map from cache.");
				routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
			}
			//这里循环获取routerMap中的信息
			for (String className : routerMap) {
				//如果className = "com.alibaba.android.arouter.routes.ARouter$$Root"格式，则将路由组信息添加到Warehouse.groupsIndex中
				if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
					// This one of root elements, load root.
					((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
				//如果className = "com.alibaba.android.arouter.routes.ARouter$$Interceptors"格式，则将拦截器信息添加到Warehouse.interceptorsIndex中
				} else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
					// Load interceptorMeta
					((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
				//如果className = "com.alibaba.android.arouter.routes.ARouter$$Providers"格式，则将服务Provider信息添加到Warehouse.providersIndex中
				} else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
					// Load providerIndex
					((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
				}
			}
		}
	} catch (Exception e) {
		throw new HandlerException(TAG + "ARouter init logistics center exception! [" + e.getMessage() + "]");
	}
}
```

**总结_ARouter的init操作：**

- 1.优先使用插件加载路由表信息到仓库中，如果没有使用插件，则使用包名com.alibaba.android.arouter.routes去dex文件中查找对应的类对象
查找到后，保存到sp文件中，非debug或者新版本的情况下，下次就直接使用sp文件中缓存的类信息即可。
- 2.查找到对应的类文件后，使用反射调用对应的类的loadInto方法，将路由组，拦截器以及服务Provider信息加载到Warehouse仓库中


继续看init方法中给的_ARouter.afterInit

```java
static void afterInit() {
	// Trigger interceptor init, use byName.
	interceptorService = (InterceptorService) ARouter.getInstance().build("/arouter/service/interceptor").navigation();
}
```
找到`/arouter/service/interceptor`注解处

```java
@Route(path = "/arouter/service/interceptor")
public class InterceptorServiceImpl implements InterceptorService 

```

这里给ARouter创建了一个`InterceptorServiceImpl`服务的实例对象，后面讲到拦截器的时候会用到

#### 步骤2分析：注册Activity路由
我们注册的Activity，Provider等路由信息，会在编译器被注解处理器处理后生成对应的路由表：

路由表在步骤1中`ARouter`初始化的时候被加载到`Warehouse`中

#### 步骤3分析：通过path启动对应的Activity

```java
ARouter.getInstance().build("/test/activity2").navigation();
```

这里我们拆分成三个部分：`getInstance`，`build`，`navigation`

- 3.1:**getInstance**

```java
public static ARouter getInstance() {
	if (!hasInit) {
		throw new InitException("ARouter::Init::Invoke init(context) first!");
	} else {
		if (instance == null) {
			synchronized (ARouter.class) {
				if (instance == null) {
					instance = new ARouter();
				}
			}
		}
		return instance;
	}
}
```

做了init检查并创建了一个ARouter对象

- 3.2：**build**

```java
public Postcard build(String path) {
	return _ARouter.getInstance().build(path);
}
调用了_ARouter的同名build方法
protected Postcard build(String path) {
	if (TextUtils.isEmpty(path)) {
		throw new HandlerException(Consts.TAG + "Parameter is invalid!");
	} else {
		PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
		if (null != pService) {
			path = pService.forString(path);
		}
		return build(path, extractGroup(path), true);
	}
}
1.使用PathReplaceService，可以替换原path为新的path
继续看build方法：
protected Postcard build(String path, String group, Boolean afterReplace) {
	if (TextUtils.isEmpty(path) || TextUtils.isEmpty(group)) {
		throw new HandlerException(Consts.TAG + "Parameter is invalid!");
	} else {
		if (!afterReplace) {
			PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
			if (null != pService) {
				path = pService.forString(path);
			}
		}
		return new Postcard(path, group);
	}
}
```

看到这里创建了一个`Postcard`，传入path和group，对Postcard前面有讲解，这里不再重复


- 3.3：**navigation**

最后会走到`_ARouter`中的同名`navigation`方法中：

```java
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
	//预处理服务
	PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
	if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
		// Pretreatment failed, navigation canceled.
		return null;
	}
	try {
		//完善PostCard信息 留个点1
		LogisticsCenter.completion(postcard);
	} catch (NoRouteFoundException ex) {
		logger.warning(Consts.TAG, ex.getMessage());

		if (debuggable()) {
			// Show friendly tips for user.
			runInMainThread(new Runnable() {
				@Override
				public void run() {
					Toast.makeText(mContext, "There's no route matched!\n" +
							" Path = [" + postcard.getPath() + "]\n" +
							" Group = [" + postcard.getGroup() + "]", Toast.LENGTH_LONG).show();
				}
			});
		}
		
		//没有找到路由信息，则直接返回callback.onLost
		if (null != callback) {
			callback.onLost(postcard);
		} else {
			// 没有callback则调用全局降级服务DegradeService的onLost方法
			DegradeService degradeService = ARouter.getInstance().navigation(DegradeService.class);
			if (null != degradeService) {
				degradeService.onLost(context, postcard);
			}
		}

		return null;
	}
	//回调callback.onFound提醒用户已经找到path
	if (null != callback) {
		callback.onFound(postcard);
	}
	//非绿色通道走到拦截器中
	if (!postcard.isGreenChannel()) {   // It must be run in async thread, maybe interceptor cost too mush time made ANR.
		interceptorService.doInterceptions(postcard, new InterceptorCallback() {
			/**
			 * Continue process
			 *
			 * @param postcard route meta
			 */
			@Override
			public void onContinue(Postcard postcard) {
				_navigation(postcard, requestCode, callback);
			}

			/**
			 * Interrupt process, pipeline will be destory when this method called.
			 *
			 * @param exception Reson of interrupt.
			 */
			@Override
			public void onInterrupt(Throwable exception) {
				if (null != callback) {
					callback.onInterrupt(postcard);
				}

				logger.info(Consts.TAG, "Navigation failed, termination by interceptor : " + exception.getMessage());
			}
		});
	} else {
		//绿色通道直接调用_navigation
		return _navigation(postcard, requestCode, callback);
	}

	return null;
}
```

方法任务：
- 1.预处理服务
- 2.完善`PostCard`信息
- 3.如果是非绿色通道，则使用拦截器处理请求
- 4.调用`_navigation`处理

这里我们看下第3点：拦截器处理

```java
interceptorService.doInterceptions{
	public void onContinue(Postcard postcard) {
		_navigation(postcard, requestCode, callback);
	}
	public void onInterrupt(Throwable exception) {
		if (null != callback) {
			callback.onInterrupt(postcard);
		}
	}
}
```

如果被拦截回调callback.onInterrupt
如果没有就执行_navigation方法

进入interceptorService.doInterceptions看下：

前面分析过interceptorService是InterceptorServiceImpl对象

```java
@Route(path = "/arouter/service/interceptor")
public class InterceptorServiceImpl implements InterceptorService {
    private static boolean interceptorHasInit;
    private static final Object interceptorInitLock = new Object();

    @Override
    public void doInterceptions(final Postcard postcard, final InterceptorCallback callback) {
        if (MapUtils.isNotEmpty(Warehouse.interceptorsIndex)) {

            checkInterceptorsInitStatus();

            if (!interceptorHasInit) {
                callback.onInterrupt(new HandlerException("Interceptors initialization takes too much time."));
                return;
            }

            LogisticsCenter.executor.execute(new Runnable() {
                @Override
                public void run() {
					//使用CancelableCountDownLatch计数器
                    CancelableCountDownLatch interceptorCounter = new CancelableCountDownLatch(Warehouse.interceptors.size());
                    try {
                        _execute(0, interceptorCounter, postcard);
                        interceptorCounter.await(postcard.getTimeout(), TimeUnit.SECONDS);
                        if (interceptorCounter.getCount() > 0) {    // Cancel the navigation this time, if it hasn't return anythings.
							//拦截器处理超时
                            callback.onInterrupt(new HandlerException("The interceptor processing timed out."));
                        } else if (null != postcard.getTag()) {    // Maybe some exception in the tag.
							//拦截器过程出现异常
                            callback.onInterrupt((Throwable) postcard.getTag());
                        } else {
							//继续执行下面任务onContinue
                            callback.onContinue(postcard);
                        }
                    } catch (Exception e) {
                        callback.onInterrupt(e);
                    }
                }
            });
        } else {
            callback.onContinue(postcard);
        }
    }
	private static void _execute(final int index, final CancelableCountDownLatch counter, final Postcard postcard) {
        if (index < Warehouse.interceptors.size()) {
            IInterceptor iInterceptor = Warehouse.interceptors.get(index);
            iInterceptor.process(postcard, new InterceptorCallback() {
                @Override
                public void onContinue(Postcard postcard) {
                    // Last interceptor excute over with no exception.
                    counter.countDown();
					//递归调用_execute执行拦截器
                    _execute(index + 1, counter, postcard);  // When counter is down, it will be execute continue ,but index bigger than interceptors size, then U know.
                }

                @Override
                public void onInterrupt(Throwable exception) {
                    // Last interceptor execute over with fatal exception.

                    postcard.setTag(null == exception ? new HandlerException("No message.") : exception);    // save the exception message for backup.
                    counter.cancel();
                    // Be attention, maybe the thread in callback has been changed,
                    // then the catch block(L207) will be invalid.
                    // The worst is the thread changed to main thread, then the app will be crash, if you throw this exception!
//                    if (!Looper.getMainLooper().equals(Looper.myLooper())) {    // You shouldn't throw the exception if the thread is main thread.
//                        throw new HandlerException(exception.getMessage());
//                    }
                }
            });
        }
    }
}
```

拦截器总结：
- 1.使用计数器对拦截器技术，执行开始计数器+1，执行结束计数器-1，如果拦截器执行时间到，计数器数大于0，则说明还有未执行完成的拦截器，这个时候就超时了退出
- 2.拦截器执行使用递归的方式进行
- 3.拦截器执行完成继续执行`_navigation`方法

我们来看_navigation方法：

```java
private Object _navigation(final Postcard postcard, final int requestCode, final NavigationCallback callback) {
	final Context currentContext = postcard.getContext();

	switch (postcard.getType()) {
		case ACTIVITY:
			// Build intent
			final Intent intent = new Intent(currentContext, postcard.getDestination());
			intent.putExtras(postcard.getExtras());

			// Set flags.
			int flags = postcard.getFlags();
			if (0 != flags) {
				intent.setFlags(flags);
			}

			// Non activity, need FLAG_ACTIVITY_NEW_TASK
			if (!(currentContext instanceof Activity)) {
				intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
			}

			// Set Actions
			String action = postcard.getAction();
			if (!TextUtils.isEmpty(action)) {
				intent.setAction(action);
			}

			// Navigation in main looper.
			runInMainThread(new Runnable() {
				@Override
				public void run() {
					startActivity(requestCode, currentContext, intent, postcard, callback);
				}
			});

			break;
		case PROVIDER:
			return postcard.getProvider();
		case BOARDCAST:
		case CONTENT_PROVIDER:
		case FRAGMENT:
			Class<?> fragmentMeta = postcard.getDestination();
			try {
				Object instance = fragmentMeta.getConstructor().newInstance();
				if (instance instanceof Fragment) {
					((Fragment) instance).setArguments(postcard.getExtras());
				} else if (instance instanceof android.support.v4.app.Fragment) {
					((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
				}

				return instance;
			} catch (Exception ex) {
				logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
			}
		case METHOD:
		case SERVICE:
		default:
			return null;
	}

	return null;
}

```

这个方法其实就是根据`PostCard`的`type`来处理不同的请求了
- 1.**Activity**，直接跳转
- 2.**Fragment，Provider，BroadcaseReceiver和ContentProvider**，直接返回类的实例对象。

整个过程我们就基本了解了。
上面还留了一个点：

**留的点1**：`ARouter`是如何完善`PostCard`信息

看LogisticsCenter.completion(postcard);

进入这个方法：

```java
public synchronized static void completion(Postcard postcard) {
        if (null == postcard) {
            throw new NoRouteFoundException(TAG + "No postcard!");
        }
		//去Warehouse.routes去取路由元数据，开始肯定是没有的
        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
		//没获取到
        if (null == routeMeta) {
            // Maybe its does't exist, or didn't load.
			//判断Warehouse.groupsIndex路由组中是否有这个group
            if (!Warehouse.groupsIndex.containsKey(postcard.getGroup())) {
                throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
            } else {
                
                try {
                    //动态添加路由元信息到路由中
                    addRouteGroupDynamic(postcard.getGroup(), null);
                } catch (Exception e) {              
                }
				//重新加载。这个时候就会有路由元信息了
                completion(postcard);   // Reload
            }
        } else {
			//给postcard设置目的地，设置类型，设置优先级，设置Extra等信息
            postcard.setDestination(routeMeta.getDestination());
            postcard.setType(routeMeta.getType());
            postcard.setPriority(routeMeta.getPriority());
            postcard.setExtra(routeMeta.getExtra());

            Uri rawUri = postcard.getUri();
            ...
            switch (routeMeta.getType()) {
                case PROVIDER:  // if the route is provider, should find its instance
                    // Its provider, so it must implement IProvider
                    Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                    IProvider instance = Warehouse.providers.get(providerMeta);
                    if (null == instance) { // There's no instance of this provider
                        IProvider provider;
                        try {
                            provider = providerMeta.getConstructor().newInstance();
                            provider.init(mContext);
                            Warehouse.providers.put(providerMeta, provider);
                            instance = provider;
                        } catch (Exception e) {
                            logger.error(TAG, "Init provider failed!", e);
                            throw new HandlerException("Init provider failed!");
                        }
                    }
                    postcard.setProvider(instance);
                    postcard.greenChannel();    // Provider should skip all of interceptors
                    break;
                case FRAGMENT:
                    postcard.greenChannel();    // Fragment needn't interceptors
                default:
                    break;
            }
        }
    }

进入addRouteGroupDynamic
public synchronized static void addRouteGroupDynamic(String groupName, IRouteGroup group) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
	if (Warehouse.groupsIndex.containsKey(groupName)){
		// If this group is included, but it has not been loaded
		// load this group first, because dynamic route has high priority.
		Warehouse.groupsIndex.get(groupName).getConstructor().newInstance().loadInto(Warehouse.routes);
		Warehouse.groupsIndex.remove(groupName);
	}

	// cover old group.
	if (null != group) {
		group.loadInto(Warehouse.routes);
	}
}
```

看上面代码可知：
**数据完善过程是通过组名group去groupsIndex获取对应的组的class对象，然后调用class对象的loadInto方法，将路由元数据加载到Warehouse.routes
然后重新调用completion完善方法去Warehouse.routes中取出路由信息并加载到PostCard中，这样PostCard中就获取到了目的地址信息。**



下面我画了一张图描述了上面的调用过程
一图胜千言

![arouter调用过程.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/127693fde669493d8e09caf6ee2082d5~tplv-k3u1fbpfcp-watermark.image?)
## 总结
本文先介绍了`ARouter`使用过程中 的一些**基本概念**，理解了这些概念后，我们再从使用步骤触发，对每个使用节点进行了介绍。
最后使用一张图总结了整个**使用原理过程**：
这里我们还有一些悬念：
- 1.ARouter帮助类是如何生成的，这里使用到了**APT注解处理器**的技术
关于APT我们会在下一章：

Android开源系列-组件化框架Arouter-(三)APT技术详解

- 这里还有个有趣的现象，我们在调用路由表加载的时候：
使用了`loadRouterMap`加载，但是查看里面代码：

```java
private static void loadRouterMap() {
	registerByPlugin = false;
	// auto generate register code by gradle plugin: arouter-auto-register
	// looks like below:
	// registerRouteRoot(new ARouter..Root..modulejava());
	// registerRouteRoot(new ARouter..Root..modulekotlin());
}
```

居然是空的。。
呃呃呃
没关系看注解：

```java
auto generate register code by gradle plugin: arouter-auto-register
```

可以看到这里使用了`arouter-auto-register`插件中自动生成注册代码的方式：
这里其实就是使用到了**字节码插庄技术**，动态添加了代码，这里留到：

Android开源系列-组件化框架Arouter-(四)AGP插件详解

好了，本篇就到这里了。

**> 持续输出中。。你的点赞，关注是我最大的动力。**





​	