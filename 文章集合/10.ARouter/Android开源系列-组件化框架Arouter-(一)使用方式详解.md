> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言：
最近组里需要进行**组件化框架**的改造，用到了`Arouter`这个开源框架，为了更好的对项目进行改造，笔者花了一些时间去了解了下`Arouter`

`ARouter`是阿里巴巴团队在17年初发布的一款针对**组件化模块**之间无接触通讯的一个开源框架，经过多个版本的迭代，现在已经非常成熟了。

`ARouter`**主要作用**：组件间通讯，组件解耦，路由跳转，涉及到我们常用的`Activity`，`Provider`，`Fragment`等多个场景的跳转

接下来笔者会以几个阶段来对`Arouter`进行讲解：

> - Android开源系列-组件化框架Arouter-(一)使用方式详解
> - Android开源系列-组件化框架Arouter-(二)基本原理详解
> - Android开源系列-组件化框架Arouter-(三)APT技术详解
> - Android开源系列-组件化框架Arouter-(三)AGP插件详解

这篇文章是Arouter的第一篇：**基本使用方式**

## 前期准备

我们先来新建一个项目：

项目架构如下：

![demo架构.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e483186956d4454faf6b5221821d4662~tplv-k3u1fbpfcp-watermark.image?)
- **app**：app模块

`build.gradle`:

```java
dependencies {
	...
	implementation 'com.alibaba:arouter-api:1.5.2'
	implementation project(':module_java')
	implementation project(':module_kotlin')
	//annotationProcessor 'com.alibaba:arouter-compiler:1.5.2'
}
```

引入arouter的核心api和引用需要调用的模块即可

这里如果需要使用依赖注入的方式查找class对象，则需要添加声明`注解处理器`：
```java
annotationProcessor 'com.alibaba:arouter-compiler:1.5.2'
```
- **module_java**：java语言lib模块

`build.gradle:`

```java
android {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [AROUTER_MODULE_NAME: project.getName(), AROUTER_GENERATE_DOC: "enable"]
            }
        }
}
dependencies {
        ...
        implementation 'com.alibaba:arouter-api:1.5.2'
        annotationProcessor 'com.alibaba:arouter-compiler:1.5.2'
}
```


1.引入`arouter-api`核心api和`arouter-compiler`注解处理器api

2.设置主机处理器的外部参数`AROUTER_MODULE_NAME`和`AROUTER_GENERATE_DOC`，这两个参数会在编译器生成`Arouter路由表`的时候使用到

- **module_kotlin**：kotli语言lib模块

`build.gradle:`

```java
apply plugin: 'kotlin-kapt'
kapt {
        arguments {
                arg("AROUTER_MODULE_NAME", project.getName())
        }
}
dependencies {
    implementation 'com.alibaba:arouter-api:1.5.2'
    kapt 'com.alibaba:arouter-compiler:1.5.2'
}
```
> 这里的设置方式和java一样，区别：java使用关键字`annotationProcessor`，而kotlin使用`kapt`。

同步项目后。就可以使用Arouter的api啦。

## ARouter api

###  1.初始化Arouter
建议在application中初始化：

```java
ARouter.init(this);
```
### 2.日志开启

```java
ARouter.openLog();
```

### 3.打开调试模式

```java
ARouter.openDebug();
```

> 调试模式不是必须开启，但是为了防止有用户开启了InstantRun，
> 忘了开调试模式，导致无法使用Demo，如果使用了InstantRun，必须在
> 初始化之前开启调试模式，但是上线前需要关闭，InstantRun仅用于开
> 发阶段，线上开启调试模式有安全风险，可以使用BuildConfig.DEBUG
> 来区分环境
### 4.Activity模块的调用：
- 4.1：注解声明

```java
@Route(path = "/gtest/test") 
//或者@Route(path = "/gtest/test",group = "gtest") 
public class ArouterTestActivity extends AppCompatActivity {
	
}
```

- 4.2：组件跨模块调用
```java
ARouter.getInstance().build("/gtest/test").navigation();
```
**记住**：如果注解没有指明group,arouter默认会以path的第一个节点gtest为组名进行分组

### 5.属性依赖注入
- 5.1：注解声明

