> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
最近在`组件化`开发中准备封装一个`图片加载库`，于是乎就有了这篇文章

本篇文章对`Glide`源码过程做了一个详细的讲解，也是为了记录下自己对`Glide`的理解，以后忘记还可以从这里查找。

这里我有几点建议：
- 看源码前先问下自己：
你为什么去看源码，是为了面试？学习？或者和我一样为了封装一个自己的图片加载库？
如果是为了面试，建议先列出几种同类型的开源库，比如我们的图片加载库有Picasso，Fresco和Glide，首先你要知道他们的基本使用方式,能说出他们的优缺点，并从中挑选一个类库去深入了解，因为面试官很大可能会让你自己设计一套类似的开源库，那这个时候你有对某个类库做过深入了解，会起到事半功倍的效果，就算不能完整设计出来，也可以说出一些基本原理和大致的架构。

**个人对看源码的思路:**

先看`整体框架`：然后找到一个`切入点`，深入框架内部去学习。学习中间记得根据框架进行`分阶段总结`，这样才能不会深陷代码泥潭。好了下面我们开始吧。。




## 整体框架图

![glide整体框架图.awebp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cbbfaf9922414000842380f0250c72e7~tplv-k3u1fbpfcp-watermark.image?)

## 源码梳理
我们将源码分为三个部分来讲解：
`with` `load` 和`into`
> 这个三个部分就是我们看源码的切入点：

#### 步骤1：with：
**Glide.java**

```java
public static RequestManager with(@NonNull Context context) {
	return getRetriever(context).get(context);
}

private static RequestManagerRetriever getRetriever(@Nullable Context context) {
	return Glide.get(context).getRequestManagerRetriever();
}

public static Glide get(@NonNull Context context) {
	checkAndInitializeGlide(context);
}
private static void checkAndInitializeGlide(@NonNull Context context) {
	initializeGlide(context);
}

private static void initializeGlide(@NonNull Context context) {
	initializeGlide(context, new GlideBuilder());
}
private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
	Context applicationContext = context.getApplicationContext();
	//1.获取注解自动生成的AppGlideModule
	GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
	List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
	if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
	  //这里解析manifest中的，metedata字段，并添加到list的格式的manifestModules中
	  manifestModules = new ManifestParser(applicationContext).parse();
	}
...
	RequestManagerRetriever.RequestManagerFactory factory =
		annotationGeneratedModule != null
			? annotationGeneratedModule.getRequestManagerFactory() : null;
	//给builder设置一个RequestManagerFactory，用于创建RequestManager
	builder.setRequestManagerFactory(factory);
	for (com.bumptech.glide.module.GlideModule module : manifestModules) {
	  module.applyOptions(applicationContext, builder);
	}
	if (annotationGeneratedModule != null) {
	  annotationGeneratedModule.applyOptions(applicationContext, builder);
	}
	//创建Glide对象  --关注点1--
	Glide glide = builder.build(applicationContext);
	//这里给manifestModules中设置的GlideModule注册到应用的生命周期中
	for (com.bumptech.glide.module.GlideModule module : manifestModules) {
	  module.registerComponents(applicationContext, glide, glide.registry);
	}
	//将应用的applicationContext和build创建的glide注册到：
	//之前使用注解@GlideModule创建的GlideApp中，可以让GlideApp在退出时，可以对glide或者app做一些处理
	if (annotationGeneratedModule != null) {
	  annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
	}
	//这里将glide注册到应用的生命周期中：
	//glide实现了ComponentCallbacks接口，这个接口有两个方法onConfigurationChanged和onLowMemory，
	//在应用回调这两个接口的时候，glide可以感知到，可以对缓存或者图片做一些处理等
	applicationContext.registerComponentCallbacks(glide);
	Glide.glide = glide;
}

```

`--关注点1--`

```java
Glide build(@NonNull Context context) {
	//创建SourceExecutor线程池
    if (sourceExecutor == null) {
      sourceExecutor = GlideExecutor.newSourceExecutor();
    }
	//创建diskCacheExecutor磁盘缓存线程池
    if (diskCacheExecutor == null) {
      diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    }
	//创建animationExecutor动画线程池
    if (animationExecutor == null) {
      animationExecutor = GlideExecutor.newAnimationExecutor();
    }
	//创建内存大小测量器，后期会根据这个测量器测出的内存状态，对缓存进行操作
    if (memorySizeCalculator == null) {
      memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
    }
	//这个类里面对Activity或者Fragment的生命周期做了监听，在Activity或者Fragment处于活跃状态时去监听网络连接状态，在监听中做一些重启等操作
    if (connectivityMonitorFactory == null) {
      connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
    }
	//创建一个BitMap的缓存池，内部FIFO
    if (bitmapPool == null) {
      int size = memorySizeCalculator.getBitmapPoolSize();
      if (size > 0) {
        bitmapPool = new LruBitmapPool(size);
      } else {
        bitmapPool = new BitmapPoolAdapter();
      }
    }
	//创建一个缓存池列表
    if (arrayPool == null) {
      arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }
	//创建一个Resource缓存池
    if (memoryCache == null) {
      memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }
	//创建磁盘缓存池
    if (diskCacheFactory == null) {
      diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }
	//将上面创建的所有对象封装到一个Engine对象中，以后所有需要的缓冲池或者线程池都在这里面获取即可
    if (engine == null) {
      engine =
          new Engine(
              memoryCache,
              diskCacheFactory,
              diskCacheExecutor,
              sourceExecutor,
              GlideExecutor.newUnlimitedSourceExecutor(),
              animationExecutor,
              isActiveResourceRetentionAllowed);
    }
	//这里是回调列表，用户设置的Listener有可能会有多个
    if (defaultRequestListeners == null) {
      defaultRequestListeners = Collections.emptyList();
    } else {
      defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
    }
	//这里是设置一些Glide的实验性数据，这里不用太过关注
    GlideExperiments experiments = glideExperimentsBuilder.build();
    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory, experiments);
	//最后将上面创建的所有对象都保存到一个Glide对象中，并返回
    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptionsFactory,
        defaultTransitionOptions,
        defaultRequestListeners,
        experiments);
}
```

