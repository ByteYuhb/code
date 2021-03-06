## 前言

- 开发过程中经常需要使用到网络请求，而OkHttp作为第一代网络框架，经久不衰，在众多框架中脱颖而出，且在近期已被谷歌官方推荐使用，一定有其原因：

- 今天我们就来讲下OkHttp这个高效框架背后的秘密。

         **Android体系课学习 之 开源框架系列**

[Android体系课学习 之 网络请求库OkHttp看这一篇就够了]()

[Android体系课学习 之 网络请求库Retrofit使用方式(附Demo)]()

[Android体系课学习 之 网络请求库Retrofit源码分析]()

[Android体系课学习 之 RxJava看这篇就够了]()

[Android体系课学习 之 RxJava操作符详解]()

[Android体系课学习 之 RxJava源码分析 ]()

## 目录

## 简介

> OkHttp是一个高效的网络请求开源框架

### 特点：

#### 1.支持同一个主机的所有请求使用同一个套接字，这个特点让OkHttp对连接池的复用成为了可能

#### 2.连接池的复用，大大减少了请求的延时

#### 3.使用透明的Gzip压缩请求体大大缩小了请求传输大小，关于Gzip我们只需要知道其是一种高效的数据压缩方式即

#### 4.对于响应，OkHttp会对其做一个缓存，在某些情况下，可以使用缓存中的内容

#### 5.基于<mark>拦截器</mark>的<mark>责任链模式</mark>对请求进行阶段处理，可以自定义拦截器，方便对请求二次处理

## 基本使用介绍

- 本次使用的OkHttp版本为：3.14.9
  
  OkHttp使用起来非常方便，下面几行代码就可以实现一个简单的网络请求
  
  ```java
   fun testOkHttp(){
      val client = OkHttpClient()//步骤1：创建一个OkHttpClient对象
         
      val rd1:RequestBody = formBodyBuilder.build()
      val request = Request.Builder().get().url("http://host:port/api").build()
      val response = client.newCall(request).execute()
      print(response.body()?.bytes())
   }
  ```
  
  ### 使用步骤：
  
  #### 步骤1：可以通过OkHttpClient.Builder配置一个新的OkHttpClient，重新配置连接超时时间，添加拦截器，添加代理，设置缓存策略等
  
  #### 步骤2：配置请求体：本例中配置的是Form表单格式的请求体，并添加了两个字段key1和key2
  
  #### 步骤3：这个步骤可以添加请求的url信息和添加RequestBody等操作
  
  #### 步骤4：创建网络请求执行器并执行execute操作，这里也可以使用enqueue异步调用请求
  
  

## 高级使用

        这里先介绍下我们平时使用网络请求的场景：

        **1.上传<mark>Form</mark>表单数据，如登录场景需要删除用户名和密码**
        **2.上传字符串数据**
        **3.上传文件**
        **4.下载文件**



        `现在来分别介绍下这几种情况`

#### 1.上传Form表单数据，如登录场景需要删除用户名和密码

- 创建一个FormBody对象，并传入对应的key和value值

```java
val formBodyBuilder:FormBody.Builder = FormBody.Builder()
                .add("key1","value1")
                .add("key2","value2");         
```

#### 2.上传字符串数据

- 上传字符串需要在创建网络请求的时候传入一个RequestBody
  
  ```java
  RequestBody requestStrBody  = RequestBody.create(MediaType.parse("text/plain;charset=utf-8"),"username:admin password 123456");
  Request.Builder requestBuilder = new Request.Builder().post(requestStrBody)
  ```

> 这个RequestBody是一个“text/plain”格式，“utf-8”编码格式的body类型，body内容为："username:admin password 123456"



#### 3.使用Post上传文件

-    上传文件擦操作和字符串类似，只是MediaType类型是application/octet-stream表示以任意二进制格式上送，当然也可以使用image/png格式
  
  ```java
  RequestBody requestBody2 = RequestBody.create(MediaType.parse("application/octet-stream"), file);
  
  
  ```

- 这里记得添加文件的写操作权限：
  
  ```java
  <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
  ```

#### 4.使用get进行文件下载

-   请求时指定get方式请求 

```java
Request request = new Request.Builder()
            .get()
            .url("https://www.baidu.com/img/bd_logo1.png")
			.build()    
```

- 回调的时候将数据放入文件：

```java
InputStream is = response.body().byteStream();
			int len = 0;
			File file  = new File(Environment.getExternalStorageDirectory(), "n.png");
			FileOutputStream fos = new FileOutputStream(file);
			byte[] buf = new byte[128];

			while ((len = is.read(buf)) != -1){
				fos.write(buf, 0, len);
			}

			fos.flush();
			//关闭流
			fos.close();
			is.close();    
```

- 这里记得添加文件的写操作权限：
  
  ```java
  <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
  ```

#### 5.既实现Form表单上送也可以实现文件上送

