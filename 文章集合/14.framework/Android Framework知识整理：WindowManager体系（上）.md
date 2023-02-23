***本文正在参加[「金石计划 . 瓜分6万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")***
> 🔥 **Hi，我是小余。本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
大家好，由于**工作和面试需要**，笔者结合大佬们的经验以及自身对源码理解，写了一篇关于Android framework层：**WindowManager体系的讲解**。

本篇是Android framework的第一讲《WindowManager体系-上》，重在讲解Window在被添加到WindowManagerService前的流程。


## 1.Android WindowManager体系
Android中的窗口体系可以分为下面三部分：

- 1.**Window**：可以简单理解为一个窗口，这个窗口承载了需要绘制的View，他是一个抽象类，具体实现类是`PhoneWindow`。

- 2.**WindowManager**：也是一个接口类，继承`ViewManager`，用来对Window进行管理，
实现类：`WindowManagerImpl`，其对Window的操作具体是通过WMS来实现的。理解为一个app层面的WMS和Window的中间人。

- 3.**WindowManagerService**(AMS)：是系统服务中的一重要成员，WindowManager和WMS通过binder进行通讯，**真正对Window添加，更新，删除等操作的执行者**。

三者之间关系：

![三种关系.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a07ad073b21e44c78a61b24caae61ecf~tplv-k3u1fbpfcp-watermark.image?)

前面分析了窗口体系中的类关系，下面我们从源码角度和分析下：

**为了大家不会迷路在源码中，笔者会`根据面试中可能被问到的几个问题出发，有目的性的介绍`。**

- 问题1：**Window和Activity以及WindowManager什么时候建立的关系？**
- 问题2：**Window什么时候和View进行关联？**
- 问题3：**Window有哪些属性？类型？层级关系？z-order？Window标志？软键盘模式都了解么？**
- 问题4：**View是如何一步一步添加到屏幕上的？更新？删除呢？**

`那么就开始我们的源码（mo）遨（yu）游吧。`

### 1.Window和Activity以及WindowManager什么时候建立的关系？

前面几篇文章我们分析过：Activity在启动的过程中，会调用它的attach方法进行Window的创建。
那就直接定位到Activity的attach方法：

```java
final void attach(Context context, ActivityThread aThread,
		Instrumentation instr, IBinder token, int ident,
		Application application, Intent intent, ActivityInfo info,
		CharSequence title, Activity parent, String id,
		NonConfigurationInstances lastNonConfigurationInstances,
		Configuration config, String referrer, IVoiceInteractor voiceInteractor,
		Window window, ActivityConfigCallback activityConfigCallback) {
	attachBaseContext(context);
	...
	mWindow = new PhoneWindow(this, window, activityConfigCallback);//1
	...
	mWindow.setWindowManager(
			(WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
			mToken, mComponent.flattenToString(),
			(info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);//2
}
```

先看attach注释1处：**创建了一个PhoneWindow，并传入当前Activity的this对象**，
PhoneWindow类，继承了Window

```java
public class PhoneWindow extends Window {
	public PhoneWindow(Context context, Window preservedWindow,
            ActivityConfigCallback activityConfigCallback) {
		//这里最终会调父类Window的构造方法，并传入Activity的实例context。
        this(context);
        //只有Activity的Window才会使用Decor context,其他依赖使用的Context
        mUseDecorContext = true;
        ...
   }

}
```

Window构造方法：

```java
public Window(Context context) {
	mContext = context;
	mFeatures = mLocalFeatures = getDefaultFeatures(context);
}

```

PhoneWindow的构造方法中传入当前Activity对象，
并最终在父类Window构造方法中给mContext赋值，**Window得到Activity的引用，由此就建立了Window和Activity的联系。**


下面继续看attach注释2：mWindow.setWindowManager

setWindowManager在PhoneWindow的父类Window中实现：

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
		boolean hardwareAccelerated) {
	//如果wm是空，则创建一个WindowManager服务：注释1
	if (wm == null) {
		wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
	}
	//注释2
	mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

进入setWindowManager的注释1：

wm =(WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);

**这里的mContext是ContextImpl类的对象**：

```java
@Override
public Object getSystemService(String name) {
	return SystemServiceRegistry.getSystemService(this, name);
}
进入SystemServiceRegistry的getSystemService方法，name = Context.WINDOW_SERVICE
public static Object getSystemService(ContextImpl ctx, String name) {
	ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);//1
	return fetcher != null ? fetcher.getService(ctx) : null;//2
}
```

先看getSystemService方法的注释1：
**SYSTEM_SERVICE_FETCHERS是一个HashMap的集合，在哪里初始化的？**

我们注意到SystemServiceRegistry的static 方法,这个方法内部注册了很多服务，我们可以在方法里面找到name = Context.WINDOW_SERVICE的服务

PS：**static方法是在类加载的时候进行初始化的，所以在整个类使用过程中只会执行一次。**

```java
static {
	...
	registerService(Context.WINDOW_SERVICE, WindowManager.class,
		new CachedServiceFetcher<WindowManager>() {
	@Override
	public WindowManager createService(ContextImpl ctx) {
		return new WindowManagerImpl(ctx);
	}});
}

```

来看下registerService方法：

```java
private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) 
```

第三个参数是ServiceFetcher类型对象，实现的createService方法返回了一个WindowManagerImpl对象。

所以getSystemService方法的注释1：**fetcher = CachedServiceFetcher对象，且其createService方法返回了一个WindowManagerImpl对象**

回到getSystemService注释2处：**fetcher.getService(ctx)最终会返回调用到createService方，返回WindowManagerImpl对象。**

继续回到Window的setWindowManager方法的注释2处：

mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this)，