看Glide构造方法：


```java
Glide(...){
	1.创建了一个Registry，这个方法主要是用来注册modelClass和dataClass和factory对应，由此可以通过modelClass找到对应的dataClass和factory
	registry = new Registry();
    registry.register(new DefaultImageHeaderParser());
	
	registry
        .append(ByteBuffer.class, new ByteBufferEncoder())
        .append(InputStream.class, new StreamEncoder(arrayPool))
		...
		//这里是一个比较重要的，标注TODO1。后面调用DataFetch会用到这个注册的factory 
		.append(GlideUrl.class, InputStream.class, new HttpGlideUrlLoader.Factory())
		..
	//这里省略了很多append调用：
}
```




回到调用 “`关注点1`”的代码处
`initializeGlide`方法讲解完后，我们回调最开始的`with`调用方法处：

```java
public static RequestManager with(@NonNull Context context) {
	return getRetriever(context).get(context);
}
这里有个get方法：我们进入看看
public RequestManager get(@NonNull Context context) {
	//传入的context不能为null
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        return get((Activity) context);
      } else if (context instanceof ContextWrapper
          // Only unwrap a ContextWrapper if the baseContext has a non-null application context.
          // Context#createPackageContext may return a Context without an Application instance,
          // in which case a ContextWrapper may be used to attach one.
          && ((ContextWrapper) context).getBaseContext().getApplicationContext() != null) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }
    return getApplicationManager(context);	
}
```

这里我们看到：

- **1**.如果`woth`方法在`子线程`运行，则会走到`getApplicationManager(context`)，这里会创建一个`全局的Glide`，不会随着控件`生命周期`销毁
- **2**.如果是`主线程`：
	- **2.1**：context是`FragmentActivity`，返回一个Fragment的生命周期的Glide RequestManager
	- **2.2**：context是`Activity`，返回一个Activity的生命周期的Glide RequestManager
	- **2.3**：context是`ContextWrapper`，返回一个ContextWrapper的生命周期的Glide RequestManager
- 3.其他情况就返回应用层的**RequestManager**，

`注意`：
> 所以在子线程中创建的RequestManager都是全局应用的RequestManager，只能感知应用状态，无法感知控件状态

​		

总结with方法：
> 作用：初始化`glide`：创建了多种`线程池`，多种`缓存池`，将glide注册到应用控件的`生命周期`中，可以感知应用的内存状态以及界面配置等信息
>
> 返回值：`RequestManager`


### 步骤2：load