-   使用MultipartBody可以实现既上传表单又上传文件
  
  ```java
  FormBody.Builder formBodyBuilder = new FormBody.Builder()
                  .add("key1","value1")
                  .add("key2","value2");
  		RequestBody requestBody = formBodyBuilder.build();
          RequestBody rd = RequestBody.create(MediaType.parse("application/octet-stream"), new File("path/1.png"));
          RequestBody bodyPart  = new MultipartBody.Builder()
                  .addPart(requestBody)
                  .addPart(rd)
                  .build(); 
  ```
  
  > 使用上面的方式将表单数据username和password上传到服务器，也将本地文件path/1.png进行了上传

   

    上面已经讲解几种常用的网络请求场景：[*Demo*](https://github.com/ByteYuhb/demoOkhttp)

## 源码分析

上面简单讲解了OkHttp的使用方式：下面我将会从源码的角度来分析OkHttp的强大之处。

`此次分析的源码版本：3.14.9`

`大部分具体分析会在代码内部根据流程来执行，源码外部是对过程的一个总结`

我们按<mark>四个步骤</mark>来查看源码：

**1.分析使用流程**

**2.分析源码中的重要组件**

**3.根据大概使用流程画出流程图**

**4.根据使用流程对代码进行具体源码分析**

> 使用流程贯彻了整个分析过程

### 我们再来列下OkHttp的使用步骤：

##### 步骤1：可以通过OkHttpClient.Builder配置一个新的OkHttpClient

```java
val client:OkHttpClient = OkHttpClient.Builder()
                .connectTimeout(5000,TimeUnit.MILLISECONDS)
                .addInterceptor(userInterceptor)
                .build()  
```

##### 步骤2：配置请求体：本例中配置的是Form表单格式的请求体，并添加了两个字段key1和key2

```java
val formBody:FormBody = FormBody.Builder()
                .add("username","admin")
                .add("password","123456")
                .build();
```

##### 步骤3：这个步骤可以添加请求的url信息和添加RequestBody等操作

```java
val request = Request.Builder().post(requestBody).url(baseUrl).build()
```

##### 步骤4：创建网络请求执行器并执行execute操作，这里也可以使用enqueue异步调用请求

```java
val response = client.newCall(request).execute()
```

##### 步骤5：根据响应结果头部数据长度获取响应body

```java
printR(response.body()?.bytes())
```

基本流程分析完，下面来分析下源码中的几个重要组件

### 再来介绍下源码部分的几个主要角色：

- <mark>OkHttpClient</mark>：框架核心管理类，内部创建了很多默认属性，和应用直接接触

- <mark>RealCal</mark>l：负责请求的调度，对于同步请求直接调用execute方法执行，对于异步请求使用enqueue方法，将请求放到他的默认线程池中最终还是调用了execute方法实现后续操作

- <mark>Request</mark>：请求实体类，封装了客户端的请求信息，包括url，method，header还有body等信息

- <mark>Response</mark>：响应实体类，封装了服务器返回的应答信息，如code，message，header和body等信息

- <mark>Dispatcher</mark>：分发器，对请求任务进行异步分发，内部使用线程池进行分发

- <mark>Cache</mark>：响应缓存，默认的缓存策略使用的<mark>DiskLruCache</mark>，写入缓存过程也是使用线程池操作，不会阻塞请求线程。
      只对Get请求有相应的缓存，这样做性价比比较高，因为大部分Get请求是客户端需要响应数据，而post或者其他情况请求大部分是为了把数据发送给服务器，而不是获取服务器的结果，
      如果需要使用其他缓存策略可以在创建OkHttpClient请求的时候从外部设置

- <mark>ConnectionPool</mark>：对原有连接的一种复用机制

- <mark>dns</mark>：dns服务器解析后的ip和端口信息

- <mark>Interceptor</mark>：拦截器使用的是责任链模式对网络请求进行处理
  
  主要包括以下几种：
  
  **自定义用户拦截器：**
  
  > 在拦截器开始前使用自定义拦截器对请求状态进行检查，增加网络请求粘性，可以不设置
  
  **重试拦截器(RetryAndFollowUpInterceptor)：**
  
  > 这个拦截器主要是对失败进行重试或者重定向操作
  
  重试条件：
  
  ` 1.应用层禁止重试，则不能重试`
   `2.如果请求已经发送，且返回了FileNotFoundException或者isOneShot标志位true，oneShot指请只发送一次，则不会进行重试`
    `3.异常是一个崩溃导致的不会进行重试`
   ` 4.没有其他路由可以尝试了，不能重试`
    `5.以上情况都没有，则进行重试``

        

        **桥接拦截器(BridgeInterceptor)：**

        这个拦截器主要作用个如下：

            `1.负责把用户构造的请求转换为发送到服务器的请求 、把服务器返回的响应转换为用户友好的响应，是从应用程序代码到网络代码的桥梁`
            `2.设置内容长度，内容编码`
            `3.设置gzip压缩，并在接收到内容后进行解压。省去了应用层处理数据解压的麻烦`
            `4.添加cookie`
             `5.设置其他报头，如User-Agent,Host,Keep-alive等。其中Keep-Alive是实现连接复用的必要步骤`

       **缓存拦截器(CacheInterceptor)**

        缓存拦截器的使用机制：

            `1.如果禁止使用网络请求且缓存响应为空，则直接返回一个504的错误码`
            `2.如果网络请求被禁止，且获取到cacheResponse不为空，则直接返回缓存数据`
            `3.如果缓存数据不为空，且服务器返回了HTTP_NOT_MODIFIED 304则返回缓存数据`
            `4.如果cache不为null且Header头部的hasBody为true，且缓存器可使用状态，则将响应数据写到缓存中，并返回`
            `5.缓存器如果在使用的过程中出现异常，记得清空缓存，防止内存溢出`

        **连接拦截器(ConnectInterceptor)**

            连接拦截器做了哪些事情？
            `1.先检查transmitter中的connection是否可用，可用标志位是connection的noNewExchanges，如果为true，则说明不可用，并释放socket，这个是在CallServerIntercept中赋值的`，
            `2.再检查连接池中是否有可用的连接
        3.如果1或2中找到可以复用的连接，则可以省略重新创建socket和dns解析，连接，握手等操作，最大限度的缩短通讯时间，提升性能
        4.如果没有可复用的连接，则创建一个新的RealConnection,并调用tcp的握手操作，如果是https则还会调用TLS的握手操作，调试过程中如果出现握手失败的情况，可以通过这里面查看握手协议是否有更改
        5.连接成功后，再次查看连接池中是否有可以复用的连接池。这种情况出现在同一主机对服务器有并发连接的情况
        6.如果5中没有在连接池中获取到可复用的连接，则将4中创建的连接实例放入到连接池中，方便下次连接复用使用
        7.如果5中在连接池中找到可复用的连接，则将4中创建的连接关闭，返回5中的连接池的连接`

``

            **自定义网络拦截器(networkInterceptors)：**

> 这个一般情况下不用设置，只有对网络连接有要求的情况下才会去设置

            **请求服务拦截器(CallServerInterceptor)：**

主要职责：

`   发送数据头，发送数据body，接收数据头，如果数据头显示有content，则创建一个body的response，最后调用response.body().bytes()才会去取body数据
    核心操作都是之前获取的HttpCodec 进行发送和接收`

### 下面来看下OkHttp请求的实体流程图

图

### 现在来具体进入源码看看

##### 步骤1：可以通过OkHttpClient.Builder配置一个新的OkHttpClient

```java

        val client:OkHttpClient = OkHttpClient.Builder()
                .connectTimeout(5000,TimeUnit.MILLISECONDS)
                .addInterceptor(userInterceptor)
                .build()
```

我们进入OkHttpClient源码看看：

`OkHttpClient继承了Call.Factory, WebSocket.Factory 实现了Cloneable接口
        所以这是一个网路请求器，也可以作为WebSocket请求器`

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
		  //默认支持HTTP1.1和HTTP2
		  final List<Protocol> DEFAULT_PROTOCOLS = Util.immutableList(Protocol.HTTP_2, Protocol.HTTP_1_1);
		  //这里指定了默认的连接属性： --关注点1--
		  final List<ConnectionSpec> DEFAULT_CONNECTION_SPECS = Util.immutableList(ConnectionSpec.MODERN_TLS, ConnectionSpec.CLEARTEXT);
		  //这个是请求分发器，用于对请求任务进行同步和异步分发  --关注点2--
		  final Dispatcher dispatcher;
		  //如果是请求的代理，可以通过设置代理转发请求事件
		  final @Nullable Proxy proxy;
		  //支持的协议类型，默认支持HTTP1.1和HTTP2
		  final List<Protocol> protocols;
		  //支持的TLS协议类型，默认支持TLS1.2和1.3
		  final List<ConnectionSpec> connectionSpecs;
		  //请求拦截器，用于对事件进行拦截处理。拦截器后面会详细讲解
		  final List<Interceptor> interceptors;
		  //网络拦截器，一般不需要设置
		  final List<Interceptor> networkInterceptors;
		  //事件回调器，比如请求状态，dns执行状态回调等
		  final EventListener.Factory eventListenerFactory;
		  //代理选择器
		  final ProxySelector proxySelector;
		  //设置Cookie，可以在外部设置
		  final CookieJar cookieJar;
		  //缓存策略， --关注点3--
		  final @Nullable Cache cache;
		  //框架内置的缓存策略， --关注点3--
		  final @Nullable InternalCache internalCache;
		  //创建Socket请求的工厂类：默认为：DefaultSocketFactory实例对象
		  final SocketFactory socketFactory;
		  //Https请求的SSLSocketFactory工厂类
		  final SSLSocketFactory sslSocketFactory;
		  //证书链清理者，看名字就知道是用于对证书进行清理的
		  final CertificateChainCleaner certificateChainCleaner;
		  //主机名称校验者，通讯过程中可以使用这个方法回调对主机名进行合法性确认
		  final HostnameVerifier hostnameVerifier;
		  //这个类主要用来针对有对固定的证书进行限制的情况：
		  //如你不想要请求访问某个主机名或者代理主机，可以将主机的主机名和对应证书编号sha设置到这个certificatePinner中即可,有一定风险，不建议使用
		  final CertificatePinner certificatePinner;
		  //在连接到代理服务器前对代理服务器进行合法身份验证
		  final Authenticator proxyAuthenticator;
		  //在连接到web服务器前对web服务器进行合法身份验证
		  final Authenticator authenticator;
		  //连接复用池 --关注点4--
		  final ConnectionPool connectionPool;
		  //主机dns信息 --关注5--
		  final Dns dns; //dns解析器，使用系统的dns解析器
		  ...

		}
```

`来看--<mark>关注点1</mark>--：`

```java
public static final ConnectionSpec MODERN_TLS = new Builder(true)
			  .cipherSuites(APPROVED_CIPHER_SUITES)
			  .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
			  .supportsTlsExtensions(true)
			  .build();
			}
```

- 这里看到OkHttp默认支持的TLS版本为TLS1.3和TLS1.2，这两个TLS版本可以满足大部分服务器要求，所以OkHttp做了这个默认配置

`再来看--<mark>关注点2</mark>--，我们来看下分发器内部结构：`

> 开源作者对Dispatcher的描述：
> 
> 用于对请求任务进行异步分发，内部通过线程池进行处理，用户也可以设置自己的线程池，但是应该提供线程池并发执行的线程最大数量

```java
public final class Dispatcher {
	  //分发器最大请求数
	  private int maxRequests = 64;
	  //单个主机最大请求数
	  private int maxRequestsPerHost = 5;
	  //分发器空闲状态回调，可以让分发器的没有任务的时候可以回调做一些事情
	  private @Nullable Runnable idleCallback;
	  //线程池执行器
	  /** Executes calls. Created lazily. */
	  private @Nullable ExecutorService executorService;

	  //即将被执行的请求任务会放到这个readyAsyncCalls队列中
	  /** Ready async calls in the order they'll be run. */
	  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();
	  
	  //正在执行的异步任务会被放入到这个runningAsyncCalls队列中
	  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
	  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

	  //正在执行的同步任务会被放入这个runningSyncCalls队列中
	  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
	  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
	  
	  //可以在外部设置请求执行器，但根据要求需要设置最大并发线程数，默认的线程池执行器：ThreadPoolExecutor
	  public Dispatcher(ExecutorService executorService) {
		this.executorService = executorService;
	  }

	  public Dispatcher() {
	  }
	  //这里是设置默认的executorService为ThreadPoolExecutor
	  //核心线程数为0，非核心线程数最大值无限制，非核心线程保留时间为60s，阻塞队列使用SynchronousQueue队列FIFO处理
	  public synchronized ExecutorService executorService() {
		if (executorService == null) {
		  executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
			  new SynchronousQueue<>(), Util.threadFactory("OkHttp Dispatcher", false));
		}
		return executorService;
	  }

}
```

对Dispatcher进行总结：

**内部使用三个队列**：

    <mark>readyAsyncCalls</mark>：保存“准备执行的异步请求”

    <mark>runningAsyncCalls</mark>：保存“正在执行的异步请求”

   <mark> runningSyncCalls</mark>：保存“正在执行的同步请求”

> 默认线程池执行器：使用核心线程数为0，非核心线程数最大值无限制，非核心线程保留时间为60s，阻塞队列使用SynchronousQueue队列FIFO处理
>             用户也可以自定义线程池执行器，但必须确保最大并发线程数



`来看--关注点3--`

**缓存器Cache**

```java
public final class Cache implements Closeable, Flushable {
	//这是OkHttp内置的一个缓存策略，调用自身的get方法，请求实体作为参数去获取，
	//使用内置缓存而不直接继承缓存并实现，主要是满足开闭原则，让内部接口只对自身开放，用一个内置对象和外部沟通
	final InternalCache internalCache = new InternalCache() {
		@Override public @Nullable Response get(Request request) throws IOException {
		  return Cache.this.get(request);
		}

		@Override public @Nullable CacheRequest put(Response response) throws IOException {
		  return Cache.this.put(response);
		}
		...
	};
	  //这里表面系统默认使用的是DiskLruCache磁盘缓存
	  final DiskLruCache cache;
	  //缓存的key是通过对url进行md5加密得到，所以多同一url的请求都会优先从缓存中获取
	  public static String key(HttpUrl url) {
		return ByteString.encodeUtf8(url.toString()).md5().hex();
	  }
	  //这是获取缓存的方式
	  @Nullable Response get(Request request) {
		//这里获取缓存的key，上面介绍过使用的是url的md5加密作为key
		String key = key(request.url());
		/
		DiskLruCache.Snapshot snapshot;
		Entry entry;
		try {
		  //获取缓存的快照
		  snapshot = cache.get(key);
		  //没有缓存直接返回null给上级
		  if (snapshot == null) {
			return null;
		  }
		} catch (IOException e) {
		  // Give up because the cache cannot be read.
		  return null;
		}

		try {
		  //从缓存快照中拿到Entry对象
		  entry = new Entry(snapshot.getSource(ENTRY_METADATA));
		} catch (IOException e) {
		  Util.closeQuietly(snapshot);
		  return null;
		}
		//从entry对象中获取响应数据，从这里也可以推断出，
		//缓存存储步骤：1.将响应数据放入一个entry对象中，2.然后将entry对象作为数据元放入snapshot缓存快照中，3.最后将快照存入本地磁盘中
		Response response = entry.response(snapshot);
		//检测快照是否合法
		if (!entry.matches(request, response)) {
		  Util.closeQuietly(response.body());
		  return null;
		}
		//合法直接返回对应的请求的缓存数据
		return response;
	  }

	  @Nullable CacheRequest put(Response response) {
		...
		if (!requestMethod.equals("GET")) {
		  //这里可以看出默认缓存策略只会缓存get请求的响应数据，对于其他方式不会做缓存，
		  //如果需要缓存其他数据，可以通过自定义缓存策略实现
		  return null;
		}
		//创建一个Entry对象包裹response
		Entry entry = new Entry(response);
		DiskLruCache.Editor editor = null;
		try {
		//创建一个url对应的editor对象
		  editor = cache.edit(key(response.request().url()));
		  if (editor == null) {
			return null;
		  }
		  //将entry写入editor，这里面会将缓存数据写到磁盘中
		  entry.writeTo(editor);
		  //返回一个包含editor的CacheRequestImpl对象
		  return new CacheRequestImpl(editor);
		} catch (IOException e) {
		  abortQuietly(editor);
		  return null;
		}
	  }


}
```

**这里总结下Cache：**

- 系统针对Get请求内置了一个DiskLruCache的磁盘缓存策略，缓存写入过程也是通过线程池（只提供了一个非核心线程，说明同一时间只能有一个线程操作磁盘数据）做的异步缓存操作，不会阻塞网络请求线程
  

- 用户可以根据需求自定义缓存策略。

`<mark>来看 --关注点4--</mark>`

**ConnectionPool.java**

```java
public final class ConnectionPool {
//这里可以看到使用的是一个代理模式：ConnectionPool代理了RealConnectionPool的执行
  final RealConnectionPool delegate;
	
  //这里对连接池官方的解释：
  //连接池的参数在以后的OkHttp版本中可能会发生变化，连接池中会有5个默认的空闲连接，如果5分钟没有活动的connection则，将会把这些connections删除
  /**
   * Create a new connection pool with tuning parameters appropriate for a single-user application.
   * The tuning parameters in this pool are subject to change in future OkHttp releases. Currently
   * this pool holds up to 5 idle connections which will be evicted after 5 minutes of inactivity.
   */
  public ConnectionPool() {
	this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
  //这里创建一个真实的RealConnectionPool连接池
	this.delegate = new RealConnectionPool(maxIdleConnections, keepAliveDuration, timeUnit);//关注点4.1--
  }
	
	//返回连接池中空闲连接的个数，如果没有空闲连接，则不会再创建新的连接，会等待前面连接使用完毕
  /** Returns the number of idle connections in the pool. */
  public int idleConnectionCount() {
	return delegate.idleConnectionCount();
  }
	//返回连接池中的所有连接个数
  /** Returns total number of connections in the pool. */
  public int connectionCount() {
	return delegate.connectionCount();
  }
	//关闭连接池中所有的连接
  /** Close and remove all idle connections in the pool. */
  public void evictAll() {
	delegate.evictAll();
  }
}
来看4.1处：
RealConnectionPool构造方法
	public final class RealConnectionPool {
		//使用ArrayDeque来存储连接池
		private final Deque<RealConnection> connections = new ArrayDeque<>();
		//这个方法是请求实体去连接池中查看可复用的连接情况：
		
		boolean transmitterAcquirePooledConnection(Address address, Transmitter transmitter,
			  @Nullable List<Route> routes, boolean requireMultiplexed) {
			assert (Thread.holdsLock(this));
			for (RealConnection connection : connections) {
			  if (requireMultiplexed && !connection.isMultiplexed()) continue;
			  if (!connection.isEligible(address, routes)) continue;//这里判断连接情况关注点4.1.1
			  transmitter.acquireConnectionNoEvents(connection);
			  return true;
			}
			return false;
		}
	}
	
	
来看 --关注点4.1.1--
这个方法用来判断一个连接是否可用
boolean isEligible(Address address, @Nullable List<Route> routes) {
	//如果连接这个链接不能接受新的请求exchanges，则返回false
	// If this connection is not accepting new exchanges, we're done.
	if (transmitters.size() >= allocationLimit || noNewExchanges) return false;
	//如果传入的地址的dns和connection中的dns不一致，则返回false，这个equalsNonHost在OkHttpClient中
	// If the non-host fields of the address don't overlap, we're done.
	if (!Internal.instance.equalsNonHost(this.route.address(), address)) return false;
	
	//如果到这里主机名对应上则返回true
	// If the host exactly matches, we're done: this connection can carry the address.
	if (address.url().host().equals(this.route().address().url().host())) {
	  return true; // This connection is a perfect match.
	}
	
	//如果以上都对应不上，还有一种复用的可能：要求如下
	//1.请求是HTTP/2 2 路由使用的是同一个ip地址 3.连接使用的服务器证书涵盖了新的请求 4.新的请求可以通过原网络connection的挂起证书
	//满足以上条件的请求也可以复用
	
	// 1. This connection must be HTTP/2.
	if (http2Connection == null) return false;

	// 2. The routes must share an IP address.
	if (routes == null || !routeMatchesAny(routes)) return false;

	// 3. This connection's server certificate's must cover the new host.
	if (address.hostnameVerifier() != OkHostnameVerifier.INSTANCE) return false;
	if (!supportsUrl(address.url())) return false;

	// 4. Certificate pinning must match the host.
	try {
	  address.certificatePinner().check(address.url().host(), handshake().peerCertificates());
	} catch (SSLPeerUnverifiedException e) {
	  return false;
	}

	return true; // The caller's address can be carried by this connection.
	
}
```

**总结ConnectionPool：**

> 使用代理模式实现了对连接吃的复用操作，实际其作用给的是RealConnectionPool，连接池内部存储了5个空闲的连接，如果五分钟内没有任何请求，则会删除连接池中的空闲连接

- 连接池复用逻辑如下：
  
  **1.如果超过了连接的最大连接数，则不能复用**
  **2.如果传入的地址的dns和connection中的dns不一致，则不能复用**
  **3.以上两种情况都可以后，判断主机名是否一致，如果一致则可复用**
  **4.在上面情况下，如果只有主机名不一致，则还可以有一种复用情况**：
     `  1)请求是HTTP/2 `
  
  `   2)路由使用的是同一个ip地址 
      3).连接使用的服务器证书涵盖了新的请求 4.新的请求可以通过原网络connection的挂起证书
      满足上面几个条件，也是可以拿来复用的`

来看下 --关注点5--

```java
public interface Dns {
	//这里使用lambda表达式创建了一个系统的DNS解析器SYSTEM：其实就是lookup方法的实现
	  Dns SYSTEM = hostname -> {
		if (hostname == null) throw new UnknownHostException("hostname == null");
		try {
		  return Arrays.asList(InetAddress.getAllByName(hostname));
		} catch (NullPointerException e) {
		  UnknownHostException unknownHostException =
			  new UnknownHostException("Broken system behaviour for dns lookup of " + hostname);
		  unknownHostException.initCause(e);
		  throw unknownHostException;
		}
	  };
	  
	  List<InetAddress> lookup(String hostname) throws UnknownHostException;
	}
	继续查看InetAddress.getAllByName(hostname)：
	public static InetAddress[] getAllByName(String host)
		throws UnknownHostException {
		//看官方解释目前Android使用的是Libcore.os动态库对主机dns进行解析
		// Android-changed: Resolves a hostname using Libcore.os.
		// Also, returns both the Inet4 and Inet6 loopback for null/empty host
		return impl.lookupAllHostAddr(host, NETID_UNSET).clone();
}
```

**总结下DNS解析:**

> Android系统的dns解析过程是通过Libcore.os库对主机名进行解析获取，返回的是一个List的ip和端口InetAddress对象，后面就涉及到NDK的内容，这里就不在深入





<u>现在我们已经对OkHttpClient中的所有属性做了一个比较深入的讲解，有以上基础再去看下面步骤应该会相对简单点了，</u>
        <mark><u>继续看后面的步骤</u></mark>





##### 步骤2：配置请求体：本例中配置的是Form表单格式的请求体，并添加了两个字段key1和key2

```java
val formBody:FormBody = FormBody.Builder()
                .add("username","admin")
                .add("password","123456")
				.build();    
```

这里没有太多可讲的，内部就是创建一个FormBody实体，实体内部包括两个List分别存储names和values    



##### 步骤3：这个步骤可以添加请求的url信息和添加RequestBody等操作

```java
val request = Request.Builder().post(requestBody).url(baseUrl).build()
//来看Request的构造器：
		  Request(Builder builder) {
			this.url = builder.url;
			this.method = builder.method;
			this.headers = builder.headers.build();
			this.body = builder.body;
			this.tags = Util.immutableMap(builder.tags);
		  }
```

构造器中包括：url请求，method请求方法，header头文件信息，body请求信息等信息



##### 步骤4：创建网络请求执行器并执行execute操作，这里也可以使用enqueue异步调用请求

- 1.网路请求器创建
  
  ```java
  client.newCall(request)
  ```

```java
 public Call newCall(Request request) {
	return RealCall.newRealCall(this, request, false /* for web socket */);
	}
	继续看：RealCall.newRealCall
	static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
	// Safely publish the Call instance to the EventListener.
	RealCall call = new RealCall(client, originalRequest, forWebSocket);
	call.transmitter = new Transmitter(client, call);
	return call;
}   
```

这里创建了一个RealCall执行器，并将请求信息封装到一个transmitter对象中，并把这个transmitter对象放到RealCall对象中，返回RealCall对象

`来看RealCall内部：`

```java
final class RealCall implements Call {
			final OkHttpClient client;
			private Transmitter transmitter;
		    final Request originalRequest;	
		  }
```

可以看到RealCall内部就是对OkHttpClient和Request请求做了一次再封装



2.网络请求执行：

call..execute():这个call是RealCall的实例

```java

public void enqueue(Callback responseCallback) {
	//这里判断请求是否已经执行，如果已经执行则报异常退出
	synchronized (this) {
	  if (executed) throw new IllegalStateException("Already Executed");
	  executed = true;
	}
	//这里面是将请求信息报告给eventListener
	transmitter.callStart();
	//这里是真实调用的地方：先用装饰器模式将responseCallback封装到AsyncCall中，关注点1
	client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

`继续看<mark>关注点1</mark> client.dispatcher().enqueue(new AsyncCall(responseCallback));：`

实际调用的是：Client中的Dispatcher的enqueue方法，传参new AsyncCall(responseCallback)

进入看看：

```java
void enqueue(AsyncCall call) {
	//调用synchronized同步
	synchronized (this) {
	  //将call请求放入到readyAsyncCalls中，这个是准备发送的异步请求
	  readyAsyncCalls.add(call);
	  ...
	}
	promoteAndExecute();
}
继续看promoteAndExecute
private boolean promoteAndExecute() {

	List<AsyncCall> executableCalls = new ArrayList<>();
	boolean isRunning;
	synchronized (this) {
	  for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
		AsyncCall asyncCall = i.next();
		//将readyAsyncCalls队列中的请求放入到executableCalls中，待请求
		executableCalls.add(asyncCall);
		//将readyAsyncCalls队列中的请求放入到runningAsyncCalls中
		runningAsyncCalls.add(asyncCall);
	  }
	}

	for (int i = 0, size = executableCalls.size(); i < size; i++) {
	  AsyncCall asyncCall = executableCalls.get(i);
	  //这里发送executableCalls中的请求使用线程池进行分发
	  asyncCall.executeOn(executorService());
	}
	...
	return isRunning;
}
```

网络请求到这里就是使用线程池对请求进行分发了：
分发的任务：在AsyncCall的父类中：

```java
public abstract class NamedRunnable implements Runnable {
				 
