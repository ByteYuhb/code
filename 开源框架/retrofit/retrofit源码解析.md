## 前言

- 网络请求在我们开发中起的很大比重，有一个好的网络框架可以节省我们的开发工作量，也可以避免一些在开发中不该出现的bug

- <mark>Retrofit</mark>是一个轻量级框架，基于<mark>OkHttp</mark>的一个<mark>Restful</mark>框架
  
  ![图2](C:\Users\Administrator\Desktop\笔记\开源框架\retrofit\微信图片_20220527220425.png)
  
  今天我们就从<mark>使用方式和源码</mark>两个角度来分析下Retrofit v2.0
  
  Android体系课学习 之 开源框架学习
  
  [Android体系课学习 之 网络请求库OkHttp使用方式(附Demo)]()
  
  [Android体系课学习 之 网络请求库OkHttp看这一篇就够了]()
  
  [Android体系课学习 之 网络请求库Retrofit使用方式(附Demo) - 掘金](https://juejin.cn/post/7103518717471883277)
  
  [Android体系课学习 之 网络请求库Retrofit源码分析]()
  
  [Android体系课学习 之 RxJava看这篇就够了]()
  
  [[Android体系课学习 之 RxJava操作符详解]()
  
  [[Android体系课学习 之 RxJava源码分析]()

## 1.目录

   <img src="file:///C:/Users/Administrator/Desktop/笔记/开源框架/retrofit/Retrofit%202.0目录介绍.png" title="" alt="目录" width="548">

## 2.简介

- **功能**
  
  - 使用接口方法注解的方式配置网络请求参数

- **优点**
  
  - 功能强大：支持同步和异步，支持自定义数据解析类型，常用如<mark>Json</mark>
  
  - 支持多平台，如<mark>Android</mark>，<mark>Java8</mark>等
  
  - 使用简单：通过注释的方式动态配置交易请求参数
  
  - 使用多种<mark>设计模式</mark>，对模块之间解耦，拓展性好
  
  - 目前最新版本支持协程的方式进行网络请求，不需要再指定使用同步还是异步方式

- **使用场景**
  
  > 支持所有的网络请求场景，特别是请求次数比较频繁的场景，因为内部是基于OkHttp，而OkHttp内部使用了各种拦截器，且对请求有做缓存IO复用操作，防止短时间重复连接操作，浪费资源。

**需要注意：Retrofit请求的本质是通过OkHttp发送网路请求，而Retrofit只是负责网络前期接口的封装**

![图](C:\Users\Administrator\Desktop\笔记\开源框架\retrofit\retrofit网络流程.png)

## 3.具体使用

由于本文主要讲的是Retrofit的源码分析这块，具体使用方式可以看这：

[Android体系课学习 之 网络请求库Retrofit使用方式(附Demo)](https://juejin.cn/post/7103518717471883277)

> **如果只是为了查看Retrofit基本使用方式直接跳转到上面链接**
> 
> **有想进阶的可以继续往下看**

## 4.源码分析

 `本文分析的源码版本：`

```java
implementation 'com.squareup.retrofit2:retrofit:2.9.0
```

### **先列出几个在Retrofit使用过程中很重要的角色：**

- **网络请求执行器** <mark>Call</mark>
  
  作用：创建Http网络请求

- **网络请求适配器** <mark>CallAdapter</mark>
  
  网络请求执行器的的适配器，将默认的网络请求执行器OkhttpCall适配到不同的平台。
  
  Retrofit支持四种不同的平台：
  
  *<mark>Android</mark>：* 对应ExcutorCallAdapterFactory
  
  *<mark>RxJava</mark>* : 对应RxJavaCallAdapterFactory
  
  *<mark>Java8</mark>* ：对应Java8CallAdapterFactory
  
  *<mark>Guava</mark>* ：对应GuavaCallAdapterFactory

- **数据转换器** <mark>Converter</mark>
  
   将返回数据解析出我们需要的类型：
  
   支持的数据类型如下：XML,Gson,Json,protoBuf等

- **回调执行器** <mark>CallBackExcutor</mark>
  
  - 主要作用是切换线程使用：在数据返回过程中将线程从子线程切换到主线程，让主线程处理数据

### **下面介绍下Retrofit网络请求过程：**

<img src="file:///C:/Users/Administrator/AppData/Roaming/marktext/images/48bf17a19a6ab172c560a32e2af0d34c26be8cbd.png" title="" alt="涂" width="531">

    

### **然后我们来回忆下Retrofit的大致使用流程：**

**`步骤1.创建Retrofit实例`**

**`步骤2.创建请求的实体类，并配置网络请求参数`**

**`步骤3.创建请求实例并发送网络请求`**

**`步骤4.响应数据处理`**

- **接下来我们依次来分析下每个步骤内部源码：**

#### **步骤1.1.创建Retrofit实例：**

![图整体](C:\Users\Administrator\Desktop\笔记\开源框架\retrofit\源码分析\创建retrofit整体.png)

        我们将步骤1分为5个模块，现在依次来分析下这5个模块：

##### 

##### 模块1：new Retrofit

```java
//Retrofit 类
  public final class Retrofit {
  //这个map存储的是网络请求配置缓存对象，以请求接口方法名为Key,存储的value为一个ServiceMethod对象，
  //内部包含了一个请求需要的所有信息
  private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();

  //网络请求器的工厂
  final okhttp3.Call.Factory callFactory;
//baseUrl信息
  final HttpUrl baseUrl;
//数据转换器工厂的集合，某些情况下可能会需要几层转换，所以是一个List集合
  final List<Converter.Factory> converterFactories;
//网络适配器工厂的集合，某些情况下可能需要几层网络适配器，分别用装饰器模式装饰，所以也是一个List集合
  final List<CallAdapter.Factory> callAdapterFactories;
//网络请求回调处理处，用于切换线程从子线程到主线，内部使用Handler切换线程
  final @Nullable Executor callbackExecutor;
//是否提前对接口中的注解进行验证转换
  final boolean validateEagerly;

  }
 //Retrofit类的构造函数
Retrofit(
      okhttp3.Call.Factory callFactory,
      HttpUrl baseUrl,
      List<Converter.Factory> converterFactories,
      List<CallAdapter.Factory> callAdapterFactories,
      @Nullable Executor callbackExecutor,
      boolean validateEagerly) {
    this.callFactory = callFactory;
    this.baseUrl = baseUrl;
    this.converterFactories = converterFactories; // Copy+unmodifiable at call site.
    this.callAdapterFactories = callAdapterFactories; // Copy+unmodifiable at call site.
    this.callbackExecutor = callbackExecutor;
    this.validateEagerly = validateEagerly;
  }
```

- 由源码可知一个标准的Retrofit实例应该包括以下元素：
  
  - `<mark>ServiceMethod</mark>：请求接口的方法注解对象信息`
  
  - `<mark>callFactory</mark>：网络请求工厂`
  
  - `<mark>baseUrl</mark>：baseUrl信息`
  
  - `<mark>converterFactories</mark>：数据转换工厂集合`
  
  - `<mark>callAdapterFactories</mark>：网络请求适配器工厂集合`
  
  - `<mark>callbackExecutor</mark>：网络请求回调处理器`
    
    这里我们看到有converterFactories和callAdapterFactories和callFactory等，从这些元素的字面意思就可以看到使用的是设计模式中的工厂模式：关于工厂模式可以看下面这篇文章：
    
    > [Android体系课 之 设计模式--工厂模式解析]()

- 这里我们来介绍下<mark>CallAdapterFactory</mark>这个类的使用
  
  - **作用：将默认的网络请求执行器OkHttpCall转换成适合不同平台的的网络请求执行器**
    
        `1.OkHttp默认的网络执行器为OkHttpCall`
    
        `2.Retrofit内部支持四种平台CallAdapterFactory：分别是Android，Java8，RxJava，Guava`
    
       ` 3.一般我们Android上使用默认的Android平台网络适配器就可以`
  
  **`继续来看步骤1.2,1.3,1,4`**
  
        

##### 模块2：创建请求的实体类，并配置网络请求参数，看2处

![图整体](C:\Users\Administrator\Desktop\笔记\开源框架\retrofit\源码分析\创建retrofit整体.png)

 进入Builder类中：

```java
public static final class Builder {
//Platform 标识具体使用哪个平台：Android？Java8？IOS？
    private final Platform platform;
//网络请求执行器
    private @Nullable okhttp3.Call.Factory callFactory;
//请求baseUrl
    private @Nullable HttpUrl baseUrl;
//响应数据转换器
    private final List<Converter.Factory> converterFactories = new ArrayList<>();
//网络请求适配器
    private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
//回调处理器   
    private @Nullable Executor callbackExecutor;
//验证标志位   
     private boolean validateEagerly;

    Builder(Platform platform) {
      this.platform = platform;
    }
//Builder构造函数：传入一个Platform为参数
    public Builder() {
      this(Platform.get());//...关注点1
    }
//Builder构造函数,传入一个Retrofit 对象，可用于已有Retrofit 对象，拷贝一个新的对象使用

    Builder(Retrofit retrofit) {
      platform = Platform.get();
      callFactory = retrofit.callFactory;
      baseUrl = retrofit.baseUrl;

      // Do not add the default BuiltIntConverters and platform-aware converters added by build().
      for (int i = 1,
              size = retrofit.converterFactories.size() - platform.defaultConverterFactoriesSize();
          i < size;
          i++) {
        converterFactories.add(retrofit.converterFactories.get(i));
      }

      // Do not add the default, platform-aware call adapters added by build().
      for (int i = 0,
              size =
                  retrofit.callAdapterFactories.size() - platform.defaultCallAdapterFactoriesSize();
          i < size;
          i++) {
        callAdapterFactories.add(retrofit.callAdapterFactories.get(i));
      }

      callbackExecutor = retrofit.callbackExecutor;
      validateEagerly = retrofit.validateEagerly;
    }    

//关注点1
  private static final Platform PLATFORM = findPlatform();
  static Platform get() {
    return PLATFORM;
  }  
  private static Platform findPlatform() {
    return "Dalvik".equals(System.getProperty("java.vm.name"))
        ? new Android() //如果系统参数"java.vm.name = Dalvik"
        : new Platform(true);//如果是其他的则new 一个Platform，这个true表示是否支持java8
  }  
//总结：关注点1：对于Android平台我们直接返回Android 的Platform，而如果是其他则创建一个默认Platform

//来看下Android Platform ：
static final class Android extends Platform {
    Android() {
      super(Build.VERSION.SDK_INT >= 24);
    }

    @Override
    public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();//这里创建一个默认MainThreadExecutor
    }

    static final class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());
        //这里创建了一个主线程上的handler
      @Override
      public void execute(Runnable r) {
        handler.post(r); //将任务放到主线程执行
      }
    }
  }


//总结Android:和普通的Platform的区别是创建了一个默认的MainThreadExecutor主线程执行器
```

-    我们注意到Platform中有一个<mark>hasJava8Type</mark>s字段，这个 字段是做什么的呢？
  
  我们来看下代码：
  
  ```java
  //在Retrofit 的build方法中调用了下面的两行代码
  
  public Retrofit build() {
  ...
  
  callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
  converterFactories.addAll(platform.defaultConverterFactories());
  ...
  
  }
  
  //继续跟踪到platform.defaultConverterFactories()
  
  List<? extends Converter.Factory> defaultConverterFactories() {
      return hasJava8Types ? singletonList(OptionalConverterFactory.INSTANCE) : emptyList();
      //这里看到如果hasJava8Types 为true，则会创建一个OptionalConverterFactory的单例类
  
  }
  ```

```
>  **可以看大这个字段会增加默认的 ConverterFactories和默认的CallAdapterFactories，这里点到为止，后面还会具体介绍这块**

##### 模块3： .baseUrl("url")

```java
  public Builder baseUrl(String baseUrl) {
    return baseUrl(HttpUrl.get(baseUrl));//步骤1
  }
  //继续看HttpUrl.get(baseUrl)
  public static HttpUrl get(String url) {
      //这里创建一个OkHttp的HttpUrl
      return new Builder().parse(null, url).build();
  }
  //步骤1
  public Builder baseUrl(HttpUrl baseUrl) {
      ...
      this.baseUrl = baseUrl;//设置Retrofit的baseUrl
  }
```

- 模块3只是对传入的baseUrl做了验证并将baseUrl传递给Retrofit

##### 模块4：

```java
addConverterFactory(GsonConverterFactory.create())
```

**对模块4我们从外到内来：**

`首先查看GsonConverterFactory.create()`

```java
public static GsonConverterFactory create() {  
  return create(new Gson());  
}
//创建了一个GsonConverterFactory 对象内部包含了一个Gson 对象
public static GsonConverterFactory create(Gson gson) {
    return new GsonConverterFactory(gson);
}

//GsonConverterFactory.java
@Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonResponseBodyConverter<>(gson, adapter);//关注点1
//这里可以看到Gson响应的时候是创建了一个GsonResponseBodyConverter对象并传入之前创建的gson对象和
//响应的 TypeAdapter适配器 ,后面对响应数据转换的时候需要使用到这个GsonResponseBodyConverter转换器
    }

  @Override
  public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonRequestBodyConverter<>(gson, adapter);
//这里可以看到Gson请求的时候是创建了一个GsonRequestBodyConverter对象并传入之前创建的gson对象和
//请求的 TypeAdapter适配器

//关注点1中创建了一个GsonResponseBodyConverter
//我们来看下这个类中的转换逻辑
@Override public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
//将数据放入一个JsonReader中
    try {
      return adapter.read(jsonReader);//使用adapter读取jsonReader中的数据，返回给客户端
    } finally {
      value.close();
    }
  }
```

    `再来看addConverterFactory（converter）`

```java
public Builder addConverterFactory(Converter.Factory factory) {  
  converterFactories.add(Objects.requireNonNull(factory, "factory == null"));  
  return this;
}
```

- 可以看到这里是将前面创建的GsonConverterFactory对象传递给Retrofit。

- 这里使用了<mark>抽象工厂</mark>创建ConverterFactory和<mark>策略模式</mark>根据传入的转换策略进行转换
  
  > **抽象工厂模式将实例使用和创建分离，解除类之间的耦合关系**
  > 
  > **策略模式可以抽取出程序执行过程中的关键逻辑，提供给不同的类去处理，客户端根据不同的需求传入对应的策略，满足开闭原则**

##### 模块5：.build()

查看内部源码

```java
public Retrofit build() {

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();//默认情况下callFactory 都是null
        //所有默认情况callFactory OkHttpClient等于
      }
//默认情况下callbackExecutor为null
      Executor callbackExecutor = this.callbackExecutor;    
      if (callbackExecutor == null) {    
//所以会进入这里面
//这个platform对于Android系统则子类为Android，这个之前分析过
//defaultCallbackExecutor = new MainThreadExecutor()是一个子线程切换主线程操作的回调执行器
        callbackExecutor = platform.defaultCallbackExecutor();        
      }

//defaultCallAdapterFactories = 
      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));//关注点1

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories =
          new ArrayList<>(
              1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);//这里设置之前传入的GsonConverter
      converterFactories.addAll(platform.defaultConverterFactories());//关注点2
