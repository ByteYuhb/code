
## 前言

-   网络请求在我们开发中起的很大比重，有一个好的网络框架可以节省我们的开发工作量，也可以避免一些在开发中不该出现的bug

-   Retrofit是一个轻量级框架，基于OkHttp的一个Restful框架

    

![微信图片_20220527220425.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28dc09d824494cca9e666326f99bfbd5~tplv-k3u1fbpfcp-watermark.image?)
    **今天我们来看下Retrofit的具体使用方式**

 [Android体系课学习 之 网络请求库OkHttp使用方式(附Demo)]()

 [Android体系课学习 之 网络请求库OkHttp看这一篇就够了]()

 [Android体系课学习 之 网络请求库Retrofit使用方式(附Demo)](https://juejin.cn/post/7103518717471883277)

 [Android体系课学习 之 网络请求库Retrofit源码分析](https://juejin.cn/post/7104121838749843492)

 [Android体系课学习 之 RxJava看这篇就够了]()

 [Android体系课学习 之 RxJava操作符详解]()

 [Android体系课学习 之 RxJava源码分析]()


## 1.目录


![Retrofit 2.0目录介绍.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6861e6525da4526ba024fa07cb40aae~tplv-k3u1fbpfcp-watermark.image?)

## 2.简介

-   **功能**

    -   使用接口方法注解的方式配置网络请求参数

-   **优点**

    -   功能强大：支持同步和异步，支持自定义数据解析类型，常用如Json
    -   支持多平台，如Android，Java8等
    -   使用简单：通过注释的方式动态配置交易请求参数
    -   使用多种设计模式，对模块之间解耦，拓展性好
    -   目前最新版本支持协程的方式进行网络请求，不需要再指定使用同步还是异步方式

-   **使用场景**

    > 支持所有的网络请求场景，特别是请求次数比较频繁的场景，因为内部是基于OkHttp，而OkHttp内部使用了各种拦截器，且对请求有做缓存IO复用操作，防止短时间重复连接操作，浪费资源。

> **需要注意：Retrofit请求的本质是通过OkHttp发送网路请求，而Retrofit只是负责网络前期接口的封装**


![retrofit网络流程.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c75d73d3f8514d6c9a7b648fd66af772~tplv-k3u1fbpfcp-watermark.image?)

# 3.使用教程

Retrofit使用步骤如下： 

-   步骤1：添加Retrofit库依赖
-   步骤2：根据服务器响应格式创建响应类
-   步骤3：创建网络请求接口并添加注解
-   步骤4：创建Retrofit实例
-   步骤5：创建网络请求实例
-   步骤6：调用网络请求方法
-   步骤7：处理网络请求返回

#### 步骤1：添加Retrofit库依赖

在模块的build.gradle中添加

```
   implementation 'com.squareup.retrofit2:retrofit:2.9.0' 
   implementation 'com.squareup.retrofit2:converter-gson:2.0.2'
```

在manfest中添加网络权限

`<uses-permission android:name="android.permission.INTERNET"/>`

#### 步骤2： 根据服务器响应格式创建响应类、

假如服务器给了以下请求响应格式：对应了Translation这个响应类，

一个使用中括号括起来的数据格式代表一个新的类：如格式中的*message，baesInfo*等由于是另起一个中括号，所以需要创建一个内部类

```
{
    "status" ：1，
    “message”：{    
            “baesInfo”：{
                    “from”：“eng”
                    “to” ： “CN”
                    “translate_result”：“hello”
            }
            “word_name”：“你好”
    }
}
```

```
/**
 * 创建解析类
 */
public class Translation {
    private int status;
    private message message;
    private static class message {
        private baesInfo baesInfo;
        private String word_name;
    }
    private static class baesInfo {
        private String from;
        private String to;
        private String translate_result;
    }
}
```

#### 步骤3： 创建网络请求接口并添加注解

Retrofit将网络请求抽象成一个接口，并使用注解的方式配置网络请求参数和方法

> 1.用*动态代理*的方式将请求解析为一个Http请求，并执行这个请求
>
> 2.接口中每个方法的参数都需要使用注释的方式完成。

```
/**  

* 创建用于描述网络请求的接口  
  */  
  public interface RequestInterface{  
   @GET("/dictionary/word/query/v2?client=1&timestamp=1637143127&isChangeTo=0&sign=63243e0fb77fa81e&uuid=d6ceb7789ff04ff69355cea0a3325e40&sv=android10&v=11.1.7&uid=&translateType=-1&key=1000001&identity=12&word=%25E4%25BD%25A0%25E5%25A5%25BD%25E5%25B0%258F%25E5%25A7%2590&signature=7cdc8cee1566bde7c34a817fbab31a20")  
   Call<Translation> getCall();  
   // 注解里传入 网络请求 的部分URL地址  
   // Retrofit把网络请求的URL分成了两部分：一部分放在Retrofit对象里，另一部分放在网络请求接口里  
   // 如果接口里的url是一个完整的网址，那么放在Retrofit对象里的URL可以忽略  
   // getCall()是接受网络请求数据的方法  
  }
```

##### 注解详细说明：

##### 第一类：网络请求方法注释

| 类型     | 名称                                                | 解释                                                         |
| -------- | --------------------------------------------------- | ------------------------------------------------------------ |
| 请求方法 | @GET，@POST ，@PUT，@DELETE，@PATH，@HEAD，@OPTIONS | 所有请求都对应一个网络请求方法，接收一个URL地址，如果设置了baseUrl，则完整地址= baseUrl+这里设置的url |
|          | @HTTP                                               | 替换以上七个注解                                             |

**这里对URL地址组成做个详细说明：**

-   网络请求完整url = 创建Retrofit的时候设置的baseUrl+网络请求接口注解设置的路径path

> 如果path是一个完整路径，则无需再配置baseUrl,否则会报错

**来讲下HTTP注释**

-   HTTP注释主要是用来取代之前请求方式GET，POST等的操作，扩展性更强

-   如下面这个调用方式：

    `下面的例子使用HTTP取代POST方式，并在path中传递一个参数，来动态配置url`

```
public interface RequestInterface {  
    /**  
     * method:请求方式  
     * path：请求路径 {user}：表示需要从参数中传入  
     * hasBody：请求体是否存在  
     * @param user  
     * @return  
     */  
    @HTTP(method = "POST",path = "select/{user}",hasBody = false)  
    Call<Translation> getCall(@Path("user") int user);  

}
```

##### 第二类：网络请求标记注释

标记类注释主要包括三个：

-   *@FormUrlEncoded*：表示请求体是一个form表单

    > **每个键值对以Field的形式传入**

    `使用方式如下：`

    ```
        /**
        * FormUrlEncoded:表示该请求体是一个表单格式(Content-Type:application/x-www-form-urlencoded)
        * @Field ("userName") String name：表示以userName为key，以name为值的表单数据
        * */
        @HTTP(method = "POST",path = "select/{user}",hasBody = false)
        @FormUrlEncoded
        Call<Translation> showFULAnnotation(@Field ("userName") String name,@Field ("pwd") String pwd,@Path("user") int user);
    具体调用：  
        RequestInterface rq = retrofit.create(RequestInterface.class);
        Call<Translation> call1 = rq.showFULAnnotation("name","123456",123);
    ```

-   *@Multipart*：表示请求实体是一个form，适用于有文件上传的场景

    > 每个键值对以Part形式传入

    `使用方式如下：`

    ```
    /**
         * Multipart:表示该请求体是一个表单格式(Content-Type:application/x-www-form-urlencoded)，且支持文件上传
         * @Part ("userName") String name：表示以userName为key，以name为值的表单数据
         * @Par MultipartBody.Part file ,表示fileText是一个Part，内部包含了一个key和body
         * Part后面可以是：RequestBody或者MultipartBody.Part
         * */
        @HTTP(method = "POST",path = "select/{user}",hasBody = false)
        @Multipart
        Call<Translation> showMultipart(@Part("userName") RequestBody name,
                                        @Part ("pwd") RequestBody pwd,
                                        @Part MultipartBody.Part fileText ,
                                        @Path("user") int user);

    //调用处：
    RequestInterface rq1 = retrofit.create(RequestInterface.class);
    RequestBody bodyName = RequestBody.create(MediaType.get("text/plain"),"yuhb");
    RequestBody bodyPwd = RequestBody.create(MediaType.get("text/plain"),"123");
    //这里获取文件系统中的数据，并传给MultipartBody.Part
    File file = new File("/aa/bb/file.png");
    RequestBody bodyFile = RequestBody.create(MediaType.get("image/png"),file);
    MultipartBody.Part part = MultipartBody.Part.createFormData("MyFile","testFile.text",bodyFile);
    rq1.showMultipart(bodyName,bodyPwd,part,123);
    ```

-   *@Streaming*：适用于一些返回数据量比较大的场景，如下载大数据等，并且以流的形式返回

##### 第三类：网络请求参数注解

**a）@Header和@Headers**

    定义：设置请求头
    
    **两者区别：**

-   Headers添加的是固定的头而Header是添加不固定的头

-   Headers作用在方法上，而Header作用在参数上

    `使用方式如下：`

    ```
    /**
         *这里使用Header注解参数动态传入Header数据，可以设置多个Header
         */
        @GET("/get")
        Call<Translation> showHeader(@Header("Content") String content);
    
        /**
         *这里使用Headers注解方法设置头数据，只能设置一次
         */
        @Headers("Content:content")
        @GET("/get")
        Call<Translation> showHeaders();
    ```

**b）@Body**

    定义：传入一个对象作为Body传入参数，作为POST的请求body

> 作用类似于FieldMap

  使用方式如下：

```
/**
     *这里使用Body注解修饰了一个Bean对象，
     */
    @POST("/post")
    Call<Translation> showBody(@Body Bean bean);

    //这里调用：
    rq1.showBody(new Bean("yuhb","28","love you"));
```

####

#### 步骤4：创建Retrofit实例

```
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://dict.iciba.com/") // 设置 网络请求 Url
                .addConverterFactory(GsonConverterFactory.create()) //设置使用Gson解析(记得加入依赖            )
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create()) // 支持RxJava平台             
                .build();
```

##### a 关于数据解析器

-   Retrofit支持多种数据解析方式

-   需要在项目build.gradle中添加对应依赖

    常用的是Gson解析器：

    ```
    implementation 'com.squareup.retrofit2:converter-gson:2.0.2' 
    ```

##### b 关于网络适配器

    Retrofit支持多种适配器

> 使用时如使用的是 `Android` 默认的 `CallAdapter`，则不需要添加网络请求适配器的依赖，否则则需要按照需求进行添加\
> Retrofit 提供的 `CallAdapter`

#### 步骤5：创建网络请求实例

```
GetRequest_Interface request = retrofit.create(RequestInterface.class);
Call<Translation> call = request.getCall();
```

#### 步骤6：调用网络请求方法

这里使用了异步的方式，还可以使用同步的方式，同步会阻塞线程，所以切记在主线程上调用同步的方式

```
//异步调用
call.enqueue(new Callback<Translation>() {
            //请求成功时回调
            @Override
            public void onResponse(Call<Translation> call, Response<Translation> response) {
                // 步骤7：处理返回的数据结果
                response.body().show();
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        Toast.makeText(RetrofitTestActivity.this,""+response.body().getResult(),Toast.LENGTH_SHORT).show();
                    }
                });
            }

            //请求失败时回调
            @Override
            public void onFailure(Call<Translation> call, Throwable throwable) {
                System.out.println("连接失败");
            }
        });
//同步调用
call.execute();
```

# 4.项目实战

-   **首先我们使用ApiPost 的mock server功能mock出一个登录api接口**


![mockserver.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ce788167953482288d9a4551e5798d6~tplv-k3u1fbpfcp-watermark.image?)

**接口类型：*POST***

**mock服务器地址：**

<https://console-mock.apipost.cn/app/mock/project/a050bb0d-28d9-48fd-c7db-b2953f7f1caf/api/demo/login>

**响应数据格式：**

```
{
    "code": "0000",
    "data": {
        "verifySuccess": true,
        "userInfo": {
            "username": "admin",
            "email": "t.opfmyej@hkhygiel.lv",
            "address": "@address"
        }
    },
    "desc": "成功"
}
```

-   **步骤1：根据响应数据格式，我们创建响应数据转换类：**

> `每个中括号代表一个类`

```
/**
*响应数据类
* */
public class Result{
    String code;
    Data data;
    class Data{
        boolean verifySuccess;
        UserInfo userInfo;

        @Override
        public String toString() {
            return "Data{" +
                    "verifySuccess=" + verifySuccess +
                    ", userInfo=" + userInfo +
                    '}';
        }

        class UserInfo{
            String username;
            String email;
            String address;

            @Override
            public String toString() {
                return "UserInfo{" +
                        "username='" + username + ''' +
                        ", email='" + email + ''' +
                        ", address='" + address + ''' +
                        '}';
            }
        }

    }

    @Override
    public String toString() {
        return "Result{" +
                "code='" + code + ''' +
                ", data=" + data +
                '}';
    }
}
```

**请求api：**


![请求接口.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3f0c4456a9944bfab687127ba3fc93e~tplv-k3u1fbpfcp-watermark.image?)

**步骤2：根据请求api创建请求接口类**

```
public interface RequestInterface {
    /**
     * FormUrlEncoded:只是请求Body为一个Form表单
     **/
    @FormUrlEncoded
    @POST("api/demo/login")
    Call<Result> login(@FieldMap Map<String, Object> map);

    /**
     * FormUrlEncoded:只是请求Body为一个Form表单
     **/
    @FormUrlEncoded
    @POST("api/demo/login")
    Call<Result> login(@Field("username") String username,
                       @Field("password") String password,
                       @Field("mobile") String mobile,
                       @Field("ver_code") String ver_code);
}
```

`请求方法：POST` 
`请求URL：这里使用baseUrl+url路径(api/demo/login)` 
`请求接口：使用了@Field和@FieldMap两种模式，在参数比较多的情况下，应该使用@FieldMap的注解表示参数输入`

> **调用了FormUrlEncoded表示该请求实体是一个表单。**

**步骤3:创建Retrofit实例** 
`1.设置baseUrl` 
`2.设置GsonConverterFactory数据解析器；`

**记得添加依赖：**

```
implementation com.squareup.retrofit2:converter-gson:2.0.9
```

```
Retrofit retrofit =  new Retrofit.Builder().baseUrl("https://console-mock.apipost.cn/app/mock/project/a050bb0d-28d9-48fd-c7db-b2953f7f1caf/")
                .addConverterFactory(GsonConverterFactory.create())
                .build();
```

**步骤4：获取请求代理类**

```
RequestInterface request = retrofit.create(RequestInterface.class);
Call<Result> call = request.login("admin","123456","18289454846","123456");
```

**步骤5：调用enqueue异步发送请求**

```
call.enqueue(new Callback<Result>() {
            @Override
            public void onResponse(Call<Result> call, Response<Result> response) {
                System.out.println(response.body());
            }

            @Override
            public void onFailure(Call<Result> call, Throwable t) {
                System.out.println(t.getMessage());
            }
        });
```

**步骤6：对响应进行处理** `打印成功或者失败信息`

**运行结果：**

```
Result{code='0000', data=Data{verifySuccess=true, userInfo=UserInfo{username='admin', email='w.fbckyti@rgpvhn.as', address='@address'}}}`
```

**Demo地址**：*<https://github.com/ByteYuhb/DemoRetrofit>*

# 5.总结

-   **静下心来看完本篇文章，相信你一定会有所收获**

-   **如果想了解Retrofit源码部分，请看这篇：[Android体系课学习 之 网络请求库Retrofit源码分析]()**

-   **Android体系课学习 之 开源框架系列**

    [Android体系课学习 之 网络请求库OkHttp使用方式(附Demo)]()

    [Android体系课学习 之 网络请求库OkHttp看这一篇就够了]()

    [Android体系课学习 之 网络请求库Retrofit使用方式(附Demo)](https://juejin.cn/post/7103518717471883277)

    [Android体系课学习 之 网络请求库Retrofit源码分析](https://juejin.cn/post/7104121838749843492)

    [Android体系课学习 之 RxJava看这篇就够了]()

    [Android体系课学习 之 RxJava操作符详解]()

    [Android体系课学习 之 RxJava源码分析]()