进入WindowManagerImpl的createLocalWindowManager方法：

```java
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
	return new WindowManagerImpl(mContext, parentWindow);
}
```
这里也是创建了一个WindowManagerImpl，和前面getSystemService创建的WindowManagerImpl区别：
**这里传入了parentWindow，让WindowManagerImpl持有了Window的引用，可以对此Window进行操作了。**

`总结下Activity的attach方法`：
- 1.**new一个PhoneWindow对象并传入当前Activity引用，建立Window和Activity的一一对应关系。此Window是Window类的子类PhoneWindow的实例。Activity在Window中是以mContext属性存在。**
- 2.**调用PhoneWindow的setWindowManager方法，在这个方法中让Window和WindowManager建立了一一关系。此WindowManager是WindowManagerImpl的实例、**

笔者画了一张Window关系类图：

![window关系图.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5dd01ff87a58413fa562df83222883a5~tplv-k3u1fbpfcp-watermark.image?)
### 2.Window什么时候和View进行关联？

我们来看Activity的setContentView方法

```java
public void setContentView(@LayoutRes int layoutResID) {
	getWindow().setContentView(layoutResID);//1
	initWindowDecorActionBar();//2
}
```

注释1中的getWindow前面分析了是：PhoneWindow对象
进入PhoneWindow的setContentView方法：

```java
public void setContentView(int layoutResID) {
	...
	if (mContentParent == null) {
		installDecor();//注释1
	} 
	mLayoutInflater.inflate(layoutResID, mContentParent);//注释2
	...
}

private void installDecor() {
	...
	if (mDecor == null) {
		mDecor = generateDecor(-1);
	} else {
		mDecor.setWindow(this);
	}
}

protected DecorView generateDecor(int featureId) {
	...
	return new DecorView(context, featureId, this, getAttributes());
}
```

PhoneWindow：setContentView注释1处：**创建一个DecorView给当前Window的mDecor**。
PhoneWindow：setContentView注释2处：**解析当前layoutResID为id的xml文件并传递给mContentParent。**

> 这里的mContentParent其实就是当前Window对应的ContentView。


继续回到Activity的setContentView方法注释2处：initWindowDecorActionBar这里是初始化Window的ActionBar

