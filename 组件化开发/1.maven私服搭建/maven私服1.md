## 前言
组件化，我们都有一些了解，基本上对于一些大型互联网App来说，都会采用组件化来开发，也是面试中的重中之重，本专栏会通过将一个实战项目扩展开发讲解整个项目知识体系。

下面这幅图就是我们后面需要讲解的实战项目框架：

![组件化开发.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b0e2df764dc4e3ebbe6d75e9a748357~tplv-k3u1fbpfcp-watermark.image?)

今天我们先不讲解项目框架，我们先来创建一个maven私服，来为我们的组件化框架项目先铺一条路子。

## 简介

`maven`私服其实就是在部门·局域网·中设置一个`maven`仓库，所有在局域网中的开发人员都可以使用该仓库：


![1.maven库介绍.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/710a4d4102a74afbb6dea31d815f2d47~tplv-k3u1fbpfcp-watermark.image?)

PS:私服中可以添加自己本地的仓库，也可以代理`中央仓库`中的包。毕竟对于一些网络比较差的环境，去中央仓库里面获取数据是一个很耗时的操作

## 优点

**1.节省自己的外部带宽：**

**2.加速构建过程**

**3.可以部署第三方构件**

**4.提高稳定性，增强控制**

**5.降低中央仓库的负荷**


![maven私服优势.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc99d203312345ae9734ab91216be761~tplv-k3u1fbpfcp-watermark.image?)

### 如何搭建Maven私服

#### 1.去`官网下载` maven私服启动器   `nexus`:

地址：[https://www.sonatype.com/]()

#### 2.下载后，配置`环境变量`后：

在命令行输入：`nexus /run`

#### 3.nexus启动成功后：在浏览器中输入：

```
http://localhost:8081/
```

-   启动界面如下：


![2.nexus界面.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1810ebb2071f48bf9ec81827f82d8ad5~tplv-k3u1fbpfcp-watermark.image?)

点击右上角的`sign in`按钮：

输入`用户名和密码`：

> 用户名和密码系统会提示在哪个目录下

#### 4.登录成功后：

-   点击导航栏的设置按钮-->repository进入仓库列表

![3.仓库搭建.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90c817c2a9224a09a96f3d3466d40983~tplv-k3u1fbpfcp-watermark.image?)


![4仓库.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9369419b60ac4ff18198fb865d102f38~tplv-k3u1fbpfcp-watermark.image?)


-   点击`create Repository`创建一个新的本地局域网仓库：

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

        `always`：每次都需要拉去 `    daliy`：表示每天需要拉取 `    interval`：按分钟拉取 `    nerver`：和release版本一样，不需要重复拉取

点击确定后就创建的一个maven私服：

点击该仓库就可以看到对应的url，这个url就是我们私服的地址。需要在项目中引用：


![7maven.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa76e3ad7f9941ffa21df6b974bd5211~tplv-k3u1fbpfcp-watermark.image?)

经过上面的步骤后，我们就搭建好了一个`maven`私服，`局域网内`用户都可以拉取使用

### 实战中maven私服接入：

##### 1.第三方lib库的上传操作

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

## 总结
关于maven仓库就讲到这里了，后续会推出关于组件化开发各个组件的封装功能以及组件直接的前依赖关系的解耦也就是阿里开源框架aroute的使用，以及如何实现组件单独测试等问题。




