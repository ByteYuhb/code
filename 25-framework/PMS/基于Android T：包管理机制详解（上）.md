> 🔥 **Hi，我是小余。**
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [[小余的自习室](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)]，在成功的路上不迷路！**
## 前言
**PackageManagerService**（简称PKMS）是Android系统核心服务之一，和AMS，WMS，IMS并列”**Android四大金刚服务**“，其管理着整个Android应用的安装更新和卸载等操作。

PKMS在我们开发中经常会碰到，了解其底层原理对我们开发也是很有帮助的，比如**包体积优化，应用启动优化等**。

注：本文源码全部**基于Android T**

## 目录

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/Androidpmsmulu.png)
## 0.Framework包管理框架

**Android包管理**主要体现在以下几个部分：

- **1.系统启动过程中PKMS对系部分统配置文件进行读取，如package.xml文件,然后对外提供app信息查询接口（IPackageManager）**.
- **2.提供apk/apex的安装，更新，卸载等操作api接口（IPackageInstaller）,apex是谷歌提供的类似apk的系统更新模块。**
- **3.应用运行过程中对系统权限的检查**



Android Framework包管理框架（基于**Android T**）如下图:

![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/packageframwork.png)

基于Android T的Android包管理机制主要分为以下三层：

#### 1.应用层

应用层需要获取某个安装包信息或者安装应用时，需要获取PKMS的实例，PKMS是在系统启动的时候注册到ServiceManager中。应用通过调用ContextImpl的getPackageManager接口获取PKMS的实例，**实际返回的是一个ApplicationPackageManager对象**，这个对象实现了PackageManager抽象接口且内部有个IPackageManager对象是远程接口IPackageManager的代理对象，通过这个对象可以访问PKMS层的接口。

IPackageManager对象有个方法getPackageInstaller可以返回远程接口IPackageInstaller的实例对象PackageInstallerService。**PackageInstallerService对象可以用于应用的安装，卸载，更新等操作**。

#### **2.PKMS服务层**

和AMS，WMS一样，也是在SystemServer进程启动过程中启动的，PKMS层可以被应用层调用，返回对应的app信息。其内部有三个文件目录保存相关包管理信息。

- **1./data/system/package.list**
- **2./system/etc/permissions**
- **3./data/system/packages.xml** 

**PKMS在启动的时候会从这三个目录中解析相关的XML文件，从而建立庞大的信息树，应用程序可以间接的从这个信息树种查询所需的程序包信息**。下面依次来看这三个文件结构：

**1.data/system/package.list**

**用于记录系统中所有的应用的一些基本信息，包括应用名称，uid，gid，数据存放路径等信息**
下面是8.0设备上的package.list文件结构：每个版本都大同小异，差别不是很大。

```
com.google.android.youtube 10076 0 /data/user/0/com.google.android.youtube default:targetSdkVersion=26 3003
```

每行格式如下：

- 1.`com.google.android.youtube `//app的包名,也就是AndroidManifest.xml文件中的package=”xxx.xxx.xxx”设置的内容
- 2.`10076`//uid 如果没有在AndroidManifext.xml里使用android:sharedUserId属性指定UID, 在app安装的时候，系统会给这个app自动分配一uid，以后app运行时，就用这个UID运行
- 3.`0 `//debugable app是否处于调试模式，由AndroidManifext.xml里android:debuggable指定
- 4.`/data/user/0/com.google.android.youtube` //app的数据存放路径，一般是”/data/data/${package_name}”这样的文件夹 /data/user/0指向/data/data/
- 5.`default:targetSdkVersion=26` //默认的targetSdkVersion
- 6.`3003` //app所属的user group

**2./system/etc/permissions**

该目录下的文件**主要用于权限的管理**，包含两件事：1.定义系统中包含了哪些feature，应用程序可以在AndroidManifest.xml中使用标签声明程序需要哪些feature。
2.该目录还有一个还有一个platform.xml文件，该文件为一些特别uid和gid分配一些默认权限，给uid分配权限使用标签，给gid分配权限使用标签。

**3.data/system/packages.xml**

文件中记录了系统的所有权限信息，系统中所有安装应用的基本信息，系统中所有shared-user信息以及应用打包时的签名信息等。packages文件有点类似注册表，比如包的名称、路径、权限等等。

下面是8.0设备上的package.xml文件结构：

```
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<packages>
	//节点version 存储了当前设备的基本信息：包括sdk版本，database版本，以及指纹等信息。
    <version sdkVersion="26" databaseVersion="3" fingerprint="google/sdk_gphone_x86/generic_x86:8.0.0/OSR1.180418.024/6450276:userdebug/dev-keys" />
    <version volumeUuid="primary_physical" sdkVersion="26" databaseVersion="26" fingerprint="google/sdk_gphone_x86/generic_x86:8.0.0/OSR1.180418.024/6450276:userdebug/dev-keys" />
    //节点permission-trees代表了一组权限信息 ：如有com.google.android.googleapps.permission.GOOGLE_AUTH：p1 或者com.google.android.googleapps.permission.GOOGLE_AUTH：p2等
	<permission-trees>
        <item name="com.google.android.googleapps.permission.GOOGLE_AUTH" package="com.google.android.gsf" />
    </permission-trees>
	//permissions:表示系统中所有的权限信息 package表示该权限归属于某个应用，没有package表示归属于系统，protection表示当前权限的级别，如normal或dangerous等。
    <permissions>
        <item name="com.google.android.apps.nexuslauncher.permission.READ_SETTINGS" package="com.google.android.apps.nexuslauncher" />
        <item name="com.google.android.gms.auth.api.phone.permission.SEND" package="com.google.android.gms" protection="2" />
        <item name="android.permission.REAL_GET_TASKS" package="android" protection="18" />
        <item name="android.permission.ACCESS_CACHE_FILESYSTEM" package="android" protection="18" />
        <item name="android.permission.REMOTE_AUDIO_PLAYBACK" package="android" protection="2" />
        //省略一大串permission item。。。
    </permissions>
	//package：表示一个应用的信息 name：应用包名 codePath;代码放置的路径 nativeLibraryPath：表示apk的so库存放路径 。。。。 
    <package name="com.google.android.youtube" codePath="/system/app/YouTube" nativeLibraryPath="/system/app/YouTube/lib" primaryCpuAbi="x86" publicFlags="945536581" privateFlags="0" ft="171cdd40748" it="171cdd40748" ut="171cdd40748" version="121741370" userId="10076" isOrphaned="true">
        //签名信息
		<sigs count="1">
            <cert index="1" key="308204a830820390a003020102020900847e4f..." />
        </sigs>
		//权限信息
        <perms>
            <item name="com.google.android.c2dm.permission.RECEIVE" granted="true" flags="0" />
            <item name="android.permission.USE_CREDENTIALS" granted="true" flags="0" />
            <item name="com.google.android.providers.gsf.permission.READ_GSERVICES" granted="true" flags="0" />
            <item name="com.google.android.youtube.permission.C2D_MESSAGE" granted="true" flags="0" />
            //省略
        </perms>
		//app使用的公钥信息的id。对应下面的keysets节点中的公钥信息
        <proper-signing-keyset identifier="15" />
		//域名验证
        <domain-verification packageName="com.google.android.youtube" status="0">
            <domain name="youtu.be" />
            <domain name="m.youtube.com" />
            <domain name="youtube.com" />
            <domain name="www.youtube.com" />
        </domain-verification>
    </package>
	
   //shared-user节点代表一个shareuser的属性信息。声明了shareuser的应用的userid是固定一样的、
    <shared-user name="android.uid.system" userId="1000">
        <sigs count="1">
            <cert index="3" />
        </sigs>
        <perms>
            <item name="android.permission.BIND_INCALL_SERVICE" granted="true" flags="0" />
            <item name="android.permission.WRITE_SETTINGS" granted="true" flags="0" />
            <item name="android.permission.CONFIGURE_WIFI_DISPLAY" granted="true" flags="0" />
            <item name="android.permission.CONFIGURE_DISPLAY_COLOR_MODE" granted="true" flags="0" />
            <item name="android.permission.ACCESS_WIMAX_STATE" granted="true" flags="0" />
            <item name="android.permission.USE_CREDENTIALS" granted="true" flags="0" />
            <item name="android.permission.MODIFY_AUDIO_SETTINGS" granted="true" flags="0" />
            //省略
        </perms>
    </shared-user>
    //keyset存储了所有的应用对应的公钥信息。
    <keyset-settings version="1">
        <keys>
            <public-key identifier="1" value="MIIBIDANBgkqhkiG9w0BAQEFAAOCAQ..." />
            <public-key identifier="2" value="MIIBIDANBgkqhkiG9w0BAQEFAAOCAQ0AMIIBCAK..." />
			//省略
		</keys>
        <keysets>
            <keyset identifier="1">
                <key-id identifier="1" />
            </keyset>
            <keyset identifier="2">
                <key-id identifier="2" />
            </keyset>
            //省略。。。
        </keysets>
		//表示最近一次使用的公钥id
        <lastIssuedKeyId value="29" />
        <lastIssuedKeySetId value="29" />
    </keyset-settings>
</packages>
```