```java
private void initWindowDecorActionBar() {
	Window window = getWindow();

	// Initializing the window decor can change window feature flags.
	// Make sure that we have the correct set before performing the test below.
	window.getDecorView();

	if (isChild() || !window.hasFeature(Window.FEATURE_ACTION_BAR) || mActionBar != null) {
		return;
	}

	mActionBar = new WindowDecorActionBar(this);
	mActionBar.setDefaultDisplayHomeAsUpEnabled(mEnableDefaultActionBarUp);
	...
}
```


通过上面两步骤：**我们就给当前Activity的Window创建了一个DecorView，这个DecorView就是当前Window的rootView、并对rootView的ContentView设置了mContentParent
实现了将Window和DecorView绑定**。

**如果你`够细心`，你会发现Activity所有的对View进行的操作几乎都是通过Window进行的。而我们的Activity又是通过AMS来控制生命周期，所以AMS和View的通讯其实就是通过WindowManager这个介子进行的、**

比如：

```java
1.设置主题
public void setTheme(int resid) {
    super.setTheme(resid);
    mWindow.setTheme(resid);
}
2.设置Window属性等
public final boolean requestWindowFeature(int featureId) {
    return getWindow().requestFeature(featureId);
}

3.获取Layout解析器
public LayoutInflater getLayoutInflater() {
    return getWindow().getLayoutInflater();
}
```

### 3.Window有哪些属性？类型？层级关系？z-order？Window标志？软键盘模式都了解么

上面我们说过了Window、WindowManager和WMS之间的体系关系，WMS就好比老板，Window就好比员工，老板为了方便管理员工，就给员工定了很多规则.

这些规则就是Window的属性：**这些属性都定义在WindowManager.LayoutParams类中**。

应用开发中比较常见的，主要有以下几类：
- **1.Window窗口类型（Type）**
- **2.Window窗口层级关系（Z-Order）**
- **3.Window窗口标志（Flag）**
- **4.Window 软键盘模式（SoftInputModel）**

下面我们就分别来讲解下这几种属性

#### 1.Window窗口类型（Type）

Window窗口类型有很多种：比如：应用程序窗口，PopupWindow，Toast，Dialog，输入法窗口，系统错误窗口等。总体来说主要分为以下三大类：
- 1.**应用程序窗口**（Application Window）
- 2.**子窗口**（Sub Window）
- 3.**系统窗口**（System Window）

> 窗口类型在WindowManager的静态内部类LayoutParams中定义

##### 应用程序窗口

应用程序窗口指我们应用程序使用的（非Dialog）界面Window，如**Activity就是一个典型的应用窗口。**

```java
public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {
	//应用程序窗口的起始值
	public static final int FIRST_APPLICATION_WINDOW = 1;
	//应用程序窗口的基准值
    public static final int TYPE_BASE_APPLICATION   = 1;
	//标准的应用程序窗口：依附于Activity存在
	public static final int TYPE_APPLICATION        = 2;
	public static final int TYPE_APPLICATION_STARTING = 3;
	public static final int TYPE_DRAWN_APPLICATION = 4;
	//应用程序窗口结束值
	public static final int LAST_APPLICATION_WINDOW = 99;
	...
}
```
应用程序窗口起始值为1，结束值为99，说明我们应用程序窗口的类型为：`1~99`.
**这个值直接影响了窗口在屏幕中的层级关系**，后面再讲。

##### 子窗口
子窗口看名字就知道其是一个窗口的子窗口，所以**不能独立存在，需要依附于父窗口存在**，比如PopupWindow，Dialog等就属于子窗口，子窗口的类型定义如下所示:

```java
public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {
	public static final int FIRST_SUB_WINDOW = 1000;//子窗口类型初始值
	public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW;
	public static final int TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW + 1;
	public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW + 2;
	public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW + 3;
	public static final int TYPE_APPLICATION_MEDIA_OVERLAY  = FIRST_SUB_WINDOW + 4; 
	public static final int TYPE_APPLICATION_ABOVE_SUB_PANEL = FIRST_SUB_WINDOW + 5;
	public static final int LAST_SUB_WINDOW = 1999;//子窗口类型结束值
	...
}

```