```java
public RequestBuilder<Drawable> load(@Nullable String string) {
   return asDrawable().load(string);
}
public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
}
public <ResourceType> RequestBuilder<ResourceType> as(
	@NonNull Class<ResourceType> resourceClass) {
	return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

- 看到`asDrawable`其实就是创建了一个`RequestBuilder`

来看`RequestBuilder`的`load`方法：

```java
public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
}
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
	//这个判断默认为false
	if (isAutoCloneEnabled()) {
	  return clone().loadGeneric(model);
	}
	this.model = model;
	isModelSet = true;
	return selfOrThrowIfLocked();
}
```

`load`方法很简单：
> 就是创建了一个`RequestBuilder`，并设置了请求的`url`信息

### 步骤3：into

**这里才是glide的重点**

前面两个步骤都是初始化操作
我们将into方法分为三个阶段：
- `decodeJob前`
- `decodeJob开始`
- `decodeJob结束`


#### `阶段1`：decodeJob前

```java
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
	//先断言是在主线程上，如果是子线程则抛异常退出，
	//这就话可以看出glide可以在子线程初始化，但是into操作一定需要在主线程执行
	Util.assertMainThread();
	//传入的view不能为null，否则会抛异常
	Preconditions.checkNotNull(view);

	BaseRequestOptions<?> requestOptions = this;
	//这里是设置view的scaleType，glide默认提供了四种scaleType
	if (!requestOptions.isTransformationSet()
		&& requestOptions.isTransformationAllowed()
		&& view.getScaleType() != null) {
	  // Clone in this method so that if we use this RequestBuilder to load into a View and then
	  // into a different target, we don't retain the transformation applied based on the previous
	  // View's scale type.
	  switch (view.getScaleType()) {
		case CENTER_CROP:
		  requestOptions = requestOptions.clone().optionalCenterCrop();
		  break;
		case CENTER_INSIDE:
		  requestOptions = requestOptions.clone().optionalCenterInside();
		  break;
		case FIT_CENTER:
		case FIT_START:
		case FIT_END:
			//默认是使用这个状态设置图片
		  requestOptions = requestOptions.clone().optionalFitCenter();
		  break;
		case FIT_XY:
		  requestOptions = requestOptions.clone().optionalCenterInside();
		  break;
		case CENTER:
		case MATRIX:
		default:
		  // Do nothing.
	  }
	}
	return into(
        glideContext.buildImageViewTarget(view, transcodeClass), //·关注点2·
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
}
```

来看·`关注点2`·

```java
public <X> ViewTarget<ImageView, X> buildImageViewTarget(
	@NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
	return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
}
public class ImageViewTargetFactory {
	@NonNull
	@SuppressWarnings("unchecked")
	public <Z> ViewTarget<ImageView, Z> buildTarget(
	  @NonNull ImageView view, @NonNull Class<Z> clazz) {
	if (Bitmap.class.equals(clazz)) {
	  return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
	} else if (Drawable.class.isAssignableFrom(clazz)) {
	  return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
	} else {
	    throw new IllegalArgumentException(
		  "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
		}
	}
}
```

可以看到
- 传入的是`Bitmap`.class.则创建`BitmapImageViewTarget`
- 如果是`Drawable`.class，则创建`DrawableImageViewTarget`

并将传入的`ImageView`包裹进

> 步骤2 load方法中有了解过，load方法默认调用了asDrawable，所以返回的是Drawable类型的数据，
> 这里如果需要拿到的是Bitmap的类型图片数据，则需要调用asBitmap，跳过默认asDrawable方法的执行

继续回到`关注点2调用`的地方：调用了一个内部的`into`方法

```java
private <Y extends Target<TranscodeType>> Y into(
	  @NonNull Y target,
	  @Nullable RequestListener<TranscodeType> targetListener,
	  BaseRequestOptions<?> options,
	  Executor callbackExecutor) {
	...
	//关注点3
	Request request = buildRequest(target, targetListener, options, callbackExecutor);

	Request previous = target.getRequest();
	if (request.isEquivalentTo(previous)
		&& !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
	  // If the request is completed, beginning again will ensure the result is re-delivered,
	  // triggering RequestListeners and Targets. If the request is failed, beginning again will
	  // restart the request, giving it another chance to complete. If the request is already
	  // running, we can let it continue running without interruption.
	  if (!Preconditions.checkNotNull(previous).isRunning()) {
		// Use the previous request rather than the new one to allow for optimizations like skipping
		// setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
		// that are done in the individual Request.
		previous.begin();
	  }
	  return target;
	}

	requestManager.clear(target);
	target.setRequest(request);
	requestManager.track(target, request);

	return target;
}
```

来看`关注点3`：

```java
private Request buildRequest(
	  Target<TranscodeType> target,
	  @Nullable RequestListener<TranscodeType> targetListener,
	  BaseRequestOptions<?> requestOptions,
	  Executor callbackExecutor) {
	return buildRequestRecursive(
		/*requestLock=*/ new Object(),
		target,
		targetListener,
		/*parentCoordinator=*/ null,
		transitionOptions,
		requestOptions.getPriority(),
		requestOptions.getOverrideWidth(),
		requestOptions.getOverrideHeight(),
		requestOptions,
		callbackExecutor);
}
```

继续看buildRequestRecursive
为了不让大家看的头晕，非关键代码我这边省略了：

```java
private Request buildRequestRecursive(...) {

    ...
	//这里创建一个缩略图的请求：关注点4
    Request mainRequest =
        buildThumbnailRequestRecursive(
            requestLock,
            target,
            targetListener,
            parentCoordinator,
            transitionOptions,
            priority,
            overrideWidth,
            overrideHeight,
            requestOptions,
            callbackExecutor);
	...
	//这里创建一个errorRequest请求
    Request errorRequest =
        errorBuilder.buildRequestRecursive(
            requestLock,
            target,
            targetListener,
            errorRequestCoordinator,
            errorBuilder.transitionOptions,
            errorBuilder.getPriority(),
            errorOverrideWidth,
            errorOverrideHeight,
            errorBuilder,
            callbackExecutor);
    errorRequestCoordinator.setRequests(mainRequest, errorRequest);
    return errorRequestCoordinator;
  }
```


进入`关注点4`


```java
private Request buildThumbnailRequestRecursive(...)
  {
	...
	if (thumbnailBuilder != null) {
		Request fullRequest = obtainRequest(...)
		...
		Request thumbRequest = thumbnailBuilder.buildRequestRecursive(...)
		coordinator.setRequests(fullRequest, thumbRequest);
		return coordinator;
	}else if(thumbSizeMultiplier != null){
		Request fullRequest = obtainRequest(...)
		Request thumbnailRequest = obtainRequest()
		coordinator.setRequests(fullRequest, thumbnailRequest);
		return coordinator;
	}else{
		return obtainRequest(
          requestLock,
          target,
          targetListener,
          requestOptions,
          parentCoordinator,
          transitionOptions,
          priority,
          overrideWidth,
          overrideHeight,
          callbackExecutor);
	}
	
  
  }
  
进入obtainRequest看看：

private Request obtainRequest(
	  Object requestLock,
	  Target<TranscodeType> target,
	  RequestListener<TranscodeType> targetListener,
	  BaseRequestOptions<?> requestOptions,
	  RequestCoordinator requestCoordinator,
	  TransitionOptions<?, ? super TranscodeType> transitionOptions,
	  Priority priority,
	  int overrideWidth,
	  int overrideHeight,
	  Executor callbackExecutor) {
	return SingleRequest.obtain(
		context,
		glideContext,
		requestLock,
		model,
		transcodeClass,
		requestOptions,
		overrideWidth,
		overrideHeight,
		priority,
		target,
		targetListener,
		requestListeners,
		requestCoordinator,
		glideContext.getEngine(),
		transitionOptions.getTransitionFactory(),
		callbackExecutor);
	}
}
```
- 这里创建了一个`SingleRequest`的请求类，先记住这里

总结下关注点3：`buildRequest`

> 如果有要求加载缩略图请求的，会将缩略图请求和图片真实请求放到一起。
> 如果没有缩略图请求，则创建一个SingleRequest的请求返回给调用测

我们回调`关注点3`被调用处：这里我将前面代码再次拷贝一次大家就不用再次向前翻找了。

```java
private <Y extends Target<TranscodeType>> Y into(
	  @NonNull Y target,
	  @Nullable RequestListener<TranscodeType> targetListener,
	  BaseRequestOptions<?> options,
	  Executor callbackExecutor) {
	...
	//关注点3
	Request request = buildRequest(target, targetListener, options, callbackExecutor);
	
	
	Request previous = target.getRequest();
	//这里判断当前创建的请求和前面的请求是否是同一个请求，我们看不同的情况，所以跳过这里
	if (request.isEquivalentTo(previous)
		&& !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
	  // If the request is completed, beginning again will ensure the result is re-delivered,
	  // triggering RequestListeners and Targets. If the request is failed, beginning again will
	  // restart the request, giving it another chance to complete. If the request is already
	  // running, we can let it continue running without interruption.
	  if (!Preconditions.checkNotNull(previous).isRunning()) {
		// Use the previous request rather than the new one to allow for optimizations like skipping
		// setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
		// that are done in the individual Request.
		previous.begin();
	  }
	  return target;
	}
	//请求前先调用clear一下这个target
	requestManager.clear(target);
	target.setRequest(request);
	//这里是真实请求入口 我们进去看看
	requestManager.track(target, request);

	return target;
}
track方法:
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
	//这里将target放入一个targets集合中
	targetTracker.track(target);
	//请求入口
	requestTracker.runRequest(request);
}
runRequest方法：
public void runRequest(@NonNull Request request) {
    requests.add(request);
	//如果不是暂停状态，则调用begin启动请求，如果是，则将request放入pendingRequests，延迟启动，
    if (!isPaused) {
      request.begin();
    } else {
      request.clear();
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Paused, delaying request");
      }
      pendingRequests.add(request);
    }
}