从文件中可以看出每个节点会有多个相同节点，说明这些节点并不是唯一的。所以写入到内存中也是一个集合的形式存在。

packages文件中各节点分析如下：

| 节点名称         | 节点信息                                                     |
| ---------------- | ------------------------------------------------------------ |
| version          | 展示了当前设备的基本信息：包括sdk版本，database版本，以及指纹等信息。 |
| permission-trees | 代表了一组权限信息 ：如有com.google.android.googleapps.permission.GOOGLE_AUTH：p1 或者com.google.android.googleapps.permission.GOOGLE_AUTH：p2等<br/>而com.google.android.googleapps.permission.GOOGLE_AUTH代表了这两种权限的集合。 |
| permissions      | 表示系统中所有的权限信息 package表示该权限归属于某个应用，没有package表示归属于系统，protection表示当前权限的级别，如normal或dangerous等。 |
| package          | 表示一个应用的在系统中的信息： <br/>**name**：应用包名  	<br/>**codePath**：代码放置的路径 ，即class文件存放的目录，如果是系统app，存放在system分区，如果是第三方app，存放在data分区<br/> **nativeLibraryPath**：表示apk的so库存放路径。<br/> **primaryCpuAbi**：表示app以哪种abi架构运行是armabi还是armabi-v7a，x86等<br/> **ft**：表示apk文件上次被更改的时间，**it**：表示app第一次安装的时间，**ut**：表示app上次被更新时间，它的值好像一直和ft相同, ota或app重装之后，这里的ft和ut可能会改变。<br/> **version**：版本号：也就是versioncode信息。<br/> **userId**：是为app分配的user id, 如果有使用shareUserId, 这里出现的是SharedUserId。**sigs count**：表示这个app有多少个签名信息，cert里的index表示这个app使用的证书的序号，key是app使用的证书内容的ascii码值 perms;表示当前apk申请的权限信息。 |
| shared-user      | 代表一个shareuser的属性信息。声明了同一个shareuser的应用的userid是同一个。<br/>	perms表示当前shared-user拥有的权限信息 |
| keyset-settings  | **keyset-settings块里收集了所有app签名的公钥信息**，和前面介绍的package信息有关<br/>	keysets块中包含了很多keyset, 每个keyset都有一个编号用identifier表示，keyset里包含的key-id里的identifier和上面keys中public-key的identifier的值相对应。<br/>	keys块中public-key里的value值就是从apk包里的签名文件里提取出来的的公钥的值。<br/>	lastIssuedKeyId和lastIssuedKeySetId表示的是最近一次取出来的公钥所属的set编号<br/>可以看到package.xml文件在系统启动过程中起着一个非常重要的作用，主要体现在对应用清单配置以及应用的权限把控方面。 |

**PKMS中还初始化了一个PackageInstallerService服务，用于应用的安装卸载和更新等操作，最终是通过binder通讯和系统层的installd系统服务进行通讯。**

#### **3.文件系统层**

除了上面说的几个的系统包配置文件。还有应用安装的文件系统，包括：第三方应用以及系统应用。
所有的系统app保存在**/system/app**目录下，所有的第三方app保存在**/data/app**目录下。
对于非系统应用，在安装前，apk文件会被复制到一个**临时文件夹**下面，安装成功后会被放到**/data/app**下面，且使用apk的包名为路径存储。

**data/dalvik-cache**目录保存着着程序执行代码，
当Android启动时，DalvikVM监视所有的程序(APK文件)和框架，并且为他们创建一个依存关系树。
DalvikVM通过这个依存关系树来为每个程序优化代码并存储在Dalvik缓存中。这样，所有程序在运行时都会使用优化过的代码。

**data/data**是应用的私有目录，其他应用对此是没有访问权限的，**/sdcard/Android/data**属于应用的外部存储目录。

**system/framework**下存储了系统运行的各种jar包，**framework-res.apk**则存储了framework系统需要的各种资源文件。

#### 4.Installd系统服务层
Installd系统服务主要用来运行apk的**安装和卸载，dex优化等工作**，而PKMS收到应用安装任务时，会把最终任务提交给Installd进行处理。

**Installd进程拥有root权限，而PKMS只拥有系统权限**。

Installd进程在6.0之前使用的是socket通讯，之后使用的是binder通讯。

**有了一个包管理的整体认知过程，下面我们再来具体分析过程**

小余会使用下面两个角度来讲解PKMS在系统中的具体功能以及实现逻辑：

- **1.PKMS的启动过程**
- **2.第三方应用的安装过程。**


