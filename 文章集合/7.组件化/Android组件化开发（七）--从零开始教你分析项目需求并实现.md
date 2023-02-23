
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言：
前面几篇文章我们封装了几个组件化功能组件：包括：**网络请求组件，图片加载请求组件，应用保活组件，音乐播放组件封装。**

> 每个组件都可以直接拿到自己项目中使用,当然还需根据自己项目要求进行优化。



![组件化开发.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0127f2cbd10640128c437e163c7dea80~tplv-k3u1fbpfcp-watermark.image?)
**组件化相关系列文章：**

Android组件化开发（一）--**Maven私服的搭建**(https://juejin.cn/post/7118646272323485709)

Android组件化开发（二）--**网络请求组件**封装(https://juejin.cn/post/7119281692350611493)

Android组件化开发（三）-**-图片加载组件**封装(https://juejin.cn/post/7120791098775044103)

Android组件化开发（四）--**进程保活组件**的封装(https://juejin.cn/post/7121643256495996936)

Android组件化开发（五）--**完整版音乐播放组件**的封装(https://juejin.cn/post/7122869564299280415)

**今天我们先来聊个话题：**各位在接到一个项目，或者一个需求的时候，是如何去实行的？


- **1.先分析需求，然后一边代码，一边想思路，最后实现了项目组给的需求任务**。
> 这种方式写出来的代码，往往质量较差，类与类之间，模块与模块之间耦合一般都比较严重，而且很容易出bug，现实中这类型的开发其实不占少数，
> **笔者在刚开始入行的时候，就是这样，没有一个很好的事件流去参考。**
- 2.**分析需求后，再画`流程图`，根据流程图，设计好你需要的类以及待实现的功能**，`类与类之间的关系`是继承还是组合关系，`回调机制`该如何设计，是否需要使用到`线程池`，使用哪种线程池？对象或者代码是否可以`复用`？`可扩展性`？设计出一张`UML`类图，最后根据我们前面分析的步骤去写代码.

> **注意写代码的时候也是有技巧的，不要急于实现某个类的具体功能，**
> **先把类的`架子`搭起来。然后，再去一部一部实现内部的逻辑，这样我们的类就会`越来越饱满`**

**下面笔者用一个例子来带大家了解下第二种方式的项目实践过程**。

**任务**：项目组让我们去封装一个**应用更新的组件**

首先我们需要对需求进行分析：我们参考大部分应用更新的过程：

> UI上有个按钮，点击之后，会去服务器获取是否有版本需要更新，检测到新版本后，这个时候会弹出一个提示Dialog，点击确认后我们就会去服务器下载数据，下载到一个文件中，然后调用原生的安装方法去安装。

根据上面的需求分析：**笔者画了一个基本的流程图**

![下载更新流程图.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09036f8eaf6445d6a1cd26d9a77d77e8~tplv-k3u1fbpfcp-watermark.image?)

好了有了流程图后：就要开始设计我们的类了。

- **1.为了响应点击事件，可以设计一个UpdateHelper类：内部提供一个`checkUpdate`的方法:**

```java
public class UpdateHelper{
	//这个方法是直接供UI层调用的方法，可能需要传入一个上下文下来
	public static checkUpdate(Context context){
		//1.传入url去服务器拿版本信息，这里如果获取失败，需要提供一个异常响应给上层，不然UI层会以为我们代码有问题了
		//2.对比当前版本信息和获取的版本信息
		//3.弹出一个提示更新的Dialog,并显示当前版本更新的内容。
		//4.用户点击确认后就开始去服务器下载apk了
	}
}
```

- 2**.通过对checkUpdate方法的分析，我们了解到：**

  - 1.需要设计一个版本更新提示的`Dialog`。暂且起名：`UpdateDialog`

  - 2.因为需要去服务器下载`apk`数据，可能得需要起一个后台`Service`去下载数据。


- **3.好接下来我们设计这个下载任务的的`Service`**：

`UpdateService`


```java
public UpdateService extends Service{
	
	onStartCommand(){
		//1.startDownLoad()
	}
	//实现数据的下载以及监听
	public startDownLoad(){
	
	}
}
```


这个`UpdateService`中我们目前只让他实现一个`startDownLoad`方法去下载数据。接下来就是设计这个`startDownLoad`了，我们可不可以把下载代码都写在这个方法中呢，当然是可以的，但是笔者觉得这样代码写的`太笨重了，不够优雅`，且不符合设计模式中说的`单一职责`，

- **4.于是笔者设计了一个`UpdateManager`的类**:**这个类是用来对当前下载请求做一些前期的准备工作**：
  比如：

  - **1.检查当前是否已经有下载请求在处理了，忽略重复请求**

  - **2.判断当前文件和文件夹是否存在，是否有必要预先新建一个文件。**

  - **3.创建一个下载任务`Runnable`放到线程池中。**


根据上面分析，我们需要创建一个下载任务的`Runnable`，
新建一个`UpdateRequestRunnable`继承`Runnable`

```java
class UpdateRequestRunnable extends Runnable{
	//这个方法中才是真正去实现下载任务请求,创建一个realDownLoadApk，让用户知道哦这里才是真正的去下载。
	run{
		realDownLoadApk()
	}
	
	realDownLoadApk(){
		//1下载开始。。
		//2下载中 
		//3下载结束
	
	}

}
```

我们想下，下载进度，下载成功还是失败等是不是都要通知到用户呢？
是的，**我们需要在`通知栏`起一个下载的通知提示用户我们目前的`下载进度和下载状态`。**

那这个通知在哪个类里面设计呢？

其实你在`UpdateManager`或者`UpdateService`中去处理都可以。
笔者是在`UpdateService`的`OnCreate`方法中初始化了一个通知`Notification`。

那有了通知，通知如何知道到底现在的下载进度，就需要使用到回调了，

- **5.所以我们创建一个`UpdateStatusListener`，用来监听当前下载进度.**

别忘了，我们还有最重要的一部，将下载的apk进行安装，安装任务放在哪个类里面好呢

- **6.笔者这里用了一个`LocalBroadcastManager`**

  - **注册时机**：启动`UpdateService`的时候。

  - **发送时机**：回调下载完成的时候。

  - **反注册**：`UpdateService Destroy`的时候。


`尽量做到闭环`。

**以上就是整个应用更新任务的分析过程:**

- 7.有了上面的分析，**可以大概画出一个基本的UML类图**：有了类图我们就可以将代码功能补全啦。
  ![应用更新.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d6c9ae664a7429ebd3ea34ba4aab32f~tplv-k3u1fbpfcp-watermark.image?)

- 1.**开发中注意点：`Android10权限升级`：对于访问文件系统,需要添加**：

```java
android.permission.MOUNT_UNMOUNT_FILESYSTEM
```
并在application中设置：

```java
android:requestLegacyExternalStorage="true"
```

- **2.如果需要应用间访问文件：需要使用：FileProvider进行文件分享**

```java
<!--文件共享-->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

- **3.高版本上会出现通知栏无法显示的情况：需要判断通知栏是否打开，并调用显示通知栏界面的api，手动开启通知权限。**

## 总结：
笔者使用一个应用更新的例子，阐述了我们接到需求到需求实践的整个过程。

> **学习最好的方式是学习别人的思维方式**

目前已将这个应用更新的组件作为一个类库，成为了我们组件化大家庭的一部分。
,**：https://github.com/ByteYuhb/anna_music_app


![组件化开发.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ffc7494a8cd24eccad95641612f84d73~tplv-k3u1fbpfcp-watermark.image?)