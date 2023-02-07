> 🔥 **Hi，我是小余。**
> **本文已收录到 [GitHub · Androider-Planet](https://github.com/ByteYuhb/Androider-Planet) 中。这里有 Android 进阶成长知识体系，关注公众号 [[小余的自习室](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)] ，在成功的路上不迷路！**
## 前言

前面一篇文章我们讲解了PKMS的启动过程。

PKMS启动过程中主要做了以下事情：

-   **1.会对某些配置文件进行解析扫描，放到PKMS对象内存中**
-   **2.会对系统中的应用包括：overlay，system，vendor，app等路径下的应用进行扫描，如果发现有版本更新，则进行应用更新操作。**
-   **3.初始化包管理过程中需要使用到一些环境对象等。**

接下面我们再来讲解下第三方应用的安装过程

## 应用安装过程

应用安装的方式有如下几种：

### 1.普通安装方式

**在7.0之后，为了进一步提升文件读写的安全性，Android框架执行的StrictMode API政策禁止在您的应用外部公开file://URI**。 如果一项包含文件URI的intent离开您的应用，则应用出现故障，并出现FileUriExposedException异常。

这个时候需要**使用FileProvider来授权外部文件读写权限**。

**FileProvider**

具体使用方式如下：

-   1.在AndroidManifest文件中定义：

    ```
    <provider
        android:authorities="${applicationId}.fileprovider"
        android:name="androidx.core.content.FileProvider"
        android:exported="false"
        android:grantUriPermissions="true"
        >
        <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/update_files"
            />
    </provider>
    ```

-   2.在xml中定义文件update_files.xml：

    ```
    <?xml version="1.0" encoding="utf-8"?>
    <paths xmlns:android="http://schemas.android.com/apk/res/android">
        <external-path
            name="external_storage_install"
            path="yuhb/install"/>
    </paths>
    ```

-   3.在代码中调用

    ```
    /**
    普通应用安装方式
    7.0以后需要使用FileProvider进行申请
    @param apkFile
    @param context
    */
    public static void generateInstall(File apkFile, Context context){
        if(!apkFile.exists()){
            return;
        }
        Intent intent = getInstallIntent(apkFile, context);
        context.startActivity(intent);
    }
    //获取安装应用的Intent
    private static Intent getInstallIntent(File apkFile, Context context) {
        Uri data;
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        data = getInstallUri(context,apkFile);
        //7.0以后使用FileProvider处理
        if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.N){
            intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);//授权其他应用的读权限
            intent.addFlags(Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION);//防止app加固下出现授权失败情况
        //            intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);//授权其他应用写权限
        }
        intent.setDataAndType(data,"application/vnd.android.package-archive");
        return intent;
    }
    //获取安装文件的uri
    private static Uri getInstallUri(Context context,File apkFile) {
        Uri data;
        //7.0以后使用FileProvider处理
        if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.N){
            data = FileProvider.getUriForFile(context,context.getPackageName()+".fileprovider",apkFile);
        }else {
            data = Uri.fromFile(apkFile);
        }
        return data;
    }
    ```

### 2.静默安装方式（需要有root权限）

你是不是尝试了N种方法，打了N个debug，然后得到的却是各种各样的安装失败 ~ **首先类似静默功能一般是被系统所禁止的，只有厂商在自已平台才会开发权限**（好比小米的系统应用，默认实现了静默功能，但是如果小米应用移植到vivo就无效了）。

**具体使用方式如下：**

```
/**静默安装方式，一般需要root权限或者是厂商自己的系统应用。
@param context
@param apkFilePath
*/
public static void silenceInstallApk(Context context,String apkFilePath) {
    /*apkFilePath:这里我们首先传入的是安装包的路径   installObserver:自定义安装的回调，不需要可以删了*/
    File apkFile = new File(apkFilePath);
    //判断路径下的文件是否存在
    if (!apkFile.exists()) {
        Log.e(TAG, "apkFile is null...");
        return;
    }
    String packageName = "";
    //获取安装包的信息
    PackageInfo packageInfo = context.getPackageManager().getPackageArchiveInfo(apkFilePath,
            PackageManager.GET_ACTIVITIES | PackageManager.GET_SERVICES);
    if (packageInfo != null) {
        packageName = packageInfo.packageName;
        String versionName = packageInfo.versionName;
    }
    //获取packageInstaller，后面用来创建PackageInstaller.Session
    PackageInstaller packageInstaller = context.getPackageManager().getPackageInstaller();
    //获取创建PackageInstaller.Session的参数
    PackageInstaller.SessionParams sessionParams = new PackageInstaller.SessionParams(
            PackageInstaller.SessionParams.MODE_FULL_INSTALL);
    /*指示将在此会话中交付的所有APK的总大小（以字节为单位），系统可以使用它来确保在继续之前存在足够的磁盘空间，
    或者估计安装在外部存储上的容器大小*/
    sessionParams.setSize(apkFile.length());
    PackageInstaller.Session session = null;
    try {
        //代表一个session的唯一ID,这里我是在全局变量中声明，因为后面的另外一个方法用到了这个sessionId
        int mSessionId = packageInstaller.createSession(sessionParams);
        if (mSessionId != -1) {
            //也就是在这个外部的onTransfesApkFile()方法中，将会用到sessionId
            boolean copySuccess = onTransfesApkFile(mSessionId,context,apkFilePath, packageName);
            if (copySuccess) {
                session = context.getPackageManager().getPackageInstaller().openSession(mSessionId);
                //设置安装完成后需要发送的一个自定义安装结果广播，这里我设置了App的NAME，VERSION，PACKAGE
                Intent intent = new Intent(context,
                        InstallResultReceiver.class);
                intent.setAction(PackageInstaller.EXTRA_STATUS);
                intent.putExtra("APP_VERSION", "1.0");
                intent.putExtra("APP_PACKAGE", "com.allinpay.manager");
                //执行结束后，发送intent
                PendingIntent pendingIntent = PendingIntent.getBroadcast(context,1,
                        intent, PendingIntent.FLAG_UPDATE_CURRENT);
                //这里最终进行session的提交
                session.commit(pendingIntent.getIntentSender());
            } else {
                //此处是安装失败的回调，不需要可以删除
    //                    if (installObserver != null) {
    //                        installObserver.observer(false, apkFilePath, packageName);
    //                    }
            }
        }
    } catch (Exception exception) {
        Log.e(TAG, "installApk exception = " + exception.getLocalizedMessage());
    } finally {
        if (null != session) {
            session.close();
        }
        //安装完成需要删除文件
        if (apkFile != null && apkFile.exists()) {
    //                apkFile.delete();
        }
    }
}
private static boolean onTransfesApkFile(int mSessionId,Context context,String apkFilePath, String packageName) {
    InputStream in = null;
    OutputStream out = null;
    PackageInstaller.Session session = null;
    boolean success = false;
    try {
        File apkFile = new File(apkFilePath);
        //根据sessionId来获取其代表的session
        session = context.getPackageManager().getPackageInstaller().openSession(mSessionId);
        //向session中写入文件数据
        out = session.openWrite(packageName + "_base.apk", 0, apkFile.length());
        in = new FileInputStream(apkFile);
        int total = 0;
        int len;
        byte[] buffer = new byte[1024];
        while ((len = in.read(buffer)) != -1) {
            total += len;
            out.write(buffer, 0, len);
        }
        session.fsync(out);
        success = true;
    } catch (IOException exception) {
        exception.printStackTrace();
    } finally {
        if (null != session) {
            session.close();
        }
        try {
            if (null != out) {
                out.close();
            }
            if (null != in) {
                in.close();
            }
        } catch (IOException exception) {
            exception.printStackTrace();
        }
    }
    return success;
}
```

这里我们来分析下方式一（**普通应用更新方式**）

-   **1.提取关键代码**：调用的startActivity中的Intent属性

    -   `Action`：Intent.ACTION_VIEW
    -   `Flag`：Intent.FLAG_ACTIVITY_NEW_TASK
    -   `Uri`：content格式
    -   `Type`：application/vnd.android.package-archive

-   **2.根据以上Intent，调用了startActivity**，然后通过PKMS找到对应的Activity。 最终定位到：**InstallStart类，这个类就是启动安装时打开的第一个Activity**。

    ```
    /**
    Select which activity is the first visible activity of the installation and forward the intent to it.
    */
    public class InstallStart extends Activity {
        protected void onCreate(@Nullable Bundle savedInstanceState) {
    
            Uri packageUri = intent.getData();
            //如果Scheme是Content格式
            if (packageUri != null && packageUri.getScheme().equals(
                    ContentResolver.SCHEME_CONTENT)) {
                // [IMPORTANT] This path is deprecated, but should still work. Only necessary
                // features should be added
            // Copy file to prevent it from being changed underneath this process
            nextActivity.setClass(this, InstallStaging.class);
            //如果Scheme是package格式
            } else if (packageUri != null && packageUri.getScheme().equals(
                    PackageInstallerActivity.SCHEME_PACKAGE)) {
                nextActivity.setClass(this, PackageInstallerActivity.class);
            //如果Scheme是其他格式
            } else {
                Intent result = new Intent();
                result.putExtra(Intent.EXTRA_INSTALL_RESULT,
                        PackageManager.INSTALL_FAILED_INVALID_URI);
                setResult(RESULT_FIRST_USER, result);
                nextActivity = null;
            }
            if (nextActivity != null) {
                startActivity(nextActivity);
            }
            finish();
      
        }
    }
    ```

InstallStart的onCreate方法会对传入的Scheme格式进行判断，然后启动另外一个Activity，并结束自己。 我们重点来看Content格式的Activity。最终启动的是InstallStaging.class。看父类名字应该是一个选择框类型的Activity。

```
//frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/InstallStaging.java
public class InstallStaging extends AlertActivity {
    protected void onResume() {
        ...
        mStagingTask = new StagingAsyncTask();
        //执行StagingAsyncTask
        mStagingTask.execute(getIntent().getData());
    }
    private final class StagingAsyncTask extends AsyncTask<Uri, Void, Boolean> {
        @Override
        protected Boolean doInBackground(Uri... params) {
            if (params == null || params.length <= 0) {
                return false;
            }
            Uri packageUri = params[0];
            try (InputStream in = getContentResolver().openInputStream(packageUri)) {
                // Despite the comments in ContentResolver#openInputStream the returned stream can
                // be null.
                if (in == null) {
                    return false;
                }
    
                try (OutputStream out = new FileOutputStream(mStagedFile)) {
                    byte[] buffer = new byte[1024 * 1024];
                    int bytesRead;
                    while ((bytesRead = in.read(buffer)) >= 0) {
                        // Be nice and respond to a cancellation
                        if (isCancelled()) {
                            return false;
                        }
                        out.write(buffer, 0, bytesRead);
                    }
                }
            } catch (IOException | SecurityException | IllegalStateException e) {
                Log.w(LOG_TAG, "Error staging apk from content URI", e);
                return false;
            }
            return true;
        }
    
        @Override
        protected void onPostExecute(Boolean success) {
            if (success) {
                // Now start the installation again from a file
                Intent installIntent = new Intent(getIntent());
                installIntent.setClass(InstallStaging.this, DeleteStagedFileOnResult.class);
                installIntent.setData(Uri.fromFile(mStagedFile));
    
                if (installIntent.getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
                    installIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
                }
    
                installIntent.addFlags(Intent.FLAG_ACTIVITY_NO_ANIMATION);
                startActivity(installIntent);
    
                InstallStaging.this.finish();
            } else {
                showError();
            }
        }
    }
}
```

-   1.InstallStaging的onResume时会启用了一个ASyncTask，在后台读取apk文件，并写入到mStagedFile文件中。 **mStagedFile文件的作用是临时文件，防止在安装过程中对原文件变更**。
-   2.在文件读取完成后，调用AsyncTask的onPostExecute方法，这个方法中会再次启动一个DeleteStagedFileOnResult类Activity。

**继续进入DeleteStagedFileOnResult**

```
/**

 * Trampoline activity. Calls PackageInstallerActivity and deletes staged install file onResult.
   */
   public class DeleteStagedFileOnResult extends Activity {
   @Override
   protected void onCreate(@Nullable Bundle savedInstanceState) {
       super.onCreate(savedInstanceState);

       if (savedInstanceState == null) {
           Intent installIntent = new Intent(getIntent());
           installIntent.setClass(this, PackageInstallerActivity.class);
       
           installIntent.setFlags(Intent.FLAG_ACTIVITY_NO_ANIMATION);
           startActivityForResult(installIntent, 0);
       }

}
```

看谷歌给我们的注解：**这个类是一个过渡Activity：最终是用来启动PackageInstallerActivity并删除mStagedFile临时文件**，这在onCreate方法中也可以看出。

那就转到PackageInstallerActivity，在PackageInstallerActivity中会让引导用户点击安装按钮，点击之后会调用startInstall方法进行安装操作。

```
//frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java
private void startInstall() {
    // Start subactivity to actually install the application
    Intent newIntent = new Intent();
    newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
            mPkgInfo.applicationInfo);
    newIntent.setData(mPackageURI);
    newIntent.setClass(this, InstallInstalling.class);
    String installerPackageName = getIntent().getStringExtra(
            Intent.EXTRA_INSTALLER_PACKAGE_NAME);
    if (mOriginatingURI != null) {
        newIntent.putExtra(Intent.EXTRA_ORIGINATING_URI, mOriginatingURI);
    }
    if (mReferrerURI != null) {
        newIntent.putExtra(Intent.EXTRA_REFERRER, mReferrerURI);
    }
    if (mOriginatingUid != PackageInstaller.SessionParams.UID_UNKNOWN) {
        newIntent.putExtra(Intent.EXTRA_ORIGINATING_UID, mOriginatingUid);
    }
    if (installerPackageName != null) {
        newIntent.putExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME,
                installerPackageName);
    }
    if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
        newIntent.putExtra(Intent.EXTRA_RETURN_RESULT, true);
    }
    newIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
    if (mLocalLOGV) Log.i(TAG, "downloaded app uri=" + mPackageURI);
    startActivity(newIntent);
    finish();
}
```

startInstall重新启动了一个“InstallInstalling”去安装，并将启动应用需要的参数信息放到Intent中。

```
//frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/InstallInstalling.java

public class InstallInstalling extends AlertActivity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
                        PackageInstaller.SessionParams.MODE_FULL_INSTALL);
        ...
        //注册一个安装结果监听器launchFinishBasedOnResult
        mInstallId = InstallEventReceiver
                            .addObserver(this, EventResultPersister.GENERATE_NEW_ID,
                                    this::launchFinishBasedOnResult);
        ...
        //创建一个createSession
        mSessionId = getPackageManager().getPackageInstaller().createSession(params);
    }
    
    @Override
    protected void onResume() {
        //启动一个InstallingAsyncTask
        mInstallingTask = new InstallingAsyncTask();
        mInstallingTask.execute();
    }
    
    private final class InstallingAsyncTask extends AsyncTask<Void, Void,
        PackageInstaller.Session> {
    
        @Override
        protected PackageInstaller.Session doInBackground(Void... params) {
            PackageInstaller.Session session;
            try {
                //打开Session
                session = getPackageManager().getPackageInstaller().openSession(mSessionId);
            } catch (IOException e) {
                
            }
            //设置session安装进度
            session.setStagingProgress(0);
    
            try {
                File file = new File(mPackageURI.getPath());
    
                try (InputStream in = new FileInputStream(file)) {
                    long sizeBytes = file.length();
                    try (OutputStream out = session
                            .openWrite("PackageInstaller", 0, sizeBytes)) {
                        byte[] buffer = new byte[1024 * 1024];
                        while (true) {
                            int numRead = in.read(buffer);
    
                            if (numRead == -1) {
                                session.fsync(out);
                                break;
                            }
    
                            if (isCancelled()) {
                                session.close();
                                break;
                            }
    
                            out.write(buffer, 0, numRead);
                            if (sizeBytes > 0) {
                                float fraction = ((float) numRead / (float) sizeBytes);
                                session.addProgress(fraction);
                            }
                        }
                    }
                }
    
                return session;
            } catch (IOException | SecurityException e) {
                
            }
        }
    
        @Override
        protected void onPostExecute(PackageInstaller.Session session) {
            if (session != null) {
                //注册一个broadcastIntent监听安装结果：BROADCAST_ACTION = "com.android.packageinstaller.ACTION_INSTALL_COMMIT";
                Intent broadcastIntent = new Intent(BROADCAST_ACTION);
                broadcastIntent.setFlags(Intent.FLAG_RECEIVER_FOREGROUND);
                broadcastIntent.setPackage(getPackageName());
                broadcastIntent.putExtra(EventResultPersister.EXTRA_ID, mInstallId);
    
                PendingIntent pendingIntent = PendingIntent.getBroadcast(
                        InstallInstalling.this,
                        mInstallId,
                        broadcastIntent,
                        PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_MUTABLE);
                //调用commit进行安装
                session.commit(pendingIntent.getIntentSender());
                mCancelButton.setEnabled(false);
                setFinishOnTouchOutside(false);
            } 
        }
    }
}
    
}
```

InstallInstalling可以总结为下面几个步骤：

-   1.创建session
-   2.打开session
-   3.copy apk文件到Session中
-   4.调用commit进行安装。

仔细观察你会发现：**这里步骤和我们前面分析的静默安装方式步骤其实是一样的**。而我们的InstallInstalling是运行在系统进程中，所以拥有静默安装权限， 而第三方应用是没有这个权限的、

下面我们深入PackageInstaller看看其是如何实现安装过程的？

首先来看context.getPackageManager().getPackageInstaller()获取到的是哪个类？

如果你对Activity熟悉的话，应该知道context的实现类是ContextImpl类。

**定位到它的getPackageManager。**

```
//frameworks/base/core/java/android/app/ContextImpl.java
public PackageManager getPackageManager() {
    if (mPackageManager != null) {
        return mPackageManager;
    }
    final IPackageManager pm = ActivityThread.getPackageManager();
    if (pm != null) {
        // Doesn't matter if we make more than one instance.
        return (mPackageManager = new ApplicationPackageManager(this, pm));
    }
    
    return null;
}
```

由此可知getPackageManager返回的是一个ApplicationPackageManager，而**这里有个关键参数pm，后期操作实际都是通过pm进行的。** pm是通过ActivityThread.getPackageManager()获取。

```
//frameworks/base/core/java/android/app/ActivityThread.java
public static IPackageManager getPackageManager() {
    if (sPackageManager != null) {
        return sPackageManager;
    }
    final IBinder b = ServiceManager.getService("package");
    sPackageManager = IPackageManager.Stub.asInterface(b);
    return sPackageManager;
}
```

哦？原来就是获取ServiceManager中的package服务。如果你还有印象，前面我们分析过在PKMS的main方法中有下面这段代码。

```
//构造IPackageManagerImpl对象并将其add到ServiceManager中：name为package
IPackageManagerImpl iPackageManager = m.new IPackageManagerImpl();
ServiceManager.addService("package", iPackageManager);
```

**所以这里返回的是一个IPackageManagerImpl对象。**

好了，回到前面context.getPackageManager().getPackageInstaller()

**context.getPackageManager()：对应ApplicationPackageManager(context,IPackageManagerImpl)**

进入ApplicationPackageManager的getPackageInstaller：

```
//frameworks/base/core/java/android/app/ApplicationPackageManager.java
public PackageInstaller getPackageInstaller() {
    if (mInstaller == null) {
        try {
            mInstaller = new PackageInstaller(mPM.getPackageInstaller(),
                    mContext.getPackageName(), mContext.getAttributionTag(), getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    return mInstaller;
}
```

**由此可知getPackageInstaller返回的是一个PackageInstaller对象**，而关键看第一个参数mPM.getPackageInstaller()，这个参数也是实际进行安装的类。

前面分析过**mPM是IPackageManagerImpl对象**，进入IPackageManagerImpl.getPackageInstaller()

在IPackageManagerImpl父类IPackageManagerBase实现了getPackageInstaller

```
//frameworks/base/services/core/java/com/android/server/pm/IPackageManagerBase.java
public final IPackageInstaller getPackageInstaller() {
    ...
    return mInstallerService;
}
```

返回的是一个mInstallerService，这个mInstallerService是在哪里赋值的呢？经过几轮跳转，定位到。

**mInstallerService是在PKMS的构造方法中赋值的：mInstallerService = mInjector.getPackageInstallerService();**

**mInjector是PackageManagerServiceInjector类（PKMS的运行时环境类）。**

**最终获取的mInstallerService是在PKMS构造过程中也就是系统开机时初始化的PackageInstallerService对象**。

好了这里画了一张图来表示他们之间的关系：


![包管理框架.drawio.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ea81946faa64adab4adbd443cd6a65e~tplv-k3u1fbpfcp-watermark.image?)

由以上分析可知：context.getPackageManager().getPackageInstaller()获取的是PackageInstaller，而实际安装操作是PackageInstaller的内部PackageInstallerService对象

下面我们根据前面分析出的安装步骤进行具体分析 1.创建session 2.打开session 3.copy apk文件到Session中 4.调用commit进行安装。

### 1.创建session

```
//frameworks/base/core/java/android/content/pm/PackageInstaller.java
public int createSession(@NonNull SessionParams params) throws IOException {
    //mInstaller为PackageInstallerService
    return mInstaller.createSession(params, mInstallerPackageName, mAttributionTag,
            mUserId);
}
PackageInstallerService的createSession方法会调用内置的createSessionInternal方法
private int createSessionInternal(SessionParams params, String installerPackageName...){
    //前面一大堆对Session创建的条件进行判断，不满足创建则抛出异常
    //创建随机数的sessionId
    sessionId = allocateSessionIdLocked();
    //创建SessionDir
    stageDir = buildSessionDir(sessionId, params);
    //InstallSource持有应用安装的apk源文件信息
    InstallSource installSource = InstallSource.create(installerPackageName,
                originatingPackageName, requestedInstallerPackageName,
                installerAttributionTag, params.packageSource);
    //创建session ： PackageInstallerSession
    session = new PackageInstallerSession(mInternalCallback, mContext, mPm, this,
            mSilentUpdatePolicy, mInstallThread.getLooper(), mStagingManager, sessionId,
            userId, callingUid, installSource, params, createdMillis, 0L, stageDir, stageCid,
            null, null, false, false, false, false, null, SessionInfo.INVALID_ID,
            false, false, false, PackageManager.INSTALL_UNKNOWN, "");
    //将session放入到mSessions键值对中，key为sessionId
    synchronized (mSessions) {
        mSessions.put(sessionId, session);
    }
    //将InstallSource放入到PKMS的Setting集合中
    mPm.addInstallerPackageName(session.getInstallSource());
}
```

createSessionInternal过程主要就是创建了PackageInstallerSession对象，并将对象放入到mSessions集合中。

### 2.打开session

打开Session也是调用PackageInstallerService的createSession方法，内部调用openSessionInternal进行打开。

```
private IPackageInstallerSession openSessionInternal(int sessionId) throws IOException {
    synchronized (mSessions) {
        final PackageInstallerSession session = mSessions.get(sessionId);
        if (!checkOpenSessionAccess(session)) {
            throw new SecurityException("Caller has no access to session " + sessionId);
        }
        session.open();
        return session;
    }
}
```

内容很简单，通过sessionId去mSessions获取Session对象，然后调用session.open()打开。

```
//frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java
public void open() throws IOException {
    ...
    boolean wasPrepared;
    synchronized (mLock) {
        wasPrepared = mPrepared;
        if (!mPrepared) {
            if (stageDir != null) {
                prepareStageDir(stageDir);
            }
            mPrepared = true;
        }
    }
}
static void prepareStageDir(File stageDir) throws IOException {
    
    try {
        Os.mkdir(stageDir.getAbsolutePath(), 0775);
        Os.chmod(stageDir.getAbsolutePath(), 0775);
    } catch (ErrnoException e) {
        
    }
}
```

open方法也只是调用Os的mkdir进行stageDir文件夹创建，并且给stageDir文件夹设置了对应的权限、 **stageDir临时文件夹路径：new File("data/app", "vmdl" + sessionId + ".tmp");**

### 3.copy apk文件到Session中

Session文件的写操作openWrite最终调用的是PackageInstallerSession的doWriteInternal，写文件就不介绍了，这个大家都非常清楚了。 只要知道文件是写入到的是stageDir临时文件夹("data/app/vmdl{$sessionId}.tmp")下面

### 4.调用commit进行安装。

```
public void commit(@NonNull IntentSender statusReceiver, boolean forTransfer) {
    ...
    dispatchSessionSealed();
}
private void dispatchSessionSealed() {
    mHandler.obtainMessage(MSG_ON_SESSION_SEALED).sendToTarget();
}

private final Handler.Callback mHandlerCallback = new Handler.Callback() {
    @Override
    public boolean handleMessage(Message msg) {
        switch (msg.what) {

            case MSG_ON_SESSION_SEALED:
                //内部发射一个MSG_STREAM_VALIDATE_AND_COMMIT msg
                handleSessionSealed();
                break;
            case MSG_STREAM_VALIDATE_AND_COMMIT:
                //内部发射一个MSG_INSTALL msg
                handleStreamValidateAndCommit();
                break;
            case MSG_INSTALL:
                //处理应用安装过程
                handleInstall();
                break;
            case MSG_ON_PACKAGE_INSTALLED:
                final SomeArgs args = (SomeArgs) msg.obj;
                final String packageName = (String) args.arg1;
                final String message = (String) args.arg2;
                final Bundle extras = (Bundle) args.arg3;
                final IntentSender statusReceiver = (IntentSender) args.arg4;
                final int returnCode = args.argi1;
                args.recycle();
    
                sendOnPackageInstalled(mContext, statusReceiver, sessionId,
                        isInstallerDeviceOwnerOrAffiliatedProfileOwner(), userId,
                        packageName, returnCode, message, extras);
    
                break;
            case MSG_SESSION_VALIDATION_FAILURE:
                final int error = msg.arg1;
                final String detailMessage = (String) msg.obj;
                onSessionValidationFailure(error, detailMessage);
                break;
        }
    
        return true;
    }
};
```

发送MSG_ON_SESSION_SEALED的msg调用handleSessionSealed方法。handleSessionSealed方法内部又发送了MSG_STREAM_VALIDATE_AND_COMMIT的msg。 然后在handleStreamValidateAndCommit又发送了MSG_INSTALL。所以**最终调用了handleInstall方法进行安装。**

handleInstall方法可以大致分为：

-   1.apk文件的校验
-   2.apk文件的安装

```
@WorkerThread
private void handleInstall() {
    ...
    if (params.isStaged) {
        mStagedSession.verifySession();
    } else {
        verify();
    }
}
```

mStagedSession.verifySession最终也会走到verify，可以直接看verify方法

```
private void verify() {
    try {
        //1.创建安装apk需要的文件夹：
        prepareInheritedFiles();
        //2.解析APK文件并提取so库文件。其实就是解析AndroidManfest中的四大组件信息
        parseApkAndExtractNativeLibraries();
        //3.检验apk的过程
        verifyNonStaged();
    } catch (PackageManagerException e) {
        final String completeMsg = ExceptionUtils.getCompleteMessage(e);
        final String errorMsg = PackageManager.installStatusToString(e.error, completeMsg);
        setSessionFailed(e.error, errorMsg);
        onSessionVerificationFailure(e.error, errorMsg);
    }
}
```

**重点来看3.检验apk的过程**，verifyNonStaged在经过一系列session检查之后，最终会调用到PackageSessionVerifier的verifyAPK方法， verifyAPK内部设置了安装结果监听IPackageInstallObserver2：

```
//frameworks/base/services/core/java/com/android/server/pm/PackageSessionVerifier.java
private void verifyAPK(PackageInstallerSession session, Callback callback)
            throws PackageManagerException {
    final IPackageInstallObserver2 observer = new IPackageInstallObserver2.Stub() {
        @Override
        public void onUserActionRequired(Intent intent) {
            throw new IllegalStateException();
        }
        @Override
        public void onPackageInstalled(String basePackageName, int returnCode, String msg,
                Bundle extras) {
            if (session.isStaged() && returnCode == PackageManager.INSTALL_SUCCEEDED) {
                // Continue verification for staged sessions
                verifyStaged(session.mStagedSession, callback);
                return;
            }
            if (returnCode != PackageManager.INSTALL_SUCCEEDED) {
                String errorMessage = PackageManager.installStatusToString(returnCode, msg);
                session.setSessionFailed(returnCode, errorMessage);
                callback.onResult(returnCode, msg);
            } else {
                session.setSessionReady();
                callback.onResult(PackageManager.INSTALL_SUCCEEDED, null);
            }
        }
    };
    final VerificationParams verifyingSession = makeVerificationParams(session, observer);
    ...
    verifyingSession.verifyStage();
}
frameworks/base/services/core/java/com/android/server/pm/VerificationParams.java
public void verifyStage() {
    mPm.mHandler.post(this::startCopy);
}
```

可以看到verifyStage最终调用了mPm.mHandler post了一个startCopy的任务。

```
final void startCopy() {
    handleStartCopy();
    handleReturnCode();
}
```

handleStartCopy和handleReturnCode是两个抽象方法：具体实现是在VerificationParams类中

```
public void handleStartCopy() {
    //获取需要安装的apk包信息
    PackageInfoLite pkgLite = PackageManagerServiceUtils.getMinimalPackageInfo(mPm.mContext,
            mPackageLite, mOriginInfo.mResolvedPath, mInstallFlags, mPackageAbiOverride);
    //校验需要更新的app的VersionCode，这里面会对VersionCode版本和原始版本进行校验。
    Pair<Integer, String> ret = mInstallPackageHelper.verifyReplacingVersionCode(
            pkgLite, mRequiredInstalledVersionCode, mInstallFlags);
   
    if (!mOriginInfo.mExisting) {
        //如果是PackageManager.INSTALL_APEX不是APEX包，也就是apk包，则调用sendApkVerificationRequest对APK包进行更新
        if ((mInstallFlags & PackageManager.INSTALL_APEX) == 0) {   
            sendApkVerificationRequest(pkgLite);//关注点1
        }
        //回溯版本走的
        if ((mInstallFlags & PackageManager.INSTALL_ENABLE_ROLLBACK) != 0) {
            sendEnableRollbackRequest();
        }
    }
}
```

```
//关注点1
private void sendApkVerificationRequest(PackageInfoLite pkgLite) {
    ...
    //发送完整的校验请求
    sendIntegrityVerificationRequest(verificationId, pkgLite, verificationState);
    //发送package安装包校验
    sendPackageVerificationRequest(
            verificationId, pkgLite, verificationState);
    ...
}
```

sendPackageVerificationRequest主要校验下面几个：

-   **1.四大组件包信息校验**
-   **2.apk打包公钥校验**
-   **3.校验打包的sdk版本信息，通过添加广播的方式进行。**

好了，继续回到startCopy的handleReturnCode方法

```
//frameworks/base/services/core/java/com/android/server/pm/VerificationParams.java
void handleReturnCode() {
    sendVerificationCompleteNotification();
}
private void sendVerificationCompleteNotification() {
    ...
    try {
        mObserver.onPackageInstalled(null, mRet, mErrorMessage,
                new Bundle());
    } catch (RemoteException e) {
        Slog.i(TAG, "Observer no longer exists.");
    }
    ...
}
```

在校验完毕成功以后，回调mObserver的onPackageInstalled方法。 而mObserver之前说过是在verifyAPK方法时传入的。

```
final IPackageInstallObserver2 observer = new IPackageInstallObserver2.Stub() {

    @Override
    public void onPackageInstalled(String basePackageName, int returnCode, String msg,
            Bundle extras) {
        ...
        if (returnCode != PackageManager.INSTALL_SUCCEEDED) {
            String errorMessage = PackageManager.installStatusToString(returnCode, msg);
            session.setSessionFailed(returnCode, errorMessage);
            callback.onResult(returnCode, msg);
        } else {
            session.setSessionReady();
            callback.onResult(PackageManager.INSTALL_SUCCEEDED, null);
        }
    }
};
```

onPackageInstalled回调成功后会再次调用callback的onResult方法

callBack是在前面分析的PackageInstallerSession的verifyNonStaged方法中传入的，一层一层向外回调。

```
private void verifyNonStaged()
        throws PackageManagerException {
    mSessionProvider.getSessionVerifier().verify(this, (error, msg) -> {
        mHandler.post(() -> {
            if (dispatchPendingAbandonCallback()) {
                // No need to continue if abandoned
                return;
            }
            if (error == INSTALL_SUCCEEDED) {
                onVerificationComplete();
            } else {
                onSessionVerificationFailure(error, msg);
            }
        });
    });
}
```

最终回调到onVerificationComplete方法，可以看到前面很大一部分是在对应用进行校验的部分。

下面分析的才是具体安装的过程：

```
@WorkerThread
private void onVerificationComplete() {
	...
	install();
}
private CompletableFuture<Void> install() {
	List<CompletableFuture<InstallResult>> futures = installNonStaged();
	...
}

private List<CompletableFuture<InstallResult>> installNonStaged() {
	...	
	final InstallParams installingSession = makeInstallParams(future);	
	installingSession.installStage();
	...
}

frameworks/base/services/core/java/com/android/server/pm/InstallParams.java
public void installStage() {
	final Message msg = mPm.mHandler.obtainMessage(INIT_COPY);
	

	msg.obj = this;
	mPm.mHandler.sendMessage(msg);

}
```

installStage发送了一个INIT_COPY的msg，定位到mPm = PackageManagerImpl.java mPm.mHandler = PackageHandler

```
final class PackageHandler extends Handler {
	@Override
    public void handleMessage(Message msg) {
        try {
            doHandleMessage(msg);
        } finally {
            Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
        }
    }

    void doHandleMessage(Message msg) {
        switch (msg.what) {
            case INIT_COPY: {
                HandlerParams params = (HandlerParams) msg.obj;
                if (params != null) {
                    ...
                    params.startCopy();                
                }
                break;
            }
    	}
    }

}
```

INIT_COPY的msg调用了HandlerParams的startCopy方法处理，而这个时候的HandlerParams的实现类是InstallParams.java,前面校验过程中的实现类是VerificationParams

```
final void startCopy() {
	

	handleStartCopy();//关注点1
	handleReturnCode();//关注点2

}
```

先看关注点1

```
//frameworks/base/services/core/java/com/android/server/pm/InstallParams.java
public void handleStartCopy() {
	

	PackageInfoLite pkgLite = PackageManagerServiceUtils.getMinimalPackageInfo(mPm.mContext,
			mPackageLite, mOriginInfo.mResolvedPath, mInstallFlags, mPackageAbiOverride);
	
	if (!mOriginInfo.mStaged && pkgLite.recommendedInstallLocation
			== InstallLocationUtils.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
		//先释放一部分不需要的缓存。
		pkgLite.recommendedInstallLocation = mPm.freeCacheForInstallation(
				pkgLite.recommendedInstallLocation, mPackageLite,
				mOriginInfo.mResolvedPath, mPackageAbiOverride, mInstallFlags);
	}
	//根据默认的规则重写安装路径，主要是区分使用外置sdcard路径还是内置路径
	mRet = overrideInstallLocation(pkgLite.packageName, pkgLite.recommendedInstallLocation,
			pkgLite.installLocation);

}

再看关注点2：
void handleReturnCode() {
	processPendingInstall();
}
private void processPendingInstall() {
	//创建安装的参数信息
	InstallArgs args = createInstallArgs(this);
	if (mRet == PackageManager.INSTALL_SUCCEEDED) {
		//关注点3 拷贝apk
		mRet = args.copyApk();
	}
	if (mRet == PackageManager.INSTALL_SUCCEEDED) {
		F2fsUtils.releaseCompressedBlocks(
				mPm.mContext.getContentResolver(), new File(args.getCodePath()));
	}
	if (mParentInstallParams != null) {
		mParentInstallParams.tryProcessInstallRequest(args, mRet);
	} else {
		

		PackageInstalledInfo res = new PackageInstalledInfo(mRet);
		//关注点4
		processInstallRequestsAsync(
				res.mReturnCode == PackageManager.INSTALL_SUCCEEDED,
				Collections.singletonList(new InstallRequest(args, res)));
	}

}
```

processPendingInstall关注两个部分：

-   1.拷贝apk
-   2.安装apk

**1.拷贝apk：mRet = args.copyApk();**

而args 是FileInstallArgs类对象

```
//frameworks/base/services/core/java/com/android/server/pm/FileInstallArgs.java
int copyApk() {
    return doCopyApk();
}
private int doCopyApk() {
    //1.给StageDir分配对应的临时文件夹以及权限
    final File tempDir = mPm.mInstallerService.allocateStageDirLegacy(mVolumeUuid, isEphemeral);    
    mCodeFile = tempDir;
    //2.拷贝Package，这里面主要是四大组件信息的拷贝
    int ret = PackageManagerServiceUtils.copyPackage(
            mOriginInfo.mFile.getAbsolutePath(), mCodeFile);
    
    //3.根据abifilter 拷贝NativeLibrary ,so库到对应的lib目录下
    handle = NativeLibraryHelper.Handle.create(mCodeFile);
    ret = NativeLibraryHelper.copyNativeBinariesWithOverride(handle, libraryRoot,
            mAbiOverride, isIncremental);
    
    return ret;
}
```

先来分析1处allocateStageDirLegacy

```
public File allocateStageDirLegacy(String volumeUuid, boolean isEphemeral) throws IOException {
    synchronized (mSessions) {
        try {
            final int sessionId = allocateSessionIdLocked();
            mLegacySessions.put(sessionId, true);
            final File sessionStageDir = buildTmpSessionDir(sessionId, volumeUuid);
            prepareStageDir(sessionStageDir);
            return sessionStageDir;
        } catch (IllegalStateException e) {
            throw new IOException(e);
        }
    }
}
```

看buildTmpSessionDir，这个前面也分析过，**最后返回的File路径为：data/app/vmdl{$sessionId}.tmp**

再来分析2处PackageManagerServiceUtils.copyPackage

```
public static int copyPackage(String packagePath, File targetDir) {
    try {
        final File packageFile = new File(packagePath);
        //解析APK文件到ParseResult中
        final ParseResult<PackageLite> result = ApkLiteParseUtils.parsePackageLite(
                input.reset(), packageFile, /* flags */ 0);
        //获取apk的文件PackageLite信息
        final PackageLite pkg = result.getResult();
        //拷贝file，核心方法
        copyFile(pkg.getBaseApkPath(), targetDir, "base.apk");
      
        return PackageManager.INSTALL_SUCCEEDED;
    } catch (IOException | ErrnoException e) {
    }
}
private static void copyFile(String sourcePath, File targetDir, String targetName)
        throws ErrnoException, IOException {
    
    final File targetFile = new File(targetDir, targetName);
    final FileDescriptor targetFd = Os.open(targetFile.getAbsolutePath(),
            O_RDWR | O_CREAT, 0644);
    Os.chmod(targetFile.getAbsolutePath(), 0644);
    FileInputStream source = null;
    try {
        source = new FileInputStream(sourcePath);
        FileUtils.copy(source.getFD(), targetFd);
    } finally {
        IoUtils.closeQuietly(source);
    }
}
```

copyPackage会先去解析apk文件，然后调用copyFile方法，copyFile中调用Os.open去打开targetFile目标文件， 调用FileUtils.copy方法将原文件拷贝到目标文件中

**2.安装apk：processInstallRequestsAsync**

```
// Queue up an async operation since the package installation may take a little while.
private void processInstallRequestsAsync(boolean success,
		List<InstallRequest> installRequests) {
	mPm.mHandler.post(() -> {
		mInstallPackageHelper.processInstallRequests(success, installRequests);
	});
}
frameworks/base/services/core/java/com/android/server/pm/InstallPackageHelper.java
public void processInstallRequests(boolean success, List<InstallRequest> installRequests) {
	

	List<InstallRequest> apkInstallRequests = new ArrayList<>();
	for (InstallRequest request : installRequests) {
		...
		apkInstallRequests.add(request);
		
	}
	
	if (success) {
		for (InstallRequest request : apkInstallRequests) {
			//预安装，内部主要是做clean操作
			request.mArgs.doPreInstall(request.mInstallResult.mReturnCode);
		}
		synchronized (mPm.mInstallLock) {
			//实际安装apk过程
			installPackagesTracedLI(apkInstallRequests);
		}
	}
	...

}
private void installPackagesTracedLI(List<InstallRequest> requests) {
	...
	installPackagesLI(requests);
	...
}
```

下面我们重点来分析下installPackagesLI

```
private void installPackagesLI(List<InstallRequest> requests) {
    
    for (InstallRequest request : requests) {       
        //阶段1：prepare阶段
        repareResult = preparePackageLI(request.mArgs, request.mInstallResult);         
        //阶段2：scan阶段
        final ScanResult result = scanPackageTracedLI(
                prepareResult.mPackageToScan, prepareResult.mParseFlags,
                prepareResult.mScanFlags, System.currentTimeMillis(),
                request.mArgs.mUser, request.mArgs.mAbiOverride);       
    }   
    //阶段3：Reconcile阶段
    reconciledPackages = ReconcilePackageUtils.reconcilePackages(
            reconcileRequest, mSharedLibraries,
            mPm.mSettings.getKeySetManagerService(), mPm.mSettings);
    }
    
    commitRequest = new CommitRequest(reconciledPackages,
                            mPm.mUserManager.getUserIds());
    //阶段4：Commit阶段
    commitPackagesLocked(commitRequest);
    //阶段5：完成apk安装
    executePostCommitSteps(commitRequest);
}
```

installPackagesLI是最终安装应用的方法：主要分为4个阶段

-   阶段1：**prepare阶段**:分析当前安装包的状态，解析安装包并对其做初始化验证
-   阶段2：**scan阶段**：根据prepare阶段中收集的安装包状态信息去扫描解析出来的包
-   阶段3：**Reconcile阶段**:验证scan阶段扫描到的Package信息以及当前系统状态，确保apk的正确安装。
-   阶段4：**Commit阶段**：提交所有扫描的包并更新系统状态。这是唯一可以在安装流程和所有可预测错误中修改系统状态的地方.

在 preparePackageLI() 内使用 PackageParser2.parsePackage() 解析AndroidManifest.xml，获取四大组件等信息； 使用ParsingPackageUtils.getSigningDetails() 解析签名信息；重命名包最终路径 等。

完成了解析和校验准备工作后，最后一步就是对apk的安装了。这里调用了executePostCommitSteps准备app数据，并**执行dex优化**。

**最后通过executePostCommitSteps完成apk的安装，执行dex优化等操作**

-   阶段5：**完成apk安装**

    ```
    - private void executePostCommitSteps(CommitRequest commitRequest) {
        
        for (ReconciledPackage reconciledPkg : commitRequest.mReconciledPackages.values()) {
            
            // prepareAppDataPostCommitLIF经过一系列调用会走到Installer的createAppData方法。
            mAppDataHelper.prepareAppDataPostCommitLIF(pkg, 0);
            
            /*
            检测是否需要进行dex优化：同时满足下面三种情况就需要
            1.不是一个即时应用app或者如果是的话通过gservices进行dex优化操作
            2.debuggable为false
            3.不在增量文件系统上。
            */
            final boolean performDexopt =
                    (!instantApp || android.provider.Settings.Global.getInt(
                            mContext.getContentResolver(),
                            android.provider.Settings.Global.INSTANT_APP_DEXOPT_ENABLED, 0) != 0)
                            && !pkg.isDebuggable()
                            && (!onIncremental)
                            && dexoptOptions.isCompilationEnabled();
        
            if (performDexopt) {
                
                //获取so库所在的目录
                PackageSetting realPkgSetting = result.mExistingSettingCopied
                        ? result.mRequest.mPkgSetting : result.mPkgSetting;
                
                boolean isUpdatedSystemApp = reconciledPkg.mPkgSetting.getPkgState()
                        .isUpdatedSystemApp();
                //更新系统app信息。
                realPkgSetting.getPkgState().setUpdatedSystemApp(isUpdatedSystemApp);
                //进行dex优化
                mPackageDexOptimizer.performDexOpt(pkg, realPkgSetting,
                        null /* instructionSets */,
                        mPm.getOrCreateCompilerPackageStats(pkg),
                        mDexManager.getPackageUseInfoOrDefault(packageName),
                        dexoptOptions);
            
            }
        
            //通知BackgroundDexOptService服务当前packageName的应用进行了更新。
            BackgroundDexOptService.getService().notifyPackageChanged(packageName);
            
            notifyPackageChangeObserversOnUpdate(reconciledPkg);
        }
    }
    ```

阶段5：主要做了下面两件事

-   **任务1**：调用prepareAppDataPostCommitLIF方法,最终执行到createAppData方法进行app的安装.

    ```
    public @NonNull CreateAppDataResult createAppData(@NonNull CreateAppDataArgs args)
        throws InstallerException {
    ...
    try {
        return mInstalld.createAppData(args);
    } catch (Exception e) {
        throw InstallerException.from(e);
    }
    }
    ```

mInstalld在前面分析过了，installd进程 的执行权限为 root，所有实际的应用安装，卸载等操作都是通过这个服务进行的。 PKMS只是java层的封装。mInstalld进程和PKMS是通过binder进行通讯的。

-   任务2.调用performDexOpt进行dex优化 同时满足下面三种情况就需要

    -   1.不是一个即时应用app或者如果是的话通过gservices进行dex优化操作
    -   2.debuggable为false
    -   3.不在增量文件系统上。

然后关于dex优化部分，后面会单独出一篇文章来讲解。

关于应用安装部分就讲到这里了。

## 总结

关于Android中包管理机制，由于源码部分内容较多，小余使用了两篇文章来讲解。希望你能从中有所收获。

上一篇：[基于Android T：包管理机制详解（上）](https://juejin.cn/post/7180232678384336952)

如果文字对你帮助，帮忙给[小余](https://mp.weixin.qq.com/s?__biz=MzkwODI1NDEwMA==&mid=2247483986&idx=1&sn=57136c9c062caa1026edf9ed35915c2b&chksm=c0cd8ca9f7ba05bfcfadad10bd97006bbb57afdd048c9c46fe57d122af834f569aa9d8df0e48&token=2142008574&lang=zh_CN#rd)点个赞，谢啦。

## 参考：

[一篇文章看明白 Android PackageManagerService 工作流程](https://blog.csdn.net/freekiteyu/article/details/82774947)

[Android T谷歌官方文档](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java;l=317;drc=7d9f63d6f9d102411e7d19028fefd6fef38f6ee7;bpv=0;bpt=0?q=PackageManagerSer&sq=&ss=android%2Fplatform%2Fsuperproject)

[深入理解安卓-了解一下 Android 10 中的 APEX](https://www.wenjiangs.com/doc/79hyuxbwqmya)

[android overlay机制实践](https://blog.csdn.net/qq_41818873/article/details/124016434)

[Android 系统服务 PMS Installd 守护进程(二)](http://www.icodebang.com/article/319172)

[Android FileProvider介绍](https://blog.csdn.net/xialong_927/article/details/104003484)

[ANDROID 包管理（PACKAGEMANAGERSERVICE）](https://www.freesion.com/article/68561054572/)

[android系统源码目录system/framework下各个jar包的用途](https://blog.csdn.net/wangrengxing/article/details/38847225)

## 同类文章：

["一文读懂"系列：Android屏幕刷新机制](https://juejin.cn/post/7163858831309537294)

[Android Framework知识整理：WindowManager体系（上）](https://juejin.cn/post/7166157765570723871)

[“一文读懂”系列：Android中的硬件加速](https://juejin.cn/post/7166935241108488222)

[“framework必会”系列：Android Input系统（一）事件读取机制](https://juejin.cn/post/7169421307421917214)

[“framework必会”系列：Android Input系统（二）事件分发机制](https://juejin.cn/post/7169835311868936222)

[“一文读懂”系列：无处不在的WMS](https://juejin.cn/post/7172598258064162830)

[“一文读懂”系列：AMS是如何动态管理进程的？](https://juejin.cn/post/7174713775944138809)

[不知道如何看源码？试试这几种方式~](https://juejin.cn/post/7176591422529732645)