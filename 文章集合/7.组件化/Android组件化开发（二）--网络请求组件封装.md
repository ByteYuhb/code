> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [[小余的自习室](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)] ，在成功的路上不迷路！**
### 前言

前面一篇文章我们讲解了`maven私服`的搭建，maven私服在`组件化框架`中有一个很重要的地位就是可以将我们的`lib`库放到局域网中，供公司其他开发者使用，实现类库的分享。

下面是这个系列准备实现的一个`组件化实战项目框架`：


![组件化开发.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6d92402648a440cbacfc3bbca66c3e3~tplv-k3u1fbpfcp-watermark.image?)

-   笔者打算`从下往上`依次来实现我们项目中的组件，**毕竟地基稳固了，房子才可以搭的很结实**。

`注意`：这里不会对封装代码进行长篇大论，主要还是以`思路点拨`的方式进行，如果需要看完整代码的可以移步到github。

[GitHub - ByteYuhb/anna_music_app](https://github.com/ByteYuhb/anna_music_app)

-   这篇文章实现的是`一个lib_network`库：

实现一个组件的封装前，我们有几个步骤，这几个步骤不可或缺，如果上来就直接码代码，最后会让你从`激情到放弃`的

-   **1.先分析需求**

    -   每个类的封装都是有了新的需求一步一步实现扩大的，不可能一蹴而就，包括笔者今天讲解的`lib_network`类库，后期也会根据用户需求一步一步壮大。

        既然是要实现一个网络请求的封装库：主要包括`get`,`post`，`form表单`请求，`文件上传和下载`等一些基础网络功能的实现

-   2.根据需求进行**技术选型**

    -   技术选型在类库封装中也是一个重要步骤，这里我们只是实现一些网络基础功能，笔者打算在`HttpUrlConnection`，`Volly`和`OkHttp`中选择我们我们的封装基础库

        

![1.技术选型.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e77292039344945ab8438b8008339c3~tplv-k3u1fbpfcp-watermark.image?)

**几个待选型的技术对比：**

- `HttpUrlConnection`

这个类是一个比较底层的类库了，如果使用这个类库来封装，我们需要实现很多轮子工作，而在另外两个开源框架`Volly`和`OkHttp`已经实现了这些工作，没必要重复造轮子，在这里不考虑。

如果您对库有更深层次的要求且对自己技术比较自信，可以考虑使用这个类库去封装实现，毕竟开源库也是在这些基础库上实现的。

- `Volly`

这个类库是在`HttpUrlConnection` 基础上做的一层封装，为了减少用户使用HttpUrlConnection的复杂度。

 一开始出来是google主推的网络请求框架，google希望统一Android网路请求库推出的一个框架。那为什么后面用的人越来越少了呢。那就是因为下面我们要说的`OkHttp`框架。

- `OkHttp`

看过源码的都知道，`OkHttp`也是在`HttpUrlConnecttion`上做的封装，其继承了`Volly`的优势，且在`Volly`上构建了自己的有优点，包括：`连接池复用`，`重试重定向机制`，`拦截器模式`等、

具体关于`OkHttp`的介绍可以参考我的另外一篇文章：

[Android体系课 之 OkHttp你想知道的都在这里了--](https://juejin.cn/post/7105013511784235016)

**笔者最后选择使用`OkHttp`来做我们基础库的封装工作**

<!---->

- **3.封装思路**

    前面通过对`具体需求分析`，并且也做了`技术选型`

**接下来就是怎么去实现这个封装？需要封装哪些类？**

`我们来回忆下使用OkHttp的方式：`

```
fun testOkHttp(){
    val client = OkHttpClient()
    val r1:RequestBody = formBodyBuilder.build()
    val request = Request.Builder().get().url("http://host:port/api").build()
    val response = client.newCall(request).execute()
    print(response.body()?.bytes())
}
```

笔者思路：**首先封装的是和用户`直接打交道`的类**：

`Request`，`Response`和`OkHttpClient`以及异步情况下的`Callback`是直接和用户打交道的地方，那我们就从这几个类下手依次对其进行封装：

`我们先列出来我们技术方案：`


![2.封装思路.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf2bef643ff741339a8f73db5a550754~tplv-k3u1fbpfcp-watermark.image?)

有了上面几个分析步骤:接下来我们就来实现具体的封装流程：

-   1.`Request`请求的封装 **Request**：用户请求类，在OkHttp中封装了我们的业务请求信息

这里我们创建一个**CommonRequest**类来再次封装我们的Request

笔者只列出了部分类的框架代码：完整代码在github上

```java
public class CommonRequest {
    /**创建一个Post的请求，不包括headers
     * @return
     */
    public static Request createPostRequest(String url, RequestParams params){
        return createPostRequest(url,params,null);
    }
    /**创建一个Post的请求，包括headers
     * @param url
     * @param params body的参数集合
     * @param headers header的参数集合
     * @return
     */
    public static Request createPostRequest(String url, RequestParams params, RequestParams headers){
        FormBody.Builder mFormBodyBuilder = new FormBody.Builder();
        if(params!=null){
            for(Map.Entry<String,String> entry:params.urlParams.entrySet()){
                mFormBodyBuilder.add(entry.getKey(),entry.getValue());
            }
        }
        Headers.Builder mHeadersBuilder = new Headers.Builder();
        if(headers!=null){
            for(Map.Entry<String,String> entry:headers.urlParams.entrySet()){
                mHeadersBuilder.add(entry.getKey(),entry.getValue());
            }
        }
        return new Request.Builder()
                .url(url)
                .headers(mHeadersBuilder.build())
                .post(mFormBodyBuilder.build())
                .build();
    }

    /**创建一个不包含header的get请求
     * @return
     */
    public static Request createGetRequest(String url,RequestParams params){
        return createGetRequest(url,params,null);
    }
    /**创建一个Get的请求，包括headers
     * @return
     */
    public static Request createGetRequest(String url,RequestParams params,RequestParams headers){
        StringBuilder stringBuilder = new StringBuilder(url).append("?");
        if(params != null){
            for(Map.Entry<String,String> entry:params.urlParams.entrySet()){
                stringBuilder.append(entry.getKey()).append("=").append(entry.getValue());
            }
        }
        Headers.Builder mHeadersBuilder = new Headers.Builder();
        if(headers!=null){
            for(Map.Entry<String,String> entry:headers.urlParams.entrySet()){
                mHeadersBuilder.add(entry.getKey(),entry.getValue());
            }
        }
        return new Request.Builder()
                .url(stringBuilder.toString())
                .headers(mHeadersBuilder.build())
                .get()
                .build();
    }
    private static final MediaType FILE_TYPE = MediaType.parse("application/octet-stream");

    /**文件上传请求
     * @return
     */
    public static Request createMultiPostRequest(String url,RequestParams params){
        MultipartBody.Builder requestBuilder = new MultipartBody.Builder();
        requestBuilder.setType(MultipartBody.FORM);
        if(params != null){
            for (Map.Entry<String, Object> entry : params.fileParams.entrySet()) {
                if (entry.getValue() instanceof File) {
                    requestBuilder.addPart(Headers.of("Content-Disposition","form-data; name="" + entry.getKey() + """),
                            RequestBody.create(FILE_TYPE, (File) entry.getValue()));
                }else if (entry.getValue() instanceof String) {
                    requestBuilder.addPart(Headers.of("Content-Disposition", "form-data; name="" + entry.getKey() + """),
                            RequestBody.create(null, (String) entry.getValue()));
                }
            }
        }
        return new Request.Builder().url(url).post(requestBuilder.build()).build();
    }

    /**文件下载请求
     * @param url
     * @return
     */
    public static Request createFileDownLoadRequest(String url,RequestParams params){
        return createGetRequest(url,params);
    }
    /**文件下载请求
     * @param url
     * @return
     */
    public static Request createFileDownLoadRequest(String url){
        return createGetRequest(url,null);
    }
}
```

可以看到我们在这个类里面创建了几个方法：

-   1.`Get请求`
-   2.`Post请求`
-   3.`文件上传`
-   4.`文件下载`

这几个请求，已经可以基本满足我们实战项目的要求了

-   2.  `response`响应的封装

由于response请求是在CallBack中返回的，思路就是，自定义业务层需要的CallBack，尽量让代码轻量化

这里创建了两个`CallBack`类：

- 1.`CommonFileResponse`


这个类主要是由来对`文件类型`的请求回调进行封装:

**CommonFileResponse**封装思路：

- 1.对失败相应直接通过业务层传递下来的Listener回调给业务层失败结果

- 2.对成功的相应，我们先将输入流中的数据写入到文件中，并在主线程中回调文件下载进度给业务层。业务层可以在获取文件结果的同时，也可以获取文件下载进度。

```java
/**
 * 专门处理文件的回调
 */
public class CommonFileCallBack implements Callback {
    /**
     * the java layer exception, do not same to the logic error
     */
    protected final int NETWORK_ERROR = -1; // the network relative error
    protected final int IO_ERROR = -2; // the JSON relative error
    protected final String EMPTY_MSG = "";
    /**
     * 将其它线程的数据转发到UI线程
     */
    private static final int PROGRESS_MESSAGE = 0x01;
    private Handler mDeliveryHandler;
    private DisposeDownloadListener mListener;
    private String mFilePath;
    private int mProgress;
    public CommonFileCallBack(DisposeDataHandle handle){
        this.mListener = (DisposeDownloadListener) handle.mListener;
        this.mFilePath = handle.mSource;
        this.mDeliveryHandler = new Handler(Looper.getMainLooper()){
            @Override
            public void handleMessage(@NonNull Message msg) {
                switch (msg.what) {
                    case PROGRESS_MESSAGE:
                        mListener.onProgress((int) msg.obj);
                        break;
                }
            }
        };
    }
    @Override
    public void onFailure(Call call, IOException e) {
        mDeliveryHandler.post(new Runnable() {
            @Override
            public void run() {
                mListener.onFailure(new OkHttpException(NETWORK_ERROR, e));
            }
        });
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        final File file = handleResponse(response);
        mDeliveryHandler.post(new Runnable() {
            @Override
            public void run() {
                if (file != null) {
                    mListener.onSuccess(file);
                } else {
                    mListener.onFailure(new OkHttpException(IO_ERROR, EMPTY_MSG));
                }
            }
        });
    }

    private File handleResponse(Response response) {
        ...
            while ((length = inputStream.read(buffer)) != -1) {
                fos.write(buffer, 0, length);
                currentLength += length;
                mProgress = (int) (currentLength / sumLength * 100);
                mDeliveryHandler.obtainMessage(PROGRESS_MESSAGE, mProgress).sendToTarget();
            }
            fos.flush();
        } catch (Exception e) {
            file = null;
        } finally {
            ...
        }
        return file;
    }

}
```

- 2.`CommonJsonResponse`


这个类用处和我们的`Retrofit`类似，将请求转换为我们需要的类，通过`Gson`或者`fastJson`等框架处理：

```java
/**
 * @author anna
 * @function 专门处理JSON的回调
 */
public class CommonJsonCallback implements Callback {

    /**
     * the logic layer exception, may alter in different app
     */
    protected final String RESULT_CODE = "ecode"; // 有返回则对于http请求来说是成功的，但还有可能是业务逻辑上的错误
    protected final int RESULT_CODE_VALUE = 0;
    protected final String ERROR_MSG = "emsg";
    protected final String EMPTY_MSG = "";

    /**
     * the java layer exception, do not same to the logic error
     */
    protected final int NETWORK_ERROR = -1; // the network relative error
    protected final int JSON_ERROR = -2; // the JSON relative error
    protected final int OTHER_ERROR = -3; // the unknow error

    /**
     * 将其它线程的数据转发到UI线程
     */
    private Handler mDeliveryHandler;
    private DisposeDataListener mListener;
    private Class<?> mClass;

    public CommonJsonCallback(DisposeDataHandle handle) {
        this.mListener = handle.mListener;
        this.mClass = handle.mClass;
        this.mDeliveryHandler = new Handler(Looper.getMainLooper());
    }

    @Override
    public void onFailure(final Call call, final IOException ioexception) {
        /**
         * 此时还在非UI线程，因此要转发
         */
        mDeliveryHandler.post(new Runnable() {
            @Override
            public void run() {
                mListener.onFailure(new OkHttpException(NETWORK_ERROR, ioexception));
            }
        });
    }

    @Override
    public void onResponse(final Call call, final Response response) throws IOException {
        final String result = response.body().string();
        mDeliveryHandler.post(new Runnable() {
            @Override
            public void run() {
                handleResponse(result);
            }
        });
    }

    private void handleResponse(Object responseObj) {
        if (responseObj == null || responseObj.toString().trim().equals("")) {
            mListener.onFailure(new OkHttpException(NETWORK_ERROR, EMPTY_MSG));
            return;
        }

        try {
            /**
             * 协议确定后看这里如何修改
             */
            JSONObject result = new JSONObject(responseObj.toString());
            if (mClass == null) {
                mListener.onSuccess(result);
            } else {
                Object obj = new Gson().fromJson(responseObj.toString(), mClass);
                if (obj != null) {
                    mListener.onSuccess(obj);
                } else {
                    mListener.onFailure(new OkHttpException(JSON_ERROR, EMPTY_MSG));
                }
            }
        } catch (Exception e) {
            mListener.onFailure(new OkHttpException(OTHER_ERROR, e.getMessage()));
            e.printStackTrace();
        }
    }
}
```


- 3.  `OkHttpClient`请求的封装

`OkHttpClient`是我们`OkHttp`的核心枢纽，业务层以及核心框架层都需要使用到这个类，我们来思考下怎么去封装这个类：

1.我们使用装饰器模式：在`CommonOkHttpClient`中封装一个`OkHttpClient`对象，通过`CommonOkHttpClient`去代理这个`OkHttpClient`对象

来看下我们封装的代码:

```
public class CommonOkHttpClient {
    private static final int TIME_OUT = 30;
    private static OkHttpClient mOkHttpClient;
    static {
        OkHttpClient.Builder mOkHttpClientBuilder = new OkHttpClient.Builder();
        mOkHttpClientBuilder.hostnameVerifier(new HostnameVerifier() {
            @Override
            public boolean verify(String hostname, SSLSession session) {
                return true;
            }
        });
        //添加自定义的拦截器
        mOkHttpClientBuilder.addInterceptor(new Interceptor() {
            @Override
            public Response intercept(Chain chain) throws IOException {
                Request request = chain.request()
                        .newBuilder()
                        .addHeader("User-Agent","anna-movie")
                        .build();
                return chain.proceed(request);
            }
        });
        mOkHttpClientBuilder.cookieJar(new SimpleCookieJar());
        mOkHttpClientBuilder.connectTimeout(TIME_OUT, TimeUnit.SECONDS);
        mOkHttpClientBuilder.readTimeout(TIME_OUT, TimeUnit.SECONDS);
        mOkHttpClientBuilder.writeTimeout(TIME_OUT, TimeUnit.SECONDS);
        //设置是否支持重定向
        mOkHttpClientBuilder.followRedirects(true);
        //设置代理
       // mOkHttpClientBuilder.proxy()
        mOkHttpClientBuilder.sslSocketFactory(HttpsUtils.initSSLSocketFactory(),
                HttpsUtils.initTrustManager());
        mOkHttpClient = mOkHttpClientBuilder.build();
    }
    public static OkHttpClient getOkHttpClient() {
        return mOkHttpClient;
    }

    /**
     * 通过构造好的Request,Callback去发送请求
     */
    public static Call get(Request request, DisposeDataHandle handle) {
        Call call = mOkHttpClient.newCall(request);
        call.enqueue(new CommonJsonCallback(handle));
        return call;
    }
    public static Call post(Request request, DisposeDataHandle handle) {
        Call call = mOkHttpClient.newCall(request);
        call.enqueue(new CommonJsonCallback(handle));
        return call;
    }

    public static Call downloadFile(Request request, DisposeDataHandle handle) {
        Call call = mOkHttpClient.newCall(request);
        call.enqueue(new CommonFileCallBack(handle));
        return call;
    }
}
```

这里面封装了一个`get`，`post`，以及`文件下载`的请求:

对于业务层只需要调用：

```
CommonOkHttpClient.get(...)
CommonOkHttpClient.post(...)
CommonOkHttpClient.downloadFile(..)
```

### 总结：

本篇文章主要以封装思路的方式进行讲解，具体代码可以移步到github

[GitHub - ByteYuhb/anna_music_app](https://github.com/ByteYuhb/anna_music_app)

对于大部分类库的封装都可以使用我们上面的思路，再结合`maven私服`的使用。可以很好的将我们代码作为一个组件共享给开发同事使用

**组件化道路长远,这里我们只是封装了一个网络请求库，后面会不定期对其他类库进行封装，最后整合成一个完整的组件化框架**

![组件化开发.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a06bdcf1a9b44e9863eafc1c63d11da~tplv-k3u1fbpfcp-watermark.image?)