//将Builder中创建的所以元素传递给Retrofit变为Retrofit的元素，使用了建造者模式
      return new Retrofit(
          callFactory,
          baseUrl,
          unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories),
          callbackExecutor,
          validateEagerly);
    }
//关注点1
 List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
      @Nullable Executor callbackExecutor) {
//这里创建DefaultCallAdapterFactory：如果是Android平台则返回这里创建的DefaultCallAdapterFactory 对象
    DefaultCallAdapterFactory executorFactory = new DefaultCallAdapterFactory(callbackExecutor);
    return hasJava8Types
        ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
        : singletonList(executorFactory);
  }
//关注点2：platform.defaultConverterFactories()
List<? extends Converter.Factory> defaultConverterFactories() {
//可以看到这里面如果hasJava8Types 为false也就是为Android平台则不会创建任何转化器
    return hasJava8Types ? singletonList(OptionalConverterFactory.INSTANCE) : emptyList();
 }
```

总结下模块5：

- **创建默认的callFactory请求器**

- **创建默认的回调请求器，如果是Android平台则返回MainThreadExcutor，主要负责子线程切换到主线程执行任务**

- **创建默认的响应数据Converter转换器**

- **最后将所有的元素都传递给Retrofit**

现在我们对步骤1中的所有的五个模块做个大总结：

- 初始化baseUrl

- 初始化网络请求器：默认为OkHttpClient

- 初始化回调请求器：Android系统默认为MainThreadExcutor

- 初始化默认的响应数据Converter转换器，我们平时使用比较多的是GsonConverter

- 初始化默认的网络请求适配器：默认为DefaultCallAdapterFactory

- 内部使用了多种设计模式：包括：建造者模式，单例模式，代理模式
  
  装饰器模式，策略模式等，模块之间解耦形成高内聚低耦合

#### 步骤1.2：创建请求的实体类，并配置网络请求参数

```java
//模块1
RequestInterface request = retrofit.create(RequestInterface.class);  
//模块2
Call<Result> call = request.login()
```

##### 模块1：retrofit.create(RequestInterface.class);

```java
public <T> T create(final Class<T> service) {
    validateServiceInterface(service); 
//是否需要提前验证，默认是不需要validateEagerly：false
//
    return (T)
        Proxy.newProxyInstance(
            service.getClassLoader(),
            new Class<?>[] {service},
            new InvocationHandler() {
              private final Platform platform = Platform.get();
              private final Object[] emptyArgs = new Object[0];

              @Override
              public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                  throws Throwable {
                // If the method is a method from Object then defer to normal invocation.
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                args = args != null ? args : emptyArgs;
                return platform.isDefaultMethod(method)
                    ? platform.invokeDefaultMethod(method, service, proxy, args)
                    : loadServiceMethod(method).invoke(args);//关注点5
              }
            });
  }


