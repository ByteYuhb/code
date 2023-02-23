---
theme: smartblue
highlight: a11y-dark
---
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## `前言`：
前面几篇文章我们封装了几个组件化功能组件：
包括：`网络请求组件`，`图片加载请求组件`，`应用保活组件`，`音乐播放组件封装`。

> 每个组件都可以直接拿到自己项目中使用,当然还需根据自己项目要求进行优化。



![组件化开发.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0127f2cbd10640128c437e163c7dea80~tplv-k3u1fbpfcp-watermark.image?)
**组件化相关系列文章：**

 [Android组件化开发（一）--Maven私服的搭建](https://juejin.cn/post/7118646272323485709)

[ Android组件化开发（二）--网络请求组件封装](https://juejin.cn/post/7119281692350611493)

[Android组件化开发（三）--图片加载组件封装](https://juejin.cn/post/7120791098775044103)

[Android组件化开发（四）--进程保活组件的封装](https://juejin.cn/post/7121643256495996936)

[Android组件化开发（五）--完整版音乐播放组件的封装](https://juejin.cn/post/7122869564299280415)


**今天我们再来封装一个<mark>视频播放请求</mark>:**

我们先来讲解我们封装的思路。
> 笔者一直觉得：学习别人开发的思路比学习到当前知识内容更重要。
> 学思路不仅在工作中对自己帮助很大，在生活中用处也很大

## 首先我们需要了解我们的需求：

笔者这篇文章的需求是基于我们组件化业务中的“`朋友`”界面中的`短视频播放`请求。

`一图胜千言`

![device-2022-07-26-154741.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e2a41db1eeb4953adab9c6824015a2d~tplv-k3u1fbpfcp-watermark.image?)

## `需求分析`：

> - 1.切换到朋友界面后，需要显示一个类朋友圈功能的界面，其中核心是内部的一个`播放器`请求。
> - 2.首先在视频加载过程中显示一个视频加载的动画。
> 	- 2.1：加载成功后，视频开始播放，并在左上角显示一个可以切到全屏的icon（图像比`16:9`），此时将音频静音
> 	- 2.2：加载失败，可以根据重试次数进行重试，如果重试次数达到上限还是失败，则直接显示一个播放错误的icon
> - 3.切换到全屏图像后，从小界面播放器播放的位置开始显示视频，并将音频打开，点击返回键可以回到原朋友圈界面。

按上面的需求分析：
**下面我们来做个技术选型：**

## `技术选型`：

- `ljkPlayer`：

`ljkPlayer` 是Bilibili公司维护的一个开源工程，是基于`ffmpeg`开发的一个播放器软件，目前支持`Android和iOS`两种平台

> `优点`：
> 1.基于FFmpeg开发，结构比较清晰，对于二次开发难度比较小 
> 2.支持直播开发
>
> `缺点`：
> 1.基于ffmpeg，可扩展性比较小 
> 2.官方这几年维护力度没那么大了，版本更新较慢
>
- `ExoPlayer`：

`ExoPlayer`是谷歌推出的一个开源播放器，使用原`Android`提供的解码系统来解析视频和音频
将`MediaCodec`封装地非常完善，形成了一个性能优越，播放稳定性较好的一个开发播放器
由于`Google`的大力推广，目前非常流行

> `优点`：1.接入包比较小，1.1M左右  2.维护比较勤，版本更新较快
>
> `缺点`：
> 1.不适合做直播
> 2.视频硬解码，可扩展性一般
> 3.适合播放场景比较简单的场景


- `MediaPlayer`二次开发：
> `优点`：代码可控性较强
> `缺点`：封装难度比较大。

因为我们需求比较简单，笔者在`ExoPlayer`和`MediaPlayer`中考虑，最终还是选择使用`MediaPlayer`做二次开发
因为`ExoPlayer`内部封装的很多东西都是不必要的，也算变相的做了一次`包体积优化`咯。
后期如果有`直播需求`的，可以考虑直接使用`ljkPlayer`进行二次开发。


## `封装`：

根据前面的需求分析和技术选型，下面来做具体的封装工作。

首先你需要对MediaPlayer有一些基础的了解。
如果对`MediaPlayer`还不是很了解的，请转到[官网查看](https://developer.android.google.cn/guide/topics/media/mediaplayer?hl=zh_cn)：


引用官网的话：
> `MediaPlayer` 类是媒体框架最重要的组成部分之一。此类的对象能够获取、解码以及播放音频和视频，而且只需极少量设置。它支持多种不同的媒体源，例如：
>
> -   本地资源
> -   内部 URI，例如您可能从内容解析器那获取的 URI
> -   外部网址（流式传输）

`MediaPlayer`最重要的是：

他内部维护了一套音乐播放`状态机`，**每个步骤都需要根据上一个状态来做改变**，如果在某个状态执行了错误的步骤就会出异常甚至崩溃，你可能不想看到这种事件发生在你的app中吧。。
所以接下来你会在我的代码中看到很多关于状态的判断。

`MediaPlayer`**播放状态切换图**：

![音乐播放状态机.awebp](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cca4804d272d4701908eda69b688d646~tplv-k3u1fbpfcp-watermark.image?)


可以看到内部调用过程还是挺复杂的。好在这些都是MediaPlayer自己内部维护，大部分情况不需要人为去处理。

> **MediaPlayer不仅可以播放音频还可以播放适配，区别是播放视频需要一个`Surface`才可以显示**


笔者根据需求设计了
`类关系图`：

![视频播放.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6967babdbdb64c51b038bd0e56701c1b~tplv-k3u1fbpfcp-watermark.image?)

- `SmartVideoView`：用于小屏播放时的视频界面
- `VideoFullScreenDialog`：用于大屏播放时的界面
- `VideoAdSlot`：是对大小屏的控制类，大小屏切换就是根据这个控制类进行
- `VideoAdContext`：是业务层和`VideoAdSlot`之间通讯的桥梁，起承上启下的作用，这样可以让播放的处理操作处于一个黑盒中，隔离业务层逻辑

来看具体代码：
- 1.`SmartVideoView`

由于类的篇幅太长，下面只列出部分关键代码：
完整代码已经上传[Github](https://github.com/ByteYuhb/anna_music_app) 上。

```java
public class SmartVideoView extends RelativeLayout implements View.OnClickListener,
        MediaPlayer.OnPreparedListener, MediaPlayer.OnErrorListener, MediaPlayer.OnCompletionListener,
        TextureView.SurfaceTextureListener {
    ...
    private static final int STATE_ERROR = -1;
    private static final int STATE_IDLE = 0;
    private static final int STATE_PLAYING = 1;
    private static final int STATE_PAUSING = 2;
    private static final int STATE_COMP = 3;
	
	
    private static final int LOAD_TOTAL_COUNT = 3;

    ...
    private TextureView mVideoView;
    private RelativeLayout mPlayerView;
    ...
  
    /**
     * 屏幕宽高
     */
    private int mScreenWidth;
    private int mVideoHeight;
    ...
	private Surface videoSurface;

    /**给mMediaPlayer设置Surface
     * @param surface
     * @param width
     * @param height
     */
    @Override
    public void onSurfaceTextureAvailable(@NonNull SurfaceTexture surface, int width, int height) {
        videoSurface = new Surface(surface);
        checkMediaPlayer();
        mMediaPlayer.setSurface(videoSurface);
        load();
    }
    public SmartVideoView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        this.context = context;
        initData();
        initView();
		//注册屏幕切换广播
        registerBroadcastReceiver();
    }

    /**
     * 初始化布局
     * 1.设置点击事件
     * 2.SurfaceTexture回调事件
     * 3.初始化小屏状态
     */
    private void initView() {
        mPlayerView = (RelativeLayout) LayoutInflater.from(context).inflate(R.layout.video_player_layout,this);
        mVideoView = findViewById(R.id.xadsdk_player_video_textureView);
        mVideoView.setOnClickListener(this);
        mVideoView.setSurfaceTextureListener(this);
        initSmallLayout();
    }

    /**小模式状态
     * 1.设置各种View的点击事件
     */
    private void initSmallLayout() {
        setPlayViewlayoutParam();
        mMiniPlayBtn = mPlayerView.findViewById(R.id.xadsdk_small_play_btn);
        mFullBtn = mPlayerView.findViewById(R.id.xadsdk_to_full_view);
        mLoadingBar = mPlayerView.findViewById(R.id.loading_bar);
        mMiniPlayBtn.setOnClickListener(this);
        mFullBtn.setOnClickListener(this);
    }

    /**
     * 初始化宽高：16:9
     */
    private void initData() {
        DisplayMetrics dm = new DisplayMetrics();
        WindowManager wm = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
        wm.getDefaultDisplay().getMetrics(dm);
        mScreenWidth = dm.widthPixels;
        mVideoHeight = (int) (mScreenWidth * (9/16.0f));
    }
   ...
    /**数据元加载成功，准备调用start开启播放
     * @param mp
     */
    @Override
    public void onPrepared(MediaPlayer mp) {
        //1.显示playing状态‘
        showPlayView();
        //2.设置当前状态为Playing
        setPlayerState(STATE_PLAYING);
        //3设置静音模式
        mute(true);
        //4.开始调用start启动MediaPlayer
        mMediaPlayer.start();
        mCurrentCount = 0;
    }


    /**播放事件完成后回调：
     * 1.显示完成状态的view
     * 2.设置当前状态太为完成撞他
     * @param mp
     */
    @Override
    public void onCompletion(MediaPlayer mp) {
        if (listener != null) {
            listener.onAdVideoLoadComplete();
        }
        //显示播放完成状态
        playBack();
        //设置当前状态为完成状态
        setPlayerState(STATE_COMP);

        setIsComplete(true);
        setIsRealPause(true);
    }

    /**出现异常
     * @param mp
     * @param what
     * @param extra
     * @return
     */
    @Override
    public boolean onError(MediaPlayer mp, int what, int extra) {
        //1设置当前状态为ERROR
        setPlayerState(STATE_ERROR);
        //2回调error事件给上游
        if (this.listener != null) {
            listener.onAdVideoLoadFailed();
        }
        //3调用stop方法
        stop();
        return false;
    }

    /**各种View的点击事件处理
     * @param v
     */
    @Override
    public void onClick(View v) {
        ...
    }

    /**
     * 加载视频
     */
    public void load(){
        //1.判断状态是否是初始化状态
        if(playerState != STATE_IDLE){
            return;
        }
        //2.显示对应的Vew状态
        showLoadingView();
        //3.判断MediaPlayer是否已经初始化过
        checkMediaPlayer();
        //4.异步加载请求，回调onPrepare方法
        try {
            mMediaPlayer.setDataSource(mUrl);
            mMediaPlayer.prepareAsync();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 暂停按钮
     */
    public void pause(){
        //1检测状态是否是播放状态
       ...
        if(isPlaying()){
            //3调用MediaPlayer的pause方法
            mMediaPlayer.pause();
        }
        //2显示暂停状态
        ...
        //4设置当前状态为pause状态
        ...
    }

    /**
     * 恢复按钮
     */
    public void resume(){
        //1检测当前状态是否是暂停状态
        if(playerState != STATE_PAUSING && this.playerState != STATE_COMP){
            return;
        }
        if(!isPlaying()){
            //3调用MediaPlayer的resume方法
            mMediaPlayer.start();
        }
        //2显示恢复状态
        showPlayView();
        //4设置当前状态为播放状态
        entryResumeState();
    }

    //跳到指定点播放视频
    public void seekAndResume(int position) {
        ...
    }

    //跳到指定点暂停视频
    public void seekAndPause(int position) {
        if (this.playerState != STATE_PLAYING) {
            return;
        }
        showPauseView(false);
        setPlayerState(STATE_PAUSING);
        if (isPlaying()) {
            mMediaPlayer.seekTo(position);
            mMediaPlayer.setOnSeekCompleteListener(new MediaPlayer.OnSeekCompleteListener() {
                @Override
                public void onSeekComplete(MediaPlayer mp) {
                    mMediaPlayer.pause();
                }
            });
        }
    }

    //全屏不显示暂停状态,后续可以整合，不必单独出一个方法
    public void pauseForFullScreen() {
        ...
    }



    /**
     * 停止播放，需要seek到0的位置,一般出异常才会调用这个
     */
    private void stop(){
        //1.将MediaPlayer释放掉
        ..
        //2.设置当前状态为初始化状态
        ..
        //3.判断是否满足重试条件，根据重试次数来
        ..
    }

    /**
     * 释放MediaPlayer状态
     */
    public void destroy(){
        //1.将MediaPlayer释放掉
        ..
        //2.设置当前状态为初始化状态
        ..
        //3 显示pause状态
        ..
    }

    /**
     * 显示加载View
     */
    private void showLoadingView() {
        ...
    }

    /**
     * 显示播放View
     */
    private void showPlayView() {
        ...
    }

    /**显示状态状态
     * @param show 是否显示暂停和播放按钮
     */
    private void showPauseView(boolean show) {
        ...
    }

    //播放完成后回到初始状态
    private void playBack(){
        setPlayerState(STATE_PAUSING);
        if (mMediaPlayer != null) {
            mMediaPlayer.setOnSeekCompleteListener(null);
            mMediaPlayer.seekTo(0);
            mMediaPlayer.pause();
        }
        this.showPauseView(false);
    }

    ..

    /**
     * 设置播放器的宽高
     */
    private void setPlayViewlayoutParam() {
        ...
    }


    
    ...
    private void registerBroadcastReceiver() {
        if (mScreenReceiver == null) {
            mScreenReceiver = new ScreenEventReceiver();
            IntentFilter filter = new IntentFilter();
            filter.addAction(Intent.ACTION_SCREEN_OFF);
            filter.addAction(Intent.ACTION_USER_PRESENT);
            getContext().registerReceiver(mScreenReceiver, filter);
        }
    }
    ...

    /**
     * 监听锁屏事件的广播接收器
     */
    private class ScreenEventReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            //主动锁屏时 pause, 主动解锁屏幕时，resume
            switch (intent.getAction()) {
                case Intent.ACTION_USER_PRESENT:
                    if (playerState == STATE_PAUSING) {
                        if (mIsRealPause) {
                            //手动点的暂停，回来后还暂停
                            pause();
                        } else {
                            resume();
                        }
                    }
                    break;
                case Intent.ACTION_SCREEN_OFF:
                    if (playerState == STATE_PLAYING) {
                        pause();
                    }
                    break;
            }
        }
    }
}
```

下面我来说下该类的`设计思路`？为什么这么设计?

- 1.继承自`RelativeLAyout`，并实现了`MediaPlayer`需要的几个监听方法：`onPrepare，onCompleted，onError`，这些和音频播放都一样，其中最重要的是`onSurfaceTextureAvailable`
监听，这个监听回调是在我们的`SurfaceTexture`初始化好后，回调给应用层的一个操作。


```java
public void onSurfaceTextureAvailable(@NonNull SurfaceTexture surface, int width, int height) {
        videoSurface = new Surface(surface);
        checkMediaPlayer();
        mMediaPlayer.setSurface(videoSurface);
        load();
    }
```
应用接收到`onSurfaceTextureAvailable`回调后可以调用
`MediaPlayer`的`prepareAsync`异步加载操作，并在`onPrepare`回调中，调用`MediaPlayer`的`start`操作启动播放，


```java

/**
     * 加载视频
     */
    public void load(){
        //1.判断状态是否是初始化状态
        if(playerState != STATE_IDLE){
            return;
        }
        //2.显示对应的Vew状态
        showLoadingView();
        //3.判断MediaPlayer是否已经初始化过
        checkMediaPlayer();
        //4.异步加载请求，回调onPrepare方法
        try {
            mMediaPlayer.setDataSource(mUrl);
            mMediaPlayer.prepareAsync();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
	
 /**数据元加载成功，准备调用start开启播放
     * @param mp
     */
    @Override
    public void onPrepared(MediaPlayer mp) {
        //1.显示playing状态‘
        showPlayView();
        //2.设置当前状态为Playing
        setPlayerState(STATE_PLAYING);
        //3设置静音模式
        mute(true);
        //4.开始调用start启动MediaPlayer
        mMediaPlayer.start();
        mCurrentCount = 0;
    }
```
最后我们的视频数据渲染到对应的`SurfaceTexture`上面。

- 2.这个类的内部设置了很多播放器状态：
包括空闲状态`idle`，播放状态`Playing`，暂停状态`Pausing`，停止状态`stoped`和错误`error`状态等

```java
private static final int STATE_ERROR = -1;
private static final int STATE_IDLE = 0;
private static final int STATE_PLAYING = 1;
private static final int STATE_PAUSING = 2;
private static final int STATE_COMP = 3;
```

> 和音频类似，每个阶段的操作都是根据`MediaPlayer`这个`状态机`来处理的，每个状态下只能执行特定的任务，需要对每个状态进行判断，
> **这也是自己封装MediaPlayer做二次开发的难点所在。**

- 3.这里我们将界面宽高比设置为`16:9`

```java
/**
     * 初始化宽高：16:9
     */
    private void initData() {
        DisplayMetrics dm = new DisplayMetrics();
        WindowManager wm = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
        wm.getDefaultDisplay().getMetrics(dm);
        mScreenWidth = dm.widthPixels;
        mVideoHeight = (int) (mScreenWidth * (9/16.0f));
    }

```

- 4.注册了一个`锁屏的广播`:可以在视频处于锁屏状态下执行对应的操作


```java
/**
     * 监听锁屏事件的广播接收器
     */
    private class ScreenEventReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            //主动锁屏时 pause, 主动解锁屏幕时，resume
            switch (intent.getAction()) {
                case Intent.ACTION_USER_PRESENT:
                    if (playerState == STATE_PAUSING) {
                        if (mIsRealPause) {
                            //手动点的暂停，回来后还暂停
                            pause();
                        } else {
                            resume();
                        }
                    }
                    break;
                case Intent.ACTION_SCREEN_OFF:
                    if (playerState == STATE_PLAYING) {
                        pause();
                    }
                    break;
            }
        }
    }
```


基本上就是按上面逻辑来开发设计的，还有一些是在不同状态显示不同UI这里就不介绍了

下面来介绍下`VideoFullScreenDialog`

> 这个类是用来显示一个全屏播放页面的，其实也是按`16:9`的比例显示，并非正真的全屏

代码：

```java
public class VideoFullScreenDialog extends Dialog implements SmartVideoView.ADVideoPlayerListener {
	private SmartVideoView mVideoView;
	 public VideoFullScreenDialog(Context context, SmartVideoView mraidView, String instance,
                                 int position) {
        super(context, R.style.dialog_full_screen);
        mPosition = position;
        mVideoView = mraidView;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getWindow().setFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS,
                WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        requestWindowFeature(Window.FEATURE_NO_TITLE);
        setContentView(R.layout.dialog_video_layout);
        initVideoView();
    }
	
	private void initVideoView() {
        mParentView = (RelativeLayout) findViewById(R.id.content_layout);
        mRootView = findViewById(R.id.root_view);
        mRootView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                onClickVideo();
            }
        });
        mRootView.setVisibility(View.INVISIBLE);

        mVideoView.setListener(this);
        mVideoView.mute(false);
        mParentView.addView(mVideoView);
        mParentView.getViewTreeObserver()
                .addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
                    @Override
                    public boolean onPreDraw() {
                        mParentView.getViewTreeObserver().removeOnPreDrawListener(this);
                        prepareScene();
                        runEnterAnimation();
                        return true;
                    }
                });
    }
	
	//准备动画所需数据
    private void prepareScene() {
        mEndBundle = Utils.getViewProperty(mVideoView);
        /**
         * 将desationview移到originalview位置处
         */
        deltaY = (mStartBundle.getInt(Utils.PROPNAME_SCREENLOCATION_TOP) - mEndBundle.getInt(
                Utils.PROPNAME_SCREENLOCATION_TOP));
        mVideoView.setTranslationY(deltaY);
    }

    //准备入场动画
    private void runEnterAnimation() {
        mVideoView.animate()
                .setDuration(200)
                .setInterpolator(new LinearInterpolator())
                .translationY(0)
                .withStartAction(new Runnable() {
                    @Override
                    public void run() {
                        mRootView.setVisibility(View.VISIBLE);
                    }
                })
                .start();
    }

    //准备出场动画
    private void runExitAnimator() {
        mVideoView.animate()
                .setDuration(200)
                .setInterpolator(new LinearInterpolator())
                .translationY(deltaY)
                .withEndAction(new Runnable() {
                    @Override
                    public void run() {
                        dismiss();
                        if (mListener != null) {
                            mListener.getCurrentPlayPosition(mVideoView.getCurrentPosition());
                        }
                    }
                })
                .start();
    }
}

```

可以看到这个类就比较简单了。

`关键点`：
- 1.我们的构造方法中传入了一个`SmartVideoView`，**为什么要传入一个SmartVideoView呢**？新建一个不可以么？
这样做的目的其实是为了我们的`VideoView复用`做准备，大家想想，我们切换到大屏的时候，小屏的VideoView其实是在后台的，为啥不直接拿来复用呢，还省去了很多初始化的操作。



- 2.这里对`VideoView`做了一个入场和出场动画。防止动画切换不协调

> 所以别看这个类很简单，背面的思想还是很需要一番考量的。


好了继续下一个 `VideoAdSlot`

这个类设计来由：
前面我们对大小屏切换说过我们的SmartVideoView是可以拿来复用给的，那复用的逻辑就是这个类来控制的。

```java
public class VideoAdSlot implements SmartVideoView.ADVideoPlayerListener {
    private Context mContext;
    /**
     * UI
     */
    private SmartVideoView mVideoView;
    private ViewGroup mParentView;
    /**
     * Data
     */
    private String mXAdInstance;
    private SDKSlotListener mSlotListener;

    public VideoAdSlot(String adInstance, SDKSlotListener slotLitener) {

        mXAdInstance = adInstance;
        mSlotListener = slotLitener;
        mParentView = slotLitener.getAdParent();
        mContext = mParentView.getContext();
        initVideoView();
    }

    public void destroy() {
        mVideoView.destroy();
        mVideoView = null;
        mContext = null;
        mXAdInstance = null;
    }


    private void initVideoView() {
        mVideoView = new SmartVideoView(mContext);
        if (mXAdInstance != null) {
            mVideoView.setDataSource(mXAdInstance);
            mVideoView.setListener(this);
        }
        RelativeLayout paddingView = new RelativeLayout(mContext);
        paddingView.setBackgroundColor(mContext.getResources().getColor(android.R.color.black));
        paddingView.setLayoutParams(mVideoView.getLayoutParams());
        mParentView.addView(paddingView);
        mParentView.addView(mVideoView);
    }


    /**
     * 实现play层接口
     */
    @Override
    public void onClickFullScreenBtn() {
        //获取videoview在当前界面的属性
        Bundle bundle = Utils.getViewProperty(mParentView);
        mParentView.removeView(mVideoView);
        VideoFullScreenDialog dialog =
                new VideoFullScreenDialog(mContext, mVideoView, mXAdInstance, mVideoView.getCurrentPosition());
        dialog.setListener(new VideoFullScreenDialog.FullToSmallListener() {
            @Override
            public void getCurrentPlayPosition(int position) {
                backToSmallMode(position);
            }

            @Override
            public void playComplete() {
                bigPlayComplete();
            }
        });
        dialog.setViewBundle(bundle); //为Dialog设置播放器数据Bundle对象
        dialog.setSlotListener(mSlotListener);
        dialog.show();
    }

    private void backToSmallMode(int position) {
        if (mVideoView.getParent() == null) {
            mParentView.addView(mVideoView);
        }
        mVideoView.setTranslationY(0); //防止动画导致偏离父容器
        mVideoView.isShowFullBtn(true);
        mVideoView.mute(true);
        mVideoView.setListener(this);
        mVideoView.seekAndResume(position);
    }

    private void bigPlayComplete() {
        if (mVideoView.getParent() == null) {
            mParentView.addView(mVideoView);
        }
        mVideoView.setTranslationY(0); //防止动画导致偏离父容器
        mVideoView.isShowFullBtn(true);
        mVideoView.seekAndPause(0);
    }
	
	@Override
    public void onAdVideoLoadSuccess() {
        if (mSlotListener != null) {
            mSlotListener.onVideoLoadSuccess();
        }
    }

    @Override
    public void onAdVideoLoadFailed() {
        if (mSlotListener != null) {
            mSlotListener.onVideoFailed();
        }
    }

    @Override
    public void onAdVideoLoadComplete() {
        if (mSlotListener != null) {
            mSlotListener.onVideoComplete();
        }
        mVideoView.setIsRealPause(true);
    }
	...
}
```
`关键点：`
- 1.`ViewGroup mParentView`;

    这个是外部传入进来的作为`SmartVedioView`的`容器`，每次启动一个`SmartVedioView`会提供一个`容器`来包裹，
 由外部传入，**可以让我们的容器可扩展性更强。**

- 2.内部控制大小屏切换，切换过程中，需要将`VedioView`移除原`mParentView`，并传入到大屏中去。

我们用一张图来总结下大小屏切换的`View复用机制`：


![SmartVedioView复用.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/45072a7eddc4423faced939eff92779e~tplv-k3u1fbpfcp-watermark.image?)



最后来看下`VideoAdContext`

```java
public class VideoAdContext implements VideoAdSlot.SDKSlotListener {

    //the ad container
    private ViewGroup mParentView;

    private VideoAdSlot mAdSlot;
    private String mInstance;
    //the listener to the app layer
    private VideoContextInterface mListener;

    public VideoAdContext(ViewGroup parentView, String instance) {
        this.mParentView = parentView;
        this.mInstance = instance;
        load();
    }

    /**
     * init the ad,不调用则不会创建videoview
     */
    public void load() {
        if (mInstance != null) {
            mAdSlot = new VideoAdSlot(mInstance, this);
        } else {
            mAdSlot = new VideoAdSlot(null, this); //创建空的slot,不响应任何事件
            if (mListener != null) {
                mListener.onVideoFailed();
            }
        }
    }
	...
	
}

```

可以看到这个类在构造方法中创建了一个`VideoAdSlot`，其他都不用处理，一切逻辑控制都交由`VideoAdSlot`自身处理。

几个类都介绍完毕了：
我来说说上面代码的优缺点吧？你没听错，我在怼我自己。。

`优点`：
- 1.对ViedeoView的复用
- 2.对大小屏切换操作使用一个类来控制，而非直接跳转，让大小屏做自己的事情就好了，增加可扩展性和隔离性。
- 3.使用第三方VideoAdSlot类来承上启下，做到接口隔离。


`缺点`：

`SmartViedeoView`中的代码还是太集中了，`View`中做了很多`MediaPlayer`做的事情，可以将`MediaPlayer`的逻辑剥离开来，对外部`SmartViedeoView提供访问接口`
，让`SmartViedeoView`去调用这部分逻辑，也许代码看起来会更好，这个后面有时间会重新设计下。

## `总结`：
本文主要是基于`MediaPlayer`二次封装了一个大小屏切换的播放器功能。
背后逻辑不复杂，但是内部有些`设计思路`需要有一定功底才能想到，
> 学习过程中：希望你能`学人之鱼，也能学人之渔`

到今天我们组件化实战项目已经开发了差不多一半多了，后面还会对`分享组件`以及`应用更新组件`等进行封装，以及对`base库的封装，arout库的使用，让组件之间进行解耦`。
持续输出中。。

项目地址：https://github.com/ByteYuhb/anna_music_app
    
![组件化开发.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eee1398548e146ed86b49fa42ec2bbbe~tplv-k3u1fbpfcp-watermark.image?)
    