## 1.PKMS的启动过程：

### 1.1：启动过程：
PKMS在系统启动过程中的SystemServer进程的startBootstrapServices中启动

```
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
	//创建一个Installer服务，实际执行PKMS应用安装卸载等操作的服务。后面会详细介绍
	Installer installer = mSystemServiceManager.startService(Installer.class);
	
	Pair<PackageManagerService, IPackageManager> pmsPair = PackageManagerService.main(
			mSystemContext, installer, domainVerificationService,
			mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
	//省略
}
```

看PackageManagerService的main方法：

```
public static Pair<PackageManagerService, IPackageManager> main(Context context,
		Installer installer, @NonNull DomainVerificationService domainVerificationService,
		boolean factoryTest, boolean onlyCore) {
	/*
	PackageManagerServiceInjector内存存储了PKMS运行的环境，可以理解为PKMS环境的包装类，
	早期版本是都在PKMS中实现，新版本使用PackageManagerServiceInjector对关键类进行了一个包装。
	其中初始化了很多关键类，如：
	Settings：对apk的信息进行读取存储
	PackageDexOptimizer：用于dex优化
	PackageParser2：用于AndroidManifest中四大组件的解析操作
	ApexManager：对APEX包的管理，apex和apk类似，只是其是谷歌发布的系统更新包。
	...
	*/
	PackageManagerServiceInjector injector = new PackageManagerServiceInjector(
			context, lock, installer, installLock, new PackageAbiHelperImpl(),
			backgroundHandler,
			SYSTEM_PARTITIONS,
			(i, pm) -> new ComponentResolver(i.getUserManagerService(), pm.mUserNeedsBadging),
			(i, pm) -> PermissionManagerService.create(context,
					i.getSystemConfig().getAvailableFeatures()),
			(i, pm) -> new UserManagerService(context, pm,
					new UserDataPreparer(installer, installLock, context, onlyCore),
					lock),
			(i, pm) -> new Settings(Environment.getDataDirectory(),
					RuntimePermissionsPersistence.createInstance(),
					i.getPermissionManagerServiceInternal(),
					domainVerificationService, backgroundHandler, lock),
			(i, pm) -> AppsFilterImpl.create(i,
					i.getLocalService(PackageManagerInternal.class)),
			(i, pm) -> (PlatformCompat) ServiceManager.getService("platform_compat"),
			(i, pm) -> SystemConfig.getInstance(),
			(i, pm) -> new PackageDexOptimizer(i.getInstaller(), i.getInstallLock(),
					i.getContext(), "*dexopt*"),
			(i, pm) -> new DexManager(i.getContext(), i.getPackageDexOptimizer(),
					i.getInstaller(), i.getInstallLock()),
			(i, pm) -> new ArtManagerService(i.getContext(), i.getInstaller(),
					i.getInstallLock()),
			(i, pm) -> ApexManager.getInstance(),
			(i, pm) -> new ViewCompiler(i.getInstallLock(), i.getInstaller()),
			(i, pm) -> (IncrementalManager)
					i.getContext().getSystemService(Context.INCREMENTAL_SERVICE),
			(i, pm) -> new DefaultAppProvider(() -> context.getSystemService(RoleManager.class),
					() -> LocalServices.getService(UserManagerInternal.class)),
			(i, pm) -> new DisplayMetrics(),
			(i, pm) -> new PackageParser2(pm.mSeparateProcesses, pm.mOnlyCore,
					i.getDisplayMetrics(), pm.mCacheDir,
					pm.mPackageParserCallback) /* scanningCachingPackageParserProducer */,
			(i, pm) -> new PackageParser2(pm.mSeparateProcesses, pm.mOnlyCore,
					i.getDisplayMetrics(), null,
					pm.mPackageParserCallback) /* scanningPackageParserProducer */,
			(i, pm) -> new PackageParser2(pm.mSeparateProcesses, false, i.getDisplayMetrics(),
					null, pm.mPackageParserCallback) /* preparingPackageParserProducer */,
			// Prepare a supplier of package parser for the staging manager to parse apex file
			// during the staging installation.
			(i, pm) -> new PackageInstallerService(
					i.getContext(), pm, i::getScanningPackageParser),
			(i, pm, cn) -> new InstantAppResolverConnection(
					i.getContext(), cn, Intent.ACTION_RESOLVE_INSTANT_APP_PACKAGE),
			(i, pm) -> new ModuleInfoProvider(i.getContext()),
			(i, pm) -> LegacyPermissionManagerService.create(i.getContext()),
			(i, pm) -> domainVerificationService,
			(i, pm) -> {
				HandlerThread thread = new ServiceThread(TAG,
						Process.THREAD_PRIORITY_DEFAULT, true /*allowIo*/);
				thread.start();
				return new PackageHandler(thread.getLooper(), pm);
			},
			new DefaultSystemWrapper(),
			LocalServices::getService,
			context::getSystemService,
			(i, pm) -> new BackgroundDexOptService(i.getContext(), i.getDexManager(), pm),
			(i, pm) -> IBackupManager.Stub.asInterface(ServiceManager.getService(
					Context.BACKUP_SERVICE)),
			(i, pm) -> new SharedLibrariesImpl(pm, i));
	
	//调用PackageManagerService的构造方法，构造方法是整个启动过程的核心
	PackageManagerService m = new PackageManagerService(injector, onlyCore, factoryTest,
			PackagePartitions.FINGERPRINT, Build.IS_ENG, Build.IS_USERDEBUG,
			Build.VERSION.SDK_INT, Build.VERSION.INCREMENTAL);
	//这里是用于第一次启动firstboot的时候需要安装系统白名单中的应用。
	m.installAllowlistedSystemPackages();
	//构造IPackageManagerImpl对象并将其add到ServiceManager中：name为package
	IPackageManagerImpl iPackageManager = m.new IPackageManagerImpl();
	ServiceManager.addService("package", iPackageManager);
	//构造IPackageManagerImpl对象并将其add到ServiceManager中：name为package_native
	final PackageManagerNative pmn = new PackageManagerNative(m);
	ServiceManager.addService("package_native", pmn);
	//构造一个PackageManagerLocalImpl对象并将其add到本地LocalManagerRegistry中。
	LocalManagerRegistry.addManager(PackageManagerLocal.class, m.new PackageManagerLocalImpl());
	return Pair.create(m, iPackageManager);

}
```

总结下PKMS的main方法：

- 1.创建了一个PackageManagerServiceInjector对象用于PKMS的运行时环境
- 2.调用了PKMS的构造方法构造一个PKMS对象，构造方法内部是整个PKMS启动过程的核心，下面会分析
- 3.调用installAllowlistedSystemPackages将构造方法中获取的应用信息写入xml文件
- 4.给ServiceManager添加一个name为package的IPackageManagerImpl对象，这个对象是应用层使用getSystemService方法获取到的PMS对象,后期讲解应用安装会用到，**很关键的一个类，打码~**！！
- 5.在本地LocalManagerRegistry中注册了一个以PackageManagerLocal.class为key，以PackageManagerLocalImpl对象为值的键值对
- 6.返回一个Pair：分别对应PackageManagerService对象和iPackageManager服务对象