//是否需要提前验证，默认是不需要validateEagerly：false，既不需要提前验证，我们根据Android平台的逻辑来，这里先跳过
private void validateServiceInterface(Class<?> service) {

...
    if (validateEagerly) {
      Platform platform = Platform.get();
      for (Method method : service.getDeclaredMethods()) {//依次取出请求接口中的方法
        //这里两个判断一个是否是默认Method，另外一个是否是静态方法，两种都满足，才会调用loadServiceMethod：
        //isDefaultMethod的逻辑：hasJava8Types && method.isDefault();前面分析我们hasJava8Types = 默认为false，
        if (!platform.isDefaultMethod(method) && !Modifier.isStatic(method.getModifiers())) {
          loadServiceMethod(method);
        }
      }
    }

...
}
```

- 这里通过动态代理方式，在运行期动态创建了一个请求接口的代理类
  
  实际是对接口方法调用loadServiceMethod进行解析，并执行接口方法
  
  ```java
  Proxy.newProxyInstance(
              service.getClassLoader(),
              new Class<?>[] {service},
              new InvocationHandler() {
               private final Platform platform = Platform.get();
                private final Object[] emptyArgs = new Object[0];
  
                @Override
                public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                    throws Throwable {
                  ...
                  return platform.isDefaultMethod(method)
                      ? platform.invokeDefaultMethod(method, service, proxy, args)
                      : loadServiceMethod(method).invoke(args);
                      //我们的网络请求接口一般都会走到这里，我们直接来分析loadServiceMethod
                }
              });
  ```
  
  总结：**Retrofit的create方法使用动态代理的方式在运行期创建了一个请求接口的代理类**
  
  ##### 

##### 模块2：Call<Result> call = request.login()

**之前分析过request是一个请求接口的代理类：调用login方法
这个方法会调用动态代理的<mark>invoke</mark>方法：这个方法内部会调用<mark>loadServiceMethod(method).invok</mark>e方法**
`我们先来分析下<mark>loadServiceMethod</mark>：`

```java
ServiceMethod<?> loadServiceMethod(Method method) {
//这个serviceMethodCache是一个LinkedHashMap。从map中获取一个ServiceMethod对象
    ServiceMethod<?> result = serviceMethodCache.get(method);
//如果上面获取到ServiceMethod缓存，则直接返回result。不用继续执行下面操作
    if (result != null) return result;
//这里使用同步操作，防止多线程引起数据错误
    synchronized (serviceMethodCache) {
//再次从serviceMethodCache中获取ServiceMethod缓存对象，如果没有则调用ServiceMethod.parseAnnotations(this, method);
      result = serviceMethodCache.get(method);
      if (result == null) {
//这里调用ServiceMethod.parseAnnotations(this, method);解析方法
        result = ServiceMethod.parseAnnotations(this, method);//关注点1..分析内部操作可以看出来这里返回的是一个CallAdapted对象
//将解析后的方法注解信息放入到serviceMethodCache缓存中
        serviceMethodCache.put(method, result);//
      }
    }
//返回解析后的数据
    return result;
}
```

总结loadServiceMethod：去缓存中获取ServiceMethod对象信息，这个对象包含了我们创建请求的所有信息，
如果没有缓存，则调用ServiceMethod.parseAnnotations通过解析数据获取ServiceMethod请求信息

```java
//关注点1：
ServiceMethod.java中
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);//关注点2
    ...
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);//关注点3
  }