子窗口的Type值范围为`1000到1999`.

##### 系统窗口

输入法窗口，系统错误提示窗口，Toast等都属于系统窗口，系统窗口的类型定义如下所示：

```java
public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {
	public static final int FIRST_SYSTEM_WINDOW     = 2000;//系统窗口类型初始值
	public static final int TYPE_STATUS_BAR         = FIRST_SYSTEM_WINDOW;//系统状态栏窗口
	public static final int TYPE_SEARCH_BAR         = FIRST_SYSTEM_WINDOW+1;//搜索条窗口
	public static final int TYPE_PHONE              = FIRST_SYSTEM_WINDOW+2;//通话窗口
	public static final int TYPE_SYSTEM_ALERT       = FIRST_SYSTEM_WINDOW+3;//系统ALERT窗口
	public static final int TYPE_KEYGUARD           = FIRST_SYSTEM_WINDOW+4;//锁屏窗口
	public static final int TYPE_TOAST              = FIRST_SYSTEM_WINDOW+5;//TOAST窗口
	...

	public static final int LAST_SYSTEM_WINDOW      = 2999;//系统窗口类型结束值
}
```

系统窗口的Type值范围为`2000到2999`。


对于不同类型的窗口添加过程会有所不同，但是对于WMS处理部分，添加的过程基本上是一样的， WMS对于这三大类的窗口基本是“一视同仁”的。

![window添加过程.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/408704ce59704f149d9dc26213a7f640~tplv-k3u1fbpfcp-watermark.image?)
#### 2.Window窗口层级关系(Z-Order)
首先我们得有个概念就是窗口是叠加的，可以简单理解为：**几页纸按前后顺序叠加在一块**。

**我们可以将手机屏幕使用X,Y,Z轴来表示，X,Y表示屏幕的长和宽，而Z轴垂直于屏幕，从屏幕内指向外。**

![z-order.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7261b2d39ed407c954e3a03a126151b~tplv-k3u1fbpfcp-watermark.image?)

确定窗口的层级关系其实就是确定窗口在Z轴上的位置，这个层级关系就是Z-Order，而Z-Order的排序依据就是前面分析的Window类型值。

**类型值越大，越靠近用户**。通过上面分析可知：**系统窗口>子窗口>应用程序窗口**。
这也就是系统窗口会覆盖应用窗口最直接的原因。

**那么如果多个窗口都是应用程序窗口如何显示呢？**
WMS会结合实际情况，去显示窗口合适的层级。


#### 3.Window窗口标志（Flag）
Window Flag用于控制窗口的一些事件：如是否接收触屏事件，窗口显示时是否允许锁屏，窗口可见时屏幕常亮，隐藏窗口等。
这里我们列出几个常用的标志：

- **1.FLAG_ALLOW_LOCK_WHILE_SCREEN_ON**

窗口可见时，允许在此窗口锁屏，一般需要结合FLAG_KEEP_SCREEN_ON和FLAG_SHOW_WHEN_LOCKED使用
- **2.FLAG_DIM_BEHIND**

该窗口后面的画面会变模糊，一般会有一个模糊值dimAmount。
- **3.FLAG_NOT_FOCUSABLE**

窗口不能获取焦点，设置这个Flag的同时FLAG_NOT_TOUCH_MODAL也会被设置
- **4.FLAG_NOT_TOUCHABLE**

该窗口获取不到输入事件，不可点击状态。
- **5.FLAG_NOT_TOUCH_MODAL**

在该窗口区域外的触摸事件传递给其他的Window,而自己只会处理窗口区域内的触摸事件
- **6.FLAG_KEEP_SCREEN_ON**

只要窗口可见，屏幕就会一直亮着
- **7.FLAG_LAYOUT_NO_LIMITS**

允许窗口超过屏幕之外
- **8.FLAG_FULLSCREEN**

隐藏所有的屏幕装饰窗口，比如在游戏、播放器中的全屏显示
- **9.FLAG_SHOW_WHEN_LOCKED**

