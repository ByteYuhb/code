---
highlight: a11y-dark
---
## 前言
学了这么多年Android了，你真的了解Activity么，还是只是简单的会使用？
今天这篇文章笔者会通过问题的形式引出Activity的比较重要的几个知识点：

## 1.Activity的启动流程有了解么？
Activity启动流程分为根Activity和普通Activity的启动流程：
> 根Activity：即应用启动的第一个Activity：如SplashActivity，应用启动这个Activity前需要先创建应用的进程，并将启动进程的ActivityThread对象中的IApplicationThread对象传递给AMS，这时候AMS就持有了应用进程的binder索引，可以通过这个binder索引和应用进程通讯

> 普通Activity：即我们进程已经启动，且需要启动的Activity不是应用的第一个Activity

关于<mark>Activity启动流程的源码分析</mark>可以参考下面这篇文章:

[面试官：讲讲你对Activity启动机制的理解。。](https://juejin.cn/post/7112827024565076004/)

## 2.onSaveInstanceState(),onRestoreInstanceState的调用时机有哪些
### onSaveInstanceState
> 保存Activity当前状态，系统状态会自动保存，如果有自定义的属性状态需要重写onSaveInstanceState方法进行保存

`onSaveInstanceState系统调用场景：`
- 1.从最近应用中选择其他应用时
- 2.当用户按下Home键时
- 3.当屏幕切换时
- 4.Activity被系统异常回收时，如系统内存紧张时，可能被回收掉

### onRestoreInstanceState

> 恢复之前异常退出而保存的Activity状态

> onRestoreInstanceState(Bundle savedInstanceState)只有在activity确实是被系统回收，重新创建activity的情况下才会被调用

`onRestoreInstanceState系统调用场景：`
- 屏幕方向切换时，activity生命周期如下：onPause->onSaveInstanceState->onStop->onDestroy->onCreate->onStart->onRestoreInstanceState->onResume

- Activity在后台被回收时


### 对于onSaveInstanceState(),onRestoreInstanceState在系统中的逻辑如下：`

> 1.点击Home键后，系统会调用<mark>ActivityThread</mark>的<mark>performStopActivity</mark>，并将当前Activity状态保存到<mark>mActivity</mark>的信息表中
>
> 2.当Activity重启之后，会去<mark>mActivity</mark>中查找对应的<mark>ActivityClientRecord</mark>
> 如果有信息，则调用Activity的<mark>onResoreInstanceState</mark>方法，在<mark>ActivityThread</mark>的<mark>performLaunchActivity</mark>方法中，系统会判断<mark>ActivityClientRecord</mark>对象的<mark>state</mark>是否为空，不为空则通过Activity的<mark>onRestoreInstanceState</mark>获取其UI状态信息，将这些信息传递给Activity的<mark>onCreate</mark>方法

### 下面是onSaveInstanceState()和onRestoreInstanceState()基本使用方式
#### 1.说明：
> 这两个方法一般是由于系统约束导致的**Activity被异常回收**：则会回调onSaveInstanceState。让用户去保存自身状态，下次应用启动Activity的时候可以根据之前状态恢复
#### 2.流程
##### 破坏：onResume->onSaveInstanceState->onStop->onDestroy
##### 创建：onCreate->onStart->onRestoreIntanceState->onResume
#### 3.使用方式：
##### 3.1保存状态

```java
	static final String STATE_SCORE = "playerScore";
	static final String STATE_LEVEL = "playerLevel";
	...

	@Override
	public void onSaveInstanceState(Bundle savedInstanceState) {
		// 保存用户自定义的状态
		savedInstanceState.putInt(STATE_SCORE, mCurrentScore);
		savedInstanceState.putInt(STATE_LEVEL, mCurrentLevel);
		
		// 调用父类交给系统处理，这样系统能保存视图层次结构状态
		super.onSaveInstanceState(savedInstanceState);
	}
```
##### 3.2：恢复状态

```java
@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState); // 记得总是调用父类
	   
		// 检查是否正在重新创建一个以前销毁的实例
		if (savedInstanceState != null) {
			// 从已保存状态恢复成员的值
			mCurrentScore = savedInstanceState.getInt(STATE_SCORE);
			mCurrentLevel = savedInstanceState.getInt(STATE_LEVEL);
		} else {
			// 可能初始化一个新实例的默认值的成员
		}
		...
	}
	
	或者：
	public void onRestoreInstanceState(Bundle savedInstanceState) {
		// 总是调用超类，以便它可以恢复视图层次超级
		super.onRestoreInstanceState(savedInstanceState);
	   
		// 从已保存的实例中恢复状态成员
		mCurrentScore = savedInstanceState.getInt(STATE_SCORE);
		mCurrentLevel = savedInstanceState.getInt(STATE_LEVEL);
	}
```

## 3.activity的启动模式和使用场景，有了解么？
先来了解几个关键模型：

- `ActivityRecord`：一个`ActivityRecord`表示一个Activity的实例所有的信息
`来看下ActivityRecord源码：`

```java
frameworks/base/services/core/java/com/android/server/am/ActivityRecord.java
final class ActivityRecord extends ConfigurationContainer implements AppWindowContainerListener {

        final ActivityManagerService service; // owner
        final IApplicationToken.Stub appToken; // window manager token
        AppWindowContainerController mWindowContainerController;
        final ActivityInfo info; // all about me
        final ApplicationInfo appInfo; // information about activity's app
        
        //省略其他成员变量
        
        //ActivityRecord所在的TaskRecord
        private TaskRecord task;        // the task this is in.
        
        //构造方法，需要传递大量信息
        ActivityRecord(ActivityManagerService _service, ProcessRecord _caller, int _launchedFromPid,
                       int _launchedFromUid, String _launchedFromPackage, Intent _intent, String _resolvedType,
                       ActivityInfo aInfo, Configuration _configuration,
                       com.android.server.am.ActivityRecord _resultTo, String _resultWho, int _reqCode,
                       boolean _componentSpecified, boolean _rootVoiceInteraction,
                       ActivityStackSupervisor supervisor, ActivityOptions options,
                       com.android.server.am.ActivityRecord sourceRecord) {
        
        }
    }
```

ActivityRecord总结：

	1.实际上，ActivityRecord中存在着大量的成员变量，包含了一个Activity的所有信息。
	2.ActivityRecord中的成员变量task表示其所在的TaskRecord，由此可以看出：ActivityRecord与TaskRecord建立了联系。
	3.startActivity()时会创建一个ActivityRecord：

- `TaskRecord`：一个TaskRecord中包含一组`ActivityRecord`的集合，每个ActivityRecord只属于一个任务栈
`TaskRecord源码分析`

```java
frameworks/base/services/core/java/com/android/server/am/TaskRecord.java
final class TaskRecord extends ConfigurationContainer implements TaskWindowContainerListener {
        final int taskId;       //任务ID
        final ArrayList<ActivityRecord> mActivities;   //使用一个ArrayList来保存所有的ActivityRecord
        private ActivityStack mStack;   //TaskRecord所在的ActivityStack
        
        //构造方法
        TaskRecord(ActivityManagerService service, int _taskId, ActivityInfo info, Intent _intent,
                   IVoiceInteractionSession _voiceSession, IVoiceInteractor _voiceInteractor, int type) {
            
        }
        
        //添加Activity到顶部
        void addActivityToTop(com.android.server.am.ActivityRecord r) {
            addActivityAtIndex(mActivities.size(), r);
        }
        
        //添加Activity到指定的索引位置
        void addActivityAtIndex(int index, ActivityRecord r) {
            //...

            r.setTask(this);//为ActivityRecord设置TaskRecord，就是这里建立的联系

            //...
            
            index = Math.min(size, index);
            mActivities.add(index, r);//添加到mActivities
            
            //...
        }

        //其他代码略
    }
```
`TaskRecord`总结：

	1.可以看到TaskRecord中使用了一个ArrayList来保存所有的ActivityRecord
	2.同样，TaskRecord中的mStack表示其所在的ActivityStack
	3.startActivity()时也会创建一个TaskRecord
- `ActivityStack`：是一个`TaskRecord`的集合，内部包含N个TaskRecord：主要是包括`HomeStack`(Launcher任务栈)和`FocusStack`(应用任务栈)
`ActivityStack源码部分：`

```java
frameworks/base/services/core/java/com/android/server/am/ActivityStack.java
class ActivityStack<T extends StackWindowController> extends ConfigurationContainer
        implements StackWindowListener{
	private final RecentTasks mRecentTasks;最近的Task任务，可以在回退任务栈中看到该任务
	/**
     * The back history of all previous (and possibly still
     * running) activities.  It contains #TaskRecord objects.
     */
    private final ArrayList<TaskRecord> mTaskHistory = new ArrayList<>();TaskRecord集合
	/**
     * List of running activities, sorted by recent usage.
     * The first entry in the list is the least recently used.
     * It contains HistoryRecord objects.
     */
    final ArrayList<ActivityRecord> mLRUActivities = new ArrayList<>();包括最新缓存使用的Activity，这样就不用每次都去遍历TaskRecord集合中的ActivityRecord，浪费资源
	/** Run all ActivityStacks through this */
    protected final ActivityStackSupervisor mStackSupervisor;
	
	TaskRecord createTaskRecord(int taskId, ActivityInfo info, Intent intent,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            boolean toTop, int type) {
        TaskRecord task = new TaskRecord(mService, taskId, info, intent, voiceSession,新建一个TaskRecord
                voiceInteractor, type);
        // add the task to stack first, mTaskPositioner might need the stack association
        addTask(task, toTop, "createTaskRecord");    
        return task;
    }
	//添加Task
	void addTask(final TaskRecord task, final boolean toTop, String reason) {

		addTask(task, toTop ? MAX_VALUE : 0, true /* schedulePictureInPictureModeChange */, reason);

		//其他代码略
	}
	// TODO: This shouldn't allow automatic reparenting. Remove the call to preAddTask and deal
    // with the fall-out...
	//添加Task到指定位置
    void addTask(final TaskRecord task, int position, boolean schedulePictureInPictureModeChange,
            String reason) {
        // TODO: Is this remove really needed? Need to look into the call path for the other addTask
        mTaskHistory.remove(task);
        position = getAdjustedPositionForTask(task, position, null /* starting */);
        final boolean toTop = position >= mTaskHistory.size();
        final ActivityStack prevStack = preAddTask(task, reason, toTop);

        mTaskHistory.add(position, task);
        task.setStack(this);

        if (toTop) {
            updateTaskReturnToForTopInsertion(task);
        }

        updateTaskMovement(task, toTop);

        postAddTask(task, prevStack, schedulePictureInPictureModeChange);
    }
}
```
`ActivityStack`小结：

	1.里面存储了一个TaskRecord的List集合
	2.持有一个ActivityStackSupervisor对象，这个对象是用来管理ActivityStack

- `ActivityStackSupervisor`:这个对象是用来管理`ActivityStack`

```java
frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java
public class ActivityStackSupervisor extends ConfigurationContainer implements DisplayListener {

        ActivityStack mHomeStack;//管理的是Launcher相关的任务

        ActivityStack mFocusedStack;//管理非Launcher相关的任务
        
        //创建ActivityStack
        ActivityStack createStack(int stackId, ActivityStackSupervisor.ActivityDisplay display, boolean onTop) {
            switch (stackId) {
                case PINNED_STACK_ID:
                    //PinnedActivityStack是ActivityStack的子类
                    return new PinnedActivityStack(display, stackId, this, mRecentTasks, onTop);
                default:
                    //创建一个ActivityStack
                    return new ActivityStack(display, stackId, this, mRecentTasks, onTop);
            }
        }

    }
```
`ActivityStackSupervisor`小结：

	1.里面存储了两个ActivityStack，分别对应Launcher和非Launcher任务
	2.内部调用了创建ActivityStack的构造方法
	3.AMS初始化的时候会创建一个ActivityStackSupervisor



<mark>ActivityRecord，TaskRecord，ActivityStack</mark>三者之间关系图：

![桌面栈.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/88388603666c4e0897010d50df2a2f61~tplv-k3u1fbpfcp-watermark.image?)

> 我们介绍了启动模式关联的几个类之后，下面进入正题：

**Activity启动模式：**

- 1:`standard`:标准启动模式,默认启动模式，每次新建一个Activity都会创建一个ActivityRecord
> 				场景：邮件 默认模式

*下面我们使用命令行方式获取Activity的层级信息：*

```java
adb shell dumpsys activity
```
在启动应用界面再次以`Standard`方式启动一个Activity：

`1.1:默认模式从A-B-C`

```java
Display #0 (activities from top to bottom):
	//表示id为1的ActivityStack，应用stack
	  Stack #1:
	  ...
		//表示id为1的ActivityStack中的id为878的TaskRecord，内部包含3个ActivityRecord实例
		Task id #878
		...
		  TaskRecord{9dd2c54 #878 A=com.android.yuhb.test U=0 StackId=1 sz=3}
		  Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.android.yuhb.test/.AActivity }
			//ActivityRecord:id=2,CActivity的实例
			Hist #2: ActivityRecord{6905c8a u0 com.android.yuhb.test/.CActivity t878} 
			  Intent { cmp=com.android.yuhb.test/.CActivity }
			  //当前Activity的属于哪个进程：ProcessRecord表示一个进程记录
			  ProcessRecord{91f3fa7 24522:com.android.yuhb.test/u0a89}
			//ActivityRecord:id=1,BActivity的实例
			Hist #1: ActivityRecord{a3ae8dd u0 com.android.yuhb.test/.BActivity t878}
			  Intent { cmp=com.android.yuhb.test/.BActivity }
			  ProcessRecord{91f3fa7 24522:com.android.yuhb.test/u0a89}
			//ActivityRecord:id=0,AActivity的实例：android.intent.action.MAIN表示是根Activity
			Hist #0: ActivityRecord{1ebc783 u0 com.android.yuhb.test/.AActivity t878}
			  Intent { cmp=com.android.yuhb.test/.AActivity }
			  Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.android.yuhb.test/.AActivity bnds=[437,373][644,651] }
		//这个表示当前Display内部最近运行的Activity
		Running activities (most recent first):
		  TaskRecord{9dd2c54 #878 A=com.android.yuhb.test U=0 StackId=1 sz=2}
			Run #1: ActivityRecord{a3ae8dd u0 com.android.yuhb.test/.BActivity t878}
			Run #0: ActivityRecord{1ebc783 u0 com.android.yuhb.test/.AActivity t878}
		//这个表示当前正处于Resume状态的Activity
		mResumedActivity: ActivityRecord{a3ae8dd u0 com.android.yuhb.test/.BActivity t878}  
```
- **可以看到：在Task id #878中<mark>多出了一个Hist #1和Hist #1: ActivityRecord CActivity</mark>且放在任务顶端。**
**结果如图：A-B-C(Standard):**

![A-B-C(Standard).png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f6479ab32fa8494390efabca4f35694a~tplv-k3u1fbpfcp-watermark.image?)
- 2:`singleTop`：如果需要启动的Activity已经在任务栈的栈顶，则**不会重新创建Activity**，而是调用Activity的`onNewIntent`方法
> 				场景：登录页面，推送通知栏等

  **1.2:在1.1基础上启动singleTop模式的C:**
>
>   获取的结果和前面是一样，且C的onNewIntent回调了
>   **结果如图:A-B-C(Standard)-C(singleTop):**


![A-B-C(Standard)-C(singleTop).png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a954c21f0e94f60bc140a78d873519f~tplv-k3u1fbpfcp-watermark.image?)
- 3:`singletask`: 如果需要启动的Activity已经在任务栈中，则不会重新创建Activity，而是把该Activity栈上面的其他Activity出栈，然后调用Activity的onNewIntent方法
> 				场景：主页面,WebView页面、扫一扫页面 电商中：购物界面，确认订单界面，付款界面

**1.3:在1.1基础上启动singleTask模式的B:**

`结果：`

```java
Display #0 (activities from top to bottom):
	  Stack #1:
	  ...
		Task id #878
		...
		  TaskRecord{9dd2c54 #878 A=com.android.yuhb.test U=0 StackId=1 sz=3}
		  Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.android.yuhb.test/.AActivity }
			Hist #1: ActivityRecord{a3ae8dd u0 com.android.yuhb.test/.BActivity t878}
			  Intent { cmp=com.android.yuhb.test/.BActivity }
			  ProcessRecord{91f3fa7 24522:com.android.yuhb.test/u0a89}
			Hist #0: ActivityRecord{1ebc783 u0 com.android.yuhb.test/.AActivity t878}
			  Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.android.yuhb.test/.AActivity bnds=[437,373][644,651] }

		Running activities (most recent first):
		  TaskRecord{9dd2c54 #878 A=com.android.yuhb.test U=0 StackId=1 sz=2}
			Run #1: ActivityRecord{a3ae8dd u0 com.android.yuhb.test/.BActivity t878}
			Run #0: ActivityRecord{1ebc783 u0 com.android.yuhb.test/.AActivity t878}
```
> 在B栈上方的C栈出栈，且回调了B的`onNewIntent`。

![A-B-C(Standard)-B(singleTask).png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/61ae74576eeb48c0a0c11e26def65213~tplv-k3u1fbpfcp-watermark.image?)
- 4:`singleInstance`：启动的Activity放在单独的一个任务栈中，且该任务栈有且只有一个Activity
> 				场景：系统Launcher、锁屏键、来电显示等系统应用
> 		**1.4:在1.1基础上启动singleInstance模式的D**:
> 		`结果：`

```java
TaskRecord{7e15509 #889 A=com.android.yuhb.test U=0 StackId=1 sz=1}
        Run #3: ActivityRecord{f8cc6dc u0 com.android.yuhb.test/.DActivity t889}
      TaskRecord{4f95fb0 #888 A=com.android.yuhb.test U=0 StackId=1 sz=3}
        Run #2: ActivityRecord{d76f950 u0 com.android.yuhb.test/.CActivity t888}
        Run #1: ActivityRecord{ff2c64c u0 com.android.yuhb.test/.BActivity t888}
        Run #0: ActivityRecord{fec30c1 u0 com.android.yuhb.test/.AActivity t888}
```
> 可以看到在stack#1中新建了一个TaskRecord，里面存储了一个D的ActivityRecord
> `结果如图：`

![A-B-C(Standard)-D(singleInstance).png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/618d7f5202934bfa8f41baf35f71faeb~tplv-k3u1fbpfcp-watermark.image?)

**1.5:在1.4基础上调用A**

```java
Running activities (most recent first):
      TaskRecord{1db3cfe #892 A=com.android.yuhb.test U=0 StackId=1 sz=4}
        Run #4: ActivityRecord{986782d u0 com.android.yuhb.test/.AActivity t892}
      TaskRecord{81e0b9 #893 A=com.android.yuhb.test U=0 StackId=1 sz=1}
        Run #3: ActivityRecord{b042fbb u0 com.android.yuhb.test/.DActivity t893}
      TaskRecord{1db3cfe #892 A=com.android.yuhb.test U=0 StackId=1 sz=4}
        Run #2: ActivityRecord{12459a8 u0 com.android.yuhb.test/.CActivity t892}
        Run #1: ActivityRecord{c5c2e24 u0 com.android.yuhb.test/.BActivity t892}
        Run #0: ActivityRecord{890e0cc u0 com.android.yuhb.test/.AActivity t892}
```
> 可以看到在singleInstance任务栈上创建其他Activity会在原TaskRecord#892上创建了一个ActivityRecord(这个和taskAffinity有关)，所以singleInstance任务栈上只能有一个ActivityRecord

**讲解了Activity的四种启动方式：那我们来看下Activity启动给的时候设置的Intent的`Flag`有哪些作用呢？**
> Intent的`Flag`优先级比在Activity中设置`launchMode`高，**这样可以让第三方应用启动Activity的时候可以设置自己的启动模式**

- Intent.`FLAG_ACTIVITY_NEW_TASK`
分`两种`情况:
    **1.在`一`个应用中调用 2.在`两`个应用中调用**

- 1.在一个应用中启动Activity：结果和标准启动方式没区别：
> 如果设置了android:`taskAffinity`=“com.allinpay.usdk”之后，就会新建一个taskAffinity=“com.allinpay.usdk”的任务栈
> 模型如图：A-B(FLAG_ACTIVITY_NEW_TASK)(taskAffinity)	

![A-B(FLAG_ACTIVITY_NEW_TASK)(taskAffinity).png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/97925ad72e7448b68f959fd5b2a07017~tplv-k3u1fbpfcp-watermark.image?)

- 2.在`两个应用`中设置。不需要设置`taskAffinity`，默认是另外一个应用的包名

- Intent.`FLAG_ACTIVITY_CLEAR_TOP`
> singleTask默认具有此标记位的效果

- Intent.`FLAG_ACTIVITY_SINGLE_TOP`
> singleTop默认具有此标记位的效果

- Intent.`FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS`
> 其对应在`AndroidManifest`中的属性为`android:excludeFromRecents=“true”`,当用户按了“`最近任务列表`”时候,该`Task`不会出现在最近任务列表中，可达到<mark>隐藏应用的目的</mark>。

- 从一个应用跳转到另外一个应用：
1.不设置`flag`：新启动的Activity会自动放到当前任务栈中。

```java
Running activities (most recent first):
      TaskRecord{e27ebdf #919 A=com.android.yuhb.test U=0 StackId=1 sz=2}
        Run #1: ActivityRecord{a3793ad u0 com.landicorp.android.shouyinbao.chinalife/.db.activity.LoginActivity t919}
        Run #0: ActivityRecord{c076d88 u0 com.android.yuhb.test/.AActivity t919}
```
2.设置flag=`FLAG_ACTIVITY_NEW_TASK`：会在一个新的任务栈中，任务栈的`taskAffinity`默认为新应用的包名（会有明显跳转卡顿动作）

```java
Running activities (most recent first):
      TaskRecord{f65c6ed #922 A=com.landicorp.android.shouyinbao.chinalife U=0 StackId=1 sz=1}
        Run #1: ActivityRecord{361fc67 u0 com.landicorp.android.shouyinbao.chinalife/.db.activity.LoginActivity t922}
      TaskRecord{9401922 #921 A=com.android.yuhb.test U=0 StackId=1 sz=1}
        Run #0: ActivityRecord{4cf7d7d u0 com.android.yuhb.test/.AActivity t921}
```
*`好了，关于Activity的启动模式就讲到这里`*

## 4.Activity A跳转Activity B，再按返回键，生命周期执行的顺序
#### 正常情况：
A调到B：

```java
A onPause-> B onCreate-> B onStart-> B onResume-> A onStop
```
B到A返回：

```java
B onPause-> A onRestart-> A onStart->A onResume -> B onStop->B onDestroy
```
#### 下面是特殊情况：
- ....B 的启动模式为singleTop：
`则先判断任务栈栈顶是否元素是否和需要启动的是一个类的实例，如果不是，则启动模式和正常情况一样，如果是，则直接回调:`

```java
B onPause -> B onNewIntent->B onResume
```
- ....B 的启动模式为singleTask：
`则先判断任务栈中是否有该Activity的实例，如果已经有，则周期：`

```java
A->onPause-> B.onNewIntent -> B.onRestart -> B.onStart -> B.onResume -> A.onStop -> A onDestroy
```
- ....B 的启动模式为singleInstance
`则会新建一个任务栈：所以生命周期：`

```java
A-onPause->B onCreate->B onRestart-> B onResume->A onStop
```
## 5.横竖屏切换,按home键,按返回键,锁屏与解锁屏幕,跳转透明Activity界面,启动一个 Theme 为 Dialog 的 Activity，弹出Dialog时Activity的生命周期
- 5.1:`横竖屏切换`：

**如果配置了**:
```java
android：configChanges ="orientation|keyboardHidden|screenSize"
```
`则不会重新调用Activity的生命周期，而是回调onConfigurationChanged方法`。

`如果没有配置android:configChanges或者配置不对，则其生命周期为:` 


```java
onPause->onSaveInstanceState->onStop->onCreate->onStart-onRestoreInstanceState->onResume
```
- 5.2:`按home键`:
`按home键后Activity会置于后台：生命周期：`

```java
onPause->onSaveInstanceState->onStop->重新启动->onRestart->onStart->onResume
```
> 按Home键并没有回调onRestoreInstanceState，因为不是系统异常回收Activity
- 5.3:`按back键`：
```java
onPause->onStop->onDestroy->onCreate->onStart->onResume
```
- 5.4:`锁屏`：
    只会调用onPause-> 开屏：onResume;
- 5.5:`弹出Dialog`:直接是通过 WindowManager.addView 显示的（没有经过 AMS），所以不会对生命周期有任何影响。

- 5.6:启动theme为DialogActivity,跳转透明Activity

```java
A.onPause -> B.onCrete -> B.onStart -> B.onResume
```
> Activity 不会回调 onStop，因为只有在 Activity 切到后台不可见才会回调 onStop

## 6.Activity之间传递数据的方式Intent是否有大小限制，如果传递的数据量偏大，有哪些方案?
- 6.1：我们先来理解一个`intent`传输数据的过程：
```java
A Activity--->binder通讯-->AMS-->binder通讯--> B Activity
```
可以看出`Activity`直接传递数据是通过`binder机制`传输的，`binder机制`对应用传递数据有一个大小限制`1M-8K`，
	这个是`MMAP`接收数据的缓冲区大小，所以实际应用传递的数据要`小余1M-8K`.因为数据传输还有`head`和`body`两部分。

- 6.2：`MMAP`机制：
> 服务启动的时候，会调用`mmap`函数，在内核开辟一块`内核缓冲区空间`，这块空间指向一个`物理内存地址`，而在`用户空间`也对同一块`物理内存地址`做了`映射`，当有数据来的时候，`先放到内核缓冲区`，系统会自动将数据`映射到物理内存`，而用户空间因为也做了映射，可以`直接使用这块物理内存地址中的数据`，这就是`binder一次拷贝`的机制原理

- 6.3：binder本身就不是用来传递大数据，而传递大数据建议使用：实时性要求已经高使用`单例类`，`全局变量`等，对数据实时性要求不高的且需要`永久性`的数据可以放到文件里面或者`数据库sqlite`，`shardPreference`等

- 6.4：和其他几种ipc通讯方式的比较：
> `共享内存`:只需要一次拷贝，速度很快，但是对开发者要求比较高，难于控制

> `socket或者管道`:需要两次拷贝，性能比较低

## 7.显示启动和隐式启动
 - 7.1：显示启动：设置intent的Compoment
 - 7.2: 隐式启动:
> 1.`action`：`intent-filte`r里面可以有多个，intent中只有一个，只要intent-filter里面有这个`action`就匹配成功
>
> 2.`category`：`intent-filter`里面可以有多个，`intent`里面也可以有多个，intent里面的`category`要全部都能在intent-filter里面找到才算匹配成功
>
> 3.`data`：intent-filter中可以设置多个`data`，`intent`中只能`设置一个`且必须设置 用来传递`uri`数据，一般传递数据使用`putExtra`就可以了

## 8.ANR 的四种场景
- `Service TimeOut`: service 未在规定时间执行完成：前台服务 `20s`，后台 `200s`
- `BroadCastQueue TimeOut`: 未在规定时间内未处理完广播：前台广播` 10s` 内, 后台` 60s` 内
- `ContentProvider TimeOut`: publish 在 `10s` 内没有完成
- `Input Dispatching timeout`: `5s` 内未响应`键盘输入`、`触摸屏幕`等事件
> 可以看到`主线程阻塞`并不在`ANR`的场景里面，只是这个是导致`ANR`的前提，如果主线程阻塞那就没办法处理其他事件，如果系统再有上面所说几种情况发生就会出现`ANR`

## 9.如何控制Activity被第三方访问的权限
- 1.设置`shardUid`，两个应用设置同一个`ui`可以访问各自的应用文件
- 2.设置`permission`：服务Activity设置一个权限，客户端Activity访问的时候需要带有对应的`权限`才能访问该`Activity`，最好的控制方法
- 3.设置`export`为`false`，可以不让`第三方`访问自己，默认为`false`，如果设置了`inten-filter`，则默认为`true`，安全性不高

## 10.Activity的数据是怎么保存的,进程被Kill后,保存的数据怎么恢复的？
在Activity的`onSaveInstanceState`方法回调时，put到参数`outState（Bundle）`里面

`outState`就是`ActivityClientRecord`的`state`。
`ActivityClientRecord`实例，都存放在`ActivityThread`的`mActivities`里面.

`Activity`变得不可见时（onSaveInstanceState和onStop回调之后），在应用进程这边会通过`ActivityTaskManagerService`的`activityStopped`方法，把刚刚在`onSaveInstanceState`中满载了数据的`Bundle`对象传到`系统服务进程`那边！ 然后（在系统服务进程这边），会进一步将这个Bundle对象，赋值到对应`ActivityRecord`

`ActivityRecord`存储在一个`TaskRecord`里面，`TaskRecord`存储在一个`ActivityStack`里面。最后在下次启动的时候，通过`ActivityStack`找到`ActivityRecord`，最终判断`ActivityRecord`里面的状态，回调`onRestoreInstance`方法

 ## 11.屏幕旋转你了解多少？
 - 1.屏幕方向属性：
	- 1.`unspecified`：默认值，由系统决定，不同的手机可能不一样
	- 2.`landscape`：强制横屏显示
	- 3.`portrait`;强制竖屏显示
	- 4.`sensor`：根据物理传感器方向转动，用户90度、180度、270度旋转手机方向，activity都更着变化，会重启activity（无论系统是否设置为自动转屏，activity页面都会跟随传感器而转屏）
	- 5.`nosensor`：旋转设备时候，界面不会跟着旋转。初始化界面方向由系统控制（无论系统是否设置为自动转屏，activity页面都不会转屏）
	- 6.`sensorLandscape`;横屏两个方向上下旋转
	- 7.`sensorPortrait`：竖屏两个方向上下旋转，有些手机会屏蔽上下颠倒的竖屏旋转，我用vivo ioq3测试就是系统屏蔽了。
- 2.设置方式
    - 1.在`manfest`中设置`screenOrientation`
    - 2.`代码动态`设置：
```java
setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);//横屏设置
```
- 3.屏幕旋转后Activity生命周期：
    `两种情况`：
     - 1.设置了`android:configChanges=”orientation|screenSize“`，则`不会重建Activity`，会回调`onConfigurationChanged`。
> 但是`onConfigurationChanged`只对旋转`90`度有效果，`180`则不会回调>`onConfigurationChanged`:
>
> 例如：`sensorLandscape`属性，旋转的是`180°`不会回调`onConfigurationChanged`也不会重新创建`Activity`

    -2.没有设置,则重新创建Activity。

- 4.onConfigurationChanged总结
>1.只有设置了configChanges属性，才会在屏幕配置发生改变的时候，响应相应的onConfigurationChanged回调，例如设置了orientation|screenSize。mcc或keyboard等配置更改不会回调onConfigurationChanged
>
>2.只有Activity生命周期处于Resumed状态的时候才会回调onConfigurationChanged。如果是其他状态：则会将新的状态保存下来，待下次Activity处于resume状态后进行配置更改，回调onConfigurationChanged，
>但是这种情况下AMS可能会在处理新的配置前就调用了销毁Activity操作


```java
if (!ar.activity.mFinished && (allActivities || !ar.paused)) {
                        // If the activity is currently resumed, its configuration
                        // needs to change right now.
                        callbacks.add(a);
		} else if (thisConfig != null) {
			// Otherwise, we will tell it about the change
			// the next time it is resumed or shown.  Note that
			// the activity manager may, before then, decide the
			// activity needs to be destroyed to handle its new
			// configuration.
			if (DEBUG_CONFIGURATION) {
				Slog.v(TAG, "Setting activity "
						+ ar.activityInfo.name + " newConfig=" + thisConfig);
			}
			ar.newConfig = thisConfig;
		}
```

## 总结：
以上就是本人对Activity一些基本用法和类的概念总结，主要用于`巩固Android`的一些基本知识，后面会推出更多关于`Android的基础知识和高阶知识`，欢迎大家喜欢。