```

进入`begin`：

在讲解`SingleRequest`前，我们下来了解下一个`Request`的几种状态：

```java
private enum Status {
    /** Created but not yet running. */
    PENDING,
    /** In the process of fetching media. */
    RUNNING,
    /** Waiting for a callback given to the Target to be called to determine target dimensions. */
    WAITING_FOR_SIZE,
    /** Finished loading media successfully. */
    COMPLETE,
    /** Failed to load media, may be restarted. */
    FAILED,
    /** Cleared by the user with a placeholder set, may be restarted. */
    CLEARED,
}
```

这个`Status`是`SingleRequest`的一个内部枚举类：
- `PENDING`：挂起状态，还未运行，待运行，前面我们说过如果Request处于isPaused状态，会添加进入pendingRequests的list中，这个里面的Request就是处于PENDING状态
- `RUNNING`：这个状态的Request正在去服务器拿数据阶段
- `WAITING_FOR_SIZE`：等待获取ImageView的一个尺寸阶段
- `COMPLETE`：成功拿到请求数据
- `FAILED`：没有获取到请求数据，在一定情况下会重启，例如：在网络状态良好的时候，Glide会自动重启请求，去获取数据，因为Glide注册了网络状态的监听
- `CLEARED`：被用户清理状态，使用一个placeholder代替数据元，在一定情况下会重启Request

好了，有了上面的基础我们再来看下面代码：

这个`request`是在`关注点3`处创建的`SingleRequest`

```java
public void begin() {
    synchronized (requestLock) {
      ...
	  //请求处于RUNNING状态，这个异常会被丢弃
      if (status == Status.RUNNING) {
        throw new IllegalArgumentException("Cannot restart a running request");
      }
	  ...
	  //请求处于COMPLETE，则回调onResourceReady，让应用去内存中获取，这里的DataSource.MEMORY_CACHE：标志数据元在内存缓存中
      if (status == Status.COMPLETE) {
        onResourceReady(
            resource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
        return;
      }
	  ...
		
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
		//这里是请求入口
        onSizeReady(overrideWidth, overrideHeight);
      } else {
        target.getSize(this);
      }
      if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
          && canNotifyStatusChanged()) {
        target.onLoadStarted(getPlaceholderDrawable());
      }
		
    }
}
```

从`begin`方法中，看到对请求的状态做了判断，只有是未处理的请求才会去请求数据，其他情况根据具体状态处理

我们进入`onSizeReady`看看：

```java
public void onSizeReady(int width, int height) {
	...
	loadStatus = engine.load(...)
	...
}
public <R> LoadStatus load(...){
	EngineResource<?> memoryResource;
    synchronized (this) {
	  //去缓存池中获取数据 ，这里面内部先去活动的缓存中activeResources获取图片数据，如果没有获取到就去缓存池cache中获取，都是通过对应的key获取
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);
	  //没有取到缓存数据，则进入waitForExistingOrStartNewJob，看名字应该是去请求数据的方法，我们进入这里面看看
      if (memoryResource == null) {
        return waitForExistingOrStartNewJob(
            glideContext,
            model,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            options,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache,
            cb,
            callbackExecutor,
            key,
            startTime);
      }
    }

    // Avoid calling back while holding the engine lock, doing so makes it easier for callers to
    // deadlock.
    cb.onResourceReady(
        memoryResource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
    return null;

}