```java
@Route(path = "/test/activity1", name = "测试用 Activity")
public class Test1Activity extends BaseActivity {
    @Autowired
    int age = 10;

    @Autowired
    int height = 175;

    @Autowired(name = "boy", required = true)
    boolean girl;

    @Autowired
    char ch = 'A';

    @Autowired
    float fl = 12.00f;

    @Autowired
    double dou = 12.01d;

    @Autowired
    TestSerializable ser;

    @Autowired
    TestParcelable pac;

    @Autowired
    TestObj obj;

    @Autowired
    List<TestObj> objList;

    @Autowired
    Map<String, List<TestObj>> map;

    private long high;

    @Autowired
    String url;

    @Autowired
    HelloService helloService;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_test1);

        ARouter.getInstance().inject(this);
		
	}
}

```
- 5.2：注解跨模块调用

```java
ARouter.getInstance().build("/test/activity1")
		.withString("name", "老王")
		.withInt("age", 18)
		.withBoolean("boy", true)
		.withLong("high", 180)
		.withString("url", "https://a.b.c")
		.withSerializable("ser", testSerializable)
		.withParcelable("pac", testParcelable)
		.withObject("obj", testObj)
		.withObject("objList", objList)
		.withObject("map", map)
		.navigation();
```

传入的Object需要使用Serializable序列化

```java
public class TestSerializable implements Serializable {
    public String name;
    public int id;

    public TestSerializable() {
    }

    public TestSerializable(String name, int id) {
        this.name = name;
        this.id = id;
    }
}
```

**注意点**：		
- 1.常用数据类型直接使用：`Autowired`注解即可
- 2.如果需要指定外部传入的数据key：可以使用`name`指定，如果这个字段是必须传入的，则需要指定`required = true`，外部未传入可能会报错
- 3.传入自定义类的属性则需要先对自定义进行序列化，且需要指定一个`SerializationService`：这个类是用于`Object`对象和json进行转化用

```java
@Route(path = "/yourservicegroupname/json")
public class JsonServiceImpl implements SerializationService {
    @Override
    public void init(Context context) {

    }

    @Override
    public <T> T json2Object(String text, Class<T> clazz) {
        return JSON.parseObject(text, clazz);
    }

    @Override
    public String object2Json(Object instance) {
        return JSON.toJSONString(instance);
    }

    @Override
    public <T> T parseObject(String input, Type clazz) {
        return JSON.parseObject(input, clazz);
    }
}

```
### 6.Service模块的调用
- 6.1:声明接口,其他组件通过接口来调用服务

```java
public interface HelloService extends IProvider {
    String sayHello(String name);
}

```

- 6.2:实现接口

```java
@Route(path = "/yourservicegroupname/hello", name = "测试服务")
public class HelloServiceImpl implements HelloService {

    @Override
    public String sayHello(String name) {
    return "hello, " + name;
    }

    @Override
    public void init(Context context) {

    }
}
```

- 6.3:使用接口

`6.3.1`:使用依赖查找的方式发现服务，主动去发现服务并使用，下面两种方式分别是byName和byType

**1.使用class类型查找**
         
```java
helloService3 = ARouter.getInstance().navigation(HelloService.class);
helloService3.sayHello("Vergil");
```

**2.使用path查找**
            
```java
helloService4 = (HelloService) ARouter.getInstance().build("/yourservicegroupname/hello").navigation();
 helloService4.sayHello("Vergil");
```

`6.3.2`:(**推荐**)使用依赖注入的方式发现服务,通过注解标注字段,即可使用，无需主动获取

```java
@Autowired
 HelloService helloService;
 @Autowired(name = "/yourservicegroupname/hello")
 HelloService helloService2;
 helloService.sayHello("mikeaaaa");
 helloService2.sayHello("mikeaaaa");
```

### 7.Fragment和BOARDCAST以及CONTENT_PROVIDER
使用方式和`Activity`类似，但是这三者都是直接获取对应的Class类对象，拿到对象后，可以操作对象,这里不再讲解

### 8.拦截器的使用
拦截器就是在跳转的过程中，设置的对跳转的检测和预处理等操作。