```

来看关注点2：

```java
  static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
  }
```

看到这里也是使用了建造者模式创建了一个RequestFactory：

`进入Builder.build()方法中看看`

```java
    RequestFactory build() {
      for (Annotation annotation : methodAnnotations) {
        //这里我们对方法注解进行解析
        parseMethodAnnotation(annotation);//关注点3
      }
//这里开始一直到返回处都是对注解的检测是否合规   
...
    if (!hasBody) {//这两个判断表示isMultipart和isFormEncoded修饰的情况下，一定需要有Body
        if (isMultipart) {
          throw methodError(
              method,
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError(
              method,
              "FormUrlEncoded can only be specified on HTTP methods with "
                  + "request body (e.g., @POST).");
        }
     }

...
//返回一个RequestFactory并持有这个Builder类
      return new RequestFactory(this); 
    }
```

`继续查看RequestFactory构造方法：`

```java
  RequestFactory(Builder builder) {
    method = builder.method;
    baseUrl = builder.retrofit.baseUrl;
    httpMethod = builder.httpMethod;
    relativeUrl = builder.relativeUrl;
    headers = builder.headers;
    contentType = builder.contentType;
    hasBody = builder.hasBody;
    isFormEncoded = builder.isFormEncoded;
    isMultipart = builder.isMultipart;
    parameterHandlers = builder.parameterHandlers;
    isKotlinSuspendFunction = builder.isKotlinSuspendFunction;
  }
```

可以看到这个类持有了一个网络请求的所有信息：

**method，baseUrl，httpMethod，headers，contentType，hasBody，isFormEncoded，isMultipart，isKotlinSuspendFunction等等**

`回到关注点1：`

```java
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);//关注点2
    ...
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);//关注点3
  }