窗口可以在锁屏的窗口之上显示
- **10.FLAG_IGNORE_CHEEK_PRESSES**

当用户的脸贴近屏幕时（比如打电话），不会去响应此事件
- **11.FLAG_TURN_SCREEN_ON**

窗口显示时将屏幕点亮
- **12.FLAG_DISMISS_KEYGUARD**

窗口显示时，键盘会关闭


**窗口标志Flag的2种设置方式：**

- 1.通过Window的addFlags方法

```java
Window mWindow =getWindow(); 
mWindow.addFlags(LayoutParams.FLAG_KEEP_SCREEN_ON);
```

- 2.通过Window的setFlags方法

```java
mWindow.setFlags(LayoutParams.FLAG_KEEP_SCREEN_ON,LayoutParams.FLAG_KEEP_SCREEN_ON);

```
addFlags方法内部也是调用setFlags

- 3.给LayoutParams设置Flag，并通过WindowManager的addView方法进行添加

```java
WindowManager.LayoutParams wl = new WindowManager.LayoutParams();
wl.flags = LayoutParams.FLAG_KEEP_SCREEN_ON;
Log.d("TAG","window:"+window.getWindowManager());
WindowManager mWindowManager = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
TextView mTextView=new TextView(this);
mTextView.setText("ttttttttdddddd");
Log.d("TAG","mWindow:"+mWindowManager);
mWindowManager.addView(mTextView,wl);
```


#### 4.Window 软键盘模式（SoftInputModel）
**我们的软键盘也是窗口的一种，属于系统窗口，层级较高，所以在一些场景下会覆盖层级较低的应用窗口**。

比如我们的登录场景，如果键盘弹出窗口处理不好，会覆盖输入登录按钮等，这样用户体验会非常糟糕。

于是WindowManager给我们提供了关于软键盘模式的Window窗口处理方式：

- **1.SOFT_INPUT_STATE_UNSPECIFIED**

没有指定状态,系统会选择一个合适的状态或依赖于主题的设置
- **2.SOFT_INPUT_STATE_UNCHANGED**	

不会改变软键盘状态
- **3.SOFT_INPUT_STATE_HIDDEN**

当用户进入该窗口时，软键盘默认隐藏
- **4.SOFT_INPUT_STATE_ALWAYS_HIDDEN**

当窗口获取焦点时，软键盘总是被隐藏
- **5.SOFT_INPUT_ADJUST_RESIZE**

当软键盘弹出时，窗口会调整大小

- **6.SOFT_INPUT_ADJUST_PAN**

当软键盘弹出时，窗口不需要调整大小，要确保输入焦点是可见的

软键盘模式设置方式：**getWindow().setSoftInputMode**

到这里，我们已经将Window的属性：`窗口类型（Type）`，`窗口层级（Z-Order）`，`窗口标志（Flag）`，`软键盘模式（softInputModel)`做了一个系统的讲解

**接下来我们来分析我们的View是如何显示到屏幕上**：

### 4.View是如何一步一步添加到屏幕上的？更新？删除呢？
通过前面我们对WindowManager体系分析知道，我们屏幕中所有的View首先需要经过WindowManager的处理，最后提交给WMS来处理。

我们先来看WindowManager的父类ViewManager：