  @Override public final void run() {
	...
	try {
	  execute();
	} finally {
	  Thread.currentThread().setName(oldName);
	}
  }

  protected abstract void execute();
}
//可以看到还是执行的AsyncCall的execute方法
protected void execute() {
  ...
  try {
	//调用getResponseWithInterceptorChain方法 -- 关注点2--
	Response response = getResponseWithInterceptorChain();
	//这里调用responseCallback回调响应数据
	responseCallback.onResponse(RealCall.this, response);
  } catch (IOException e) {
	
  } catch (Throwable t) {
	
  }
}
```



很多同学看我上面可能会一头雾水了，我这边画了一张类图总结上面的分析:

图



<mark>**<u>下面就是我们框架的核心点：拦截器的使用：</u>**</mark>

首先我们要了解这里使用的是一种<mark>责任链模式</mark>的设计模式

关于责任链模式可以参考这边文章：。。。



我们继续来看关注点2：<u><mark>getResponseWithInterceptorChain</mark></u>

```java
  Response getResponseWithInterceptorChain() throws IOException {
	// Build a full stack of interceptors.
	List<Interceptor> interceptors = new ArrayList<>();
	//这个是应用自定义拦截器，放在所有的责任链head部位，可以在这里对请求做个再次封装或者判断 --拦截器1--
	interceptors.addAll(client.interceptors());
	//重试或重定向拦截器 --拦截器2--
	interceptors.add(new RetryAndFollowUpInterceptor(client));
	//桥接拦截器 --拦截器3--
	interceptors.add(new BridgeInterceptor(client.cookieJar()));
	//缓存拦截器 --拦截器4--
	interceptors.add(new CacheInterceptor(client.internalCache()));
	//连接拦截器 --拦截器5--
	interceptors.add(new ConnectInterceptor(client));
	if (!forWebSocket) {
	  //用户自定义网络拦截器 --拦截器6--
	  interceptors.addAll(client.networkInterceptors());
	}
	//请求服务器拦截器 --拦截器7
	interceptors.add(new CallServerInterceptor(forWebSocket));

	//这里创建一个RealInterceptorChain并将拦截器列表和交易请求信息封装到这个对象内部
	Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
		originalRequest, this, client.connectTimeoutMillis(),
		client.readTimeoutMillis(), client.writeTimeoutMillis());
	try {
	  //调用proceed方法 --关注点3--
	  Response response = chain.proceed(originalRequest);
	  return response;
	} catch (IOException e) {
	  ...
	} finally {
	  ...
	}
}
来查看 --关注点3--
  public Response proceed(Request request) throws IOException {
	return proceed(request, transmitter, exchange);
  }
	
   public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange)
	  throws IOException {
	//首先需要知道这个index是目前执行的拦截器索引：如果超过索引，直接抛出异常
	if (index >= interceptors.size()) throw new AssertionError();
	calls++;
	...
	// Call the next interceptor in the chain.
	//这里创建了一个新的RealInterceptorChain请求，且index做了+1操作
	RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
		index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
	Interceptor interceptor = interceptors.get(index);
	//这里执行了拦截器的intercept方法并传入下一个RealInterceptorChain拦截器责任链 --关注点4--
	Response response = interceptor.intercept(next);
	...
	return response;
  }