### 1.2：PKMS的构造过程
在main方法中调用了PKMS的构造方法，PKMS构造方法中主要是对某些xml文件信息进行读取，然后写入到内存中。

下面我们就来分析PKMS的构造方法,**构造方法大致可以分为以下两个阶段：**

- **阶段1**：读取应用相关的文件如package.list以及package.xml等文件
- **阶段2**：扫描系统中的apk并写入到PKMS内存或者文件中。

#### 阶段1：读取应用相关的文件如package.list以及package.xml等文件
注意这里只提取关键方法：

```
public PackageManagerService(PackageManagerServiceInjector injector, boolean onlyCore..){
	//前面省略一大堆初始化的工作
	mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
			ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
	mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
			ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
	//...
	mFirstBoot = !mSettings.readLPw(computer,
                    mInjector.getUserManagerInternal().getUsers(
                    /* excludePartial= */ true,
                    /* excludeDying= */ false,
                    /* excludePreCreated= */ false));
}
```

可以看到阶段1中使用了一个mSettings字段来处理。这个字段是干嘛的呢？进入内部看看

```
/**
Holds information about dynamic settings.
*/
public final class Settings implements Watchable, Snappable 
```

源码中注释很清楚：这个类是用来保存动态设置的信息，在这里就是PKMS用来保存当前系统中的应用相关信息的。
看Settings构造方法：

```
Settings(File dataDir, RuntimePermissionsPersistence runtimePermissionsPersistence,...){
	...
	mSettingsFilename = new File(mSystemDir, "packages.xml");

	mBackupSettingsFilename = new File(mSystemDir, "packages-backup.xml");
	mPackageListFilename = new File(mSystemDir, "packages.list");
	FileUtils.setPermissions(mPackageListFilename, 0640, SYSTEM_UID, PACKAGE_INFO_GID);
	
	final File kernelDir = new File("/config/sdcardfs");
	mKernelMappingFilename = kernelDir.exists() ? kernelDir : null;
	
	// Deprecated: Needed for migration
	mStoppedPackagesFilename = new File(mSystemDir, "packages-stopped.xml");
	mBackupStoppedPackagesFilename = new File(mSystemDir, "packages-stopped-backup.xml");

}
```

在其构造方法中初始化了几个文件类:包括前面分析的两个关键文件packages.xml和packages.list。
packages-backup.xml是packages.xml的备份文件，而packages-stopped在Android T中被注明已过期。
**packages-backup备份文件如果存在的情况，系统启动过程中会优先加载备份文件中的packages信息到PKMS中，这是因为packages.xml文件是存在被损坏的可能性的，所以backup的优先级更高**。这在后面源代码处也可以看到

好了，我们回到PKMS的构造方法第一阶段，调用了下面两个关键方法。

**关键方法1**：mSettings.addSharedUserLPw

```
SharedUserSetting addSharedUserLPw(String name, int uid, int pkgFlags, int pkgPrivateFlags) {
	SharedUserSetting s = mSharedUsers.get(name);
	if (s != null) {
		if (s.mAppId == uid) {
			return s;
		}
		PackageManagerService.reportSettingsProblem(Log.ERROR,
				"Adding duplicate shared user, keeping first: " + name);
		return null;
	}
	s = new SharedUserSetting(name, pkgFlags, pkgPrivateFlags);
	s.mAppId = uid;
	if (mAppIds.registerExistingAppId(uid, s, name)) {
		mSharedUsers.put(name, s);
		return s;
	}
	return null;
}
```

其实就是初始化一些SharedUser的name和SharedUserSetting的一一对应关系。
如对于name为“android.uid.system”的SharedUser，其对应了SharedUserSetting
**pkgFlags**：ApplicationInfo.FLAG_SYSTEM 
**pkgPrivateFlags**：ApplicationInfo.PRIVATE_FLAG_PRIVILEGED 
**uid** ：Process.SYSTEM_UID

**这个uid就对应了前面解析的packages.xml中的shared-user name="android.uid.system" userId="1000"。**

**关键方法2**：mSettings.readLPw

```
boolean readLPw(@NonNull Computer computer, @NonNull List<UserInfo> users) {
	FileInputStream str = null;
	//mBackupSettingsFilename是前面分析的构造方法创建的package_backup.xml文件
	if (mBackupSettingsFilename.exists()) {
		try {
			str = new FileInputStream(mBackupSettingsFilename);		
			if (mSettingsFilename.exists()) {	
				mSettingsFilename.delete();
			}
		} catch (java.io.IOException e) {
			// We'll try for the normal settings file.
		}
	}

	mPendingPackages.clear();
	mPastSignatures.clear();
	mKeySetRefs.clear();
	mInstallerPackages.clear();
	
	try {
		if (str == null) {
			if (!mSettingsFilename.exists()) {
				
				return false;
			}
			/在这之前的代码都是表示packages_backup的优先级高级packages文件
			str = new FileInputStream(mSettingsFilename);
		}
		//使用PullParser对文件进行读取
		final TypedXmlPullParser parser = Xml.resolvePullParser(str);
		//循环读取packages_backup.xml或者packages.xml中的文件
		while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
				&& (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
			if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
				continue;
			}
	
			String tagName = parser.getName();
			
			if (tagName.equals("package")) {//读取package节点
				readPackageLPw(parser, users, originalFirstInstallTimes);
			} else if (tagName.equals("permissions")) {//读取permissions节点
				mPermissions.readPermissions(parser);
			} else if (tagName.equals("permission-trees")) {//读取permission-trees节点
				mPermissions.readPermissionTrees(parser);
			} else if (tagName.equals("shared-user")) {//读取shared-user节点
				readSharedUserLPw(parser, users);
			} else if (tagName.equals("database-version")) {
				// Upgrade from older XML schema
				final VersionInfo internal = findOrCreateVersion(
						StorageManager.UUID_PRIVATE_INTERNAL);
				final VersionInfo external = findOrCreateVersion(
						StorageManager.UUID_PRIMARY_PHYSICAL);
	
				internal.databaseVersion = parser.getAttributeInt(null, "internal", 0);
				external.databaseVersion = parser.getAttributeInt(null, "external", 0);
			} else if (tagName.equals("keyset-settings")) {
				mKeySetManagerService.readKeySetsLPw(parser, mKeySetRefs.untrackedStorage());
			} else if (TAG_VERSION.equals(tagName)) {
				final String volumeUuid = XmlUtils.readStringAttribute(parser,
						ATTR_VOLUME_UUID);
				final VersionInfo ver = findOrCreateVersion(volumeUuid);
				ver.sdkVersion = parser.getAttributeInt(null, ATTR_SDK_VERSION);
				ver.databaseVersion = parser.getAttributeInt(null, ATTR_DATABASE_VERSION);
				ver.fingerprint = XmlUtils.readStringAttribute(parser, ATTR_FINGERPRINT);
			} else if (tagName.equals(DomainVerificationPersistence.TAG_DOMAIN_VERIFICATIONS)) {
				mDomainVerificationManager.readSettings(computer, parser);
			} else if (tagName.equals(
					DomainVerificationLegacySettings.TAG_DOMAIN_VERIFICATIONS_LEGACY)) {
				mDomainVerificationManager.readLegacySettings(parser);
			} else {
				Slog.w(PackageManagerService.TAG, "Unknown element under <packages>: "
						+ parser.getName());
				XmlUtils.skipCurrentTag(parser);
			}
		}
	
		str.close();
	} catch (IOException | XmlPullParserException e) {
		...
	}
	return true;

}
```