```

- 刚才分析了关注点2中会返回一个请求的所有信息，在关注点3处作为HttpServiceMethod.parseAnnotations的参数传入

`我们继续看下面的HttpServiceMethod.parseAnnotations`

```java
  static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;//判断是否是Kotlin suspend方法

    Annotation[] annotations = method.getAnnotations();//获取method的注解信息
    Type adapterType;
    //这里返回一个网络请求适配器，内部是使用之前设置的适配器工厂DefaultCallAdapterFactory创建一个请求适配器
    CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit, method, adapterType, annotations);//关注点x1：
    ...
    //这里返回一个响应数据转换器ResponseConverter，内部是使用之前设置的响应转换器工厂创建一个响应转换器
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType)；//后面我们分析这里：记关注点x2；

    //网络请求器默认为okhttp3.Call.Factory
    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    //目前是用的Java不是kotlin协程挂起函数，返回的是一个CallAdapted
    //内部元素：包含所有注解信息的requestFactory，网络请求器callFactory，响应数据转换器responseConverter还有网络适配器callAdapter
    if (!isKotlinSuspendFunction) {
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } else if (continuationWantsResponse) {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForResponse<>(...);
    } else {
//如果是协程挂起函数则返回一个SuspendForBody
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>)
          new SuspendForBody<>(...);
    }
  }    