private <R> LoadStatus waitForExistingOrStartNewJob(
	...
	//这里创建一个EngineJob
	EngineJob<R> engineJob =
        engineJobFactory.build(
            key,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache);
	//这里创建一个decodeJob
    DecodeJob<R> decodeJob =
        decodeJobFactory.build(
            glideContext,
            model,
            key,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            onlyRetrieveFromCache,
            options,
            engineJob);
	//将engineJob放入jobs中
    jobs.put(key, engineJob);
	//给engineJob添加回调和线程池执行器
    engineJob.addCallback(cb, callbackExecutor);
	//启动engineJob，并传入decodeJob
    engineJob.start(decodeJob);

    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob)

}
```

进入`engineJob.start(decodeJob)`方法看看：

```java
public synchronized void start(DecodeJob<R> decodeJob) {
	this.decodeJob = decodeJob;
	GlideExecutor executor =
		decodeJob.willDecodeFromCache() ? diskCacheExecutor : getActiveSourceExecutor();
	executor.execute(decodeJob);
}
```

- 内部其实就是使用线程池去请求`decodeJob`：

- 执行`decodeJob`前，我们先来梳理下我们阶段1分析的流程


```java
RequestBuilder.java
into{
	1.对ScaleType的RequestOption做处理
	2.调用内部into方法执行
	into(参数1：DrawableImageViewTarget(view),参数2：requestOptions,参数3：主线程执行器);
	{	
		1.创建Request：buildRequest，返回值：SingleRequest对象
		2.检查是否和前一个请求是同一个，如果是，则直接使用前一个请求发送begin方法，否就走第3步
		3.清除当前target请求;
		4.进入requestManager.track(target, request):
		  track{
			  1.调用requestTracker.runRequest(request)：
			  runRequest{
				//如果不是暂停状态，则调用begin启动请求，如果是，则将request放入pendingRequests，延迟启动，
				1.request.begin():这个request是一个SingleRequest对象
				begin{
					1.对请求状态做判断
					2.调用onSizeReady
					onSizeReady{
						engine.load{
							1.去缓存中获取数据：loadFromMemory
							2.1中没有获取到缓存数据，则调用waitForExistingOrStartNewJob
							waitForExistingOrStartNewJob{
								1.这里面启动//启动engineJob，并传入decodeJob：engineJob.start(decodeJob)
								start{
									1.调用线程池启动decodeJob
								}
							}
							
						}
					}
				}			
			  }	  	  
		  }		
	}
}
```


#### 阶段2：`decodeJob`执行阶段

在讲解`decodeJob`执行阶段前我们先来了解下`Job`的几种状态：

```java
/** Why we're being executed again. */
private enum RunReason {
	/** The first time we've been submitted. */
	INITIALIZE,
	/** We want to switch from the disk cache service to the source executor. */
	SWITCH_TO_SOURCE_SERVICE,
	/**
	 * We retrieved some data on a thread we don't own and want to switch back to our thread to
	 * process the data.
	 */
	DECODE_DATA,
}
```


这个`RunReason`看名字就知道：这里是表示你启动这个Job是要做什么的，主要有三种状态：

- `INITIALIZE`：
就是一个新的请求必须经过的第一步，这里job主要任务是去缓存中取数据

- `SWITCH_TO_SOURCE_SERVICE`：
就是我们希望这个job实现从磁盘服务到资源获取阶段，
这个怎么理解呢，了解OkHttp源码都知道，OkHttp内部使用的是拦截器的模式对请求做处理，每个请求都需要经过拦截器依次处理，最后得到请求数据，
如果OkHttp发现缓存拦截器中有数据，则返回，没有数据，则执行下一个拦截器。
这里也是一样，每个job类似一个拦截器，每次都是优先去缓存中获取数据，也就是第一步的INITIALIZE状态的job，如果没有数据才会去网络中请求，
SWITCH_TO_SOURCE_SERVICE就表示我们缓存中没有获取到数据，需要启动下一个job去网络中拿数据。

- `DECODE_DATA`：
这个很好理解，就是在异步线程上获取到数据，希望跳转到我们自己的线程上去解码数据

这里再介绍一个内部枚举类：

```java
/** Where we're trying to decode data from. */
private enum Stage {
	/** The initial stage. */
	INITIALIZE,
	/** Decode from a cached resource. */
	RESOURCE_CACHE,
	/** Decode from cached source data. */
	DATA_CACHE,
	/** Decode from retrieved source. */
	SOURCE,
	/** Encoding transformed resources after a successful load. */
	ENCODE,
	/** No more viable stages. */
	FINISHED,
}
```

- 这个类标识我们应该去哪个类中执行当前`Job`：
每个`Job`都有对应的执行器，执行完后，调用`startNext`执行下一个`Job`，下一个job又是对应另外一个`stage`的执行器

主要是在下面一个方法中使用：

```java
private DataFetcherGenerator getNextGenerator() {
	switch (stage) {
	  case RESOURCE_CACHE:
		return new ResourceCacheGenerator(decodeHelper, this);
	  case DATA_CACHE:
		return new DataCacheGenerator(decodeHelper, this);
	  case SOURCE:
		return new SourceGenerator(decodeHelper, this);
	  case FINISHED:
		return null;
	  default:
		throw new IllegalStateException("Unrecognized stage: " + stage);
	}
}
```

- `RESOURCE_CACHE`：对应`ResourceCacheGenerator`执行器
- `DATA_CACHE`：对应`DataCacheGenerator`执行器
- `SOURCE`：对应`SourceGenerator`执行器

经过上面的分析，我们已经知道：
> 你可以把一个Job理解为一个拦截器，每次都会根据当前Job的状态是要不同的执行器去执行任务
- `任务总线`：

1.`ResourceCacheGenerator`:缓存中获取数据，这个数据是被修改过的降采样缓存 

2.`DataCacheGenerator`:也是缓存中获取数据，这个数据是没有被修改的网路请求元数据

3.`SourceGenerator`:这个执行器执行的任务才真正是用于网络请求数据

是不是和我们的`OkHttp`很像？

有了上面的基础我们再来对`decodeJob`源码进行分析：


DecodeJob继承了Runnable接口，我们来看他的run方法

```java
public void run() {
	...
	runWrapped();
	...
}
private void runWrapped() {
	switch (runReason) {
	  case INITIALIZE:
		//关注点1
		stage = getNextStage(Stage.INITIALIZE);
		//关注点2
		currentGenerator = getNextGenerator();
		//关注点3
		runGenerators();
		break;
	  case SWITCH_TO_SOURCE_SERVICE:
		runGenerators();
		break;
	  case DECODE_DATA:
		decodeFromRetrievedData();
		break;
	  default:
		throw new IllegalStateException("Unrecognized run reason: " + runReason);
	}
}
```

`runWrapped`方法中对`runReason`做一个判断：分别指向不同的请求：

//关注点1
我们看INITIALIZE，调用了getNextStage传入Stage.INITIALIZE

```java
private Stage getNextStage(Stage current) {
    switch (current) {
      case INITIALIZE:
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE
            : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE
            : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        // Skip loading from source if the user opted to only retrieve the resource from cache.
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
}
```

这里面是根据传入的`diskCacheStrategy`磁盘缓存策略对应不同的状态
我们假设`diskCacheStrategy`.`decodeCachedResource`()返回都是`true`
则`next`关系如下：

```java
INITIALIZE->RESOURCE_CACHE->DATA_CACHE->SOURCE->FINISHED
```


回到前面`runWrapped`方法：

//关注点2
getNextGenerator获取执行器前面已经讲过，我们再发下对应关系：

- `RESOURCE_CACHE`：对应`ResourceCacheGenerator`执行器
- `DATA_CACHE`：对应`DataCacheGenerator`执行器
- `SOURCE`：对应`SourceGenerator`执行器

//关注点3
调用runGenerators开始执行对应的job

```java
private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled
        && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
}
```


> 可以看到这里有个while循环就是根据当前stage状态循环执行不同的请求：

`按前面我们的分析`：

优先会执行：
`ResourceCacheGenerator` ，然后`DataCacheGenerator`，最后`SourceGenerator`

篇幅问题：我们这只具体分析SourceGenerator的方法：其他两个执行器前面已经介绍过

```java
SourceGenerator.java
public boolean startNext() {
	...
    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
              || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        startNextLoad(loadData);
      }
    }
	...
    return started;
}