- 8.1：拦截器声明
可以添加拦截器的`优先级`，优先级越高，越优先执行

```java
@Interceptor(priority = 7)
public class Test1Interceptor implements IInterceptor {
    /**
     * The operation of this interceptor.
     *
     * @param postcard meta
     * @param callback cb
     */
    @Override
    public void process(final Postcard postcard, final InterceptorCallback callback) {
        if ("/test/activity4".equals(postcard.getPath())) {

            // 这里的弹窗仅做举例，代码写法不具有可参考价值
            final AlertDialog.Builder ab = new AlertDialog.Builder(postcard.getContext());
            ab.setCancelable(false);
            ab.setTitle("温馨提醒");
            ab.setMessage("想要跳转到Test4Activity么？(触发了\"/inter/test1\"拦截器，拦截了本次跳转)");
            ab.setNegativeButton("继续", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    callback.onContinue(postcard);
                }
            });
            ab.setNeutralButton("算了", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    callback.onInterrupt(null);
                }
            });
            ab.setPositiveButton("加点料", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    postcard.withString("extra", "我是在拦截器中附加的参数");
                    callback.onContinue(postcard);
                }
            });

            MainLooper.runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    ab.create().show();
                }
            });
        } else {
            callback.onContinue(postcard);
        }
    }

    /**
     * Do your init work in this method, it well be call when processor has been load.
     *
     * @param context ctx
     */
    @Override
    public void init(Context context) {
        Log.e("testService", Test1Interceptor.class.getName() + " has init.");
    }
}
```

可以设置绿色通道模式：`greenChannel`()不做拦截器处理

### 9.监听跳转状态

```java
.navigation(this, new NavCallback() {
	@Override
	public void onArrival(Postcard postcard) {
		Log.d("ARouter", "onArrival");
		Log.d("ARouter", "成功跳转了");
	}

	@Override
	public void onInterrupt(Postcard postcard) {
		super.onInterrupt(postcard);
		Log.d("ARouter", "被拦截了");
	}
	@Override
	public void onFound(Postcard postcard) {
		super.onFound(postcard);
		Log.d("ARouter", "onFound:path:"+postcard.getPath());
		Log.d("ARouter", "找到了对应的path");
	}

	@Override
	public void onLost(Postcard postcard) {
		super.onLost(postcard);
		Log.d("ARouter", "onLost:path:"+postcard.getPath());
		Log.d("ARouter", "没找到了对应的path");
	}
});
```

### 10.自定义全局降级策略

```java
// 实现DegradeService接口，并加上一个Path内容任意的注解即可
@Route(path = "/xxx/xxx")
public class DegradeServiceImpl implements DegradeService {
    @Override
    public void onLost(Context context, Postcard postcard) {
        // do something.
    }

    @Override
    public void init(Context context) {

    }
}
```

### 11.动态注册路由信息 
适用于部分插件化架构的App以及需要动态注册路由信息的场景，可以通过 ARouter 提供的接口实现动态注册 路由信息，
目标页面和服务可以不标注 @Route 注解，注意：同一批次仅允许相同 group 的路由信息注册

```java
ARouter.getInstance().addRouteGroup(new IRouteGroup() {
	@Override
	public void loadInto(Map<String, RouteMeta> atlas) {
		atlas.put("/dynamic/activity",      // path
			RouteMeta.build(
				RouteType.ACTIVITY,         // 路由信息
				TestDynamicActivity.class,  // 目标的 Class
				"/dynamic/activity",        // Path
				"dynamic",                  // Group, 尽量保持和 path 的第一段相同
				0,                          // 优先级，暂未使用
				0                           // Extra，用于给页面打标
			)
		);
	}
});
```
### 12:使用自己的日志工具打印日志

```java
ARouter.setLogger();
```
### 13.使用自己提供的线程池

```java
ARouter.setExecutor();
```

### 14.重写跳转URL