```

`来看关注点x1：`

```java
CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit, method, adapterType, annotations);//关注点x1：
//一层一层进入最终调用到：DefaultCallAdapterFactory的get方法
public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    return new CallAdapter<Object, Call<?>>() {
                public Type responseType() {
                    return responseType;
                }

                public Call<Object> adapt(Call<Object> call) {
                    return (Call)(executor == null ? call : new DefaultCallAdapterFactory.ExecutorCallbackCall(executor, call));
                }
            };
}
```

- 看到这里返回了一个CallAdapter实现了adapt方法:

- adapt方法返回一个ExecutorCallbackCall，我们来看这个类内部信息

```java
static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
        this.callbackExecutor = callbackExecutor;
        this.delegate = delegate;
    }

    public void enqueue(final Callback<T> callback) {}
    public Response<T> execute() throws IOException {}
}
```

- 这个类是一个网路请求器，内部调用<mark>代理模式</mark>代理Okhttp的请求
  
  **来看关注点x2：**

```java
Converter<ResponseBody, ResponseT> responseConverter = createResponseConverter(retrofit, method, responseType)；
//看如何创建的：
    private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
      Retrofit retrofit, Method method, Type responseType) {

        return retrofit.responseBodyConverter(responseType, annotations);

    }
    //继续跟进responseBodyConverter
    public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
        return nextResponseBodyConverter(null, type, annotations);
    }
    //继续
    public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
        for (int i = start, count = converterFactories.size(); i < count; i++) {
          Converter<ResponseBody, ?> converter =
              converterFactories.get(i).responseBodyConverter(type, annotations, this);
          if (converter != null) {
            //noinspection unchecked
            return (Converter<ResponseBody, T>) converter;
          }
        }
    }
    //可以看到这里面是通过遍历converterFactories，来创建一个Converter。
    //这里的converterFactory: = GsonConverterFactory
    //找到GsonConverterFactory的responseBodyConverter方法：
      public Converter<ResponseBody, ?> responseBodyConverter(
        Type type, Annotation[] annotations, Retrofit retrofit) {
        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
        return new GsonResponseBodyConverter<>(gson, adapter);
      }