readLPw方法使用PullParser方式读取package.xml或者package_backup中的文件。

其中:

- **package节点**的文件被初始化为PackageSetting对象放入到mPackages集合中。

  ```
  final WatchedArrayMap<String, PackageSetting> mPackages;
  ```

  

- **permissions节点**的文件被初始化为LegacyPermission对象放入到LegacyPermissionSettings的mPermissions集合中。

  ```
  private final ArrayMap<String, LegacyPermission> mPermissions = new ArrayMap<>();
  ```

  

- **permission-trees节点**的文件被初始化为LegacyPermission对象放入到LegacyPermissionSettings的mPermissionTrees集合中。

  ```
  private final ArrayMap<String, LegacyPermission> mPermissionTrees = new ArrayMap<>();
  ```

  

**而Settings类又持有LegacyPermissionSettings对象的引用，也就间接持有mPermissions和mPermissionTrees两个集合对象。**

- shared-user节点的文件被初始化为SharedUserSetting对象放入到mSharedUsers集合中。

  ```
  final WatchedArrayMap<String, SharedUserSetting> mSharedUsers = new WatchedArrayMap<>();
  ```

  

  ..
  还有其他节点这里不再描述，读者可以自行源码查看

**阶段1中通过读取系统配置文件信息，写入到mSettings对象中，PKMS持有mSettings对象引用，也就间接获取了所有系统安装包信息。**这一步对后面package的安装至关重要。

下面来看阶段2

#### 阶段2：扫描安装系统中的apk,PackageParser2类负责文件解析

```
public PackageManagerService(PackageManagerServiceInjector injector, boolean onlyCore..){
	mInitAppsHelper = new InitAppsHelper(this, mApexManager, mInstallPackageHelper,
	            mInjector.getSystemPartitions());
	//初始化系统app
	mOverlayConfig = mInitAppsHelper.initSystemApps(packageParser, packageSettings, userIds,
	                startTime);
	//初始化非系统app
	mInitAppsHelper.initNonSystemApps(packageParser, userIds, startTime);	
}
```

第二阶段主要是扫码系统应用以及非系统应用并进行安装。
- **1.初始化系统app**

  ```
  InitAppsHelper.java
  
  public OverlayConfig initSystemApps(PackageParser2 packageParser,...){
  	...
  	//Apex是用于系统安装包的升级。关注点1
  	mApexManager.scanApexPackagesTraced(packageParser, mExecutorService);
  	//扫描System系统 关注点2
  	scanSystemDirs(packageParser, mExecutorService);
  	// Parse overlay configuration files to set default enable state, mutability, and
  	// priority of system overlays.
  	final ArrayMap<String, File> apkInApexPreInstalledPaths = new ArrayMap<>();
  	//查找APEX包中的apk，放入到apkInApexPreInstalledPaths集合中。这个集合元素存储需要在升级APEX前需要预安装的应用路径。
  	for (ApexManager.ActiveApexInfo apexInfo : mApexManager.getActiveApexInfos()) {
  		for (String packageName : mApexManager.getApksInApex(apexInfo.apexModuleName)) {
  			apkInApexPreInstalledPaths.put(packageName, apexInfo.preInstalledApexPath);
  		}
  	}
  	final OverlayConfig overlayConfig = OverlayConfig.initializeSystemInstance(
  			consumer -> mPm.forEachPackage(mPm.snapshotComputer(),
  					pkg -> consumer.accept(pkg, pkg.isSystem(),
  							apkInApexPreInstalledPaths.get(pkg.getPackageName()))));
  
  	if (!mIsOnlyCoreApps) {
  		// do this first before mucking with mPackages for the "expecting better" case
  		updateStubSystemAppsList(mStubSystemApps);
  	}
  	
  	return overlayConfig;
  
  }
  ```

  

关注点1处涉及到了“APEX”概念：

**Apex**

Apex是在Android10中谷歌为了解决系统升级而提出的一个概念。**和APK类似，Apex把Framework层中那些关键的东西搞成一个个的模块，然后可以单独升级这些模块**。
这些模块和就和一个一个的APK类似，就是一个压缩包，后缀名叫.apex。来看官方文档中对.apex文件格式的描述：