```

**总结这个方法作用：主要是设置多个拦截器，然后执行拦截器**

##### <mark><u>**下面来看下拦截器的执行过程：**</u></mark>

`来看 --关注点4--`
       我们知道第一个拦截器是用户自定义拦截器，如果我们没有自定义则默  认跳过， 那么直接来看 ：

###### **<mark>拦截器2--RetryAndFollowUpInterceptor</mark>**：重试和重定向拦截器

```java
public Response intercept(Chain chain) throws IOException {
	//获取网络请求Request
	Request request = chain.request();
	//将chain强制转换为RealInterceptorChain，本身也是RealInterceptorChain
	RealInterceptorChain realChain = (RealInterceptorChain) chain;
	//获取realChain中的Transmitter，这个Transmitter中包含了所有的网络请求信息，包括网络执行器OkHttpClient中的信息
	Transmitter transmitter = realChain.transmitter();
	
	//初始化重定向次数为0
	int followUpCount = 0;
	Response priorResponse = null;
	//看到这里使用了一个While循环，如果需要重试会调用continue回到这里
	while (true) {
	  //准备连接请求 关注点4.1
	  transmitter.prepareToConnect(request);
	...
	  try {
		// 这里准备调用下一个拦截器 。根据拦截器插入逻辑，下一个是CacheInterceprer
		response = realChain.proceed(request, transmitter, null);
		success = true;
	  } catch (RouteException e) {--关注点4.2--
		//RouteException 表示通过单个路由连接时出现了异常，且可能已经尝试了多次均失败，
		// The attempt to connect via a route failed. The request will not have been sent.
		//看这里逻辑，如果recover返回false，则不再重试，直接抛出异常 --关注点4.2--
		if (!recover(e.getLastConnectException(), transmitter, false, request)) {
		  throw e.getFirstConnectException();
		}
		continue;//重试
	  } catch (IOException e) {
		//这里如果是一个服务器通讯出现了异常，需要考虑请求是否已经被发送出去
		// An attempt to communicate with a server failed. The request may have been sent.
		//如果是除了ConnectionShutdownException的异常，则requestSendStarted为true
		boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
		//此时recover传入的requestSendStarted参数为true，前面RouteException传入的是false --关注点4.2--
		if (!recover(e, transmitter, requestSendStarted, request)) throw e;
		continue;//重试
	  } finally {
		// The network call threw an exception. Release any resources.
		if (!success) {
		  //如果失败或者异常情况下需要释放资源
		  transmitter.exchangeDoneDueToException();
		}
	  }

	  //如果priorResponse存在，则将之前的priorResponse作为响应
	  if (priorResponse != null) {
		response = response.newBuilder()
			.priorResponse(priorResponse.newBuilder()
					.body(null)
					.build())
			.build();
	  }
	  //通过原响应获取原响应的请求Exchange信息
	  Exchange exchange = Internal.instance.exchange(response);
	  //通过exchange.connection().route()获取原响应信息的路由信息
	  Route route = exchange != null ? exchange.connection().route() : null;
	  //这里发送重定向请求，传入原响应信息和原路由信息 --关注点4.3
	  Request followUp = followUpRequest(response, route);
	//响应成功或者不需要重定向的情况，直接返回response
	  if (followUp == null) {
		if (exchange != null && exchange.isDuplex()) {
		  transmitter.timeoutEarlyExit();
		}
		return response;
	  }
	//如果重定向不为空且有body
	  RequestBody followUpBody = followUp.body();
	  //只能发送一次则直接返回response
	  if (followUpBody != null && followUpBody.isOneShot()) {
		return response;
	  }
	  //重定向次数大于MAX_FOLLOW_UPS，则抛出异常
	  if (++followUpCount > MAX_FOLLOW_UPS) {
		throw new ProtocolException("Too many follow-up requests: " + followUpCount);
	  }
	  //将重定向请求体作为请求体
	  request = followUp;
	  //将请求作为原响应
	  priorResponse = response;
	}
}				  
/**
来看关注点 “4.1”：
这个方法内部将connectionPool，请求器RealCall还有主机地址等信息封装到一个exchangeFinder对象中

**/
	public void prepareToConnect(Request request) {
		...
		this.request = request;
		this.exchangeFinder = new ExchangeFinder(this, connectionPool, createAddress(request.url()),
			call, eventListener);
	}