```java
// 实现PathReplaceService接口，并加上一个Path内容任意的注解即可
@Route(path = "/xxx/xxx") // 必须标明注解
public class PathReplaceServiceImpl implements PathReplaceService {
    /**
    * For normal path.
    *
    * @param path raw path
    */
    String forString(String path) {
    return path;    // 按照一定的规则处理之后返回处理后的结果
    }

/**
    * For uri type.
    *
    * @param uri raw uri
    */
    Uri forUri(Uri uri) {
        return url;    // 按照一定的规则处理之后返回处理后的结果
    }
}
```


### 15.使用agp插件动态路由表加载
- 1.在项目级`build.gradle`中：

```java
buildscript {
	dependencies {
        classpath "com.alibaba:arouter-register:$arouter_register_version"
    }
}
```

- 2.在模块级app下引入路由表自动加载插件

```java
id 'com.alibaba.arouter'
```

同步代码后，在编译期会自动进行路由表的加载，可以加快`arouter`的第一次初始化时间，为什么是第一次呢，**因为arouter初始化会缓存路由表信息到文件中。**

## Q&A
### 1."W/ARouter::: ARouter::No postcard![ ]"
这个Log正常的情况下也会打印出来，如果您的代码中没有实现DegradeService和PathReplaceService的话，因为ARouter本身的一些功能也依赖 自己提供的Service管理功能，ARouter在跳转的时候会尝试寻找用户实现的PathReplaceService，用于对路径进行重写(可选功能)，所以如果您没有 实现这个服务的话，也会抛出这个日志

推荐在app中实现`DegradeService、PathReplaceService`

### 2."W/ARouter::: ARouter::There is no route match the path [/xxx/xxx], in group [xxx][ ]"
通常来说这种情况是没有找到目标页面，目标不存在
如果这个页面是存在的，那么您可以按照下面的步骤进行排查
- a.检查目标页面的注解是否配置正确，正确的注解形式应该是 (@Route(path="/test/test"), 如没有特殊需求，请勿指定group字段，废弃功能)
- b.检查目标页面所在的模块的gradle脚本中是否依赖了 arouter-compiler sdk (需要注意的是，要使用apt依赖，而不是compile关键字依赖)
- c.检查编译打包日志，是否出现了形如 ARouter::�Compiler >>> xxxxx 的日志，日志中会打印出发现的路由目标
- d.启动App的时候，开启debug、log(openDebug/openLog), 查看映射表是否已经被扫描出来，形如 D/ARouter::: LogisticsCenter has already been loaded, GroupIndex[4]，GroupIndex > 0

### 3.开启InstantRun之后无法跳转(高版本Gradle插件下无法跳转)？
因为开启InstantRun之后，很多类文件不会放在原本的dex中，需要单独去加载，ARouter默认不会去加载这些文件，因为安全原因，只有在开启了openDebug之后 ARouter才回去加载InstantRun产生的文件，所以在以上的情况下，需要在init之前调用openDebug

### 4.TransformException:java.util.zip.ZipException: duplicate entry ....
ARouter有按组加载的机制，关于分组可以参考 6-1 部分，ARouter允许一个module中存在多个分组，但是不允许多个module中存在相同的分组，会导致映射文件冲突

### 5.Kotlin类中的字段无法注入如何解决？
首先，Kotlin中的字段是可以自动注入的，但是注入代码为了减少反射，使用的字段赋值的方式来注入的，Kotlin默认会生成set/get方法，并把属性设置为private 所以只要保证Kotlin中字段可见性不是private即可，简单解决可以在字段上添加 @JvmField

### 6.通过URL跳转之后，在intent中拿不到参数如何解决？
需要注意的是，如果不使用自动注入，那么可以不写 ARouter.getInstance().inject(this)，但是需要取值的字段仍然需要标上 @Autowired 注解，因为 只有标上注解之后，ARouter才能知道以哪一种数据类型提取URL中的参数并放入Intent中，这样您才能在intent中获取到对应的参数

### 7.新增页面之后，无法跳转？
ARouter加载Dex中的映射文件会有一定耗时，所以ARouter会缓存映射文件，直到新版本升级(版本号或者versionCode变化)，而如果是开发版本(ARouter.openDebug())， ARouter 每次启动都会重新加载映射文件，开发阶段一定要打开 Debug 功能

## 参考

[Arouter官方文档](https://github.com/alibaba/ARouter/blob/master/README_CN.md)