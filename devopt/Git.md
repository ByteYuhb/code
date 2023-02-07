> 🔥 **Hi，我是小余。** **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [[小余的自习室](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)] ，在成功的路上不迷路！**

## 前言



相信大家对Git已经比我还熟悉了，所以本篇文章不会大篇幅的去讲解Git中的各种命令，也没必要，去官网或者一些博客网站一搜一大堆，本文章主要来讲解Git中一个比较标准化的流程 **Git-Flow**。

以前笔者使用Git的代码管理是这样的：

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/%E4%BD%BF%E7%94%A8git-flow%E5%89%8D%E7%9A%84%E7%89%88%E6%9C%AC.png)



可以看到版本非常混乱，对后期版本维护是非常不利的，分支一多很容易出现之前发布过的版本找不到的情况，甚至引发一些bug，头大的。。
![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/006APoFYly1fuegbeqktmg3085085t9i.jpg)

作为一个小菜鸟，笔者去网上搜罗了一番，果然找到一个比较好的东西，**Git-FLow**。

什么是Git-Flow呢？**其实就是一个Git的标准控制流程或者说是一种约定**。

看下图：

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/git-model@2x.png)



**看不懂不要紧，下面我一一来分析。**

## 1.分支情况

此图中有5个分支类型：每个分支属性是不一样的

#### master分支：

这个不用介绍了吧，就是一个代码主线流程，注意哦，这个master分支**只用来存储已经发布的版本分支**：

> tag0.1，tag0.2表示使用tag来标记当前发布的版本，这样做的好处就是通过选中TAG就可以定位到某个发布版本，相信还有一些程序员是用commit的message来标记版本的情况，可以改改啦。

#### develop分支：

就是开发用的主线分支，如果项目比较小，直接在这里面开发是可以的，

但是对于一些需要多人协助异步开发的情况，这个分支呢就会是一个开发的主线分支，一些工具类以及api文档的编写可以在这上面commit，但是大部分的修改工作就要按feature进行分类了。

#### feature分支

假设我们有个开发专门做UI的，一个开发专门做NET的，一个开发专门做数据处理DATA的，为了项目顺利进展就会在develop分支上开辟三个feature分支：UI分支，NET分支，DATA分支。

> 假如哪个分支做好了，且单元测试没啥问题，就会将代码合并（使用merge或者rebase）到develop分支上，并隐藏当前feature分支。待三个分支都做好了，且都同步到develop分支上，develop分支在基础测试没问题后就会创建一个release分支。

#### release分支

release分支主要就是用来提供给客户的发布版本，因此对release版本还需要一个完整的测试，客户验收没问题后，再将release分支代码同步到master分支上，使用tag标记版本信息，并删除当前release分支以及前面创建的feature分支。这样整个需求通过细致的分工完成了，且版本统一在master分支上管理。

#### hotfix分支

什么是hotfix呢，言外之意就是热修复的意思，这里的热修复和Android中的热修复有点类似。

看图中：

> 假设当前版本已经到了1.0了，但是有一天客户说我机器出问题了，应用版本是0.1的，那么此时你可能需要在0.1的版本做修复（当然在最新版本修复也可以，这里说的是一定需要在旧版本做的修复），那么就需要使用到hotfix分支了。

我们checkout到master分支的tag0.1处，创建一个hotfix分支，然后巴拉巴拉在上面一通操作之后呢，将修复好bug的hotfix分支合并到develop分支以及master分支，注意哦。因为develop分支是开发主线，假设**使用hotfix修复了一个bug，不能仅仅同步到master，也要同步到develop分支**，因为后期新增业务还是在develop分支上进行。

**介绍完这几个分支，再回头去看这张图是不是好像顿悟了呢？**

好了，下面我来演示下如何在git中使用这套流程呢。

其实市面上大部分的git管理工具都集成了git-flow，如smartgit，fork等，下面小余使用fork来演示下git-flow的使用流程。

> fork是一款免费的git管理软件，而smartgit是一款收费软件，用哪款大家自己选吧，功能都差不多。

## 2. git-flow实战演练

- **1.首先我们在github上创建一个仓库**

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/github%E5%88%9B%E5%BB%BA%E4%BB%93%E5%BA%93.png)

- **2.使用fork工具将仓库clone到本地**

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/git-clone.png)



- **3.此时仓库中只有main分支，点击fork菜单栏的Repository->Git Flow->Initialize Git Flow..**

  ![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/011014554776_0gitflowinit.png)

  

  可能你要说了，这不就是给我们创建了一个develop分支么，额，确实是只创建了一个，别急，我们继续后面的步骤。

- 4.假设此时我们有两个分支任务，一个UI，一个NET。则点击fork菜单的**Repository->Git Flow->Start Feature**

  ![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/011015335024_0startfeature.png)

  

  **可以看到此时在feature下面创建了两个分支feature-UI和feature-NE**T。