//--关注点4.2--：
/**
如果proceed返回RouteException，则需要通过下面recover判断是否需要重试：这里的数据并没有发生出去
为false不需要重试，true需要重试
**/
	private boolean recover(IOException e, Transmitter transmitter,
	  boolean requestSendStarted, Request userRequest) {
		//应用层禁止重试
		// The application layer has forbidden retries.
		if (!client.retryOnConnectionFailure()) return false;
		
		//如果请求已经发送，且返回了FileNotFoundException或者isOneShot标志位true
		// We can't send the request body again.
		if (requestSendStarted && requestIsOneShot(e, userRequest)) return false;
		
		//异常是一个崩溃导致的
		// This exception is fatal.
		if (!isRecoverable(e, requestSendStarted)) return false;
	
		//没有其他路由可以尝试了
		// No more routes to attempt.
		if (!transmitter.canRetry()) return false;
		
		//以上都不是，则返回true，需要重试
		// For failure recovery, use the same route selector with a new connection.
		return true;
	}


```

**这个方法内部主要是对重试情况进行判断：**

重试逻辑
    **1.应用层禁止重试，则不能重试
    2.如果请求已经发送，且返回了FileNotFoundException或者isOneShot标志位true，则不会进行重试
    3.异常是一个崩溃导致的不会进行重试
    4.没有其他路由可以尝试了，不能重试
    5.以上情况都没有，则进行重试**





<mark>`--关注点4.3--`</mark>

> 发送重定向信息：这里篇幅比较长，只列出了部分关键代码

```java
private Request followUpRequest(Response userResponse, @Nullable Route route) throws IOException {
					
	int responseCode = userResponse.code();

	final String method = userResponse.request().method();
	switch (responseCode) {
	  case HTTP_PROXY_AUTH:
		//此处需要对代理服务器进行验证返回一个验证后的请求
		return client.proxyAuthenticator().authenticate(route, userResponse);

	  case HTTP_UNAUTHORIZED:
		//此处需要对服务器进行验证并返回一个验证后的请求
		return client.authenticator().authenticate(route, userResponse);

	  case HTTP_PERM_REDIRECT:
	  case HTTP_TEMP_REDIRECT:
		//对于HTTP_PERM_REDIRECT和HTTP_TEMP_REDIRECT请求，需要判断是否是GET或者HEAD
		//如果都不是，则直接返回null，如果是GET或者HEAD中的一种，则继续下面重定向逻辑
		//此处说明HTTP_PERM_REDIRECT和HTTP_TEMP_REDIRECT只有GET请求或者HEAD请求发出
		if (!method.equals("GET") && !method.equals("HEAD")) {
		  return null;
		}
		// fall-through
	  case HTTP_MULT_CHOICE:
	  case HTTP_MOVED_PERM:
	  case HTTP_MOVED_TEMP:
	  case HTTP_SEE_OTHER:
		
		//如果应用不支持重定向，则返回null
		if (!client.followRedirects()) return null;
		//获取新的重定向地址location
		String location = userResponse.header("Location");
		//location为空，则返回null
		if (location == null) return null;
		//通过新的location创建一个新的HttpUrl请求
		HttpUrl url = userResponse.request().url().resolve(location);

		// url为空，则直接返回null
		if (url == null) return null;
		
		// If configured, don't follow redirects between SSL and non-SSL.
		boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
		//url的Scheme不一致，都不能使用重定向
		if (!sameScheme && !client.followSslRedirects()) return null;
		
		
		//大部分的重定向情况不会返回一个request，需要重新创建一个request
		Request.Builder requestBuilder = userResponse.request().newBuilder();
		//这里如果是GET或者HEAD请求，则不会使用原响应中的body，且需要删除Header中的Transfer-Encoding，Content-Length，Content-Type等字段
		if (HttpMethod.permitsRequestBody(method)) {
		  final boolean maintainBody = HttpMethod.redirectsWithBody(method);
		  if (HttpMethod.redirectsToGet(method)) {
			requestBuilder.method("GET", null);
		  } else {
			RequestBody requestBody = maintainBody ? userResponse.request().body() : null;
			requestBuilder.method(method, requestBody);
		  }
		  if (!maintainBody) {
			requestBuilder.removeHeader("Transfer-Encoding");
			requestBuilder.removeHeader("Content-Length");
			requestBuilder.removeHeader("Content-Type");
		  }
		}
		
		//如果url不一致，则删除Header中的Authorization字段
		if (!sameConnection(userResponse.request().url(), url)) {
		  requestBuilder.removeHeader("Authorization");
		}
		//返回一个新的重定向request
		return requestBuilder.url(url).build();

	  case HTTP_CLIENT_TIMEOUT:
		//...这种情况比较少见，但是如HAProxy等服务器类型还是在使用这个响应码
	  case HTTP_UNAVAILABLE:
		//如果响应中的priorResponse不为null，则返回null
		if (userResponse.priorResponse() != null
			&& userResponse.priorResponse().code() == HTTP_UNAVAILABLE) {
		  // We attempted to retry and got another timeout. Give up.
		  return null;
		}
		//判断如果userResponse中的header("Retry-After")字段为0，则可以使用userResponse.request()作为重定向请求体
		if (retryAfter(userResponse, Integer.MAX_VALUE) == 0) {
		  // specifically received an instruction to retry without delay
		  return userResponse.request();
		}

		return null;

	  default:
		//其他情况，返回null
		return null;	
	}
}
```

<u><mark>上面已经将重试和重定向拦截器的逻辑讲清楚了，现总结如下：</mark></u>

**<u>重试条件</u>：**
    1.应用层禁止重试                    
    2.如果请求已经发送，且返回了<mark>FileNotFoundException</mark>或者isOneShot标志位true
    3.异常是一个崩溃导致的
    4.没有其他路由可以尝试了
    5.以上都不是，则请求需要重试
       应用重试分<mark>RoutException</mark>和<mark>IOException</mark>，前置数据没有发出，后置数据可能已经发出，重试的时候需要对请求是否已经发出进行判断


<u>**重定向条件：**</u>
    1.返回407需要对代理服务器做验证，返回401需要对服务器进行验证
    2.返回307和308需要，且返回的是Get请求或者Head请求，则无需发出重定向，其他情况继续按后面逻辑判断
    3.如果是300,301,302,303或2中的307和308的情况则判断应用是否支持重定向，不支持返回null，支持就去原响应信息中取出重定向地址，返回对应的重定向请求Request
    4.还有其他如408和503的情况，这些比较少见，且重定向逻辑要求比较复杂，可以看源码中的分析                    
    5.如果重定向次数超过最大重定向允许次数，则不进行重定向
    6.如果只允许一次通讯oneShot则不能重定向。



###### 缓存拦截器CacheIntercepter：

直接来看他的intercept方法：

```java
public final class CacheInterceptor implements Interceptor {
	public Response intercept(Chain chain) throws IOException {
	//这里从cache获取缓存数据，之前讲解OkHttpClient的时候分析过，这个内部是使用DiskLruCache磁盘对数据进行缓存，如果数据比较大，可能会有一定延迟
	Response cacheCandidate = cache != null
		? cache.get(chain.request())
		: null;
	//记录当前时间
	long now = System.currentTimeMillis();
	//这里创建了一个缓存策略
	CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
	Request networkRequest = strategy.networkRequest;
	Response cacheResponse = strategy.cacheResponse;

	// 如果禁止使用网络请求且缓存响应为空，则直接返回一个504的错误码
	if (networkRequest == null && cacheResponse == null) {
	  return new Response.Builder()
		  .request(chain.request())
		  .protocol(Protocol.HTTP_1_1)
		  .code(504)
		  .message("Unsatisfiable Request (only-if-cached)")
		  .body(Util.EMPTY_RESPONSE)
		  .sentRequestAtMillis(-1L)
		  .receivedResponseAtMillis(System.currentTimeMillis())
		  .build();
	}

	// 如果网络请求被禁止，且获取到cacheResponse不为空，则直接返回缓存数据
	if (networkRequest == null) {
	  return cacheResponse.newBuilder()
		  .cacheResponse(stripBody(cacheResponse))
		  .build();
	}

	Response networkResponse = null;
	try {
		//这里调用下一个拦截器，直接使用networkRequest
	  networkResponse = chain.proceed(networkRequest);
	} finally {
		//如果出现io错误等，且缓存不为空，则需要将缓存清空
	  if (networkResponse == null && cacheCandidate != null) {
		closeQuietly(cacheCandidate.body());
	  }
	}

	// If we have a cache response too, then we're doing a conditional get.
	//如果缓存数据不为空，且服务器返回了HTTP_NOT_MODIFIED 304则返回缓存数据
	if (cacheResponse != null) {
	  if (networkResponse.code() == HTTP_NOT_MODIFIED) {
		Response response = cacheResponse.newBuilder()
			.headers(combine(cacheResponse.headers(), networkResponse.headers()))
			.sentRequestAtMillis(networkResponse.sentRequestAtMillis())
			.receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
			.cacheResponse(stripBody(cacheResponse))
			.networkResponse(stripBody(networkResponse))
			.build();
		...
		return response;
	  } 
	}

	Response response = networkResponse.newBuilder()
		.cacheResponse(stripBody(cacheResponse))
		.networkResponse(stripBody(networkResponse))
		.build();
	//如果cache不为null且Header头部的hasBody为true，且缓存器可使用状态，则将响应数据写到缓存中，并返回
	if (cache != null) {
	  if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
		// Offer this request to the cache.
		CacheRequest cacheRequest = cache.put(response);
		return cacheWritingResponse(cacheRequest, response);
	  }
	}
	//返回响应数据
	return response;
				
}
```

**总结下CacheInterceprer：**

- **1.如果禁止使用网络请求且缓存响应为空，则直接返回一个504的错误码
  2.如果网络请求被禁止，且获取到cacheResponse不为空，则直接返回缓存数据
  3.如果缓存数据不为空，且服务器返回了HTTP_NOT_MODIFIED 304则返回缓存数据
  4.如果cache不为null且Header头部的hasBody为true，且缓存器可使用状态，则将响应数据写到缓存中，并返回
  5.缓存器如果在使用的过程中出现异常，记得清空缓存，防止内存溢出**



###### 连接拦截器(ConnectInterceptor)：

直接看他的intercept方法：

```java

