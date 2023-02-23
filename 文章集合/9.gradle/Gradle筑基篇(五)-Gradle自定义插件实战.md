携手创作，共同成长！这是我参与「掘金日新计划 · 8 月更文挑战」的第4天，[点击查看活动详情](https://juejin.cn/post/7123120819437322247 "https://juejin.cn/post/7123120819437322247") >>
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
前面几篇文章笔者对Gradle的一些基础认知，groovy基础语法，以及Gradle 项目中常用的一些api进行了讲解。

Gradle系列文章如下：
> Gradle筑基篇：
- [Gradle筑基篇(一)-Gradle初探](https://juejin.cn/post/7129780699145437215)

- [Gradle筑基篇(二)-Groovy语法的详解](https://juejin.cn/post/7129336080112812039)

- Gradle筑基篇(三)-Gradle生命周期

- [Gradle筑基篇(四)-Gradle APi详解](https://juejin.cn/post/7130177944189665310/)

- [Gradle筑基篇(五)-Gradle自定义插件实战](https://juejin.cn/post/7130828529167515679/)

- Gradle筑基篇(六)-Gradle Maven仓库管理

> Gradle进阶篇
- Gradle进阶篇(六)-AGP详解


今天笔者再来讲解一些关于`Gradle插件`的使用

## 1.定义
首先来讲下`Gradle`和`Gradle插件`有啥区别？
> `Gradle`是一套构建工具，其内部构建过程主要是以`Project`组成一个树形的生态系统，整个构建流程有自己的生命周期。每个Project又是由若干个Task组成。
>
> `Gradle插件`你可以理解为是运行在`Gradle`这套构建系统上的单个`task`,
> 如**执行脚本的编写**，**字节码插庄**等，都可以依靠`Gradle`插件实现。

> 我们常用的`Android Gradle Plugin`也是一个Gradle插件模块：
> ```js
> 应用插件的ID：‘com.android.application’
> 或者lib库：‘com.android.library’
> ```

## 2.有哪些优势
- **1.逻辑复用**:Gradle插件将一个公共模块单独的抽离出来，然后上传到共享平台，供其他项目使用
- **2.插件配置扩展**：Gradle插件声明插件扩展，将插件内部参数暴露给对应的Project进行配置，大大提高了插件的可扩展性。

## 3.插件的形式
- 1.**build script**：直接在build.gradle构建脚本中创建对应的插件，这种方式只对当前Project有效，不支持对外提供调用，无复用性，一般不推荐使用
- 2.**buildSrc模块**：这种方式是编译器提供的特殊模块，编译器可以自动识别该模块的，对所有的Project可见。但是在项目外不可见，无法给其他工程使用，复用性差。
- 3.**独立插件项目**：替插件创建一个单独的项目，这个项目可以单独的打包成一个jar包，然后发布到企业服务器上供其他项目使用，通常这个插件中包含了一个或多个任务的组合，实现具体的插件功能
	
	

## 4.自定义插件`实战`
下面我会以第三种形式来大家实现一个简单的Gradle插件功能：

**需求如下：**

> 在编译过程中实现：将当前编译的版本信息发布到公司服务器上，可以在本地服务器上实时查看编译的版本日志,通过日志的分析可以对当前编译版本进行优化。


**步骤如下**：
> - 1.初始化插件模块目录结构
> - 2.创建插件实现类
> - 3.创建插件扩展Bean
> - 4.创建插件实现的任务:上传版本信息
> - 5.将插件扩展和插件任务集成到Project生命周期中
> - 6.插件发布
> - 7.插件引入

### 步骤1.初始化插件模块目录结构
首先创建一个`Java or Kotlin Library`的模块，
		
![步骤1创建.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc2d6ccb4f5e4f44af79cc6c666b360c~tplv-k3u1fbpfcp-watermark.image?)
在创建的模块`build.gradle`中引入:

```java
plugins {
    id 'groovy' // Groovy Language
    //id 'org.jetbrains.kotlin.jvm' // Kotlin 
    id 'java-gradle-plugin' // Java Gradle Plugin
}
```

- **groovy**:使用groovy语言开发
- **org.jetbrains.kotlin.jvm**：使用kotlin开发引入kotlin核心插件库
- **java-gradle-plugin**：Gradle插件的一个辅助插件，可以在我们build目录下自动生成资源属性

设置`sourceSets`：

```java
sourceSets {
    main {
        groovy {
            srcDir 'src/main/groovy'
        }
        resources {
            srcDir 'src/main/resources'
        }
    }
}
```
工程目录结构如下：

![插件目录结构.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ba1a6c051596475696a627ebb5f24c73~tplv-k3u1fbpfcp-watermark.image?)
### 步骤2.创建插件实现类

```java
class UploadVersionPlugin implements Plugin<Project>{
    @Override
    void apply(Project project) {
            println "begin:now this is a ${project.name} 's upload plugin"
    }
}
```

### 步骤3.创建插件扩展Bean

```java
class VersionInfo {
    //版本名称
    String versionName
    //版本代码
    int versionCode
    //版本更新信息
    String versionUpdateInfo
}
```
### 步骤4.创建插件实现的任务:上传版本信息

```java
class UploadTask extends DefaultTask{
    String url = 'http://127.0.0.1/api/v3/upload/version'
    @TaskAction
    void upload(){
        //1.获取版本信息
        def version = getCurrentVersion()
        //2.发送版本信息
        def response = sendAndReceive(version)
        //3.处理响应：将版本信息以及响应写入到本地文件中
//        checkResponse(response)

    }
    //1.获取版本信息
    def getCurrentVersion(){
        def name = project.extensions.versionInfo.versionName
        def code = project.extensions.versionInfo.versionCode
        def info = project.extensions.versionInfo.versionUpdateInfo
        println "name:$name code:$code info:$info"
        return new VersionInfo(versionName: name,
                     versionCode: code,
                     versionUpdateInfo: info)
    }
    //2.发送版本信息
    void sendAndReceive(VersionInfo version){
        OkHttpClient client = new OkHttpClient()
        FormBody body = new FormBody.Builder()
                .add('versionName',version.versionName)
                .add('versionCode',""+version.versionCode)
                .add('versionUpdateInfo',version.versionUpdateInfo)
                .build()
        Request.Builder builder = new Request.Builder()
                .url(url)
                .post(body)

        def call1 = client.newCall(builder.build())
        call1.enqueue(new Callback() {
            @Override
            void onFailure(@NotNull Call call, @NotNull IOException e) {
                println "push version fail:reason:"+e.message
            }

            @Override
            void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {
                checkResponse(response);
            }
        })
    }
    //3.处理响应：将版本信息以及响应写入到本地文件中
    void  checkResponse(response){
        println "response:"+new String(response.body().bytes())
        
    }
}
```

**记住，在需要执行的方法上面添加TaskAction注解：在我们任务执行的时候就会执行到这个方法。**
### 步骤5.将插件扩展和插件任务集成到Project生命周期中

```java
@Override
void apply(Project project) {
    println "begin:now this is a ${project.name} 's upload plugin"
    //1.在插件中引入extensions中的字段，就是我们Project中配置的扩展字段
    project.extensions.create(EXTENSIVE,VersionInfo.class)
    //2.创建待处理的Task
    project.tasks.create(TASK_NAME,UploadTask.class)
    //3.将uploadTask任务挂架到Project的生命周期中
    def build = project.tasks.getByName('clean')
    def uploadTask = project.tasks.getByName(TASK_NAME)
    //这里使用dependsOn强依赖任务关系
    build.dependsOn(uploadTask)
}
```

### 步骤6.插件发布	

笔者为了测试，将jar包只发布在本地，测试使用。

使用如下方式发布：

```java
gradlePlugin {
    plugins {
        modularPlugin {
                id = 'com.yuhb.upload'
                implementationClass = 'com.yuhb.upload.UploadVersionPlugin'
        }
    }
}
```
这个配置在`build`后自动生成`resources`文件：这个插件扩展配置是引入的：`java-gradle-plugin`中。

![resources文件自动生成.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bad4c2c4668a44cbbc7ffc3cda838889~tplv-k3u1fbpfcp-watermark.image?)
当然也可以直接在`resources`文件夹中上手动写入该文件

在插件的build.gradle实现下面的逻辑：
```java
uploadArchives {
    repositories {
            mavenDeployer {
                    repository(url:uri('D:/maven_local'))
                    pom.groupId = 'com.yuhb.upload'
                    pom.artifactId = 'uploader'
                    pom.version = '1.0.0'
            }
    }
}
```

在命令行执行：

```java
./gradlew :uploadversion:uploadArchives	
```

然后去本地文件夹下面看看是否上传成功：

![本地文件成功.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4daea734c5c448099dbd33d993fff742~tplv-k3u1fbpfcp-watermark.image?)

这里要说明下：
> 一般情况下都会将自定义插件发布到maven私服或者中央仓库，才可以供其他项目使用
> 关于如何发布到maven私服，可以查看这篇文章
> 后期也会出一期文章教大家如何将数据发布到中央仓库

### 步骤7.插件引入
- **步骤1**：在工程的根`build.gradle`文件中引入：

```java
buildscript {
        repositories {
                ...
                maven {
                        url uri('D:/maven_local')
                }
        }
        dependencies {
                ...
                classpath 'com.yuhb.upload:upload:1.0.0'
        }
}
```

说明：

`com.yuhb.upload:uploader:1.0.0`格式：

| 引入字段        | 发布字段       |
| --------------- | -------------- |
| com.yuhb.upload | pom.groupId    |
| uploader        | pom.artifactId |
| 1.0.0           | pom.version    |

- **步骤2**：在子Project中引入插件：

```java
apply plugin: 'com.yuhb.upload'
```
- **步骤3**：配置`extensive`插件扩展：

```java
versionInfo {
    versionName = '1.0.0'
    versionCode = 1
    versionUpdateInfo = '当前是第一个版本：初始apk'
}
```


这个`versionInfo`扩展是怎么来的呢？

我们看下之前我们配置插件的时候，使用了：

```java
EXTENSIVE = 'versionInfo'
project.extensions.create(EXTENSIVE,VersionInfo.class)

```
在插件中引入`extensions`中的字段，就是我们`Project`中配置的扩展字段：

```java
versionInfo {
    versionName = '1.0.0'
    versionCode = 1
    versionUpdateInfo = '当前是第一个版本：初始apk'
}
```
就是这里，如果外部配置了`versionInfo`的扩展字段，就会通过`project.extensions`获取到，并将数据写入`project.extensions`的`versionInfo`属性中：之后就可以使用`project.extensions`的`versionInfo`属性访问外部传入的配置数据:
	
```java
def name = project.extensions.versionInfo.versionName
def code = project.extensions.versionInfo.versionCode
def info = project.extensions.versionInfo.versionUpdateInfo
```

- **步骤4**：运行`root`的`build` 任务查看编译信息：		
```java
./gradlew build
```
	结果：
	> Task :app:uploadTask
	name:1.0.0 code:1 info:当前是第一个版本：初始apk


​	
这里运行build可以执行插件中的任务是因为前面笔者将插件Task挂接到了build任务之前：
`挂接代码`：

```java
//3.将uploadTask任务挂架到Project的生命周期中
def build = project.tasks.getByName('build')
def uploadTask = project.tasks.getByName(TASK_NAME)
//这里使用dependsOn强依赖任务关系
build.dependsOn(uploadTask)
	
```
**项目Demo完整代码已经上传Github：**
https://github.com/ByteYuhb/a_gradle_plugin_sample	
## 5.总结
本文主要针对我们自定义插件定义以及优势做了一些说明，且使用一个实战项目对自定义插件制作，发布，引入流程做了一个详细的讲解
，Gradle插件部分还有Gradle的上传流程和AGP插件讲解没有讲，后面都会陆续推出。


## 参考资料
- [Gradle自定义插件](https://docs.gradle.org/current/userguide/custom_plugins.html#header)
_官网文档
-   [Using Gradle Plugins](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.gradle.org%2F6.0.1%2Fuserguide%2Fplugins.html%23sec%3Aplugin_management "https://docs.gradle.org/6.0.1/userguide/plugins.html#sec:plugin_management") _ Gradle 官方文档
-   [Gradle 系列（2）手把手带你自定义 Gradle 插件](https://juejin.cn/post/7098383560746696718)_胡飞洋