```

- 这里创建了一个GsonResponseBodyConverter转换器，所以之后数据转换就是通过这个转换器
  
   `篇幅太长不再深入，需要看具体如何转换可以去看里面代码`

**现在总结下我们的loadServiceMethod：**

**内部通过对注解进行解析，并对每个方法创建了一个包含网络请求的所有信息的CallAdapted对象这个<mark>CallAdapted</mark>是ServiceMethod的子类
内部元素：包含所有注解信息的requestFactory，网络请求器callFactory，响应数据转换器responseConverter还有网络适配器callAdapter**

****

`然后在关注点5：执行了CallAdapted对象的invoke方法`

`来看下这个方法：`

```java
   static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
    private final CallAdapter<ResponseT, ReturnT> callAdapter;
    ...
    @Override
    protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
      return callAdapter.adapt(call);
    }
  }
```

**这个方法内部并没有invoke方法，猜测应该是在父类中，去看他的父类HttpServiceMethod**

```java
abstract class HttpServiceMethod<ResponseT, ReturnT> extends ServiceMethod<ReturnT> {
  final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }
}
```

果然在父类中实现了invoke方法：

可以看到这个方法会创建一个OkHttpCall对象，并将请求信息传递给他，了解OkHttp的源码的同学都知道这个已经走到了Okhttp层面了

并回调adapt，这个方法在子类CallAdapted中实现，

`我们来看子类中的实现：`

```java
    @Override
    protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
      return callAdapter.adapt(call);
    }