- 5.**下面我们在两个feature分支下面加几个文件或者改点东西。**

  首先在NET分支创建src以及doc文件夹，并在文件夹内部创建了两个文件。然后切换到UI分支，同样创建src		以及doc文件夹，并在文件夹内部创建了两个文件。

  修改后的支线图如下：

  ![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/feature%E6%94%AF%E7%BA%BF%E5%9B%BE.png)

  可以看到UI以及NET两个分支分别作了两次修改，现在我**们的UI完成了，需要合并到develop分支**，如何进行呢？**鼠标右击选择Git Flow->Finish 'Feature/feature-ui'**，finish后feature-UI分支在branchs菜单栏就消失了，查看分支线图：**可知UI分支是被merge到main分支了**。

  ![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/011016013733_0finish-ui.png)

  用同样的方式，对NET分支做finish操作，但是此时你会发现出现conflict了。

  ![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/%E5%86%B2%E7%AA%811.png)

  > 这是因为由于UI分支的合并，此时develop分支的README.md文件被修改了，而在NET分支也对README.md文件进行了更新，所以产生了冲突，这是一个很经典的问题，如何解决呢？

- **6.使用merge工具对冲突文件进行合并**：

  ![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/conflict1.png)



最后冲突文件选择哪个分支的文件就看UI和NET程序员沟通后的结果啦，

> 实际在开发过程中，对于都要使用的文件可以在develop分支修改或者再使用一个feature分支修改大家都要用到的接口类，还有个办法就是使用Cherry pick，关于什么是Cherry pick自行百度看看吧。

另外如果是**组件化开发的项目就不会有这个冲突了**，因为组件化每个feature使用的文件都是隔离的。

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/conflict3.png)



这里假设选择了NET分支的README.md文件，然后对develop进行依次commit，最后finish掉feature-NET

分支。

**最终分支图如下**：

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/finishnet%E5%88%86%E6%94%AF%E5%9B%BE.png)



可以看到当前只剩下了develop分支，feature-UI以及feature-NET已经被合并到develop中了。

- 6.到这步我们就需要开始准备发布我们的版本啦，老操作，到**Git Flow中点击start Release**。

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/011016454972_0start-release.png)

release的名字最好起的能让人一眼就明白当前版本的信息：**release+版本号+日期**。

- 7.创建release分支之后，前release就是要发布的版本，如果测试下来发现有bug，则就在当前release版本上进行修改了，测试没问题后就可以将交付产物提交给客户了。此时develop分支依赖可以做自己的事情。和release分支是不冲突的。**只是提交之后需要将release分支上修改的内容同步到develop分支，防止修复的bug未在develop分支中修复**。



- 8.在成功交付给客户以后，**调用Git Flow的finish release删除当前release版本，并合并到主分支main上**

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/011016552709_0finish-release.png)

最后的分支图如下：

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/1%E4%B8%BB%E5%88%86%E6%94%AFmain.png)



> 可以看到这个支线图就很清晰啦，主线main上只有一个1.0的发布版本，且使用release-1.0这个Tag可以轻松定位当前版本所在位置。

### hotfix如何使用

假设此时我发布了两个版本1.0和2.0。客户说在1.0版本上出现了问题，那如何进行修复呢？

首先我们切换到main分支，然后checkout到1.0版本的tag，右击Git Flow->Start Hotfix.

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/011017530353_0start-hotfix.png)

切换到hotfix-net分支后，我们给hotfix中改写内容，然后commit，最后使用finish hotfix看看。

修改点：**README.md增加内容：**

```java
时间/日期：17：44/2023-01-10
改动点：
1.测试hotfix-net
```

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/finishhotfix.png)

最后分支图：

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/finishhotfix2.png)

可以看到，分支图在hotfix后，main主线上出现了第三个hotfix-net的tag，且最后的更改内容也成功移植到了main主线上，develop也同步了该TAG，说明此时的hotfix成功了。

小余在测试hotfix注意到：**虽然我们checkout到出现问题的1.0版本，但是实际创建的hotfix是在2.0版本为基础版本的，这个可能也是为了最新版本可以修复bug吧。**

## 总结

本篇文章主要讲解了关于Git-Flow在我们工作的使用方式，**GitFlow更多的应该是一种约定而不是工具**，

> 当然如果你只是一个小的项目，遵守一些简单的规则约定也就可以了，不一定要使用到Git_flow，对于一些稍微大点的项目可以考虑。毕竟学习成本也并不高。

好了，我是[小余](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)，欢迎点赞关注，我们下期见。

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/%E8%87%AA%E6%88%91%E4%BB%8B%E7%BB%8D.png)

**参考**：

https://nvie.com/posts/a-successful-git-branching-model/