看startNextLoad方法：
private void startNextLoad(final LoadData<?> toStart) {
    loadData.fetcher.loadData(
        helper.getPriority(),
        new DataCallback<Object>() {
          @Override
          public void onDataReady(@Nullable Object data) {
            if (isCurrentRequest(toStart)) {
              onDataReadyInternal(toStart, data);
            }
          }

          @Override
          public void onLoadFailed(@NonNull Exception e) {
            if (isCurrentRequest(toStart)) {
              onLoadFailedInternal(toStart, e);
            }
          }
        });
}
```

这个fetcher = `MultiFetcher`对象
进入MultiFetcher的`loadData`

```java
public void loadData(@NonNull Priority priority, @NonNull DataCallback<? super Data> callback) {
	...
	fetchers.get(currentIndex).loadData(priority, this);
	...
}
```

`fetchers`是一个`List`，内部存储的是一个`HttpUrlFeacher`
这里传入了一个this作为callback回调，后面会用到：this = MultiFetcher


```java
HttpUrlFeacher.java

public void loadData(
      @NonNull Priority priority, @NonNull DataCallback<? super InputStream> callback) {
	...
	try {
	  InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
	  callback.onDataReady(result);
	} catch (IOException e) {	  
	  callback.onLoadFailed(e);//异常回调onLoadFailed接口给应用层
	} finally {
	  ...
	}
}
loadDataWithRedirects方法：
private InputStream loadDataWithRedirects(
      URL url, int redirects, URL lastUrl, Map<String, String> headers) throws HttpException {
    ...
	//关注点1
    urlConnection = buildAndConfigureConnection(url, headers);

    try {
      // Connect explicitly to avoid errors in decoders if connection fails.
      urlConnection.connect();
      // Set the stream so that it's closed in cleanup to avoid resource leaks. See #2352.
      stream = urlConnection.getInputStream();
    } catch (IOException e) {
      throw new HttpException(
          "Failed to connect or obtain data", getHttpStatusCodeOrInvalid(urlConnection), e);
    }
	...
	//这里获取返回code
    final int statusCode = getHttpStatusCodeOrInvalid(urlConnection);
    if (isHttpOk(statusCode)) {
		//关注点2
      return getStreamForSuccessfulRequest(urlConnection);
    } else if (isHttpRedirect(statusCode)) {
      String redirectUrlString = urlConnection.getHeaderField(REDIRECT_HEADER_FIELD);
      if (TextUtils.isEmpty(redirectUrlString)) {
        throw new HttpException("Received empty or null redirect url", statusCode);
      }
      URL redirectUrl;
      try {
        redirectUrl = new URL(url, redirectUrlString);
      } catch (MalformedURLException e) {
        throw new HttpException("Bad redirect url: " + redirectUrlString, statusCode, e);
      }
      // Closing the stream specifically is required to avoid leaking ResponseBodys in addition
      // to disconnecting the url connection below. See #2352.
      cleanup();
      return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
    } else if (statusCode == INVALID_STATUS_CODE) {
      throw new HttpException(statusCode);
    } else {
      try {
        throw new HttpException(urlConnection.getResponseMessage(), statusCode);
      } catch (IOException e) {
        throw new HttpException("Failed to get a response message", statusCode, e);
      }
    }
  }