public Response intercept(Chain chain) throws IOException {
	RealInterceptorChain realChain = (RealInterceptorChain) chain;
	Request request = realChain.request();
	Transmitter transmitter = realChain.transmitter();

	// We need the network to satisfy this request. Possibly for validating a conditional GET.
	boolean doExtensiveHealthChecks = !request.method().equals("GET");
	Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

	return realChain.proceed(request, transmitter, exchange);
	}
/**
看到这个方法代码很简单：核心代码：Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);
我们进入到这里面看看：核心代码exchangeFinder.find(client, chain, doExtensiveHealthChecks);返回一个ExchangeCodec对象
**/
	
	Exchange newExchange(Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
	//关注点1
	ExchangeCodec codec = exchangeFinder.find(client, chain, doExtensiveHealthChecks);
	//将返回的Http1ExchangeCodec又放入到一个Exchange对象中并返回这个Exchange
	Exchange result = new Exchange(this, call, eventListener, exchangeFinder, codec);
	...
	return result;

	}
	//进入关注点1看看：
	public ExchangeCodec find(
	  OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {

	try {
	//返回一个RealConnection，可能是连接池中的可能是新创建的
	  RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
		  writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
	//返回一个Http1ExchangeCodec的对象并包含new Http1ExchangeCodec(client, this, source, sink);这个sink= BufferedSink ,这是okio部分内容了
	  return resultConnection.newCodec(client, chain);
	} catch (RouteException e) {
	  ...
	}
	}

	调用了findHealthyConnection方法
	private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
	  int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
	  boolean doExtensiveHealthChecks) throws IOException {
	 //这里调用了一个死循环，内部循环调用findConnection方法直到获取到一个RealConnection对象
	while (true) {
	  RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
		  pingIntervalMillis, connectionRetryEnabled);

	  // If this is a brand new connection, we can skip the extensive health checks.
	  synchronized (connectionPool) {
		if (candidate.successCount == 0 && !candidate.isMultiplexed()) {
		  return candidate;
		}
	  }

	  // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
	  // isn't, take it out of the pool and start again.
	  if (!candidate.isHealthy(doExtensiveHealthChecks)) {
		candidate.noNewExchanges();
		continue;
	  }
		//返回获取的连接
	  return candidate;
	}
	}
	来看findConnection方法：
	private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
	int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
	if (result == null) {						
		//去连接池中查找可用的连接，这个方法在前面讲解OkHttpClient的时候解析过，是在内存缓存列表中查找对应的缓存连接，可以回头看看，篇幅问题这里不再讲解
		if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, null, false)) {//关注点2
		  foundPooledConnection = true;
		  result = transmitter.connection;
		} else if (nextRouteToTry != null) {
		 //如果没有找到，看nextRouteToTry是否为空，不为空则返回selectedRoute = nextRouteToTry
		  selectedRoute = nextRouteToTry;
		  nextRouteToTry = null;
		} else if (retryCurrentRoute()) {
		  selectedRoute = transmitter.connection.route();
		}
		
	}
	//如果找到可复用的连接，则直接返回
	if (result != null) {
	  // If we found an already-allocated or pooled connection, we're done.
	  return result;
	}
	//这个方法内部包含了对主机dns的解析，这里不再深入，有需要可以自行查看
	routeSelection = routeSelector.next();
	//没有找到
	if (!foundPooledConnection) {
		if (selectedRoute == null) {
		  selectedRoute = routeSelection.next();
		}

		//创建一个RealConnection并设置connectingConnection为这个result
		result = new RealConnection(connectionPool, selectedRoute);
		connectingConnection = result;
	}
	return result;		
}
```

**总结连接池拦截器：**

> 主要是去连接池中获取可复用的连接，如果没有就自行创建一个RealConnection，创建之前会对主机进行dns解析。返回的RealConnection包括了解析后的ip和port信息
>         

类关系:

**`Exchange(Http1ExchangeCodec(RealConnection(selectedRoute)))多层包裹`**




###### 请求服务器拦截器CallServerInterceptor：

```java
public Response intercept(Chain chain) throws IOException {
	RealInterceptorChain realChain = (RealInterceptorChain) chain;
	Exchange exchange = realChain.exchange();
	Request request = realChain.request();
	
	//写入请求头部数据
	exchange.writeRequestHeaders(request);
	//读取请求头响应数据
	responseBuilder = exchange.readResponseHeaders(false);
	//根据请求头创建响应Response
	Response response = responseBuilder
		.request(request)
		.handshake(exchange.connection().handshake())
		.sentRequestAtMillis(sentRequestMillis)
		.receivedResponseAtMillis(System.currentTimeMillis())
		.build();

	response = response.newBuilder()
		  .body(exchange.openResponseBody(response))
		  .build();
	
	return response;
}
/**
进入exchange.openResponseBody(response)：
看到返回了一个RealResponseBody对象
**/
public ResponseBody openResponseBody(Response response) throws IOException {
	try {
	  eventListener.responseBodyStart(call);
	  String contentType = response.header("Content-Type");
	  long contentLength = codec.reportedContentLength(response);
	  Source rawSource = codec.openResponseBodySource(response);
	  ResponseBodySource source = new ResponseBodySource(rawSource, contentLength);
	  return new RealResponseBody(contentType, contentLength, Okio.buffer(source));
	} catch (IOException e) {
	  eventListener.responseFailed(call, e);
	  trackFailure(e);
	  throw e;
	}
}
/**

可以看到在请求服务器拦截过程中是没有获取服务器数据的，只是将数据头发送给后台，
并根据头部响应数据创建了RealResponseBody对象，内部包含数据body长度等信息，待应用调用response.body().bytes()的时候去后台取数据
我们来看bytes方法
**/

public final byte[] bytes() throws IOException {
	...
	byte[] bytes;
	try (BufferedSource source = source()) {
	  bytes = source.readByteArray();
	//这个内部方法很简单就是使用okio去取后台取数据,长度数据根据之前返回的contentLen去取
	}
	return bytes;
	...

}
```



**<u>上面文章后，我们已经将OKHttp所有涉及到的知识点已经通过源码解析了一遍了，这篇文章虽然长，但是可以说是保姆级别的文章，自行阅读源码过程中有不懂的均可以
        拿这篇文章来参考。</u>**



## 总结

> OkHttp核心处理逻辑内部拦截器在起作用，每个拦截器都有自己的职责，使用责任链模式创接起来，各司其职
> 
> 

相信能坚持看完本文章的同学一定会有所收获，加油，一起进步



 **Android体系课学习 之 开源框架系列**

[Android体系课学习 之 网络请求库OkHttp看这一篇就够了]()

[Android体系课学习 之 网络请求库Retrofit使用方式(附Demo)]()

[Android体系课学习 之 网络请求库Retrofit源码分析]()

[Android体系课学习 之 RxJava看这篇就够了]()

[Android体系课学习 之 RxJava操作符详解]()

[[Android体系课学习 之 RxJava源码分析]()




