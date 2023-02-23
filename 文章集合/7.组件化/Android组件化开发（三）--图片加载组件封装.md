> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [[小余的自习室]](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd) ，在成功的路上不迷路！**
## 前言
前面一篇文章我们做了一个组件化网络请求库：lib_network的封装

相关文章：
[Android组件化开发（二）--网络请求组件封装](https://juejin.cn/post/7119281692350611493)

今天我们来封装一个`图片加载库`：`lib_image_loader`

其实封装思想都是一样的：
主要还是根据三步走：

- **1.需求分析**
- **2.技术选型**
- **3.根据1和2进行类库封装**

## 1.需求分析
- 1.首先他是一个`图片加载库`：

那就要考虑到我们需要加载哪些种类的图片？`png`，`jpg`？`gif`？`webp`？balabala。。
- 2.具体到我们项目中：

常见图片加载`场景`：

       - 1.对ImageView设置src？
       - 2.对View或者ViewGroup设置背景色。？
       - 3.对目标对象设置图片，设置到一个Target中，设置通知栏的图片信息？
       - 4.设置通知栏的图片信息
       - 5.显示圆角图片
- 3.多张图片可以`并发`加载

- 4.图片`自适应`我们的控件

- 5.`api`使用简单，`侵入性低`。

## 2.技术选型
有了第一步的需求分析：我们再来选择对应的技术

如何选择合适的类库进行封装？


> 一般图片加载库，有`Glide`，`Fresco`，`Picasso`，而`Glide`是`Picasso`一种升级版本，可以只看`Glide`就可以，
> 为了选择合适的类库，本人去`Glide和Fresco官网`查看了下他们的优缺点:
>
> Fresco最大的优点是在5.0以下机型，将`Bitmap`的字节数组放在一个特殊的位置，
> 而不占用`堆`中的资源，这样就可以避免OOM的发生，但是目前5.0以下的设备占比已经很少了，所以这个优点可以忽略不计。
>
> 其他方面也基本能满足我们的需求，都支持`gif`和`webp`格式的图片数据，
>
> 但是`Glide`相比较`Fresco`来说，`api`使用简单，对业务代码没有侵入性，而`Fresco`需要使用他的特定控件才能使用，且需要在`application`中做初始化操作，有一定的侵入性

**基于以上几点，笔者考虑使用`Glide`来封装我们的类库。**

如果有想自己开发一套更加原生的图片加载库的同学，可以参考这篇文章：

[面试官：让你设计一套图片加载框架，你会怎么设计？](https://juejin.cn/post/7120459653682561060)

**但是不推荐大家去重复造轮子，没必要，但是你必须了解其内部原理。**

## 封装
有了前两个步骤，我们就可以开始我们的类库封装了。
我们要封装哪些东西呢？还是根据我们的需求来

- 1.对`ImageView`设置src
- 2.对`View`或者`ViewGroup`设置背景色
- 3.对目标对象设置图片，设置到一个`Target`中或者说是直接回调一个`Bitmap`给上层
- 4.设置通知栏的图片信息
- 5.显示圆角图片


> 基于以上需求笔者做了以下封装

#### 1.对ImageView设置src

```java
/**创建一个设置ImageView src的接口
     * @param imageView
     * @param url
     */
    public void displayImageForIView(ImageView imageView, String url, CustomRequestListener listener){
        Glide.with(imageView.getContext())
                .load(url)
                .apply(initCommonOptions())
                .into(imageView);
    }
```


#### 2.对View或者ViewGroup设置背景色

```java
/**创建一个设置View背景的接口
     * @param view
     * @param url
     * @param listener
     */
    public void displayImageForView(final View view, String url,CustomRequestListener listener){
        Glide.with(view.getContext())
                .asBitmap()
                .load(url)
                .listener(listener)
                .apply(initCommonOptions())
                .into(initCustomViewTarget(view));
    }
```


#### 3.对目标对象设置图片，设置到一个Target中或者说是直接回调一个Bitmap给上层，用户希望自己处理Bitmap显示

```java
/**回调一个Bitmap给上层
     */
    public void displayImageForCallBack(Context context, String url, BitmapRequestListener listener){
        Glide.with(context)
                .asBitmap()
                .load(url)
                .into(new JustReadyTarget(listener));

    }
```

#### 4.设置通知栏的图片信息，一般用在音乐播放器等场景


```java
/**创建一个给通知栏RemoteViews设置src的接口
     * @param context
     * @param remoteViews
     * @param id
     * @param notification
     * @param notificationId
     * @param url
     */
    public void displayImageForNotification(Context context, RemoteViews remoteViews, int id,
                                            Notification notification,int notificationId,String url){
        Glide.with(context)
                .asBitmap()
                .load(url)
                .apply(initCommonOptions())
                .into(new NotificationTarget(context,id,remoteViews,notification,notificationId));

    }
```


#### 5.显示圆角图片

```java
/**给ImageVIew设置一个圆形的view
     * @param imageView
     * @param url
     */
    public void displayImageForCircleView(ImageView imageView,String url){
        Glide.with(imageView)
                .asBitmap()
                .load(url)
                .into(new BitmapImageViewTarget(imageView){
                    @Override
                    protected void setResource(Bitmap resource) {
                        RoundedBitmapDrawable drawable = RoundedBitmapDrawableFactory
                                .create(imageView.getResources(),resource);
                        drawable.setCircular(true);
                        imageView.setImageDrawable(drawable);
                    }
                });
    }

```

这里对常见的一些需求做了封装，完整代码可以看`demo`

[GitHub - ByteYuhb/anna_music_app](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FByteYuhb%2Fanna_music_app "https://github.com/ByteYuhb/anna_music_app")
## 总结
此篇文章主要是对组件化框架中功能组件:图片加载框架的封装，


对于大部分类库的封装都可以使用我们上面的思路，再结合`maven私服`的使用。可以很好的将我们代码作为一个组件共享给开发同事使用

**组件化道路长远,到目前为止我们封装两个功能框架：网络请求库和图片加载库，后面会不定期对其他类库进行封装，最后整合成一个完整的组件化框架**



![组件化开发.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/165ceb78a8634810bc5d8adf70f41dcf~tplv-k3u1fbpfcp-watermark.image?)