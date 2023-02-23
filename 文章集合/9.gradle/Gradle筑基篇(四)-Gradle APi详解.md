---
theme: smartblue
highlight: a11y-dark
---


携手创作，共同成长！这是我参与「掘金日新计划 · 8 月更文挑战」的第3天，[点击查看活动详情](https://juejin.cn/post/7123120819437322247 "https://juejin.cn/post/7123120819437322247") >>
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言：
前面我们使用两篇文章讲解了Gradle一些基础知识和Groovy语法详解

> 工欲善其事必先利其器

今天我们来讲解下`Gradle的Api`相关知识

**相关系列文章：**
> Gradle筑基篇：
- [Gradle筑基篇(一)-Gradle初探](https://juejin.cn/post/7129780699145437215)

- [Gradle筑基篇(二)-Groovy语法的详解](https://juejin.cn/post/7129336080112812039)

- Gradle筑基篇(三)-Gradle生命周期

- [Gradle筑基篇(四)-Gradle APi详解](https://juejin.cn/post/7130177944189665310/)

- Gradle筑基篇(五)-Gradle自定义插件

- Gradle筑基篇(六)-Gradle Maven仓库管理

> Gradle进阶篇
- Gradle进阶篇(六)-AGP详解

## `Gradle`Api
这里我将Gradle api分为以下五个部分

1.**Project api**:

2.**Task ap**i

3.**File api**

4.**Property api**

5.**其他 api**

既然是讲解api，那就首先去他们源码中看看：

笔者使用的是最新版本的：`Gradle7.5.1`

查看源码方式：

更改：`gradle-wrapper.properties`文件中的
```java
distributionUrl=https\://services.gradle.org/distributions/gradle-7.5.1-bin.zip
为：
distributionUrl=https\://services.gradle.org/distributions/gradle-7.5.1-all.zip
```
重新编译之后就可以看到我们`Gradle`的源码了

我们先来看Project部分
## 1.`Project` api
由于Project源码篇幅太长：这里只列出类的部分方法和属性：

我们前面分析过，每个build.gradle对应一个Project，Project在初始过程中会被构建为`树`形结构：
如下：

![gradleproject树.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d5db199dd1945cf90112d923d898582~tplv-k3u1fbpfcp-watermark.image?)

每个Project都有自己的`子Project`和`父Project`

Gradle给我们提供了一系列对Project的操作:


- 1.`getAllprojects`:可以获取工程中的所有Project
这个方法最常见使用场景：就是给我们的项目配置仓库地址：

```java
//统一配置所有子project的集合
allprojects {
    repositories {
        maven {url "https://maven.aliyun.com/repository/google/"}
        maven {url "https://maven.aliyun.com/repository/public/"}
        mavenCentral()
        maven {
            url uri('D:/maven_local')
        }
    }
    group 'cpm_group'
    version 1.0
}
```

当然也可以配置所有项目的其他属性：如`group`,`version`，`description`等

- 2.`getSubprojects`：获取所有的子Project

**使用场景列举**：将所有的lib模块上传到maven中

```js
//包括子Project
subprojects {Project project ->
    if(plugins.hasPlugin('com.android.library')){
        apply from:'../publishMaven.gradle'
    }
}
```
- 3.`getProject`：获取当前Project实例

我们所有的build.gradle中的代码，都是以当前Project实例为delegate展开的：
在脚本中，你可以使用下面方法调用project方法：

```java
1.this.project
2.project
3.this
4.什么不不写，直接调用方法或者属性
```

以上方法调用方式结果都是一样的

- 4.`getRootProject`：获取root脚本就是我们根工程的Project

获取根Project的用处也很大，我们平时在根工程中定义的一系列变量，task等都可以通过这个方式在子Project中获取

- 5.`getParent`：获取父Project实例

- 6.`findProject`：查找Project，需要传入Project名称获取路径

- 7.`project(String path)`：定位一个外部或者内部Project。

关于Project操作的api就在上面了

**下面我们来讲解关于Task相关的api**

## 2.`Task` api
Gradle中整个工程由若干个Project组成，而每个Project由若干个Task组成，
在Gradle中Task由`TaskContainer`统一管理，工程全局只有一个`TaskContainer`，project中可以使用tasks访问TaskContainer方法
### 1.**创建**

```java
//使用Project的方法创建：

task task1{
	group 'yuhb'
}
task task1(group:'yuhb'){

}

//使用TaskContainer创建：
tasks.create('task1'){
	group 'yuhb'

}

//带任务类型的Task：一般在插件中使用
class MyTestTask extends DefaultTask {

    @TaskAction
    void doActon(){
		//do something
    }
}
tasks.create('task1',MyTestTask.class)
//注册一个task，在必要的时候创建，不是立即创建
tasks.register('task1',MyTestTask.class)
```

### 2.**查找**

```java
//findByName:
def task1 = tasks.findByName('task6')：
//getByName:
def task2 = tasks.getByName('task6')
//两者区别：findByName没有找到返回null，getByName没有找到返回异常UnknownTaskException
//findAll：
Set<Task> taskSet = this.tasks.findAll()
//查找当前TaskContainer中所有的任务
//matching:
tasks.matching {
    group = 'yuhb'
}
//获取匹配某些条件的task	
```


### 3.**删除**

Gradle没有提供删除方法，也不需要，因为每个任务都只会执行一次，
如果确切不需要就不要引入这个task即可

### 4.**设置`task属性`**

```java
//创建task的时候传入：
//方法1：在参数中传入
task task2(group:'yuhb',description: 'this is task2'){

}
//方法2：在闭包中传入
tasks.create('task3'){
    group 'yuhb'
    description 'this is task3'
}
//两种方法效果是一样的
```

**Task属性：**

| 属性        | 描述                         |
| ----------- | ---------------------------- |
| name        | 唯一标识符                   |
| group       | 组别                         |
| description | 描述信息                     |
| type        | Task类型，默认为 DefaultTask |
| actions     | 有哪些动作                   |
| dependsOn   | 依赖的task列表               |

### 5.**Task依赖管理**

5.1:**使用`dependOn`**

```java
task task1{
    //单个task
    dependsOn 'task2'
    //多个task使用列表
    dependsOn = ['task2','task3']
}
//这里task1强依赖task2和task3
```


5.2:**使用mustRunAfter**
    
	task task1{
		//单个task
		mustRunAfter 'task2'
		//多个task使用列表
		mustRunAfter = ['task2','task3']
	}
5.3:**使用Task输入和输出**

每个task都会有自己的输入和输出：产出数据可能会提供给下一个任务使用
`TaskInputs`：管理输入

`TaskOutputs`：管理输出

输入和输出有三种类型：

**1.文件，文件夹**

**2.单个映射属性**

**3.多个映射属性，Map**

```java
task task2(group:'yuhb',description: 'this is task2'){
        inputs.file file('release.xml')
}
task task3(group:'yuhb',description: 'this is task2'){
        outputs.file file('release.xml')
}
```

**使用上面的方式通过输入和输出的挂接，将task2和task3实现依赖关系。**
### 6.**Task执行**
使用task的doFirst和doLast可以在任务执行前后设置一些Action

```java
task task3(group:'yuhb',description: 'this is task2'){
    outputs.file file('release.xml')
    doFirst {
          'task3执行前'
    }
    doLast {
          'task3执行后'
    }
}
```
Task执行方式：

1.使用`gradlew`命令行：如要执行build任务：
```java
gradlew build
```

2.使用`IDE`中的`Gradle`面板

![gradle面板.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c2e6b941e08b478894b0ecfe264d6acd~tplv-k3u1fbpfcp-watermark.image?)

3.将task挂接到Gradle生命周期中

我们创建任务后，在执行构建过程中并没有挂接到Gradle生命周期中，也就是不会执行

```java
def prebuild = this.tasks.findByName('prebuild')
prebuild.dependsOn('task1')
```

上面的例子`prebuild`是编译前需要执行的一个预编译任务，使用`dependsOn`依赖关系，将任务`task1`使用`dependsOn`挂接到`prebuild`执行前


关于Task api就讲解到这里，以上api基本涵盖我们对Task的使用

## 3.文件

Gradle中的文件操作和java中的文件操作是可以互相混用的。也就是说
> 在Gradle中可以直接使用java中的文件操作。


下面介绍几种Gradle中文件使用方式：

### 1.**文件创建以及获取方式**

`方式1`：file
```java
def file1 = file('release.xml')
def file2 = file('release.xml',PathValidation.FILE)
```
file2使用的`第二个参数`是校验文件使用：

有以下几个校验方式：

```java
public enum PathValidation {
    NONE(), EXISTS(), FILE(), DIRECTORY()
}
```
默认使用的是`NONE`：

`EXISTS(), FILE(), DIRECTORY()`：表示如果不满足当前条件会报对应的异常
- `EXISTS`：文件是否存在
- `FILE`：是否是文件
- `DIRECTORY`:是否是文件夹

`方式2`：files

```java
ConfigurableFileCollection files(Object... paths);
//获取一个文件集合，返回类型ConfigurableFileCollection
def _files = files('release.xml','release2.xml')
```


`方式3`：fileTree
获取一个文件夹下面的所有的文件

```java
def files = fileTree(dir: 'libs',includes: ['*.jar']){
    excludes = ['a*.jar','b*.jar']
    builtBy = ['task1','task2']
}
```

**也可以使用fileTree来对文件进行遍历**

`方式4`：zipTree

```java
FileTree zipTree(Object zipPath);
```

获取zip文件下面的所有文件

### 2.**文件路径设置及获取**

`getRootDir`:获取根路径

`setBuildDir`:设置编译路径

`getBuildDir`：获取编译路径

`getProjectDir`：获取当前Project的路径

### 3.**文件拷贝**

```java
copy {
    from file('release.xml')
    into getRootProject().getBuildDir().path+'/test/'
}
```

### 4.**文件遍历**

普通文件夹遍历
```java
fileTree('build/outputs/apk/'){ FileTree fileTree ->
    fileTree.visit { FileTreeElement element ->
        copy {
            from element.file
            into getRootProject().getBuildDir().path+'/test/'
        }
    }
}
```
zip/tar压缩文件遍历

```java
FileTree ziptr = zipTree('release1.zip')
FileTree ziptr = tarTree('release1.zip')
```
然后用`FileTree`的`visit`方法进行遍历

### 5.**文件写入和读出**

使用java文件的`InputStream`和`OutputStream`就可以了，这个大家都很熟悉了

文件Api就讲到这里，下面来看下**属性api**这块

## 4.属性Api

### 属性分类：
- 1.在`gradle.properties`中设置的全局属性

```java
org.gradle.jvmargs =-Dfile.encoding=UTF-8
android.useAndroidX=true
android.enableJetifier=true
isLoadTest=true
GRADLE_USER_HOME = '../../user'
```
这里面包括系统属性和开发者自定义的属性，工程全局都可以访问
其他地方使用访问方式：
- 2.在rootProject中设置的`root项目属性`：一般用于统一全局版本信息使用

```java
ext {
    mCompileSdk = 31
    versionName = '1.0.2'
    versionCode = 2
    versionInfo = 'App的第2个版本，更新了一些基础核心功能'
}
```

**注意：** 其他地方需要使用：则最好加上`rootProject.ext.xxProperty`

- 3.**当前Project中的属性：**

包括自定义的属性或者Project自带的属性：
如：
```java
this.project.gradle:当前Gradle
this.project.tasks:当前工程的TaskContainer
this.project.task1:获取当前Project中的task1任务
```
- 4.**当前Project定义的ext属性**
一般用于当前Project使用的ext属性
- 5.**`Extension` 扩展属性**

Extension 扩展是插件为外部构建脚本提供的配置项，用于支持外部自定义插件，我们项目中给的`android{}`
其实就是`Android Gradle Plugin`给我们提供的`Extension` 扩展，插件内部可以获取这个扩展属性，然后执行对应流程

```java
ReleaseInfo {
    versionCode = 1
    versionName = '1.0.0'
    versionInfo = "12345566版本发布"
    fileName = "releaseinfo.xml"
}
```
这里的`ReleaseInfo`是笔者自定义插件中的一个`Extension`扩展，插件中的`Task`可以使用这个`Extension`获取到用户提供的属性。

### 属性访问方式：
`hasProperty('key')`：是否包含该属性

`findProperty('key')`;找属性，没有找到返回null

`property('key')`：找属性，没有找到返回
`MissingPropertyException`异常

`getProperties()`：获取当前Project的所有属性

`setProperty('key','value')`;设置属性

一般我们访问属性：直接使用key访问

如：

```java
定义：GRADLE_USER_HOME = '../../user' =>等价：project.setProperty('GRADLE_USER_HOME','../../user')
访问：GRADLE_USER_HOME  =>等价于：project.getProperty('GRADLE_USER_HOME')
定义：project.name = 'pp1' =>等价：project.setProperty('name','pp1')
访问：name =>等价于：project.getProperty('name')
```

**关于自定义插件这块内容，后面会单独出一期文章**


## 总结：
今天这篇文章主要是对`Gradle`中我们比较常用给的一些api进行了讲解。
主要包括`Project相关api`，`Task相关api`，`文件相关api`和`属性相关api`等，其实还有一些其他的比如`外部命令的api`，这些很少会用到，就不再讲解了.

可以结合这篇文章，自己再去看源码和相关官网文档，会让自己对api的认识更加深刻。
后面会持续推出Gradle的一些高级语法，如`自定义插件`，`优秀开源框架插件`的解读以及`AGP的解析`；
> **点关注，不迷路，助你进阶移动端高级开发。**






​	