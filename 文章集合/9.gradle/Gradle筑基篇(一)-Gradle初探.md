---
theme: smartblue
highlight: a11y-dark
---


携手创作，共同成长！这是我参与「掘金日新计划 · 8 月更文挑战」的第2天，[点击查看活动详情](https://juejin.cn/post/7123120819437322247 "https://juejin.cn/post/7123120819437322247") >>
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言:
大家回想一下自己第一次接触`Gradle`是什么时候？
相信大家也都是和我一样，在我们打开第一个AS项目的时候，
发现有很多带gradle字样的文件：`setting.gradle,
build.gradle,gradle.warpper`,以及在`gradle`文件中各种配置，
这些都是啥wy啊。。


特别对于一些小公司开发人员，因为接触架构层面的机会很少，可能在使用AS几年后都不一定对`Gradle`有太多深入了解，这是实话，因为笔者就是这么过来的。。

而`Gradle`又是`进阶高级`开发的必经之路。

好了，接下来进入正题，此系列笔者会由浅入深的方式，带领大家来了解下，`Gradle`背后究竟有哪些奥秘。

本系列文章：

> Gradle筑基篇：
- Gradle筑基篇(一)-Gradle初探

- [Gradle筑基篇(二)-Groovy语法的详解](https://juejin.cn/post/7129336080112812039)

- Gradle筑基篇(三)-Gradle生命周期

- Gradle筑基篇(四)-Gradle APi详解

- Gradle筑基篇(五)-Gradle自定义插件

- Gradle筑基篇(六)-Gradle Maven仓库管理

> Gradle进阶篇
- Gradle进阶篇(六)-AGP详解

**本篇是这个系列的第一篇文章：Gradle初探**
## 1.`Gradle`定义：
很多开发喜欢把`Gradle`简单定义为一种构建工具，和`ant，maven`等作用类似，
诚然Gradle确实是用来做构建，但是如果简单得把Gradle拿来做构建，就太小看Gradle了.

笔者更愿意将Gradle看做一种`编程框架`。在这个框架中，你可以做很多ant，maven等常用构建工具做不了的事情，
如将自己的任务task集成到构建生命周期中，完成文件拷贝，脚本编写等操作。

## 2.`Gradle`优缺点：
相较早期的构建工具：ant，maven等。

### 优点如下：
- **1.使用DSL Grovvy语言来编写：**:了解ant的同学应该都知道：ant使用的是xml配置的模式，而Gradle使用的是`表达性的Groovy`来编写，
Groovy同时`支持面向对象和面向过程`进行开发，这个特性让`Groovy`可以写出一些脚本的任务，这在传统ant，maven上是不可能实现的

- **2.基于java虚拟机：**:`Groovy`是基于`jvm`的语言，`groovy`文件编译后其实就是class文件，和我们的`java`一样。

所以在gradle构建过程中，我们完全可以使用java/kotlin去编写我们的构建任务以及脚本，极大的降低我们学习的成本。

- **3.Gradle自定义task**：可以构建自己的任务，然后挂接到gradle构建`生命周期`中去，这在ant，maven上也是不可能实现的，

- **4.扩展性好**：gradle将关键配置扔给我们开发者，开发者配置好任务后，无需关心gradle是如何构建的。

- **5.支持增量更新**：增量更新可以大大加快我们的编译速度

关于Groovy的语法篇：可以参考这篇文章：
[Gradle筑基篇(二)-groovy语法详解](https://juejin.cn/post/7129336080112812039)
### 缺点：
> 用过gradle都知道，低版本gradle的项目在高版本的gradle中经常出现很多莫名其妙的错误，向后兼容性较差。


## 3.`Gradle`工程结构：
gradle标准工程代码如下

```java
├── moduleA
│   └── build.gradle
├── moduleB
│   └── build.gradle
├── build.gradle
├── settings.gradle
├── gradle.properties
├── local.properties
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
└── gradlew.bat
```

- 1.`build.gradle`:可以理解为一个Project脚本，Project脚本中有自己的任务，最外层的Project为rootProject
- 2.`settings.gradle`：主要用来配置我们项目中需要用到的模块。用include关键字给包裹进
- 3.`gradle.properties`：这个文件主要是设置一些全局变量，包括jvm运行以及自定义的一些全局参数
- 4.`local.properties`：这个文件主要配置一些本地的sdk和ndk版本信息以及路径
- 5.`gradle-wrapper.jar`：负责自动下载Gradle脚本运行环境
- 6.`gradle-wrapper.properties`：用来配置当前使用的Gradle的版本以及存储的路径

```java
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-6.5-bin.zip

```

	distributionBase + distributionPath：指定Gradle安装路径；
	zipStoreBase + zipStorePath：指定Gradle安装包的存储路径；
	distributionUrl：Gradle版本的下载地址。
> 注意这里如果将bin改为all，则可以查看当前Gradle的源码信息。

- 7.`gradlew和gradlew.bat`：用来执行构建任务的脚本，可以在命令行使用gradlew xxxTask

## 4.Gradle生命周期

Gradle作为新兴的构建工具，其内部也有自己的生命周期阶段，每个阶段做的事情都层次分明，
了解Gradle生命周期，才能很好的使用我们的Gradle工具。

### 1.初始化阶段
做了哪些事情?：
- 1.初始化`Setting.gradle`文件，获取`setting`实例，
- 2.执行`setting`中的脚本，根据`include`字段，创建对应的`project`实例
- 3.设置构建需要的环境

> 注意：初始化阶段执行任何任务都会执行一次。

Project实例关系如下：

![gradleproject树.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/569765f2c63743ecb7f7915a23d6e4cf~tplv-k3u1fbpfcp-watermark.image?)
### 2.配置阶段
- 1.下载所有插件和构建脚本依赖项
- 2.执行`build.gradle`文件中的脚本信息
- 3.实现task任务的拓扑图，这个图是一个`有向无环图`，防止任务执行进入死循环。

> 注意：配置阶段执行任何任务都会执行一次。
### 3.执行阶段
执行阶段就是根据当前task拓扑图进行执行task任务。

需要注意以下几点：
- 1.在项目中配置的`doLast，doFirst`操作，都会在任务执行阶段执行，而不会在配置阶段执行，
	而如果任务需要执行，需要挂接到`gradle执行生命周期`中，笔者开始接触gradle时就踩过这个坑。。这块后面讲解`task`的时候在来具体讲解
- 2.前面也说了初始化阶段和配置阶段在每个任务执行前都会执行，所以**不要在前两个阶段进行一些耗时的操作**，这样可能每次编译执行你都会崩溃的
	
## 5.Gradle生命周期监听：
要查找Gradle是如何监听生命周期，可以到`Gradle`源码中看看：

- 1.**监听初始化阶段**
初始化阶段主要用来初始化`Setting.gradle`文件，获取`setting`实例，创建`Project`实例等，所以其可用下面代码监听:

```java
//开始初始化Setting.gradle前
this.gradle.beforeSettings {
    println "beforeSettings"
}
//Setting.gradle配置完毕后，创建了setting实例
this.gradle.settingsEvaluated {
    println "settingsEvaluated"
}
//执行解析Setting.gradle文件后，创建了project实例列表
this.gradle.projectsLoaded {
    println "projectsLoaded"
}
```

- 2.**监听配置阶段**

2.1:**监听当前project的配置阶段前后：**
    
在`Project源码`中可以看到：

```java
/**
 * Adds an action to execute immediately before this project is evaluated.
 *
 * @param action the action to execute.
 */
void beforeEvaluate(Action<? super Project> action);

/**
 * Adds an action to execute immediately after this project is evaluated.
 *
 * @param action the action to execute.
 */
void afterEvaluate(Action<? super Project> action);

/**
 * <p>Adds a closure to be called immediately before this project is evaluated. The project is passed to the closure
 * as a parameter.</p>
 *
 * @param closure The closure to call.
 */
void beforeEvaluate(Closure closure);

/**
 * <p>Adds a closure to be called immediately after this project has been evaluated. The project is passed to the
 * closure as a parameter. Such a listener gets notified when the build file belonging to this project has been
 * executed. A parent project may for example add such a listener to its child project. Such a listener can further
 * configure those child projects based on the state of the child projects after their build files have been
 * run.</p>
 *
 * @param closure The closure to call.
 */
void afterEvaluate(Closure closure);
```



看这两个方法的说明就是用来监听配置阶段，传入的是一个Action或者传入一个闭包，闭包的代理为当前Project

**使用方式如下**：

```java
//监听project被配置前
this.beforeEvaluate {Project project ->
    println "${project.name} :beforeEvaluate"
}
//监听project被配置后
this.afterEvaluate {Project project ->
    println "${project.name}:afterEvaluate"
}
```

> 注意：这个监听只是针对当前Project的配置阶段而不是所有Project的配置

你也可以使用：

```java
this.project.beforeEvaluate
this.project.afterEvaluate
```


**那么有没有可以监听所有Project的配置阶段的api呢？安排**

2.2：**监听每个Project的配置前后：**

使用this.gradle的内部方法，因为**gradle是相对于整个工程作为作用域**

```java
//监听所有的Project的被配置前
this.gradle.beforeProject {Project project ->
    println "${project.name}:beforeProject"
}
//监听所有的Project的被配置后
this.gradle.afterProject {Project project ->
    println "${project.name}:afterProject"
}

```

编译下看看：

```java
> Configure project :
gradle_source_plugin:afterProject

> Configure project :app
app:beforeProject
do app evaluating
app:afterProject

> Configure project :application
application:beforeProject
do application evaluating
application:afterProject
```
看到当前工程所有的project都调用了一次`beforeProject和afterProject`

**那有同学又要问了，有没有监听整个project配置阶段的：当然有**

2.3：**监听全部project配置阶段的前后**

```java
this.gradle.projectsEvaluated {
    println "all projectsEvaluated"
}
```

> 这个闭包可以监听整个项目的配置完毕后的事件

配置阶段还有一些监听如下：

2.4：**监听任务的添加操作**

```java
this.project.tasks.whenTaskAdded {Task task->
    println "${task.name}:whenTaskAdded"
}
```
2.5：**监听任务拓扑图的执行**

```java
//task拓扑图构造完毕
this.gradle.taskGraph.whenReady {TaskExecutionGraph graph->
    println "taskGraph:->"+graph
}
```

**监听拓扑图完毕后其实才是真正的配置阶段完毕**，瞧瞧源码：

在BasePlugin中：

```java
threadRecorder.record(
		ExecutionType.BASE_PLUGIN_PROJECT_CONFIGURE,
		project.getPath(),
		null,
		this::configureProject);

threadRecorder.record(
		ExecutionType.BASE_PLUGIN_PROJECT_BASE_EXTENSION_CREATION,
		project.getPath(),
		null,
		this::configureExtension);

threadRecorder.record(
		ExecutionType.BASE_PLUGIN_PROJECT_TASKS_CREATION,
		project.getPath(),
		null,
		this::createTasks);
```

看到配置阶段最后一步才是创建Task，所以可以使用this.gradle.taskGraph.whenReady监听整个配置阶段的结束


- 3.**监听执行阶段**

3.1：监听任务执行：

```java
gradle.taskGraph.beforeTask { Task task ->
    println "${task.name}:beforeTask"
}
gradle.taskGraph.afterTask {Task task ->
    println "${task.name}:afterTask"
}
执行下面任务：
task clean(type: Delete) {
    doFirst {
        println 'clean:doFirst'
    }
    doLast {
        println 'clean:doLast'
    }
    delete rootProject.buildDir
}
结果：
> Task :clean
clean:beforeTask
clean:doFirst
clean:doLast
clean:afterTask
```


可以看到在task执行前后调用了监听中的方法

3.2：**监听执行任务阶段开始**

其实可以使用配置阶段的this.gradle.taskGraph.whenReady，这个就是所有project配置完毕，且生成了task拓扑图
下一步就是开始执行任务了

3.3：**监听执行任务阶段结束**

this.gradle.buildFinished {}
这个可以监听所有任务执行完毕后事件回调：


## 6.Gradle Api
Gradle为我们提供了很多丰富的api操作
主要有几下几种：
- Project api 
- Task api
- 属性 api
- 文件 api
- 以及一些其他api

由于api这块篇幅比较多，就不展开讲解了，后面会单独出一篇文章来讲解这块内容

## 7.Gradle插件
Gradle插件在我们的项目中使用的还是比较多的，在一些优秀的开源框架：

如鹅厂的`Tinker`，滴滴的`VirtualApk`，阿里的`Arouter`等
内部都使用了`Gradle`插件知识

笔者Gradle插件开始学习的时候，也是一脸懵逼，
> 其实你把Gradle插件理解为一个第三方jar包就可以了，只是这个jar包是用于我们apk构建的过程
> 内部其实也是使用一些Task，挂接到我们的apk构建生命周期中。
> 这里也不会过多讲解

下面我们来讲下Gradle一个特性：

## 8.增量更新

有没发现你在构建过程中，如果修改的地方对整个任务容器影响不大情况下，你的编译速度会很快，其实就是Gradle默认支持增量更新功能。

 - 1.`定义`:

 官方：
>  An important part of any build tool is the ability to avoid doing work that has already been done. 
>  Consider the process of compilation. Once your source files have been compiled, 
>  there should be no need to recompile them unless something has changed that affects the output, 
>  such as the modification of a source file or the removal of an output file. And compilation can take a significant amount of time, 
>  so skipping the step when it’s not needed saves a lot of time.


>  简单点说就是Gradle目前对Task的输入和输出做了判断，如果发现文件的输入和输出没有变化，
>  就直接使用之前缓存的输入输出数据，不再重新执行，缩短编译时间


![taskInputsOutputs.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dcf999eadc69494a8f7fdf136f4ea829~tplv-k3u1fbpfcp-watermark.image?)
 这里就涉及到了Task的一些知识点：
 Task是我们apk构建过程中给的最少单位，每个任务都有输入和输出，将输入的信息传递给下一个任务作为下一个任务的输入，这就是整个构建体系正常运行的核心。

 - 2.`Task`输入和输出

 任务的执行离不开输入和输出，和我们方法执行一样，依赖输入参数和输出返回值

` Gradle`中使用：

`TaskInputs`：来管理输入

`TaskOutputs`:来管理输出

 我们来看下这个两个类的内部代码：

```java
TaskInputs.java
public interface TaskInputs {
    /**
     * Returns true if this task has declared the inputs that it consumes.
     *
     * @return true if this task has declared any inputs.
     */
    boolean getHasInputs();

    /**
     * Returns the input files of this task.
     *
     * @return The input files. Returns an empty collection if this task has no input files.
     */
    FileCollection getFiles();

    /**
     * Registers some input files for this task.
     *
     * @param paths The input files. The given paths are evaluated as per {@link org.gradle.api.Project#files(Object...)}.
     * @return a property builder to further configure the property.
     */
    TaskInputFilePropertyBuilder files(Object... paths);

    /**
     * Registers some input file for this task.
     *
     * @param path The input file. The given path is evaluated as per {@link org.gradle.api.Project#file(Object)}.
     * @return a property builder to further configure the property.
     */
    TaskInputFilePropertyBuilder file(Object path);

    /**
     * Registers an input directory hierarchy. All files found under the given directory are treated as input files for
     * this task.
     *
     * @param dirPath The directory. The path is evaluated as per {@link org.gradle.api.Project#file(Object)}.
     * @return a property builder to further configure the property.
     */
    TaskInputFilePropertyBuilder dir(Object dirPath);

    /**
     * Returns a map of input properties for this task.
     *
     * The returned map is unmodifiable, and does not reflect further changes to the task's properties.
     * Trying to modify the map will result in an {@link UnsupportedOperationException} being thrown.
     *
     * @return The properties.
     */
    Map<String, Object> getProperties();

    /**
     * <p>Registers an input property for this task. This value is persisted when the task executes, and is compared
     * against the property value for later invocations of the task, to determine if the task is up-to-date.</p>
     *
     * <p>The given value for the property must be Serializable, so that it can be persisted. It should also provide a
     * useful {@code equals()} method.</p>
     *
     * <p>You can specify a closure or {@code Callable} as the value of the property. In which case, the closure or
     * {@code Callable} is executed to determine the actual property value.</p>
     *
     * @param name The name of the property. Must not be null.
     * @param value The value for the property. Can be null.
     */
    TaskInputPropertyBuilder property(String name, @Nullable Object value);

    /**
     * Registers a set of input properties for this task. See {@link #property(String, Object)} for details.
     *
     * <p><strong>Note:</strong> do not use the return value to chain calls.
     * Instead always use call via {@link org.gradle.api.Task#getInputs()}.</p>
     *
     * @param properties The properties.
     */
    TaskInputs properties(Map<String, ?> properties);

    /**
     * Returns true if this task has declared that it accepts source files.
     *
     * @return true if this task has source files, false if not.
     */
    boolean getHasSourceFiles();

    /**
     * Returns the set of source files for this task. These are the subset of input files which the task actually does work on.
     * A task is skipped if it has declared it accepts source files, and this collection is empty.
     *
     * @return The set of source files for this task.
     */
    FileCollection getSourceFiles();
}
```

源文件中我们可以看出：
输入可以有以下种类：
- 1.**文件，文件夹以及一个文件集合**
- 2.**普通的key value属性**
- 2.**Map：传递一个Map的属性集合**

`TaskInputs`还可以通过`getHasInputs`判断是否有输入

同理我们来看下`TaskOutputs`的源码，篇幅原因，这里直接看下TaskOutputs的方法框架：

![Outputs.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/089bdbabcbb84db6a4e72667ac9f5c30~tplv-k3u1fbpfcp-watermark.image?)

大部分情况和inputs类似，可以输出为文件，属性properties等

注意到这里有几个关键的方法：
`upToDateWhen和cacheIf`
这两个方法就是用来对构建中的是否对输出操作进行缓存的点，用于`增量构建`使用

## 总结
本篇文章主要是讲解了Gradle一些基础认识，Gradle工程项目的概括以及Gradle构建生命周期管理和监听等操作。
后面文章会陆续推出关于GradleApi，Gradle插件以及AGP插件的详细介绍，希望大家能从中会有一些收获。







​	