```

直接调用了网络请求适配器的callAdapter的adapt方法：

这个callAdapter = DefaultCallAdapterFactory中的内部类CallAdapter

`下面进入DefaultCallAdapterFactory内部类中的adapt方法看看`

`这个内部类在DefaultCallAdapterFactory.get方法中实现`

```java
DefaultCallAdapterFactory.java
public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    ...
    final Executor executor =
        Utils.isAnnotationPresent(annotations, SkipCallbackExecutor.class)
            ? null
            : callbackExecutor;

    return new CallAdapter<Object, Call<?>>() {
      @Override
      public Type responseType() {
        return responseType;
      }

      @Override
      public Call<Object> adapt(Call<Object> call) {
        return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
        //这里返回了一个ExecutorCallbackCall
        //包含元素：executor：线程回调执行器 前面分析过Android平台是一个MainThreadExcutor
        //包含元素：call：之前分析过是OkHttpCall的一个对象

      }
    };
  }
```

**从以上分析可知：
我们的invoke方法最终返回的是一个ExecutorCallbackCall的网络执行器：
这和使用中：Call<T> call = request.login()对应上
所以这个call = ExecutorCallbackCall的实例对象，后期调用同步和异步操作都是在这个基础上的**

#### 步骤1.3：请求执行器的执行操作

 `我们使用异步来分析call.enqueue()`

```java
从步骤2中给我们分析出call = ExecutorCallbackCall的实例对象
那我们把源码定位到ExecutorCallbackCall类中：
    static final class ExecutorCallbackCall<T> implements Call<T> {
        final Executor callbackExecutor;//回调执行器用于线程切换
        final Call<T> delegate;//这个Call = OkHttpCall

        @Override
        public void enqueue(final Callback<T> callback) {
          delegate.enqueue(
              new Callback<T>() {
                @Override
                public void onResponse(Call<T> call, final Response<T> response) {
                  callbackExecutor.execute(
                      () -> {
                        if (delegate.isCanceled()) {
                          // Emulate OkHttp's behavior of throwing/delivering an IOException on
                          // cancellation.
                          callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
                        } else {
                          callback.onResponse(ExecutorCallbackCall.this, response);
                        }
                      });
                }

                @Override
                public void onFailure(Call<T> call, final Throwable t) {
                  callbackExecutor.execute(() -> callback.onFailure(ExecutorCallbackCall.this, t));
                }
              });
    }
```

- **1.可以看到这里面使用了代理模式：将请求转发给OkHttpCall的enqueue方法，详细对OkHttp了解的同学对这块已经很熟悉了**

- **2.数据响应成功后通过回调执行器CallbackExcutor将数据返回**

到这里，我们已经分析了，从接口-注解到数据通过回调执行器返回的过程
整个网络逻辑已经分析完毕

### 请求过程中的UML图如下：

> 为了更加直观的看到类与类之间的关系图，我划了两幅UML类图希望大家看到可以更加明白

 **请求前的UML类图**

<img src="file:///C:/Users/Administrator/Desktop/笔记/开源框架/retrofit/源码分析/源码分析前.png" title="" alt="图1" width="714">

**到OkHttp层面的UML图：**

<img src="file:///C:/Users/Administrator/Desktop/笔记/开源框架/retrofit/源码分析/源码分析.png" title="" alt="图2" width="668">

**上面两幅UML类图关系可以通过请求器：ExecutorCallbackCall连接起来看，类和类之间使用组合代理等模式进行调用，**

## 5.总结

> Retrofit本质是通过多种设计模式，简化我们对OkHttp请求的构建，使用注解的方式对请求进行配置，大大节约了我们开发的工作量

**看完本篇，相信你对Retrofit已经有了更加清晰的认识，喜欢的可以关注下，后期会推出更多Android体系课，希望你能学到更多知识。**

Android体系课学习 之 开源框架学习

[Android体系课学习 之 网络请求库OkHttp使用方式(附Demo)]()

[Android体系课学习 之 网络请求库OkHttp看这一篇就够了]()

[Android体系课学习 之 网络请求库Retrofit使用方式(附Demo) - 掘金](https://juejin.cn/post/7103518717471883277)

[Android体系课学习 之 网络请求库Retrofit源码分析]()

[Android体系课学习 之 RxJava看这篇就够了]()

[Android体系课学习 之 RxJava操作符详解]()

[Android体系课学习 之 RxJava源码分析]()