```java
public interface ViewManager{
	public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

ViewManager只提供了三个接口方法：分别对应添加，更新，删除
- 1.**addView**：添加一个View到WMS中，第二个参数为当前View的参数。
- 2.**updateViewLayout**：更新当前View，第二个参数为需要更新的view参数
- 3.**removeView**：删除当前View

而这几个方法具体实现是在WindowManagerImpl类中（前面分析过）
直接来看WindowManagerImpl的三个对应方法:

```java
public final class WindowManagerImpl implements WindowManager{
	private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }	
    @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
	...
}
```


可以看到这三个方法都是委托给了WindowManagerGlobal进行处理，这是设计模式中的**桥接模式**。

进入WindowManagerGlobal中：
先分析addView：

```java
public void addView(View view, ViewGroup.LayoutParams params,
		Display display, Window parentWindow) {
	...
	ViewRootImpl root;
	//将params强转为WindowManager.LayoutParams类型
	final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
	synchronized (mLock) {
		
		//创建一个ViewRootImpl对象
		root = new ViewRootImpl(view.getContext(), display);
		//给View设置LayoutParams参数
		view.setLayoutParams(wparams);
		
		
		//存储view到mViews列表
		mViews.add(view);
		//存储root到mRoots列表
		mRoots.add(root);
		//存储wparams到mParams列表
		mParams.add(wparams);
		
		// do this last because it fires off messages to start doing things
		try {
			root.setView(view, wparams, panelParentView);
		} catch (RuntimeException e) {
			// BadTokenException or InvalidDisplayException, clean up.
			if (index >= 0) {
				removeViewLocked(index, true);
			}
			throw e;
		}
	}
}
```

- 1.首先创建一个ViewRootImpl对象root。
- 2.然后将当前view，当前params以及1中创建的ViewRootImpl对象root分别存储到对应的List列表中，注意这三者的index是一一对应的。
- 3.调用root的setView方法

ViewRootImpl身负了很多职责：

- 1.**管理View树，且其是View的根**
- 2.**触发三大绘制流程：测量，布局，绘制**
- 3.**输入事件中转站**
- 4.**管理Surface**
- 5.**负责与WMS通讯**

我们来看ViewRootImpl的setView方法：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
	...
	requestLayout();//注释1
	...
	res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);//注释2
	...
}
```

来看setView的注释1：**requestLayout**，屏幕绘制部分