```

//关注点1
这里主要是创建`HttpUrlConnection`请求

```java
private HttpURLConnection buildAndConfigureConnection(URL url, Map<String, String> headers)
      throws HttpException {
    HttpURLConnection urlConnection;
    try {
      urlConnection = connectionFactory.build(url);
    } catch (IOException e) {
      throw new HttpException("URL.openConnection threw", /*statusCode=*/ 0, e);
    }
    for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
      urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
    }
    urlConnection.setConnectTimeout(timeout);
    urlConnection.setReadTimeout(timeout);
    urlConnection.setUseCaches(false);
    urlConnection.setDoInput(true);
    // Stop the urlConnection instance of HttpUrlConnection from following redirects so that
    // redirects will be handled by recursive calls to this method, loadDataWithRedirects.
    urlConnection.setInstanceFollowRedirects(false);
    return urlConnection;
}

```

//关注点2
返回成功，则获取`InputStream`

```java
private InputStream getStreamForSuccessfulRequest(HttpURLConnection urlConnection)
      throws HttpException {
    try {
      if (TextUtils.isEmpty(urlConnection.getContentEncoding())) {
        int contentLength = urlConnection.getContentLength();
        stream = ContentLengthInputStream.obtain(urlConnection.getInputStream(), contentLength);
      } else {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "Got non empty content encoding: " + urlConnection.getContentEncoding());
        }
        stream = urlConnection.getInputStream();
      }
    } catch (IOException e) {
      throw new HttpException(
          "Failed to obtain InputStream", getHttpStatusCodeOrInvalid(urlConnection), e);
    }
    return stream;
}

继续回到HttpUrlFeacher的loadData方法
public void loadData(
      @NonNull Priority priority, @NonNull DataCallback<? super InputStream> callback) {
	...
	try {
	  InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
	  callback.onDataReady(result);
	} catch (IOException e) {	  
	  callback.onLoadFailed(e);//异常回调onLoadFailed接口给应用层
	} finally {
	  ...
	}
}
```

把返回的`InputStream`通过`callback`回调给上一级
这个callback 前面分析过：`callBack = MultiFetcher`
回到MultiFetcher的`onDataReady`方法中：

```java
public void onDataReady(@Nullable Data data) {
  if (data != null) {
	callback.onDataReady(data);
  } else {
	startNextOrFail();
  }
}
```

看到这里又调用了一次callback.onDataReady(data);
这个callback = loadData传入下来的`SourceGenerator`中的`startNextLoad`方法：

```java
SourceGenerator.java
private void startNextLoad(final LoadData<?> toStart) {
    loadData.fetcher.loadData(
        helper.getPriority(),
        new DataCallback<Object>() {
          @Override
          public void onDataReady(@Nullable Object data) {
            if (isCurrentRequest(toStart)) {
              onDataReadyInternal(toStart, data);
            }
          }

          @Override
          public void onLoadFailed(@NonNull Exception e) {
            if (isCurrentRequest(toStart)) {
              onLoadFailedInternal(toStart, e);
            }
          }
        });
}
```

回调这里再次调用`onDataReadyInternal`方法：

```java
void onDataReadyInternal(LoadData<?> loadData, Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
	//这里表示我们是否需要进行磁盘缓存
	if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
	  dataToCache = data;
	  // We might be being called back on someone else's thread. Before doing anything, we should
	  // reschedule to get back onto Glide's thread.
	  cb.reschedule();
	} else {
	  cb.onDataFetcherReady(
		  loadData.sourceKey,
		  data,
		  loadData.fetcher,
		  loadData.fetcher.getDataSource(),
		  originalKey);
	}
}
```


看cb.reschedule()方法：这个cb = `DecodeJob`

```java
DecodeJob.java
public void reschedule() {
    runReason = RunReason.SWITCH_TO_SOURCE_SERVICE;
    callback.reschedule(this);
}
这个callback是EngineJob
EngineJob.java
public void reschedule(DecodeJob<?> job) {
    // Even if the job is cancelled here, it still needs to be scheduled so that it can clean itself
    // up.
    getActiveSourceExecutor().execute(job);
}
```

到这里又去执行了一次job，这个job 的 `runReason = RunReason.SWITCH_TO_SOURCE_SERVICE`
前面第一次获取网络数据也调用过这个：
前面是为了获取缓存数据：这次是为了将数据写入缓存

最后将数据通过DecodeJob的onDataFetcherReady回调出去

这里对阶段2做一个总结：
- `工作`：主要负责去缓存中取数据，如果没有取到就去网络服务器拉数据，获取到数据后，将数据放入到缓存中。
使用的是Job模式，每个Job都有一个状态，每个状态对于一个执行器，类似OkHttp中的拦截器模式


#### `阶段3`：**解码数据**


```java
public void onDataFetcherReady(
      Key sourceKey, Object data, DataFetcher<?> fetcher, DataSource dataSource, Key attemptedKey) {
    this.currentSourceKey = sourceKey;
    this.currentData = data;
    this.currentFetcher = fetcher;
    this.currentDataSource = dataSource;
    this.currentAttemptingKey = attemptedKey;
    this.isLoadingFromAlternateCacheKey = sourceKey != decodeHelper.getCacheKeys().get(0);

    if (Thread.currentThread() != currentThread) {
      runReason = RunReason.DECODE_DATA;
      callback.reschedule(this);
    } else {
      GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
      try {
        decodeFromRetrievedData();
      } finally {
        GlideTrace.endSection();
      }
    }
}
```

- 这个方法中又调用了`decodeFromRetrievedData`
- 作用：解码得到的响应数据


```java
private void decodeFromRetrievedData() {
	try {
	  //这里解码数据 关注点1
      resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
      e.setLoggingDetails(currentAttemptingKey, currentDataSource);
      throwables.add(e);
    }
    if (resource != null) {
	  //通知应用 关注点2
      notifyEncodeAndRelease(resource, currentDataSource, isLoadingFromAlternateCacheKey);
    } else {
      runGenerators();
    }
}
```

看` 关注点1`：

```java
decodeFromData
	decodeFromFetcher(data, dataSource);
		runLoadPath(data, dataSource, path);
			path.load(rewinder, options, width, height, new DecodeCallback<ResourceType>(dataSource));
				loadWithExceptionList(rewinder, options, width, height, decodeCallback, throwables);
					result = path.decode(rewinder, width, height, options, decodeCallback);
						decodeResource(rewinder, width, height, options);
							decodeResourceWithList(rewinder, width, height, options, exceptions);
								result = decoder.decode(data, width, height, options);
									downsampler.decode(source, width, height, options);
										decode(...);
											decodeFromWrappedStreams
												Bitmap downsampled = decodeStream(imageReader, options, callbacks, bitmapPool);
													result = imageReader.decodeBitmap(options);
														BitmapFactory.decodeStream(stream(), /* outPadding= */ null, options);
