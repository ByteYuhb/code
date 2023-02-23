携手创作，共同成长！这是我参与「掘金日新计划 · 8 月更文挑战」的第5天，[点击查看活动详情](https://juejin.cn/post/7123120819437322247 "https://juejin.cn/post/7123120819437322247") >>
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
前面几篇文章我们讲解了关于关于`Gradle的基础`，`Gradle生命周期`，`Gradle相关Api`的讲解，以及`Gradle自定义插件`

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

现如今已经不是以前单兵作战时代，越来越多的需求，促使我们项目去实现动态组件化开发。
这个时候组件化发布共享就显的尤为重要。

这篇文章我们就来讲解下如何使用`Maven进行组件化发布`

在讲解组件化发布之前，我们先来了解一些基础概念

## 基础概念：
### 1.POM
pom:全名`Project Object Model` 项目对象模型，用来描述当前`maven`项目发布模块的基础信息

pom主要节点信息如下：

| 配置         | 描述              | 举例（'com.android.tools.build:gradle:4.1.1'） |
| ------------ | ----------------- | ---------------------------------------------- |
| `groupId`    | 组织 / 公司的名称 | com.android.tools.build                        |
| `artifactId` | 组件的名称        | gradle                                         |
| `version`    | 组件的版本        | 4.1.1                                          |
| `packaging`  | 打包的格式        | aar                                            |



### 2.仓库
我们在开发中经常使用到第二/三方插件或者第二/三方库，就是存储在仓库中的

#### 2.1：仓库种类：
- **本地仓库**：存储在本地设备中的仓库以及远程仓库中下载保存的仓库，统称为本地仓库
- **私有仓库**：公司内部仓库，比如是有maven私服搭建的局域网仓库
- **中央仓库**：开源社区仓库，我们平时使用的第三方插件或者类库一般都存储在中央仓库，比如`Maven Central`，阿里的国内镜像库等
![1.maven库介绍.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2ddd77463f44b85a719347f1310b776~tplv-k3u1fbpfcp-watermark.image?)
#### 2.2：仓库构建顺序:
- 1.在本地仓库中查找对应的类库，没有找到`执行2`
- 2.按照`repositories`中声明的仓库顺序，在私有仓库和中央仓库查找对应的类库，找到则将类库版本信息下载到本地仓库，没找到则`执行3`
- 3.前面都没找到对应的类库或者类库版本，则抛出异常‘`没找到对应的类库`’

![仓库执行顺序.awebp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e80491b0fb744532a695ed36c5a011c8~tplv-k3u1fbpfcp-watermark.image?)

#### 2.3：仓库声明方式：

`项目build.gradle`

```java
buildscript {
    repositories {
        [Gradle 插件的仓库]        
    }
}
allprojects {
    repositories {
        [项目中所有模块依赖的仓库]  
    }
}
```
`模块内build.gradle`

```java
repositories{
    [当前模块依赖的仓库]
}
```


`gradle支持的仓库类型`：

```java
repositories{
    maven { url '...' }
    ivy { url '...' }
    flatDir { dirs '...' }
}

```

`常用的中央仓库：`

```java
google()        // https://dl.google.com/dl/android/maven2/
mavenCentral()   // https://repo.maven.apache.org/maven2/
jCenter() 
```
网络不好的情况下，访问上面的中央仓库会有点慢：可以开考虑使用国内镜像代理

```java
maven { url 'http://maven.aliyun.com/nexus/content/repositories/google' }
maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
maven { url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
```


### 3.有了`Release`版本为啥还需要`SNAPSHOT`？

`区别`：

- 1.Release版本每次升级都需要更新版本，而SNAPSHOT不需要更新版本，使用原版本即可
- 2.Release版本如果版本没有更新不需要每次都去下载，除非本地仓库被清除，而SNAPSHOT版本每次编译都需要去中央仓库更新版本信息
- 3.常规Release版本是发布上线的版本，SNAPSHOT是测试版本。
- 4.版本名：Release版本：1.0.0，SNAPSHOT版本：1.0.0-SNAPSHOT

`使用场景`：
> A和B共同开发，如果A修改了代码，
> 使用常规Release版本则需要每次都发布一个新版本，如果不小心忘记增加版本，B则无法收到更新后的版本，
> 造成代码不同步，后期会出现不可预料的bug。
> 使用SNAPSHOT只要A发布了版本，B在每次编译时就可以立即收到A的类库更新信息，可以实时同步代码。

SNAPSHOT以`牺牲编译时间换取代码之间的立即可见度`，缺点就是在网络较差的情况下代码编译时间可能常常比较慢。

## 如何发布组件：
### 1.发布到本地仓库：

`模块级：build.gradle`

```java
plugins {
    id 'groovy' // Groovy Language
//    id 'org.jetbrains.kotlin.jvm' // Kotlin
    id 'java-gradle-plugin' // Java Gradle Plugin

    id 'maven'
}
...
uploadArchives {
    repositories {
        mavenDeployer {
            repository(url:uri('D:/maven_local'))
            pom.groupId = 'com.yuhb.upload'
            pom.artifactId = 'upload'
            pom.version = '1.0.1'
        }
    }
}

```

同步模块后：点击Gradle面板中`对应模块下Tasks：upload`里面的`uploadArchives`任务

![upload.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b4f3e68744d14d39b8743b8aad3849b5~tplv-k3u1fbpfcp-watermark.image?)
如果执行成功：则会在对应的本地目录下找到类库信息：

![本地文件成功.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aeed87111dbb48149fbef211227469c2~tplv-k3u1fbpfcp-watermark.image?)

### 2.搭建maven私服创建私有仓库：
#### 简介

`maven`私服其实就是在部门·局域网·中设置一个`maven`仓库，所有在局域网中的开发人员都可以使用该仓库：


![1.maven库介绍.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/710a4d4102a74afbb6dea31d815f2d47~tplv-k3u1fbpfcp-watermark.image?)

PS:私服中可以添加自己本地的仓库，也可以代理`中央仓库`中的包。毕竟对于一些网络比较差的环境，去中央仓库里面获取数据是一个很耗时的操作

#### 优点

**1.节省自己的外部带宽：**

**2.加速构建过程**

**3.可以部署第三方构件**

**4.提高稳定性，增强控制**

**5.降低中央仓库的负荷**


![maven私服优势.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc99d203312345ae9734ab91216be761~tplv-k3u1fbpfcp-watermark.image?)

#### 如何搭建Maven私服

##### 1.去`官网下载` maven私服启动器   `nexus`:

地址：[https://www.sonatype.com/]()

##### 2.下载后，配置`环境变量`后：

在命令行输入：`nexus /run`

##### 3.nexus启动成功后：在浏览器中输入：

```
http://localhost:8081/
```

-  启动界面如下：


![2.nexus界面.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1810ebb2071f48bf9ec81827f82d8ad5~tplv-k3u1fbpfcp-watermark.image?)

点击右上角的`sign in`按钮：

输入`用户名和密码`：

> 用户名和密码系统会提示在哪个目录下

##### 4.登录成功后：

-   点击导航栏的设置按钮-->repository进入仓库列表

![3.仓库搭建.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90c817c2a9224a09a96f3d3466d40983~tplv-k3u1fbpfcp-watermark.image?)


![4仓库.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9369419b60ac4ff18198fb865d102f38~tplv-k3u1fbpfcp-watermark.image?)


- 点击`create Repository`创建一个新的本地局域网仓库：

因为我们是为Android项目创建的maven仓库选择maven2：


![5maven仓库.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f42d527c17c44919be2ba9ec4af6df23~tplv-k3u1fbpfcp-watermark.image?)

这里有三个maven2类型仓库：

`*hosted*`:本地局域网私服，像官方仓库一样，提供本地私库功能

`*proxy*`：提供代理其他仓库的功能，表示这个repository可以代理远程仓库，比如`jcenter` `google` 等远程中央仓库

`*group*`：组合多个仓库为一个地址使用

> 这里我们选择`hosted`仓库即可，大家可以根据自己需求选取


![6maven.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/527965603bd648809b6f2395e7be139e~tplv-k3u1fbpfcp-watermark.image?)

**1.输入仓库名称**

**2.设置maven类型：**

`*release*`:表示是一个该仓库存储的是一个release版本的第三方库

`*snapshot*`：表示存储的是一个不稳定第三方库，需要进程去私服或者中央仓库拉数据：

```
<repositorys>
    <repository>
        <id>****</id>
        <url>***</url>
        <snapshots>
            <enabled>true</enabled>
            <updatePolicy>(always/ daliy/  interval/ never)</updatePolicy>
        </snapshot>
    </repository>
</repository>
```

`updatePolicy`:表示更新的频率：

        `always`：每次都需要拉去 `    daliy`：表示每天需要拉取 `    interval`：按分钟拉取 `    nerver`：和release版本一样，不需要重复拉取

点击确定后就创建的一个maven私服：

点击该仓库就可以看到对应的url，这个url就是我们私服的地址。需要在项目中引用：


![7maven.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa76e3ad7f9941ffa21df6b974bd5211~tplv-k3u1fbpfcp-watermark.image?)

经过上面的步骤后，我们就搭建好了一个`maven`私服，`局域网内`用户都可以拉取使用

#### 实战中maven私服接入：

##### 第三方lib库的上传操作

使用as创建一个新的项目，在新建项目下创建一个lib库，命名为:lib_vedio:并在`lib_vedio`的`build.gradle`工程文件下面，引入`maven`库:

实现`uploadArchives`的task方法：**这个方法用于上传lib库：**

```
plugins {
    id 'com.android.library'
    id 'maven'
}

//上传的类库名称
def _artifactId = this.getName() 
//类库版本
def pomVersionName = '1.0.1'
def pomName = this.getName()
//类库描述
def pomDescription = 'the vedio library for all project'

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url:NEXUS_REPOSITORY_URL) {
                authentication(userName: NEXUS_USERNAME, password: NEXUS_PASSWORD)
            }
            pom.project {
                name pomName
                version pomVersionName
                description pomDescription
                artifactId _artifactId
                groupId POM_GROUPID
                packaging POM_PACKAGING
            }
        }
    }
}
```

里面一些变量是在gradle.properties中设置的全局变量：

```
NEXUS_REPOSITORY_URL=http://localhost:8081/repository/anna_release/
POM_GROUPID=com.anna.android
POM_PACKAGING=aar
NEXUS_USERNAME=admin
NEXUS_PASSWORD=admin123456
```

设置好后，执行`uploadArchives`任务，这个时候我们本地的`lib_vedio`库就会上传到我们之前搭建的`maven`私服上：


![8maven.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87196436f29440109264e91b8d1958f6~tplv-k3u1fbpfcp-watermark.image?)

这里因为我们上传了两个版本`1.0.0和1.0.1`

**切记：**

> 如果当前仓库选择的是release属性：则每次上传的版本不能一致，需要在原版本基础上增加
>
> 如果选择的是snapshot属性:则可以在不增加版本号的前提下，修改同一版本的代码并上传。

##### maven私服仓库的接入：

我们在项目的`build.gradle`文件中引入maven库：

```
buildscript {
    repositories {
        google()
        jcenter()
        maven {
            url = 'http://localhost:8081/repository/anna_release/'
            credentials {
                username = 'admin'
                password = 'admin123456'
            }
        }

    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.1.1"

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        google()
        jcenter()
        maven {
            url = 'http://localhost:8081/repository/anna_release/'
            credentials {
                username = 'admin'
                password = 'admin123456'
            }
        }
    }
}
```

并在app模块引入lib库文件：

```
implementation 'com.anna.android:lib_video:1.0.1'
```

这样就可以在我们项目中愉快使用类库啦。

### 3.如何发布到Github仓库
如果我们需要开源我们的代码就需要将仓库发布到公共仓库中：

#### 步骤1：在`项目级build.gradle`中设置

```java
buildscript {
    repositories {
        maven { url 'https://jitpack.io' }
    }
    dependencies {
        classpath 'com.github.dcendents:android-maven-gradle-plugin:2.1'
    }
}

allprojects {
    repositories {
        maven { url 'https://jitpack.io' }
    }
}

```

#### 步骤2:在`模块级build.gradle`设置如下

```java
apply plugin: 'com.github.dcendents.android-maven'
group = 'com.github.xxx' // xxx为github 的用户名
```

#### 步骤3：将源码`push`到`github`上

#### 步骤4：在github上创建 `Release Tag`

![release tag.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/afac97b60ac14446a5f85db5244505fe~tplv-k3u1fbpfcp-watermark.image?)

#### 步骤5：打开https://jitpack.io 网址， 点击`look up`

![jitpack.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/030b09cccc884fa096e3bdc896881e8b~tplv-k3u1fbpfcp-watermark.image?)

然后会显示出对应的编译版本信息:
> 红色代表编译失败
>
> 蓝色代表编译成功
>
> 可通过日志查看编译错误原因：

![编译失败信息.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3abd6ded1f6a482881c63f698caabbc4~tplv-k3u1fbpfcp-watermark.image?)
#### 步骤6：类库引入：在`项目级 build.gradle` 声明远程仓库，在`模块级 build.gradle` 中依赖类库。

`项目级build.gradle`

```java
buildscript {
    repositories {
...
        maven { url 'https://jitpack.io' }
    }
    dependencies {
        ...
        classpath 'com.github.ByteYuhb.a_gradle_plugin_sample:uploadversion:1.0.3'
    }
}
```

**这里我们上传的是一个gradle插件库，所以需要在buildscript中的dependencies声明插件版本信息**

关于Gradle自定义插件的编写可以参考这篇文章:

[Gradle筑基篇(五)-Gradle自定义插件实战](https://juejin.cn/post/7130828529167515679/)

`模块级build.gradle`

```java
apply plugin: 'com.yuhb.upload'

versionInfo {
    versionName = '1.0.0'
    versionCode = 1
    versionUpdateInfo = '当前是第一个版本：初始apk'
}
引入插件的group和扩展信息
```


`结果`：
在编译期：打印出了Gradle插件中的信息：

```java
Task :app:uploadTask
name:1.0.0 code:1 info:当前是第一个版本：初始apk
```



### 4.指定发布二进制文件：
使用新版 `Maven 插件`，可以直接以指定二进制文件的方式发布组件。例如：

```java
apply plugin: 'maven-publish'

publishing {
    publications {
        [任务名](MavenPublication) {
            groupId MAVEN_GROUP_ID
            artifactId MAVEN_ARTIFACTID
            version MAVEN_VERSION
            artifact([文件路径])
        }
    }
    repositories {
        maven {
            // 发布仓库路径
            url MAVEN_RELEASE_URL
        }
    }
}
```


## 如何封装一个通用发布版本
### 步骤1：在项目根目录下创建：

```java
maven_publish.gradle
apply plugin: 'maven'

uploadArchives {
    repositories {
        mavenDeployer {
            // 是否快照版本
            def isSnapShot = Boolean.valueOf(MAVEN_IS_SNAPSHOT)
            def versionName = MAVEN_VERSION
            if (isSnapShot) {
                versionName += "-SNAPSHOT"
            }
            // 组件信息
            pom.groupId = MAVEN_GROUP_ID
            pom.artifactId = MAVEN_ARTIFACTID
            pom.version = versionName

            // 快照仓库路径
            snapshotRepository(url: uri(MAVEN_SNAPSHOT_URL)) {
                authentication(userName: MAVEN_USERNAME, password: MAVEN_USERNAME)
            }
            // 发布仓库路径
            repository(url: uri(MAVEN_RELEASE_URL)) {
                authentication(userName: MAVEN_USERNAME, password: MAVEN_USERNAME)
            }

            println("###################################"
                    + "\nuploadArchives = " + pom.groupId + ":" + pom.artifactId + ":" + pom.version + "." + pom.packaging
                    + "\nrepository =" + (isSnapshot ? MAVEN_SNAPSHOT_URL : MAVEN_RELEASE_URL)
                    + "\n###################################"
            )
        }
    }
}
```

> 这段脚本会读取 `MAVEN_IS_SNAPSHOT` 配置参数，如果为 true，会在版本号后追加 `-SNAPSHOT` 后缀，表示快照版本。随后声明了两个仓库：repository(...) 声明的是 Release 仓库地址，而 snapshotRepository(...) 声明的是快照仓库地址。Maven 会自动将版本号带 -SNAPSHOT 后缀的组件发布到 snapshotRepository(...) 仓库中，这样就 自动将正式版本和快照版本分发的不同仓库中。

### 步骤2：声明`项目级`gradle.properties配置参数：

```java
MAVEN_SNAPSHOT_URL = /Users/yuhb/workspace/public/DemoHall/snapshotRepository
MAVEN_RELEASE_URL = /Users/yuhb/workspace/public/DemoHall/releaseRepository
MAVEN_USERNAME = 
MAVEN_PASSWORD = 
MAVEN_IS_SNAPSHOT = true
MAVEN_GROUP_ID = com.yuhb.demo

```

| 参数描述             |                              |
| -------------------- | ---------------------------- |
| `MAVEN_SNAPSHOT_URL` | 快照仓库地址                 |
| `MAVEN_RELEASE_UR`   | 发布仓库地址                 |
| `MAVEN_USERNAME`     | 仓库账号                     |
| `MAVEN_PASSWORD`     | 仓库密码                     |
| `MAVEN_IS_SNAPSHOT`  | 是否快照版本                 |
| `MAVEN_GROUP_ID`     | 组织 / 公司的名称            |
| `MAVEN_ARTIFACTID`   | 组件的名称（在发布模块配置） |
| `MAVEN_VERSION`      | 组件的版本（在发布模块配置） |


### 步骤 3：在发布模块应用脚本

```java
apply from: '../maven.gradle'
...
```
### 步骤 4：在发布模块配置参数 （模块级配置参数会覆盖项目级配置参数）
`模块级 gradle.properties`
```java
MAVEN_ARTIFACTID = maven
MAVEN_VERSION = v1.0.0
MAVEN_IS_SNAPSHOT = true
...
```

### 然后在Gradle面板中找到对应模块下的任务uploadArchives，执行成功后就可以将组建发布对应的maven私服上了

## 本地aar文件引入：

`模块级build.gradle`

```java
dependencies {
    ...
    api(name: 'lib-debug', ext: 'aar')
}

repositories {
    flatDir {
        dirs "libs"
    }
}
```

这种方式如果其他模块需要使用就不方便：

`方法1`：

`在项目级build.gradle`

```java
allprojects {
    repositories {
        google()
        mavenCentral()
        flatDir { dirs project(':aarlib').file('libs') } // 文件夹要放在某个 module 内
    }
}
```

这种方式可以在本工程中使用，如果跨工程或者跨设备就不好使了

`方法2`：二次打包aar发布到maven仓库

```java
apply plugin: 'maven-publish'

def libPath = project.getProjectDir().getAbsolutePath()

publishing {
    publications {
        lib1(MavenPublication) {
            groupId MAVEN_GROUP_ID
            artifactId "lib"
            version "v1.0.0"
            artifact(libPath + "/libs/lib.aar")
        }

        lib2(MavenPublication) {
            groupId MAVEN_GROUP_ID
            artifactId "lib2"
            version "v1.0.0"
            artifact(libPath + "/libs/lib2.aar")
        }
    }
    repositories {
        maven {
            // 发布仓库路径
            url MAVEN_RELEASE_URL

            // 本地仓库地址不适用账号密码
            // > Failed to publish publication 'maven' to repository 'maven'
            // > Authentication scheme 'all'(Authentication) is not supported by protocol 'file'
            // credentials(PasswordCredentials) {
            //     username = MAVEN_USERNAME
            //     password = MAVEN_PASSWORD
            // }
        }
    }
}
```

## 参考资料
-   [Maven](https://link.juejin.cn/?target=https%3A%2F%2Fmaven.apache.org%2Fusers%2Findex.html "https://maven.apache.org/users/index.html")、[Maven Plugin](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.gradle.org%2F4.8%2Fuserguide%2Fmaven_plugin.html "https://docs.gradle.org/4.8/userguide/maven_plugin.html")、[Maven Publish Plugin](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fuserguide%2Fpublishing_maven.html "https://docs.gradle.org/current/userguide/publishing_maven.html")、[JitPack.io](https://link.juejin.cn/?target=https%3A%2F%2Fjitpack.io%2Fdocs%2F "https://jitpack.io/docs/") —— 官方文档

- https://juejin.cn/post/6963633839860088846 -胡飞洋