我们在前面一篇文章《["一文读懂"系列：Android屏幕刷新机制](https://juejin.cn/post/7163858831309537294)》已经对原理进行解析过了，这里我们再大概说下:
**requestLayout内部主要使用垂直同步信号VSync的方式，在收到GPU提供的VSync信号后，会触发View的三大绘制layout，mesure，draw流程...**
需要知道完整机制的可以调到原文查看

这里我们重点来看setView的注释2：**mWindowSession.addToDisplay**

顾名思义：addToDisplay就是将Window添加到屏幕上，如何添加的呢？

mWindowSession是IWindowSession类型的，它是一个binder对象，用于进程间通讯，IWindowSession是C端代理，在S端使用的是Session类实现。addToDisplay运行在WMS进程中。

```java
mWindowSession = WindowManagerGlobal.getWindowSession();
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
                IWindowManager windowManager = getWindowManagerService();//1
                sWindowSession = windowManager.openSession(//2
                        new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                        ValueAnimator.setDurationScale(scale);
                                }
                        },
                        imm.getClient(), imm.getInputContext());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowSession;
    }
}

```

注释1处：**返回一个WMS对象的代理类**。在注释2处**调用WMS的代理类的openSession方法**

**定位到WMS的openSession方法**：


```java
@Override
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
		IInputContext inputContext) {
	if (client == null) throw new IllegalArgumentException("null client");
	if (inputContext == null) throw new IllegalArgumentException("null inputContext");
	Session session = new Session(this, callback, client, inputContext);
	return session;
}
```

可以看到返回的就是一个**Session对象**。

所以之前的mWindowSession.addToDisplay方法调用的就是Session类的addToDisplay方法

**此时已经进入了WMS进程**：
```java
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
		int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
		Rect outOutsets, InputChannel outInputChannel) {
	return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
			outContentInsets, outStableInsets, outOutsets, outInputChannel);
}
```


这里的**mService = WMS**服务，最终又回调到WMS中去了：

**调用WMS的addWindow方法添加Window，在WMS眼里，一切View都是以Window形式存在的，
剩下的工作就交由WMS进行处理了：在WMS中会为这个Window分配Surface，并确定显示层级，
可见负责显示界面的是画布Surface，而不是窗口本身，WMS将他管理的Surface交由SurfaceFlinger处理，SurfaceFlinger将这些Surface合并后放入到buffer中，屏幕会定时从buffer中获取显示数据，显示到屏幕上。**

![cpu和gpu.webp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8497048abf742edae516ea5a69d23e4~tplv-k3u1fbpfcp-watermark.image?)

同理我们再来分析下View删除流程：

定位到WindowManagerGlobal的removeView

```java
public void removeView(View view, boolean immediate) {
    ...
    synchronized (mLock) {
        int index = findViewLocked(view, true);//1
        View curView = mRoots.get(index).getView();//2
        removeViewLocked(index, immediate);//3
        if (curView == view) {
                return;
        }
        throw new IllegalStateException("Calling with view " + view
                        + " but the ViewAncestor is attached to " + curView);
    }
}
private int findViewLocked(View view, boolean required) {
    final int index = mViews.indexOf(view);
    if (required && index < 0) {
        throw new IllegalArgumentException("View=" + view + " not attached to window manager");
    }
    return index;
}
```

在注释1处获取view在mViews中的索引，在注释2处通过索引获取当前mRoots列表中view。**在注释3处调用removeViewLocked，这个是重点移出方法**。

```java
private void removeViewLocked(int index, boolean immediate) {
    ...
    boolean deferred = root.die(immediate);
    if (view != null) {
        view.assignParent(null);
        if (deferred) {
                mDyingViews.add(view);
        }
    }
}

```

看root.die方法：

```java
boolean die(boolean immediate) {
    ...
    //如果是同步移除，则立即调用doDie方法
    if (immediate && !mIsInTraversal) {
        doDie();
        return false;
    }
    ...
    //异步移除，发出一个msg进行移除
    mHandler.sendEmptyMessage(MSG_DIE);
    return true;
}
```


不管是同步还是异步，最终都会调用doDie方法进行移除。
进行doDie方法：

```java
void doDie() {
    ...
    synchronized (this) {
        if (mAdded) {
                dispatchDetachedFromWindow();//1
        }
        ..
    }
    WindowManagerGlobal.getInstance().doRemoveView(this);//4
}
```

doDie注释1：
这个方法里面做一些对资源销毁的操作：

```java
void dispatchDetachedFromWindow() {
    if (mView != null && mView.mAttachInfo != null) {
        //这里可以看出我们可以在View销毁前，在View的dispatchDetachedFromWindow做一些资源释放操作
        mView.dispatchDetachedFromWindow();
    }

    //销毁硬件渲染线程
    destroyHardwareRenderer();
    //将mView的root置为mull
    mView.assignParent(null);
    mView = null;
    mAttachInfo.mRootView = null;
    //释放当前Surface
    mSurface.release();
    //核心移除View的方法。
    try {
            mWindowSession.remove(mWindow);
    } catch (RemoteException e) {
    }
    //移除同步屏障以及View的绘制task
    unscheduleTraversals();
}
```


mWindowSession.remove(mWindow) 

在addView流程分析过：**mWindowSession是WMS进程中的Session类**

Session.java

```java
public void remove(IWindow window) {
    mService.removeWindow(this, window);
}
```

在 Session 中直接调用了 WindowManagerService 的 removeWindow(Session session, IWindow client) 方法。

WindowManagerService
```java
public void removeWindow(Session session, IWindow client) {
    synchronized(mWindowMap) {
        // 得到 windowstate 对象
        WindowState win = windowForClientLocked(session, client, false);
        if (win == null) {
            return;
        }
        // 进行移除 window 操作
        removeWindowLocked(win);
    }
}
```


先得到 WindowState 对象，再调用 removeWindowLocked 去移除该 WindowState 。而具体的 removeWindowLocked 代码我们在这就不深入了，可以自行研究。

至此，整个 Window 移除机制也分析完毕了。

## 总结
本篇文章基于Window模型以及源码对Window在整个Android FrameWork体系中所涉及的知识点以及作用做了一个整体的归纳，篇幅问题，关于WMS中如何对Window的操作放在下篇进行讲解。敬请期待。。

**如果觉得我的文章对你有帮助，请帮忙点个赞，关注下，谢谢。**

下面是博主的公众号：`小余的自习室`


































​    