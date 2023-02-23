携手创作，共同成长！这是我参与「掘金日新计划 · 8 月更文挑战」的第6天，[点击查看活动详情](https://juejin.cn/post/7123120819437322247 "https://juejin.cn/post/7123120819437322247") >>
> 🔥 **Hi，我是小余。**
>
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [小余的自习室] ，在成功的路上不迷路！**
## 前言
前面几篇文章我们讲解了关于关于`Gradle的基础`，`Gradle生命周期`，`Gradle相关Api`的讲解，以及`Gradle自定义插件`,`Gradle Maven仓库管理`.

Gradle系列文章如下：
> Gradle筑基篇：
- [Gradle筑基篇(一)-Gradle初探](https://juejin.cn/post/7129780699145437215)

- [Gradle筑基篇(二)-Groovy语法的详解](https://juejin.cn/post/7129336080112812039)

- Gradle筑基篇(三)-Gradle生命周期

- [Gradle筑基篇(四)-Gradle APi详解](https://juejin.cn/post/7130177944189665310/)

- [Gradle筑基篇(五)-Gradle自定义插件实战](https://juejin.cn/post/7130828529167515679/)

- [Gradle筑基篇(六)-Gradle Maven仓库管理](https://juejin.cn/post/7131401057699102728)

> Gradle进阶篇
- [Gradle进阶篇(七)-AGP详解](https://juejin.cn/post/7132465108936032293/)

今天这篇文章我们来讲解下`Android Gradle Plugin`相关知识。
> 简化起见：本文所指`AGP`：Android Gradle Plugin

## 1.`Gradle Plugin`和`AGP`的区别？
`Gradle Plugin`是`Gradle`构建过程中使用的插件的总称，而`Android Gradle Plugin`是这个总称里面的一个插件元素.

![agp插件和gp插件区别.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a00845c5b2284efdaff90ff9fd347ac6~tplv-k3u1fbpfcp-watermark.image?)

`Android Gradle Plugin`配合`Gradle`构建我们的应用apk

## 2.apk构建流程
典型 Android 应用模块的构建流程。

![build-process_2x.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/877130633fb144a9854f3c5572a1862d~tplv-k3u1fbpfcp-watermark.image?)

按照以下常规步骤执行：
- 1.将源文件和class文件编译组合后编译为dex文件
- 2.将资源文件转换为编译后的资源文件
- 3.将dex文件和编译后的资源文件打包为apk文件
- 4.使用签名工具对文件进行签名
- 5.生成最终apk之前，会使用 zipalign 工具对应用进行优化，减小apk运行时内存

在Gradle控制面板：执行`assemble`任务看看：

```java
Line 172: > Task :application:preBuild UP-TO-DATE //编译预处理任务：空实现
	Line 176: > Task :application:preF1F3DebugBuild UP-TO-DATE //preF1F3DebugBuild F1F3变体预处理任务
	Line 180: > Task :application:compileF1F3DebugAidl NO-SOURCE //编译aidl文件
	Line 184: > Task :application:compileF1F3DebugRenderscript NO-SOURCE //编译绘制脚本文件
	Line 188: > Task :application:dataBindingMergeDependencyArtifactsF1F3Debug UP-TO-DATE //dataBinding依赖的类库或者插件合并
	Line 192: > Task :application:dataBindingMergeGenClassesF1F3Debug UP-TO-DATE //dataBinding依赖的class文件合并
	Line 196: > Task :application:generateF1F3DebugResValues UP-TO-DATE //生成ResValues 
	Line 200: > Task :application:generateF1F3DebugResources UP-TO-DATE//生成编译后的Resources
	Line 204: > Task :application:mergeF1F3DebugResources UP-TO-DATE //合并资源文件
	Line 208: > Task :application:dataBindingGenBaseClassesF1F3Debug UP-TO-DATE
	Line 212: > Task :application:dataBindingTriggerF1F3Debug UP-TO-DATE
	Line 216: > Task :application:generateF1F3DebugBuildConfig UP-TO-DATE 生成BuildConfig文件
	Line 220: > Task :application:javaPreCompileF1F3Debug UP-TO-DATE //java预编译
	Line 224: > Task :application:checkF1F3DebugAarMetadata UP-TO-DATE  //检测aar的元数据
	Line 228: > Task :application:createF1F3DebugCompatibleScreenManifests UP-TO-DATE 
	Line 232: > Task :application:extractDeepLinksF1F3Debug UP-TO-DATE
	Line 236: > Task :application:processF1F3DebugMainManifest UP-TO-DATE //处理MainManifest
	Line 240: > Task :application:processF1F3DebugManifest UP-TO-DATE //处理Manifest
	Line 244: > Task :application:processF1F3DebugManifestForPackage UP-TO-DATE//处理ManifestForPackage 
	Line 248: > Task :application:processF1F3DebugResources UP-TO-DATE//处理Resources
	Line 252: > Task :application:compileF1F3DebugJavaWithJavac UP-TO-DATE //编译原代码为class文件
	Line 256: > Task :application:compileF1F3DebugSources UP-TO-DATE//编译Sources
	Line 260: > Task :application:mergeF1F3DebugNativeDebugMetadata NO-SOURCE
	Line 264: > Task :application:mergeF1F3DebugShaders UP-TO-DATE
	Line 268: > Task :application:compileF1F3DebugShaders NO-SOURCE
	Line 272: > Task :application:generateF1F3DebugAssets UP-TO-DATE //生成Assets
	Line 276: > Task :application:mergeF1F3DebugAssets UP-TO-DATE //合并Assets
	Line 280: > Task :application:compressF1F3DebugAssets UP-TO-DATE //压缩Assets
	Line 284: > Task :application:checkF1F3DebugDuplicateClasses UP-TO-DATE //检测DuplicateClasses
	Line 288: > Task :application:desugarF1F3DebugFileDependencies UP-TO-DATE
	Line 292: > Task :application:mergeExtDexF1F3Debug UP-TO-DATE //合并des
	Line 296: > Task :application:processF1F3DebugJavaRes NO-SOURCE //处理JavaRes
	Line 300: > Task :application:mergeF1F3DebugJavaResource UP-TO-DATE //合并JavaResource
	Line 304: > Task :application:mergeLibDexF1F3Debug UP-TO-DATE //合并lib的dex
	Line 308: > Task :application:dexBuilderF1F3Debug UP-TO-DATE //dexBuilder
	Line 312: > Task :application:mergeProjectDexF1F3Debug UP-TO-DATE//mergeProjectDex
	Line 316: > Task :application:mergeF1F3DebugJniLibFolders UP-TO-DATE//合并JniLibFolders
	Line 320: > Task :application:mergeF1F3DebugNativeLibs UP-TO-DATE//合并NativeLibs
	Line 324: > Task :application:stripF1F3DebugDebugSymbols NO-SOURCE
	Line 328: > Task :application:validateSigningF1F3Debug UP-TO-DATE //检测签名
	Line 332: > Task :application:packageF1F3Debug UP-TO-DATE //打包变种
	Line 336: > Task :application:assembleF1F3Debug UP-TO-DATE//打包变种
```


可以看到打包apk的任务基本和前面图片里面描述的流程一致，整个过程都是编译然后合并，打包的过程

`主要涉及`：
> - 1.资源文件。
> - 2.源文件。
> - 3.库文件的资源 
> - 4.库文件的class文件，
> - 5.jni的静动态库信息，
> - 6.manfest清单文件的创建
> - 7.签名校验等 其他生成一些配置文件

## 3.`AGP`常用设置类型:

- 1.`buildTypes`:编译类型：是debug或者release或者其他自定义类型

```java
android {
    defaultConfig {
        manifestPlaceholders = [hostName:"www.example.com"]
        ...
    }
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }

        debug {
            applicationIdSuffix ".debug"
            debuggable true
        }

        /**
         * The `initWith` property allows you to copy configurations from other build types,
         * then configure only the settings you want to change. This one copies the debug build
         * type, and then changes the manifest placeholder and application ID.
         */
        staging {
            initWith debug
            manifestPlaceholders = [hostName:"internal.example.com"]
            applicationIdSuffix ".debugStaging"
        }
    }
}
```

- 2.`productFlavor`：产品变种

创建产品变种与创建 `build` 类型类似：将其添加到 build 配置中的 productFlavors 代码块并添加所需的设置。
产品变种支持与 defaultConfig 相同的属性，这是因为，defaultConfig 实际上属于 ProductFlavor 类。

这意味着，您可以在 defaultConfig 代码块中提供所有变种的基本配置，每个变种均可更改其中任何默认值，如 applicationId

```java
// Specifies one flavor dimension.
flavorDimensions 'abi','version'
productFlavors {
	f1 {
		dimension 'abi'
		versionName '1.0'
	}
	f2 {
		dimension 'abi'
		versionName '2.0'
	}
	f3 {
		dimension 'version'
	}
	f4 {
		dimension 'version'
	}
}
```
对应的变体：
![变体类型.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af1b0eb8bc34422787ce51893a597137~tplv-k3u1fbpfcp-watermark.image?)
- 3.**build 变体**

build 变体是 build 类型与产品变种的交叉产物，也是 Gradle 用来构建应用的配置
如上面的类型，编译时可选变体类型：

- 4.清单 (`Manifest`) 条目

在配置清单中可以设置Manifest清单中给的配置信息，如

```java
applicationId "com.example.myapp"
minSdkVersion 15
targetSdkVersion 24
versionCode 1
versionName "1.0"
```

这些信息可以单独配置在不同给的变种中：如上面的类型，编译时可选变体类型：
这样可以针对不同变体设置不同的清单Manifest信息：

```java
productFlavors {
    f1 {
        dimension 'abi'
        applicationId "com.example.myapp"
        minSdkVersion 15
        targetSdkVersion 24
        versionCode 1
        versionName "1.0"
    }
    f2 {
        dimension 'x86'
        applicationId "com.example.myapp1"
        minSdkVersion 16
        targetSdkVersion 25
        versionCode 2
        versionName "2.0"
    }
}
```

配置清单中信息会覆盖原Manifest文件中的信息,当有多个清单配置时会合并
合并工具会根据每个清单文件的优先级按顺序合并，将所有清单文件组合到一个文件中。
例如，如果您有三个清单文件，则会先将优先级最低的清单合并到优先级第二高的清单中，
然后再将合并后的清单合并到优先级最高的清单中，如图：

![manifest-merger_2x.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1f711e64e2754cbe8936cc9832cbefa4~tplv-k3u1fbpfcp-watermark.image?)
- 5.`sourceSets`：原文件文件目录

```java
sourceSets {
	main {
		java {
			srcDirs = ['src/main/java']
		}
		res {
			srcDirs = ['src/main/res']
		}
		aidl {
			srcDirs = ['src/main/aidl']
		}
	}
}
```
- 6.`signingConfigs`:签名

Android 系统要求所有 APK 必须先使用证书进行数字签名，然后才能安装到设备上或进行更新

```java
signingConfigs {
    release {
        storeFile file("myreleasekey.keystore")
        storePassword "password"
        keyAlias "MyReleaseKey"
        keyPassword "password"
    }
}
```

- 7.`compileOptions`:指定当前编译的java版本信息

```java
compileOptions {
	sourceCompatibility JavaVersion.VERSION_1_8
	targetCompatibility JavaVersion.VERSION_1_8
}
```
- 7.`buildFeatures`：编译特色属性

```java
buildFeatures {
	aidl = true
	buildConfig = true
	viewBinding = false
	dataBinding = true
}
//这个方式已经被弃用，后面源码可以看到弃用的地方
dataBinding {
	enabled = true
}

```
以上就是我们使用AGP时常用的插件配置项

下面我们从源码去看下`AGP插件`内部原理。

## 4.`AGP插件`内部原理
### 1.源码查看方法
由于AGP插件源码大概有30多个g。所以不建议直接下载源码去阅读
可以直接在模块中引入就可以：
- 1.创建一个lib模块：
- 2.修改build.gradle中的代码：

```java
apply plugin: 'java'
sourceCompatibility = 1.8
dependencies {
    implementation gradleApi()
    implementation 'com.android.tools.build:gradle:4.1.1'
}
```
同步代码后：可以在‘`External Libraries`’中查看源码：

### 2.查看源码
前面在讲解`Gradle自定义插件`的时候，说过，我们使用的每个插件都会在`resources`中进行声明：

**全局搜索：**

![appplugin属性声明.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b758fd869b34cb7adebcf10c04c6513~tplv-k3u1fbpfcp-watermark.image?)
```java
找到implementation-class=com.android.build.gradle.AppPlugin
```

进入AppPlugin看看：

```java
/**
 * The plugin applied with `com.android.application'
 */
@Suppress("DEPRECATION")
class AppPlugin: BasePlugin() {
    override fun apply(project: Project) {
        super.apply(project)

        project.apply(INTERNAL_PLUGIN_ID)
    }
}

private val INTERNAL_PLUGIN_ID = mapOf("plugin" to "com.android.internal.application")

```

看到这里使用的`INTERNAL_PLUGIN_ID`中的`plugin`：com.android.internal.application

我们再次全局搜索下：`com.android.internal.application`


```java
找到implementation-class=com.android.build.gradle.internal.plugins.AppPlugin
进入：com.android.build.gradle.internal.plugins.AppPlugin
```
`gradle`源码：

![gradle源码.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/834e8bbe64144f359fbbede75670da75~tplv-k3u1fbpfcp-watermark.image?)
**查找apply方法：**

在父类`AbstractAppPlugin`的父类`BasePlugin`找到了`apply`方法：

```java
public final void apply(@NonNull Project project) {
	CrashReporting.runAction(
			() -> {
				//方法1
				basePluginApply(project);
				//方法2
				pluginSpecificApply(project);
			});
}
```

这里我们看`方法1`：

```java
private void basePluginApply(@NonNull Project project) {
	// We run by default in headless mode, so the JVM doesn't steal focus.
	System.setProperty("java.awt.headless", "true");

	this.project = project;
	//创建Project运行需要的服务信息
	createProjectServices(project);
	//获取Project的属性Options 
	ProjectOptions projectOptions = projectServices.getProjectOptions();
	//依赖检测
	DependencyResolutionChecks.registerDependencyCheck(project, projectOptions);
	//AndroidBasePlugin内部是一个空实现，需要我们自己去扩展。
	project.getPluginManager().apply(AndroidBasePlugin.class);
	//检测文件路径
	checkPathForErrors();
	//检测模块路径
	checkModulesForErrors();

	AttributionListenerInitializer.INSTANCE.init(
			project, projectOptions.get(StringOption.IDE_ATTRIBUTION_FILE_LOCATION));
	//agp的版本检测
	AgpVersionChecker.enforceTheSamePluginVersions(project);

	RecordingBuildListener buildListener = ProfilerInitializer.init(project, projectOptions);
	//注册buildListener构建的监听逻辑
	ProfileAgent.INSTANCE.register(project.getName(), buildListener);
	threadRecorder = ThreadRecorder.get();

	ProcessProfileWriter.getProject(project.getPath())
			.setAndroidPluginVersion(Version.ANDROID_GRADLE_PLUGIN_VERSION)
			.setAndroidPlugin(getAnalyticsPluginType())
			.setPluginGeneration(GradleBuildProject.PluginGeneration.FIRST)
			.setOptions(AnalyticsUtil.toProto(projectOptions));

	/**
	Gradle构建生命周期中的Agp插件的配置流程：
	1.configureProject：构建project配置、
	2.configureExtension：配置外部Extension字段
	3.createTasks：创建Tasks
	*/
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
}
```

**方法1主要任务：**
- 做一些前期的检查工作，设置一些生命周期回调监听，创建任务等操作。
- Gradle构建生命周期中的Agp插件的配置流程：
> 1.`configureProject`：构建project配置、
> 2.`configureExtension`：配置外部Extension字段
> 3.`createTasks`：创建Tasks

#### 看第一阶段：configureProject

```java
private void configureProject() {
        final Gradle gradle = project.getGradle();
		
		//创建缓存版本变化的原因的字符串
        Provider<ConstraintHandler.CachedStringBuildService> cachedStringBuildServiceProvider =
                new ConstraintHandler.CachedStringBuildService.RegistrationAction(project)
                        .execute();
		///** Build service used to cache maven coordinates for libraries. */
        Provider<MavenCoordinatesCacheBuildService> mavenCoordinatesCacheBuildService =
                new MavenCoordinatesCacheBuildService.RegistrationAction(
                                project, cachedStringBuildServiceProvider)
                        .execute();

        new LibraryDependencyCacheBuildService.RegistrationAction(project).execute();

        extraModelInfo = new ExtraModelInfo(mavenCoordinatesCacheBuildService);

        ProjectOptions projectOptions = projectServices.getProjectOptions();
        IssueReporter issueReporter = projectServices.getIssueReporter();
		
		//创建Aapt2工作的构建Service
        new Aapt2WorkersBuildService.RegistrationAction(project, projectOptions).execute();
		//创建一个Aapt2 Daemon的构建Service
        new Aapt2DaemonBuildService.RegistrationAction(project).execute();
        new SyncIssueReporterImpl.GlobalSyncIssueService.RegistrationAction(
                        project, SyncOptions.getModelQueryMode(projectOptions))
                .execute();
        Provider<SdkComponentsBuildService> sdkComponentsBuildService =
                new SdkComponentsBuildService.RegistrationAction(
                                project,
                                projectOptions,
                                project.getProviders()
                                        .provider(() -> extension.getCompileSdkVersion()),
                                project.getProviders()
                                        .provider(() -> extension.getBuildToolsRevision()),
                                project.getProviders().provider(() -> extension.getNdkVersion()),
                                project.getProviders().provider(() -> extension.getNdkPath()))
                        .execute();

        projectOptions
                .getAllOptions()
                .forEach(projectServices.getDeprecationReporter()::reportOptionIssuesIfAny);
        IncompatibleProjectOptionsReporter.check(
                projectOptions, projectServices.getIssueReporter());

        // Enforce minimum versions of certain plugins
        GradlePluginUtils.enforceMinimumVersionsOfPlugins(project, issueReporter);

        // Apply the Java plugin
        project.getPlugins().apply(JavaBasePlugin.class);

        dslServices =
                new DslServicesImpl(
                        projectServices,
                        new DslVariableFactory(syncIssueReporter),
                        sdkComponentsBuildService);

        MessageReceiverImpl messageReceiver =
                new MessageReceiverImpl(
                        SyncOptions.getErrorFormatMode(projectOptions),
                        projectServices.getLogger());
		//将前面创建的所有服务放到globalScope对象中
        globalScope =
                new GlobalScope(
                        project,
                        creator,
                        dslServices,
                        sdkComponentsBuildService,
                        registry,
                        messageReceiver,
                        componentFactory);

        project.getTasks()
                .named("assemble")
                .configure(
                        task ->
                                task.setDescription(
                                        "Assembles all variants of all applications and secondary packages."));

        // As soon as project is evaluated we can clear the shared state for deprecation reporting.
        gradle.projectsEvaluated(action -> DeprecationReporterImpl.Companion.clean());

        createLintClasspathConfiguration(project);
    }
```


- 这个方法主要是创建了一系列构建需要的服务，并将服务放到一个`globalScope`对象中，也算是一些前期准备工作。

#### 来看阶段2：`configureExtension`

```java
private void configureExtension() {
		//获取dsl服务
        DslServices dslServices = globalScope.getDslServices();
		
		//获取构建输出
        final NamedDomainObjectContainer<BaseVariantOutput> buildOutputs =
                project.container(BaseVariantOutput.class);
		
        project.getExtensions().add("buildOutputs", buildOutputs);
		
		//创建变体的工厂类
        variantFactory = createVariantFactory(projectServices, globalScope);
		//将前面创建的几个对象都放入到variantInputModel对象中：
        variantInputModel =
                new LegacyVariantInputManager(
                        dslServices,
                        variantFactory.getVariantType(),
                        new SourceSetManager(
                                project,
                                isPackagePublished(),
                                dslServices,
                                new DelayedActionsExecutor()));
		//这里创建了外部扩展Extension，进入到这个方法看看
        extension =
                createExtension(
                        dslServices, globalScope, variantInputModel, buildOutputs, extraModelInfo);

        globalScope.setExtension(extension);

        variantManager =
                new VariantManager<>(
                        globalScope,
                        project,
                        projectServices.getProjectOptions(),
                        extension,
                        variantFactory,
                        variantInputModel,
                        projectServices,
                        threadRecorder);

        registerModels(
                registry,
                globalScope,
                variantInputModel,
                extension,
                extraModelInfo);

        // create default Objects, signingConfig first as its used by the BuildTypes.
        variantFactory.createDefaultComponents(variantInputModel);

        createAndroidTestUtilConfiguration();
    }
```
进入`createExtension`方法：

这个是抽象方法：具体实现是在子类AppPlugin中：

```java
protected AppExtension createExtension(
		@NonNull DslServices dslServices,
		@NonNull GlobalScope globalScope,
		@NonNull
				DslContainerProvider<DefaultConfig, BuildType, ProductFlavor, SigningConfig>
						dslContainers,
		@NonNull NamedDomainObjectContainer<BaseVariantOutput> buildOutputs,
		@NonNull ExtraModelInfo extraModelInfo) {
	return project.getExtensions()
			.create(
					"android",
					getExtensionClass(),
					dslServices,
					globalScope,
					buildOutputs,
					dslContainers.getSourceSetManager(),
					extraModelInfo,
					new ApplicationExtensionImpl(dslServices, 
					));
}
```

- 可以看到这里创建了一个‘`android`’的`Extensions`，bean类型为：`BaseAppModuleExtension`

这个`BaseAppModuleExtension`类就是我们可以在外部做扩展的起始点，这个类中有哪些属性包括父类中的属性都可以被扩展：
进到这个类中看看：

```java
/** The `android` extension for base feature module (application plugin).  */
open class BaseAppModuleExtension(...private val publicExtensionImpl: ApplicationExtensionImpl)
:AppExtension:InternalApplicationExtension by publicExtensionImpl,。。

```
看到该类继承了AppExtension和InternalApplicationExtension(被ApplicationExtensionImpl代理)

AppExtension又继承AbstractAppExtension继承TestedExtension继承BaseExtension
下面画出对应的类图关系

![Extentions扩展.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/40fe1d2ff5f446128fa4c6b2a71f6171~tplv-k3u1fbpfcp-watermark.image?)
这几个类组成了AGP所有的对外扩展属性

这里列举几个平时项目中开发用到的字段：

- 1.在`ApplicationExtensionImpl`中：

`buildFeatures`属性：

```java
public open val buildFeatures:com.android.build.api.dsl.ApplicationBuildFeatures
```
查看ApplicationBuildFeatures中的代码：

内部就两条属性：

```java
//用于dataBinding支持
var dataBinding: Boolean?
var mlModelBinding: Boolean?
```
ApplicationBuildFeatures 继承 `BuildFeatures`
进入`BuildFeatures` 看看:

![buidlFeatures.jt.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12d65c789f92479a8e96275e750cc5cd~tplv-k3u1fbpfcp-watermark.image?)
可以看到这里面可以配置很多信息：
```java
buildFeatures {
	aidl = true
	buildConfig = true
	viewBinding = false
	dataBinding = true
	....
}
```


- 2.在AbstractAppExtension中

```java
val applicationVariants: DomainObjectSet<ApplicationVariant> =
        dslServices.domainObjectSet(ApplicationVariant::class.java)
```


这条属性，可以让我们在外部获取到当前所有变种：
`DomainObjectSet`是一个集合，可以使用集合遍历方式：

```java
android.applicationVariants.all {variant->
	variant.outputs.all {output->
		outputFileName ="landi_dev_v${variant.versionName}_${variant.name}"+".apk"
	}
}
```

相信这个代码大家在自己项目中或多或少都见过
通过前面分析我们就知道：

```java
variant = ApplicationVariant类对象
output = BaseVariantOutput类对象
```
关于`configureExtension`就介绍到这

#### 第3阶段：`createTasks`

```java
private void createTasks() {
        threadRecorder.record(
                ExecutionType.TASK_MANAGER_CREATE_TASKS,
                project.getPath(),
                null,
                () ->
                        TaskManager.createTasksBeforeEvaluate(
                                globalScope,
                                variantFactory.getVariantType(),
                                extension.getSourceSets()));

        project.afterEvaluate(
                CrashReporting.afterEvaluate(
                        p -> {
                            variantInputModel.getSourceSetManager().runBuildableArtifactsActions();

                            threadRecorder.record(
                                    ExecutionType.BASE_PLUGIN_CREATE_ANDROID_TASKS,
                                    project.getPath(),
                                    null,
                                    this::createAndroidTasks);
                        }));
    }
```


- 1.BeforeEvaluate：配置前创建一些task
- 2.afterEvaluate:配置后创建一些task：进入createAndroidTasks看看：

```java
 final void createAndroidTasks() {
		//使用variantManager创建变体 关注点1.。。
		variantManager.createVariants();
		//创建TaskManager 关注点2.。。
		TaskManager<VariantT, VariantPropertiesT> taskManager =
                createTaskManager(
                        variants,
                        variantManager.getTestComponents(),
                        !variantInputModel.getProductFlavors().isEmpty(),
                        globalScope,
                        extension,
                        threadRecorder);
		//使用TaskManager创建任务：关注点3.。。
        taskManager.createTasks();
		//这里设置Transforms的任务，如果外部有
		new DependencyConfigurator(project, project.getName(), globalScope, variantInputModel)
			.configureDependencySubstitutions()
			.configureGeneralTransforms()
			.configureVariantTransforms(variants, variantManager.getTestComponents());
		for (ComponentInfo<VariantT, VariantPropertiesT> variant : variants) {
            apiObjectFactory.create(variant.getProperties());
        }
		// lock the Properties of the variant API after the old API because
        // of the versionCode/versionName properties that are shared between the old and new APIs.
        variantManager.lockVariantProperties();

        // Make sure no SourceSets were added through the DSL without being properly configured
        variantInputModel.getSourceSetManager().checkForUnconfiguredSourceSets();

        // configure compose related tasks.
        taskManager.createPostApiTasks();

        ...
        GradleProperty.Companion.endOfEvaluation();
	 
	 }
	 看关注点1：variantManager.createVariants();
	 /** Creates the variants. */
    public void createVariants() {
        variantFactory.validateModel(variantInputModel);
        variantFactory.preVariantWork(project);

        computeVariants();
    }
	进入computeVariants：
	 /** Create all variants. */
    private void computeVariants() {
		//获取所有flavorDimension放到List中
        List<String> flavorDimensionList = extension.getFlavorDimensionList();
		
		//获取所有buildTypes和productFlavor信息：
        DimensionCombinator computer =
                new DimensionCombinator(
                        variantInputModel,
                        projectServices.getIssueReporter(),
                        flavorDimensionList);
		//计算所有的变体
        List<DimensionCombination> variants = computer.computeVariants();
		//锁定变体
        variantApiServices.lockValues();
    }
```


关注点1主要作用就是获取所有的产品变体和`buildTypes`，创建对应的变体列表
	
关注点2：createTaskManager

抽象方法在子类`AppPlugin`中实现：

```java
protected ApplicationTaskManager createTaskManager(
            @NonNull
                    List<ComponentInfo<ApplicationVariantImpl, ApplicationVariantPropertiesImpl>>
                            variants,
            @NonNull
                    List<
                                    ComponentInfo<
                                            TestComponentImpl<
                                                    ? extends TestComponentPropertiesImpl>,
                                            TestComponentPropertiesImpl>>
                            testComponents,
            boolean hasFlavors,
            @NonNull GlobalScope globalScope,
            @NonNull BaseExtension extension,
            @NonNull Recorder threadRecorder) {
			return new ApplicationTaskManager(
                variants, testComponents, hasFlavors, globalScope, extension, threadRecorder);
		}
		}
```


创建的是一个`ApplicationTaskManager`类对象，后面会用到
	
来看关注点3：

这里面其实就是创建一系列的task,篇幅太长，不一一介绍
	
阶段三:`createTasks`主要任务：
- 1.根据`Extensions`中的信息创建所有变体，
- 2.给所有变体创建对应的任务：包括配置前任务和配置后的任务

总结我们的`AGP`主要任务：
> - 1.创建了一系列构建需要的服务，并将服务放到一个`globalScope`对象中，作为前期准备工作
> - 2.解析我们的外部扩展android{}闭包中的信息，设置到Project中
> - 3.根据buidTypes和产品变种创建对应的变种信息，创建一系列的构建任务。


## 参考
- [Android构建官方文档](https://developer.android.google.cn/studio/build)

- [补齐Android技能树——从AGP构建过程到APK打包过程](https://juejin.cn/post/6963527524609425415)


​	

​	