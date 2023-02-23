
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
前面几篇系列文章我们讲解了`组件化开发`中几个常用功能组件的开发，包括：`网络请求组件`，`图片加载请求组件`，`应用保活组件`。今天我们来封装一个`音乐播放组件`。

**组件化相关系列文章：**

 [Android组件化开发（一）--Maven私服的搭建](https://juejin.cn/post/7118646272323485709)

[ Android组件化开发（二）--网络请求组件封装](https://juejin.cn/post/7119281692350611493)

[Android组件化开发（三）--图片加载组件封装](https://juejin.cn/post/7120791098775044103)

[Android组件化开发（四）--进程保活组件的封装](https://juejin.cn/post/7121643256495996936)


![组件化开发.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/049e14f881e942aabcc4201c2062d195~tplv-k3u1fbpfcp-watermark.image?)

还记得开始写这个系列时候说过，我们这个组件化系列会以一个`实战`项目展开，那这个项目就是一个`音乐播放器`，暂且叫他`xx云音乐`吧



在写这篇文章之前，我在网上查找了一些类似文章，发现很多要么是直接贴代码，要么就是讲解了一些思路，并没有给出太多具体的代码，看了之后很容易就忘了。今天笔者使用架构思维和具体代码结合的方式带大家来看下笔者是如何去封装一个音乐播放组件的。

那我们就开始吧。

看过我之前文章都知道，笔者很注重`需求的分析`。

> 一切代码的开发都原自需求，做好需求分析直观重要。

## 需求分析：
往大了讲：
- 1.封装一个音乐播放组件：

往细了讲：

    - 1.基础功能：播放，暂停，下一首，上一首
    - 2.播放模式：单曲循环，顺序播放，随机播放
    - 3.用户登录，收藏等
    - 4.再扩展一点，开发一个类似朋友圈的功能。。。走远了哈

有了上面的需求分析，我们就有了一个切入点。
## 技术选型
要实现上面需求：

> 笔者可以想到的是使用一个`MediaPlayer`来实现我们的需求

如果对`MediaPlayer`还不是很了解的，请转到官网查看：
https://developer.android.google.cn/guide/topics/media/mediaplayer?hl=zh_cn

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

好了，有了前面的知识铺垫我们就**正式开始封装啦**。

## 封装

这里先放一张笔者设计的一个UML类图：
![音乐播放组件.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b0f68827018844eb9c76776285c2e329~tplv-k3u1fbpfcp-watermark.image?)

- `CustomMediaPlayer`:

这个类继承`MediaPlayer`，内部主要维护了`MediaPlayer`的各个阶段状态，`MediaPlayer`内部也维护了一个状态值，但是使用起来不够方便和优雅，还是自己掌控比较方便好用。

- `AudioPlayer`：

这个类主要作用是对外提供一些基础功能：如对传入的音乐进行同步或者异步加载，对音乐进行暂停，恢复等操作。内部维护了一个`CustomMediaPlayer`对象，所以操作都是使用这个`CustomMediaPlayer`来实现的.这个类还会对`MediaPlayer`的状态对外发布，让外部做出对应的处理。
- `AudioController`：

一个`单例类`，这个类很关键，是整个封装库的核心，做到承上启下的作用，
承上：管理音乐播放歌曲列表`Queue`和播放模式`PlayModel`
启下：内部维护一个`AudioPlayer`的对象，使用`AudioPlayer`提供的方法，做到对音乐列表的播放，暂停，恢复等操作也对外发布一些事件，如播放模式的改变等，需要外部UI做出一些反应。
- `MusicPlayerService`：

这是一个播放音乐用的`Service`，内部维护了一个`AudioProvider.stub`的`binder`通讯使用的类：对外提供通讯接口：如设置播放队列，暂停以及恢复等操作。这个挂接了一个播放器的`Notification`，可以让服务变为前台服务，进一步增加应用的保活能力。
- `NotificationHelper`:

这个是一个`Notification`的辅助类，用于在通知栏显示我们的音乐播放事件。
- `GreenDaoHelper`:

这个类用来处理对`收藏`的歌曲精选持久化
- `AudioBean`:

这个类是贯彻整个播放体系的`歌曲信息封装类`。

> 整个体系主要就是由这几个类撑起来的，每个类从下往上都有不同职责，相辅相成。

**经过上面的分析，我们大概了解了本次封装的一个体系架构：**

> 下面对具体代码来讲解：

笔者在代码中已经给出了很多非常详细的注解，方便大家查看：

- **CustomMediaPlayer.java**

```java
/**
 * 这个类主要是提供MediaPlayer的各种状态，内部回调
 */
public class CustomMediaPlayer extends MediaPlayer implements MediaPlayer.OnCompletionListener,
        MediaPlayer.OnPreparedListener, MediaPlayer.OnErrorListener {
    //使用枚举类表示当前播放器的各种状态
    public enum Status{
        IDLE, INITIALIZED, PREPARED,STARTED, PAUSED, STOPPED, COMPLETED
    }

    //当前播放器状态
    private Status mStatus;

    public CustomMediaPlayer(){
        super();
        //默认状态 IDLE
        mStatus = Status.IDLE;
        //设置播放完毕后的回调
        setOnCompletionListener(this);
        //设置数据准备完毕，准备播放的回调
        setOnPreparedListener(this);
        //设置发生错误事件的回调
        setOnErrorListener(this);
    }

    //设置播放完毕后的回调
    @Override
    public void onCompletion(MediaPlayer mp) {
        mStatus = Status.COMPLETED;
        listener.omCompleted();
    }

    //播放器重置：状态设置为IDLE
    @Override
    public void reset() {
        super.reset();
        mStatus = Status.IDLE;
    }

    //设置数据元的回调：状态设置为：INITIALIZED
    @Override
    public void setDataSource(String path) throws IOException, IllegalArgumentException, IllegalStateException, SecurityException {
        super.setDataSource(path);
        mStatus = Status.INITIALIZED;
    }

    //数据准备完毕后，开始播放前的状态：PREPARED
    @Override
    public void onPrepared(MediaPlayer mp) {
        mStatus = Status.PREPARED;
        listener.onPrepared();
    }

    //发送错误事件的回调，返回true表示消费掉事件，如果为false则还会回调onComplete接口回调
    @Override
    public boolean onError(MediaPlayer mp, int what, int extra) {
        listener.onError();
        return true;
    }

    //开始播放状态
    @Override
    public void start() throws IllegalStateException {
        super.start();
        mStatus = Status.STARTED;
    }

    //停止播放状态
    @Override
    public void stop() throws IllegalStateException {
        super.stop();
        mStatus = Status.STOPPED;
    }

    //暂停状态
    @Override
    public void pause() throws IllegalStateException {
        super.pause();
        mStatus = Status.PAUSED;
    }

    public void setmStatus(Status mStatus) {
        this.mStatus = mStatus;
    }

    public Status getmStatus() {
        return mStatus;
    }

    private MediaPlayerListener listener;

    public void setListener(MediaPlayerListener listener) {
        this.listener = listener;
    }

    /**
     * 这个Listener用途AudioPlayer的事件回调
     */
    public interface MediaPlayerListener{
        void onPrepared();
        void omCompleted();
        void onError();
    }
}
```

看代码中，我们可以发现，这个类只是给MediaPlayer设置了一些供回调的接口，且维护了当前MediaPlayer的播放状态。


- **AudioPlayer.java**

```
/**
 *播放器事件源
 */
public class AudioPlayer implements CustomMediaPlayer.MediaPlayerListener {
    private CustomMediaPlayer mMediaPlayer;
    private WifiManager.WifiLock mWifiLock;
    private static final String TAG = "AudioPlayer";
    private static final int TIME_MSG = 0x01;
    private static final int TIME_INVAL = 100;
    //是否是因为系统引起的焦点问题
    private boolean isPausedByFocusLossTransient;
    //音频焦点监听器
    private AudioFocusManager mAudioFocusManager;

    public AudioPlayer() {
        isPausedByFocusLossTransient = false;
        mMediaPlayer = new CustomMediaPlayer();
        init();
    }

    /**
     * 初始化mMediaPlayer的事件，如：使用唤醒锁，设置音频格式，设置wifilock，设置mAudioFocusManager
     *
     */
    private void init() {
        //使用唤醒锁
        mMediaPlayer.setWakeMode(AudioHelper.getmContext(), PowerManager.PARTIAL_WAKE_LOCK);
        //设置音频格式
        mMediaPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC);
        //设置数据准备完毕回调
        mMediaPlayer.setListener(this);
        //设置wifilock，可以让在后台也可以运行wifi下载加载数据
        mWifiLock = ((WifiManager) AudioHelper.getmContext()
                .getApplicationContext()
                .getSystemService(Context.WIFI_SERVICE)).createWifiLock(WifiManager.WIFI_MODE_FULL, "AudioPlayer");
        //设置Audio管理回调器，在一些场景下会回调，如中途遇到打电话，其他应用占用音频端口
        mAudioFocusManager = new AudioFocusManager(AudioHelper.getmContext(),audioFocusListener);
    }

    //播放进度更新handler
    private Handler mHandler = new Handler(Looper.getMainLooper()) {
        @Override public void handleMessage(Message msg) {
            switch (msg.what) {
                case TIME_MSG:
                    //暂停也要更新进度，防止UI不同步，只不过进度一直一样
                    if (getStatus() == CustomMediaPlayer.Status.STARTED
                            || getStatus() == CustomMediaPlayer.Status.PAUSED) {
                        //UI类型处理事件
                        EventBus.getDefault()
                                .post(new AudioProgressEvent(getStatus(), getCurrentPosition(), getDuration()));
                        sendEmptyMessageDelayed(TIME_MSG, TIME_INVAL);
                    }
                    break;
            }
        }
    };

    AudioFocusManager.AudioFocusListener audioFocusListener = new AudioFocusManager.AudioFocusListener() {
        // 重新获得焦点
        @Override
        public void audioFocusGrant() {
            setVolumn(1.0f,1.0f);
            if(isPausedByFocusLossTransient){
                resume();
            }
            isPausedByFocusLossTransient = false;
        }

        // 永久丢失焦点，如被其他播放器抢占
        @Override
        public void audioFocusLoss() {
            if(mMediaPlayer != null) {
                pause();
            }
            isPausedByFocusLossTransient = true;
        }
        // 短暂丢失焦点，如来电
        @Override
        public void audioFocusLossTransient() {
            if (mMediaPlayer != null) pause();
            isPausedByFocusLossTransient = true;
        }
        // 瞬间丢失焦点，如通知
        @Override
        public void audioFocusLossDuck() {
            //瞬间失去焦点,
            setVolumn(0.5f, 0.5f);
        }
    };

    /**
     * 开始启动播放音乐,由异步onPrepared回调调用
     */
    private void start(){
        mMediaPlayer.setmStatus(CustomMediaPlayer.Status.PREPARED);
        mMediaPlayer.start();
        // 启用wifi锁
        mWifiLock.acquire();
        //更新进度
        mHandler.sendEmptyMessage(TIME_MSG);
        //发送start事件，UI类型处理事件
        EventBus.getDefault().post(new AudioStartEvent());
    }

    /**
     * 对外提供的加载音频的方法
     */
    public void load(AudioBean audioBean) {
        try {
            mMediaPlayer.reset();
            mMediaPlayer.setDataSource(audioBean.mUrl);
            mMediaPlayer.prepareAsync();
            //发送加载音频事件，UI类型处理事件
            EventBus.getDefault().post(new AudioLoadEvent(audioBean));
        } catch (IOException e) {
            e.printStackTrace();
            EventBus.getDefault().post(new AudioErrorEvent());
        }
    }
    /**
     * 对外提供的播放方法
     */
    public void resume() {
        if(getStatus() == CustomMediaPlayer.Status.PAUSED){
            start();
        }
    }
    /**
     * 对外暴露pause方法
     */
    public void pause() {
        if (getStatus() == CustomMediaPlayer.Status.STARTED) {
            mMediaPlayer.pause();
            //关注wifi锁
            if(mWifiLock.isHeld()){
                mWifiLock.release();
            }
            // 取消音频焦点
            if (mAudioFocusManager != null) {
                mAudioFocusManager.abandonAudioFocus();
            }
            //发送暂停事件,UI类型事件
            EventBus.getDefault().post(new AudioPauseEvent());
        }
    }
    /**
     * 销毁唯一mediaplayer实例,只有在退出app时使用
     */
    public void release() {
        if(mMediaPlayer == null){
            return;
        }
        mMediaPlayer.release();
        mMediaPlayer = null;
        // 关闭wifi锁
        if (mWifiLock.isHeld()) {
            mWifiLock.release();
        }
        mWifiLock = null;
        // 取消音频焦点
        if (mAudioFocusManager != null) {
            mAudioFocusManager.abandonAudioFocus();
        }
        mAudioFocusManager = null;
        //发送销毁播放器事件,清除通知等
        EventBus.getDefault().post(new AudioReleaseEvent());
    }

    /**获取当前状态
     * @return
     */
    public CustomMediaPlayer.Status getStatus(){
        return mMediaPlayer.getmStatus();
    }

    /**
     * 数据元准备完毕的回调，可以直接启动start开始播放
     */
    @Override
    public void onPrepared() {
        start();
    }

    /**
     * 播放完成事件
     */
    @Override
    public void omCompleted() {
        //发送播放完成事件,逻辑类型事件
        EventBus.getDefault().post(new AudioCompleteEvent());
    }

    @Override
    public void onError() {
        //发送当次播放实败事件,逻辑类型事件
        EventBus.getDefault().post(new AudioErrorEvent());
    }

    /**
     * 获取当前音乐总时长,更新进度用
     */
    public int getDuration() {
        if (getStatus() == CustomMediaPlayer.Status.STARTED
                || getStatus() == CustomMediaPlayer.Status.PAUSED) {
            return mMediaPlayer.getDuration();
        }
        return 0;
    }

    /**呼气当前的播放进度
     * @return
     */
    public int getCurrentPosition() {
        if (getStatus() == CustomMediaPlayer.Status.STARTED
                || getStatus() == CustomMediaPlayer.Status.PAUSED) {
            return mMediaPlayer.getCurrentPosition();
        }
        return 0;
    }

    /**设置音量，
     * @param left 左声道 0-1.0f
     * @param right 右声道 0-1.0f
     */
    public void setVolumn(float left, float right) {
        if (mMediaPlayer != null) mMediaPlayer.setVolume(left, right);
    }
}
```
这个类内部做的事情就比较多了：

- 1.监听`MediaPlayer`当前状态
     如果是`prepare`状态：则启动内部的`start`方法，这个方法我将其设置为了`private`，说明是一个内部方法。
      监听当前播放位置，并回调给业务层。

- 2.对外提供：`load，pause，resume，getCurrentPosition`获取当前播放位置等接口。

这些方法会对我们当前播放器状态做判断：
如调用pause，一定要当前状态是start状态
调用resume，一定要当前状态是pause等。
这些状态都是MediaPlayer统一管理的，不能乱设置。

- 3.使用`EventBus`对外发送一些内部状态事件：
包括：`AudioLoadEvent AudioErrorEvent AudioPauseEvent AudioCompleted`等事件
外部可以在需要的地方注册这些`Event`

- 4.监听播放器`焦点事件`：如有电话来，通知或者其他获取焦点的事件，
  这些事件如何处理？
    1.重新获得焦点：则判断是不是系统行为的焦点事件：如果是则调用resume恢复播放
    2.永久失去焦点，如被其他播放器抢占了：则调用暂停pause事件
    3.短暂失去焦点：如有电话来了：也是暂停
    4.瞬间失去焦点：如通知：则将音量设置小一点即可

播放器焦点监听使用的是：`AudioManager.OnAudioFocusChangeListener`

- 5.使用`唤醒锁`和`Wifi_lock`:
唤醒锁可以让设备在息屏状态也能继续执行播放事件
Wifi_lock可以让设备在后台运行下载数据，
记得添加对应的权限：
```
<uses-permission android:name="android.permission.WAKE_LOCK"/>
```


> 可以看到我们的AudioPlayer内部执行逻辑还是比较多的，先慢慢消化下吧。

- **AudioController.java**
```java
/**
 * 控制播放逻辑类，注意添加一些控制方法时，要考虑是否需要增加Event,来更新UI
 */
public class AudioController {

    /**
     * 播放方式
     */
    public enum PlayMode {
        /**
         * 列表循环
         */
        LOOP,
        /**
         * 随机
         */
        RANDOM,
        /**
         * 单曲循环
         */
        REPEAT
    }
    private AudioPlayer mAudioPlayer;
    //播放队列,不能为空,不设置主动抛错
    private ArrayList<AudioBean> mQueue = new ArrayList<>();
    private int mQueueIndex = 0;
    private PlayMode mPlayMode = PlayMode.LOOP;

    private AudioController() {
        EventBus.getDefault().register(this);
        mAudioPlayer = new AudioPlayer();
    }

    /**设置歌曲列表
     * @param listBean
     */
    public void setmQueue(List<AudioBean> listBean) {
        setmQueue(listBean,0);
    }
    public void setmQueue(List<AudioBean> listBean,int index){
        if(mQueue == null) mQueue = new ArrayList<>();
        mQueue.addAll(listBean);
        mQueueIndex = index;
    }

    public ArrayList<AudioBean> getmQueue() {
        return mQueue;
    }

    public PlayMode getmPlayMode() {
        return mPlayMode;
    }

    public void setmPlayMode(PlayMode mPlayMode) {
        this.mPlayMode = mPlayMode;
        EventBus.getDefault().post(new AudioPlayModeEvent(mPlayMode));
    }



    /**
     * 队列头添加播放哥曲
     */
    public void addAudio(AudioBean bean) {
        this.addAudio(bean,0);
    }
    public void addAudio(AudioBean bean,int index){
        if (mQueue == null){
            mQueue = new ArrayList<>();
        }
        int queryId = queryBean(bean);
        if(queryId<=-1){
            //没添加过此id的歌曲，添加且直播番放
            addCustomBean(bean,index);
            setPlayIndex(index);
        }else{
            AudioBean currentBean = getNowPlaying();
            if(!currentBean.id.equals(queryId)){
                setPlayIndex(queryId);
            }
        }
    }

    /**
     * 对外提供获取当前播放时间
     */
    public int getNowPlayTime() {
        return mAudioPlayer.getCurrentPosition();
    }

    /**
     * 对外提供获取总播放时间
     */
    public int getTotalPlayTime() {
        return mAudioPlayer.getCurrentPosition();
    }

    /**
     * 对外提供的获取当前歌曲信息
     */
    public AudioBean getNowPlaying() {
        return getPlaying(mQueueIndex);
    }

    /**获取下一首AudioBean
     * @return
     */
    private AudioBean getNextPlaying() {
        switch (mPlayMode) {
            case LOOP:
                mQueueIndex = (mQueueIndex + 1) % mQueue.size();
                return getPlaying(mQueueIndex);
            case RANDOM:
                mQueueIndex = new Random().nextInt(mQueue.size()) % mQueue.size();
                return getPlaying(mQueueIndex);
            case REPEAT:
                return getPlaying(mQueueIndex);
        }
        return null;
    }

    /**获取前一个AudioBean
     * @return
     */
    private AudioBean getPreviousPlaying() {
        switch (mPlayMode) {
            case LOOP:
                mQueueIndex = (mQueueIndex + mQueue.size() - 1) % mQueue.size();
                return getPlaying(mQueueIndex);
            case RANDOM:
                mQueueIndex = new Random().nextInt(mQueue.size()) % mQueue.size();
                return getPlaying(mQueueIndex);
            case REPEAT:
                return getPlaying(mQueueIndex);
        }
        return null;
    }

    /**获取当前播放的AudioBean
     * @param index
     * @return
     */
    private AudioBean getPlaying(int index) {
        if (mQueue != null && !mQueue.isEmpty() && index >= 0 && index < mQueue.size()) {
            return mQueue.get(index);
        } else {
            throw new AudioQueueEmptyException("当前播放队列为空,请先设置播放队列.");
        }
    }

    /**设置当前播放列表
     * @param index
     */
    public void setPlayIndex(int index) {
        if (mQueue == null) {
            throw new AudioQueueEmptyException("当前播放队列为空,请先设置播放队列.");
        }
        mQueueIndex  = index;
        play();
    }

    /**
     * 加载当前index歌曲
     */
    public void play() {
        AudioBean bean = getPlaying(mQueueIndex);
        load(bean);
    }

    /**
     * 加载next index歌曲
     */
    public void next() {
        AudioBean bean = getNextPlaying();
        load(bean);
    }

    /**
     * 加载previous index歌曲
     */
    public void previous() {
        AudioBean bean = getPreviousPlaying();
        load(bean);
    }
    private void load(AudioBean bean) {
        mAudioPlayer.load(bean);
    }

    /**
     * 添加/移除到收藏
     */
    public void changeFavourite() {
        if (null != GreenDaoHelper.selectFavourite(getNowPlaying())) {
            //已收藏，移除
            GreenDaoHelper.removeFavourite(getNowPlaying());
            EventBus.getDefault().post(new AudioFavouriteEvent(false));
        } else {
            //未收藏，添加收藏
            GreenDaoHelper.addFavourite(getNowPlaying());
            EventBus.getDefault().post(new AudioFavouriteEvent(true));
        }
    }
    private void addCustomBean(AudioBean bean, int index) {
        if (mQueue == null) {
            throw new AudioQueueEmptyException("当前播放队列为空,请先设置播放队列.");
        }
        mQueue.add(index,bean);
    }

    /**查找bean在队列中的位置
     *  -1:表示每找到，找到返回索引位置
     * @return
     */
    private int queryBean(AudioBean bean) {
        if(mQueue==null){
            throw new AudioQueueEmptyException("当前队列为空");
        }
        return mQueue.indexOf(bean);
    }

    public void resume() {
        mAudioPlayer.resume();
    }

    public void pause() {
        mAudioPlayer.pause();
    }
    /**
     * 播放/暂停切换
     */
    public void playOrPause() {
        if (isStartState()) {
            pause();
        } else if (isPauseState()) {
            resume();
        }
    }
    public void release() {
        mAudioPlayer.release();
        EventBus.getDefault().unregister(this);
    }
    /*
     * 获取播放器当前状态
     */
    private CustomMediaPlayer.Status getStatus() {
        return mAudioPlayer.getStatus();
    }
    /**
     * 对外提供是否播放中状态
     */
    public boolean isStartState() {
        return CustomMediaPlayer.Status.STARTED == getStatus();
    }

    /**
     * 对外提提供是否暂停状态
     */
    public boolean isPauseState() {
        return CustomMediaPlayer.Status.PAUSED == getStatus();
    }

    //插放完毕事件处理
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onAudioCompleteEvent(
            AudioCompleteEvent event) {
        next();
    }

    //播放出错事件处理
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onAudioErrorEvent(AudioErrorEvent event) {
        next();
    }


    public static AudioController getInstance() {
        return AudioController.SingletonHolder.instance;
    }
    private static class SingletonHolder {
        private static AudioController instance = new AudioController();
    }
}
```
**这个类是整个体系的核心类：起承上启下作用**

1.内部维护了一个List的歌曲列表，一个当前索引index以及当前播放模式：
播放模式：
loop：顺序播放
random：随机播放
repeat：单曲循环

2.提供了很多外部方法：
play，pause，resume，next，prev等这些方法实际是将数据元传递给AudioPlayer执行，AudioPlayer传递给MediaPlayer执行。

3.接收MediaPlayer对外输出的事件，并执行对应的方法。对外输出一些Event：如播放模式的改变等

**这个方法是核心类，但是好像做的事情并没有AudioPlayer多呢。。**

- **MusicPlayerService.java**

```java
public class MusicPlayerService extends Service{
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return new AudioProvider.Stub() {

            @Override
            public void setQueue(List<AudioBean> audioList) throws RemoteException {
                AudioController.getInstance().setmQueue(audioList);
                //初始化前台Notification
                NotificationHelper.getInstance().init(new NotificationHelper.NotificationHelperListener() {
                    @Override
                    public void onNotificationInit() {
                        //让服务处于前台，提高优先级，增加保活能力
                        startForeground(NotificationHelper.NOTIFICATION_ID,NotificationHelper.getInstance().getNotification());
                    }
                });
            }

            @Override
            public void addAudio(AudioBean bean) throws RemoteException {
                AudioController.getInstance().addAudio(bean);
            }

            @Override
            public void play() throws RemoteException {
                AudioController.getInstance().play();

            }

            @Override
            public void playNext() throws RemoteException {
                AudioController.getInstance().next();
            }

            @Override
            public void playPre() throws RemoteException {
                AudioController.getInstance().previous();
            }

            @Override
            public void pause() throws RemoteException {
                AudioController.getInstance().pause();
            }

            @Override
            public void resume() throws RemoteException {
                AudioController.getInstance().resume();
            }

            @Override
            public void release() throws RemoteException {
                AudioController.getInstance().release();
            }
        };
    }

    @Override
    public void onCreate() {
        EventBus.getDefault().register(this);
        registerBroadcastReceiver();
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        EventBus.getDefault().unregister(this);
        AudioController.getInstance().release();
        unRegisterBroadcastReceiver();
    }

    private void registerBroadcastReceiver() {
        if (mReceiver == null) {
            mReceiver = new NotificationReceiver();
            IntentFilter filter = new IntentFilter();
            filter.addAction(NotificationReceiver.ACTION_STATUS_BAR);
            registerReceiver(mReceiver, filter);
        }
    }

    private NotificationReceiver mReceiver;
    private void unRegisterBroadcastReceiver() {
        if (mReceiver != null) {
            unregisterReceiver(mReceiver);
        }
    }
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onAudioLoadEvent(AudioLoadEvent event) {
        //更新notifacation为load状态
        NotificationHelper.getInstance().showLoadStatus(event.mAudioBean);
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onAudioPauseEvent(AudioPauseEvent event) {
        //更新notifacation为暂停状态
        NotificationHelper.getInstance().showPauseStatus();
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onAudioStartEvent(AudioStartEvent event) {
        //更新notifacation为播放状态
        NotificationHelper.getInstance().showPlayStatus();
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onAudioFavouriteEvent(AudioFavouriteEvent event) {
        //更新notifacation收藏状态
        NotificationHelper.getInstance().changeFavouriteStatus(event.isFavourite);
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onAudioReleaseEvent(AudioReleaseEvent event) {
        //移除notifacation
    }

    /**
     * 接收Notification发送的广播
     */
    public static class NotificationReceiver extends BroadcastReceiver {
        public static final String ACTION_STATUS_BAR =
                AudioHelper.getmContext().getPackageName() + ".NOTIFICATION_ACTIONS";
        public static final String EXTRA = "extra";
        public static final String EXTRA_PLAY = "play_pause";
        public static final String EXTRA_NEXT = "play_next";
        public static final String EXTRA_PRE = "play_previous";
        public static final String EXTRA_FAV = "play_favourite";

        @Override
        public void onReceive(Context context, Intent intent) {
            if (intent == null || TextUtils.isEmpty(intent.getAction())) {
                return;
            }
            String extra = intent.getStringExtra(EXTRA);
            switch (extra) {
                case EXTRA_PLAY:
                    //处理播放暂停事件,可以封到AudioController中
                    AudioController.getInstance().playOrPause();
                    break;
                case EXTRA_PRE:
                    AudioController.getInstance().previous(); //不管当前状态，直接播放
                    break;
                case EXTRA_NEXT:
                    AudioController.getInstance().next();
                    break;
                case EXTRA_FAV:
                    AudioController.getInstance().changeFavourite();
                    break;
            }
        }
    }
}
```

我们看到这个类：

1.内部实现了一个`AudioProvider.Stub`类，这个类实现了很多外部调用常用方法;这么做的设计原则是为了把我们封装的代码方便第三方应用调用，就不必重复开发了。

2.注册了一个`NotificationReceiver`的广播类，这个类是用于我们的音乐服务和通知栏通讯。

3.注册了一些`EventBus`事件，这些事件可以用来监听当前`MediaPlayer`的状态，并对通知栏中的`UI`进行修改。


还有其他几个类我这边就不继续讲了：

**总结下我们这个版本的`事件流程图`：**

![事件流程图.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38bbed5ca6ef4c0fb02523919dcd8313~tplv-k3u1fbpfcp-watermark.image?)


**我这边顺便封装了几个自定义View和一个播放器通知栏界面，效果如下：**

- 一个MusicBottomDialog的组件
```
/**
 * 底部弹出歌曲列表：
 * 1.功能：
 *  循环模式：
 *  收藏功能
 *  删除列表功能
 *  歌曲列表
    左边是一个圆形的可旋转专辑图片，使用的是之前封装的图片加载库加载的
 */
public class MusicBottomDialog extends BottomSheetDialog {
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9bda114f88464252b514a1179651ae28~tplv-k3u1fbpfcp-watermark.image?)




- 这是通知栏的音乐播放组件：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8443b6bbbfa4312bfdada23cfb854ec~tplv-k3u1fbpfcp-watermark.image?)


- 最后这儿是音乐播放单曲界面，可以左右滑动听其他歌曲


![device-2022-07-22-003546.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/82e188c0f7f1468ca1034611a3bb12ff~tplv-k3u1fbpfcp-watermark.image?)



**得益于前面我们封装的音乐播放组件，我们可以很快的开发出一个音乐播放界面，可以把重心放在界面的设计和渲染上面。**

**完整代码可以查看github上的Demo：**
https://github.com/ByteYuhb/anna_music_app

## 总结
**我们今天要封装的音乐播放组件在这个项目中的比重很大，算是该系列中的核心部分。希望大家可以从中获取到自己想要的东西。**

到今天我们组件化实战项目已经开发了差不多一半了，后面还会对`分享组件`以及`视频播放组件`进行封装，以及对`base库的封装，arout库的使用，让组件之间进行解耦`。
持续输出中。。


![组件化开发.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eee1398548e146ed86b49fa42ec2bbbe~tplv-k3u1fbpfcp-watermark.image?)
    