![](https://picgo-test-yuhb.oss-cn-shanghai.aliyuncs.com/imgs/apex.png)

**Apex和Apk的区别：**

- **apk是应用程序的载体**，对应用开发者而言，可以apk方式对应用功能进行升级。
- **apex是系统功能的载体**，对系统开发者（目前看主要是谷歌）而言，可以apex方式对系统功能进行升级。
  apex相当于对系统功能进行了更细粒度的划分，可以独立升级这些功能，这些apex包将来就发布在谷歌的playstore上供我们下载。

关注点2处调用了**scanSystemDirs**扫描系统目录下的apk文件进行安装，下面我们重点来分析下这个方法。

```
private void scanSystemDirs(PackageParser2 packageParser, ExecutorService executorService) {
	File frameworkDir = new File(Environment.getRootDirectory(), "framework");

	//扫描system/overlay目录下面的包信息。这个操作要在扫描其他安装包时优先操作。overlay是一种资源的动态替换机制。
	for (int i = mDirsToScanAsSystem.size() - 1; i >= 0; i--) {
		final ScanPartition partition = mDirsToScanAsSystem.get(i);
		if (partition.getOverlayFolder() == null) {
			continue;
		}
		scanDirTracedLI(partition.getOverlayFolder(), /* frameworkSplits= */ null,
				mSystemParseFlags, mSystemScanFlags | partition.scanFlag,
				packageParser, executorService);
	}
	//扫描system/framework目录下的apk文件信息
	scanDirTracedLI(frameworkDir, null,
			mSystemParseFlags,
			mSystemScanFlags | SCAN_NO_DEX | SCAN_AS_PRIVILEGED,
			packageParser, executorService);
	
	for (int i = 0, size = mDirsToScanAsSystem.size(); i < size; i++) {
		
		//扫描system/app目录下的apk文件信息
		scanDirTracedLI(partition.getAppFolder(), /* frameworkSplits= */ null,
				mSystemParseFlags, mSystemScanFlags | partition.scanFlag,
				packageParser, executorService);
	}

}
```

代码中可以看出scanSystemDirs主要扫描三个目录：

- system/overlay
- system/framework
- system/app

其中overlay目录下需要优先扫描安装，**overlay是一种资源的动态替换机制。可以实现一些静态或者动态换肤的操作**。具体可以参考这篇文章

来看使用的扫描方法**scanDirTracedLI**

```
private void scanDirTracedLI(File scanDir, List<File> frameworkSplits,
		int parseFlags, int scanFlags,
		PackageParser2 packageParser, ExecutorService executorService) {
	
	

	mInstallPackageHelper.installPackagesFromDir(scanDir, frameworkSplits, parseFlags,
			scanFlags, packageParser, executorService);

}
//进入installPackagesFromDir
public void installPackagesFromDir(File scanDir, List<File> frameworkSplits, int parseFlags,
		int scanFlags, PackageParser2 packageParser,
		ExecutorService executorService) {
	final File[] files = scanDir.listFiles();
	

	//创建一个ParallelPackageParser用于解析操作，其内部会使用线程池进行处理
	ParallelPackageParser parallelPackageParser =
			new ParallelPackageParser(packageParser, executorService, frameworkSplits);
	
	// Submit files for parsing in parallel
	int fileCount = 0;
	for (File file : files) {
		final boolean isPackage = (isApkFile(file) || file.isDirectory())
				&& !PackageInstallerService.isStageName(file.getName());
		if (!isPackage) {
			// Ignore entries which are not packages
			continue;
		}
		//关注点1
		parallelPackageParser.submit(file, parseFlags);
		fileCount++;
	}
	
	// Process results one by one
	for (; fileCount > 0; fileCount--) {
		//关注点2
		ParallelPackageParser.ParseResult parseResult = parallelPackageParser.take();
		Throwable throwable = parseResult.throwable;
		int errorCode = PackageManager.INSTALL_SUCCEEDED;
		String errorMsg = null;
		
		if (throwable == null) {
			
			try {
				addForInitLI(parseResult.parsedPackage, parseFlags, scanFlags,null);
			} catch (PackageManagerException e) {
			
			}
		} 
	}

}
```

看**关注点1**：parallelPackageParser.submit(file, parseFlags);

```
public void submit(File scanFile, int parseFlags) {
	mExecutorService.submit(() -> {
		ParseResult pr = new ParseResult();
		try {
			pr.scanFile = scanFile;
			pr.parsedPackage = parsePackage(scanFile, parseFlags);
		} 
		try {
			mQueue.put(pr);
		} catch (InterruptedException e) {
		  
		}
	});

}

```

**submit方法调用内部方法parsePackage，然后将返回的parsedPackage放入到mQueue中**。

在**关注点2处会使用take方法从mQueue中取出**解析出来的Package并调用addForInitLI进行apk的安装操作。

我们重点来看**parsePackage**方法,parsePackage内部又调用了mPackageParser的parsePackage，mPackageParser是一个PackageParser2对象。

```
PackageParser2.java
public ParsedPackage parsePackage(File packageFile, int flags, boolean useCaches,...){
	if (useCaches && mCacher != null) {
		ParsedPackage parsed = mCacher.getCachedResult(packageFile, flags);
		if (parsed != null) {
			return parsed;
		}
	}
	ParseResult<ParsingPackage> result = parsingUtils.parsePackage(input, packageFile, flags,
	            frameworkSplits);
	return result。

}
```

**解析前先看缓存中是否有对应Package，如果没有，则调用parsingUtils.parsePackage读取**

```
//frameworks/base/services/core/java/com/android/server/pm/pkg/parsing/ParsingPackageUtils.java
public ParseResult<ParsingPackage> parsePackage(ParseInput input, File packageFile, int flags,
		List<File> frameworkSplits) {
	if (((flags & PARSE_FRAMEWORK_RES_SPLITS) != 0)
			&& frameworkSplits.size() > 0
			&& packageFile.getAbsolutePath().endsWith("/framework-res.apk")) {
		return parseClusterPackage(input, packageFile, frameworkSplits, flags);
	} else if (packageFile.isDirectory()) {
		return parseClusterPackage(input, packageFile, /* frameworkSplits= */null, flags);
	} else {
		return parseMonolithicPackage(input, packageFile, flags);
	}
}
```

如果需要解析的是framework-res.apk或者是文件夹，则调用parseClusterPackage。
其他调用parseMonolithicPackage，这里我们假设是**解析apk文件，则选择走parseMonolithicPackage分支**。

```
private ParseResult<ParsingPackage> parseMonolithicPackage(ParseInput input, File apkFile,
	final ParseResult<ParsingPackage> result = parseBaseApk(input,
			apkFile,
			apkFile.getCanonicalPath(),
			assetLoader, flags);
	if (result.isError()) {
		return input.error(result);
	}

	return input.success(result.getResult()
			.setUse32BitAbi(lite.isUse32bitAbi()));

}
private ParseResult<ParsingPackage> parseBaseApk(ParseInput input, File apkFile,
		String codePath, SplitAssetLoader assetLoader, int flags) {
	final String apkPath = apkFile.getAbsolutePath();


	final AssetManager assets;
	try {
		//获取BaseAssetManager
		assets = assetLoader.getBaseAssetManager();
	} catch (IllegalArgumentException e) {
		
	}
	//查找apk路径的是否在BaseAssetManager资源路径集合中
	final int cookie = assets.findCookieForPath(apkPath);
	if (cookie == 0) {
		return input.error(INSTALL_PARSE_FAILED_BAD_MANIFEST,
				"Failed adding asset path: " + apkPath);
	}
	//使用XmlResourceParser对apk进行解析ANDROID_MANIFEST_FILENAME为AndroidManifest.xml文件
	try (XmlResourceParser parser = assets.openXmlResourceParser(cookie,
			ANDROID_MANIFEST_FILENAME)) {
		final Resources res = new Resources(assets, mDisplayMetrics, null);
	
		ParseResult<ParsingPackage> result = parseBaseApk(input, apkPath, codePath, res,
				parser, flags);
		...
		final ParsingPackage pkg = result.getResult();
		...
		return input.success(pkg);
	} catch (Exception e) {
		return input.error(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
				"Failed to read manifest from " + apkPath, e);
	}

}
```

进入同名parseBaseApk方法，parseBaseApk又调用了parseBaseApkTags方法。
**parseBaseApkTags中根据Manfest文件中的TAG为application调用了parseBaseApplication**

```
private ParseResult<ParsingPackage> parseBaseApplication(ParseInput input,
		ParsingPackage pkg, Resources res, XmlResourceParser parser, int flags)
		throws XmlPullParserException, IOException {
	final String pkgName = pkg.getPackageName();
	int targetSdk = pkg.getTargetSdkVersion();
	...
	while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
			&& (type != XmlPullParser.END_TAG
			|| parser.getDepth() > depth)) {
		if (type != XmlPullParser.START_TAG) {
			continue;
		}

		final ParseResult result;
		String tagName = parser.getName();
		boolean isActivity = false;
		switch (tagName) {
			//解析activity节点
			case "activity":
				isActivity = true;
				// fall-through
			//解析receiver节点
			case "receiver":
				ParseResult<ParsedActivity> activityResult =
						ParsedActivityUtils.parseActivityOrReceiver(mSeparateProcesses, pkg,
								res, parser, flags, sUseRoundIcon, null /*defaultSplitName*/,
								input);
	
				if (activityResult.isSuccess()) {
					ParsedActivity activity = activityResult.getResult();
					if (isActivity) {
						hasActivityOrder |= (activity.getOrder() != 0);
						pkg.addActivity(activity);
					} else {
						hasReceiverOrder |= (activity.getOrder() != 0);
						pkg.addReceiver(activity);
					}
				}
	
				result = activityResult;
				break;
			//解析service节点
			case "service":
				ParseResult<ParsedService> serviceResult =
						ParsedServiceUtils.parseService(mSeparateProcesses, pkg, res, parser,
								flags, sUseRoundIcon, null /*defaultSplitName*/,
								input);
				if (serviceResult.isSuccess()) {
					ParsedService service = serviceResult.getResult();
					hasServiceOrder |= (service.getOrder() != 0);
					pkg.addService(service);
				}
	
				result = serviceResult;
				break;
			//解析provider节点
			case "provider":
				ParseResult<ParsedProvider> providerResult =
						ParsedProviderUtils.parseProvider(mSeparateProcesses, pkg, res, parser,
								flags, sUseRoundIcon, null /*defaultSplitName*/,
								input);
				if (providerResult.isSuccess()) {
					pkg.addProvider(providerResult.getResult());
				}
	
				result = providerResult;
				break;
			case "activity-alias":
				activityResult = ParsedActivityUtils.parseActivityAlias(pkg, res,
						parser, sUseRoundIcon, null /*defaultSplitName*/,
						input);
				if (activityResult.isSuccess()) {
					ParsedActivity activity = activityResult.getResult();
					hasActivityOrder |= (activity.getOrder() != 0);
					pkg.addActivity(activity);
				}
	
				result = activityResult;
				break;
			case "apex-system-service":
				ParseResult<ParsedApexSystemService> systemServiceResult =
						ParsedApexSystemServiceUtils.parseApexSystemService(res,
								parser, input);
				if (systemServiceResult.isSuccess()) {
					ParsedApexSystemService systemService =
							systemServiceResult.getResult();
					pkg.addApexSystemService(systemService);
				}
	
				result = systemServiceResult;
				break;
			default:
				result = parseBaseAppChildTag(input, tagName, pkg, res, parser, flags);
				break;
		}
	}
	
	...
	return input.success(pkg);

}
```

最终对Manifest文件进行了完整的解析，主要是四大组件的信息。都放在ParsingPackage对象中返回。

**四大组件保存方式如下**

- 对Activity节点使用pkg.addActivity添加到pkg中
- 对Service使用pkg.addService(service);
- 对provider使用pkg.addProvider

```
public interface ParsingPackage extends ParsingPackageRead {

    ParsingPackage addActivity(ParsedActivity parsedActivity);
    
    ParsingPackage addService(ParsedService service);
    ...

}
```

可以看到这个类是一个接口类，接口中提供了很全面的信息保存方法，由子类自行实现其保存方式。
Android系统中是从这个方法中获取：

```
final ParsingPackage pkg = mCallback.startParsingPackage(
                    pkgName, apkPath, codePath, manifestArray, isCoreApp);
```

**mCallback是一个接口类，这样就将实现的方式委托给了mCallback实现类，最大限度对对象进行扩展，将对象创建操作提供给用户，这里还是挺巧妙的。**

最终定位到这个pkg的实现类为：PackageImpl extends ParsingPackageImpl，
ParsingPackageImpl中实现了addActivity，addService等操作。

以上就是apk的manifest的解析过程。



**下面回到installPackagesFromDir的关注点2处**。

```
public void installPackagesFromDir(File scanDir, List<File> frameworkSplits, int parseFlags...){
	...	
	// Process results one by one
	for (; fileCount > 0; fileCount--) {
		//关注点2
		ParallelPackageParser.ParseResult parseResult = parallelPackageParser.take();
		Throwable throwable = parseResult.throwable;
		int errorCode = PackageManager.INSTALL_SUCCEEDED;
		String errorMsg = null;
		

		if (throwable == null) {
			
			try {
				addForInitLI(parseResult.parsedPackage, parseFlags, scanFlags,null);
			} catch (PackageManagerException e) {
			
			}
		} 
	}

}
```

apk四大组件信息解析出来后被放到mQueue中，并在关注点2处使用take取出来。最后调用addForInitLI方法处理。
addForInitLI方法会将前面解析到的Package信息，写入到对应的文件中并更新某些系统文件信息且对满足版本要求的应用进行更新操作。

**一般来说启动过程中的app一般都是最新的，除非在进行一些ota操作时，需要更新某些系统的apk。**

对**第三方应用的解析过程和系统应用过程也是类似的，只是扫描的是/data/app下面的apk**，这里不再重复描述。关于PKMS的构造过程就讲这么多。

下面**总结下整个PKMS的构造方法**其实就是做了下面几件事情：

- 1.解析package.list以及package.xml，system/framework等文件信息写入到内存中。
- 2.根据1中的package name信息去加载系统中的应用，其实是加载apk的manifest文件。
- 3.将2中解析的manifest文件信息写入到PKMS中，并对满足版本要求的应用进行更新操作。**后续应用的安装操作是使用installd服务进行。**

**接下里我们来看应用是如何安装到设备中的？**


我们知道PKMS中应用的安装卸载等工作是由**Installer进程以及installd系统进程**来完成的，注意两个词的区别。
### 1.3.Installer服务与installd服务
- **Installer服务**：

````
public class Installer extends SystemService {
	//Installer服务启动的时候会调用onStart，并调用内部connect方法
	@Override
    public void onStart() {
        if (mIsolated) {
            mInstalld = null;
            mInstalldLatch.countDown();
        } else {
            connect();
        }
    }
	//connect方法
    private void connect() {
		//获取名称为installd的服务
        IBinder binder = ServiceManager.getService("installd");
        if (binder != null) {
            try {
				//binder被杀死后需要重连，调用connect，获取新的IInstalld
                binder.linkToDeath(() -> {
                    Slog.w(TAG, "installd died; reconnecting");
                    mInstalldLatch = new CountDownLatch(1);
                    connect();
                }, 0);
            } catch (RemoteException e) {
                binder = null;
            }
        }

```
    if (binder != null) {
		//给mInstalld赋值获取的binder。
        IInstalld installd = IInstalld.Stub.asInterface(binder);
        mInstalld = installd;
        
    } else { 
		//给后台服务创建一个固定时间重连机制。
        BackgroundThread.getHandler().postDelayed(this::connect, CONNECT_RETRY_DELAY_MS);
    }
}
```

}
````

**从代码中了解到Installer只是java层封装，而实际干活是installd守护进程，名称为installd的系统服务**。

- **installd服务**

installd服务是在系统启动过程中启动的

下面我们来看installd服务启动过程，installd进程的启动是在installd.rc文件开始的

````
installd.rc

frameworks/native/cmds/installd/installd.rc
service installd /system/bin/installd
    class main

installd进程运行在installd.cpp
frameworks/native/cmds/installd/installd.cpp

int main(const int argc, char *argv[]) {
    return android::installd::installd_main(argc, argv);
}
static int installd_main(const int argc ATTRIBUTE_UNUSED, char *argv[]) {
    //初始化全局文件路径：包括获取data路径，app路径，root路径，lib目录等等
    if (!initialize_globals()) {
        SLOGE("Could not initialize globals; exiting.\n");
        exit(1);
    }
	//初始化本地使用的文件夹路径
    if (initialize_directories() < 0) {
        SLOGE("Could not create directories; exiting.\n");
        exit(1);
    }
	//启动InstalldNativeService服务
    if ((ret = InstalldNativeService::start()) != android::OK) {
        SLOGE("Unable to start InstalldNativeService: %d", ret);
        exit(1);
    }
	//创建Binder通讯的线程池
    IPCThreadState::self()->joinThreadPool();

```
return 0;
```

}
class InstalldNativeService : public BinderService<InstalldNativeService>, public os::BnInstalld {
public:
    static char const* getServiceName() { return "installd"; }
}
````

由上面代码可知：**installd服务启动过程其实是启动了一个InstalldNativeService服务并注册到ServiceManager中，所以InstalldNativeService使用的是binder的方式和其他进程进行通讯，在6.0之前使用的是socket的方式**。

InstalldNativeService的服务名称是"installd"。

**installd 进程拥有root权限，而启动PKMS的SystemServer进程只有System权限**，所以installd 可以对应用进行安装和卸载，而SystemServer进程只是一个中间者而已。

最后我们来看InstalldNativeService有哪些接口：

````
class InstalldNativeService : public BinderService<InstalldNativeService>, public os::BnInstalld {
public:
    ...
    binder::Status dexopt(const std::string& apkPath, int32_t uid, const std::string& packageName,
                          const std::string& instructionSet, int32_t dexoptNeeded,
                          const std::optional<std::string>& outputPath, int32_t dexFlags,
                          const std::string& compilerFilter, const std::optional<std::string>& uuid,
                          const std::optional<std::string>& classLoaderContext,
                          const std::optional<std::string>& seInfo, bool downgrade,
                          int32_t targetSdkVersion, const std::optional<std::string>& profileName,
                          const std::optional<std::string>& dexMetadataPath,
                          const std::optional<std::string>& compilationReason, bool* aidl_return);

```
binder::Status rmdex(const std::string& codePath, const std::string& instructionSet);

binder::Status rmPackageDir(const std::string& packageName, const std::string& packageDir);
   
binder::Status linkNativeLibraryDirectory(const std::optional<std::string>& uuid,
        const std::string& packageName, const std::string& nativeLibPath32, int32_t userId);
binder::Status createOatDir(const std::string& packageName, const std::string& oatDir,
                            const std::string& instructionSet);

binder::Status deleteOdex(const std::string& packageName, const std::string& apkPath,
                          const std::string& instructionSet,
                          const std::optional<std::string>& outputPath, int64_t* _aidl_return);
...省略
```

};
````

**InstalldNativeService中定义了很多关于应用Package的操作：如对dex文件优化，修改以及删除，对so文件的关联等操作**，这里只列出了部分操作。

好了，关于PKMS的启动过程就讲到这里。

## 这里**总结**下：

PKMS主要是在SystemServer进程启动过程中启动的，PKMS启动过程中主要做了以下事情：

- **1.会对某些配置文件进行解析扫描，放到PKMS对象内存中**

- **2.会对系统中的应用包括：overlay，system，vendor，app等路径下的应用进行扫描，如果发现有版本更新，则进行应用更新操作。**

- **3.初始化包管理过程中需要使用到一些环境对象等。**

由于代码部分比较多，关于应用安装过程的讲解会单独放到下篇。我是[小余](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)，期待你的关注，我们下篇文章见。



## 参考：

[一篇文章看明白 Android PackageManagerService 工作流程](https://blog.csdn.net/freekiteyu/article/details/82774947)

[Android T谷歌官方文档](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java;l=317;drc=7d9f63d6f9d102411e7d19028fefd6fef38f6ee7;bpv=0;bpt=0?q=PackageManagerSer&sq=&ss=android%2Fplatform%2Fsuperproject)

[深入理解安卓-了解一下 Android 10 中的 APEX](https://www.wenjiangs.com/doc/79hyuxbwqmya)

[android overlay机制实践](https://blog.csdn.net/qq_41818873/article/details/124016434)

[Android 系统服务 PMS Installd 守护进程(二)](http://www.icodebang.com/article/319172)

[Android FileProvider介绍](https://blog.csdn.net/xialong_927/article/details/104003484)

[ANDROID 包管理（PACKAGEMANAGERSERVICE）](https://www.freesion.com/article/68561054572/)

[android系统源码目录system/framework下各个jar包的用途](https://blog.csdn.net/wangrengxing/article/details/38847225)

## 同类文章（推荐）：

["一文读懂"系列：Android屏幕刷新机制](https://juejin.cn/post/7163858831309537294)

[Android Framework知识整理：WindowManager体系（上）](https://juejin.cn/post/7166157765570723871)

[“一文读懂”系列：Android中的硬件加速](https://juejin.cn/post/7166935241108488222)

[“framework必会”系列：Android Input系统（一）事件读取机制](https://juejin.cn/post/7169421307421917214)

[“framework必会”系列：Android Input系统（二）事件分发机制](https://juejin.cn/post/7169835311868936222)

[“一文读懂”系列：无处不在的WMS](https://juejin.cn/post/7172598258064162830)

[“一文读懂”系列：AMS是如何动态管理进程的？](https://juejin.cn/post/7174713775944138809)

[不知道如何看源码？试试这几种方式~](https://juejin.cn/post/7176591422529732645)