```


可以看到请求数据解码过程最后也是通过`BitmapFactory.decodeStream`将数据转换为`Bitmap`返回
但是这个过程中Glide对数据做了多重处理，包括`裁剪，降采`样等操作


关注点2：`notifyEncodeAndRelease`

```java
private void notifyEncodeAndRelease(
	...核心代码
	notifyComplete(result, dataSource, isLoadedFromAlternateCacheKey);
}
private void notifyComplete(
  Resource<R> resource, DataSource dataSource, boolean isLoadedFromAlternateCacheKey) {
	setNotifiedOrThrow();
	callback.onResourceReady(resource, dataSource, isLoadedFromAlternateCacheKey);
}
调用了callback.onResourceReady：
这个callback = EngineJob
EngineJob.java
public void onResourceReady(
      Resource<R> resource, DataSource dataSource, boolean isLoadedFromAlternateCacheKey) {
	synchronized (this) {
	  this.resource = resource;
	  this.dataSource = dataSource;
	  this.isLoadedFromAlternateCacheKey = isLoadedFromAlternateCacheKey;
	}
	notifyCallbacksOfResult();
}
void notifyCallbacksOfResult() {
    ...
    for (final ResourceCallbackAndExecutor entry : copy) {
      entry.executor.execute(new CallResourceReady(entry.cb));
    }
    decrementPendingCallbacks();
  }

执行了CallResourceReady的run方法
run 
  callCallbackOnResourceReady(cb);
		cb.onResourceReady(engineResource, dataSource, isLoadedFromAlternateCacheKey); cb = SingleRequest
			 onResourceReady
				target.onResourceReady(result, animation); 这个target = BitmapImageViewTarget ，这里调用其父类ImageViewTarget的onResourceReady方法
					setResourceInternal(resource);
						setResource(resource);执行BitmapImageViewTarget的setResource
							view.setImageBitmap(resource);到这里就执行完毕了
```


- 总结阶段3：就是对网络数据根据需求进行降采样，并将数据设置到对应的target中

这里对整个步骤3：`into`方法做一个总结吧：
> `阶段1`：
> 创建对应的网络请求，对比当前请求和内存中前几个请求是否一致，如果一直，使用前一个请求进行网络请求。
> 去缓存中获取对应的数据，如果没有获取到就去网络拉取数据
>
> `阶段2`：
> 主要负责去缓存中取数据，如果没有取到就去网络服务器拉数据，获取到数据后，将数据放入到缓存中。
> 使用的是Job模式，每个Job都有一个状态，每个状态对于一个执行器，类似OkHttp中的拦截器模式
> 根据job状态的切换，执行各自的请求
>
> `阶段3`：对数据进行解码，解码会进行根据需求进行降采样，并将获取的图片数据传递给对应的Target



### Glide缓存
这里我们对`Glide三级缓存`做一个总结：因为一些面试还是经常会问到的

1.怎么缓存？
如果支持磁盘缓存，则将原`Source`缓存到磁盘，在经过解码处理后，将解码后的数据缓存到`activeResources`，`activeResources`是一个`弱引用的对象`，在内存回收的时候，会将数据放到cache缓存下面

2.怎么使用缓存

如果直接内存缓存，则去`activeResources`中取解码后的数据，如果`activeResources`中没有对应的缓存，则去`cache`缓存中获取，如找到则直接返回cache数据，并将缓存数据写入activeResources中，删除cache中的缓存数据，
如果还没有，则在调用网络请求前，去磁盘中获取，磁盘中获取的数据是图片原数据，需要经过解码处理才能被控件使用，磁盘如果还没有，才会去网络中请求数据。

以上就是Glide对三级缓存的使用机制。

### 总结
最后用一幅图来整理下过程：来自 **JsonChao**

因为是旧版本画的流程图，所以有些地方可能会不一致，但是大体流程是一致的，可以参考图片再去分析源码


![glide源码分析流程.awebp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f37bee47dfa4c2ca52dfb1f6c7a847d~tplv-k3u1fbpfcp-watermark.image?)