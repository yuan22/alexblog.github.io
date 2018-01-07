
### 一、问题引入：

#### 1.1 描述

&emsp;通过Android Beam分享联系人，接收设备出现NFC停止运行提示。

##### 1.2 日志

```log
11-16 11:39:14.320 E/AndroidRuntime( 6354): Caused by: android.os.FileUriExposedException: file:///storage/emulated/0/beam/Qw.vcf exposed beyond app through Intent.getData()
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at android.os.StrictMode.onFileUriExposed(StrictMode.java:1796)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at android.net.Uri.checkFileUriExposed(Uri.java:2346)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at android.content.Intent.prepareToLeaveProcess(Intent.java:8965)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at android.content.Intent.prepareToLeaveProcess(Intent.java:8926)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at android.app.PendingIntent.getActivity(PendingIntent.java:340)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at com.android.nfc.beam.BeamTransferManager.updateNotification(BeamTransferManager.java:331)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at com.android.nfc.beam.BeamTransferManager.updateStateAndNotification(BeamTransferManager.java:365)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at com.android.nfc.beam.BeamTransferManager.processFiles(BeamTransferManager.java:429)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at com.android.nfc.beam.BeamTransferManager.finishTransfer(BeamTransferManager.java:243)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at com.android.nfc.beam.BeamStatusReceiver.handleTransferEvent(BeamStatusReceiver.java:141)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at com.android.nfc.beam.BeamStatusReceiver.onReceive(BeamStatusReceiver.java:95)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     at android.app.LoadedApk$ReceiverDispatcher$Args.run(LoadedApk.java:1122)
11-16 11:39:14.320 E/AndroidRuntime( 6354):     ... 7 more
```

#### 1.3 代码

```java
void updateNotification() {
    ......
    if (mIncoming) {
        notBuilder.setContentText(mContext.getString(R.string.beam_tap_to_view));
        Intent viewIntent = buildViewIntent();
        PendingIntent contentIntent = PendingIntent.getActivity(
                mContext, mTransferId, viewIntent, 0, null); //line 331
        notBuilder.setContentIntent(contentIntent);
    }
    ......
}
Intent buildViewIntent() {
    if (mPaths.size() == 0) return null;
    Intent viewIntent = new Intent(Intent.ACTION_VIEW);
    String filePath = mPaths.get(0);
    Uri mediaUri = mMediaUris.get(filePath);
    Uri uri =  mediaUri != null ? mediaUri :
    Uri.parse(ContentResolver.SCHEME_FILE + "://" + filePath);
    viewIntent.setDataAndTypeAndNormalize(uri, mMimeTypes.get(filePath));
    viewIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
    return viewIntent;
}
```

#### 1.4 原因

&emsp;首先来看看官网对FileUriExposedException的介绍(https://developer.android.com/reference/android/os/FileUriExposedException.html)
```doc
The exception that is thrown when an application exposes a file:// Uri to another app.
This exposure is discouraged since the receiving app may not have access to the shared path. For example, the receiving app may not have requested the READ_EXTERNAL_STORAGE runtime permission, or the platform may be sharing the Uri across user profile boundaries.
Instead, apps should use content:// Uris so the platform can extend temporary permission for the receiving app to access the resource.
This is only thrown for applications targeting N or higher. Applications targeting earlier SDK versions are allowed to share file:// Uri, but it's strongly discouraged.
```

&emsp;即从N开始Android不允许在app中将'file://Uri'暴露给其它app,原因在于使用'file://Uri'会有风险，如：

&emsp;&emsp;<1>接收'file://Uri'的app没有申请无法访问私有文件；

&emsp;&emsp;<2>接收'file://Uri'的app没有申请READ_EXTERNAL_STORAGE权限会导致app崩溃。

&emsp;Google提供了FileProvider，用'content:// Uris'来取代'file://Uri'。


### 二、推荐方案 - FileProvider：

#### 2.1 在AndroidManifest.xml中添加provider

```xml
<provider
    android:name="android.support.v4.content.FileProvider"
    android:authorities="$packageame.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/provider_paths" />
</provider>
```

#### 2.2 新增provider_paths.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path name="external_files" path="."/>
</paths>
```

#### 2.3 修改uri相关java代码

```java
Intent buildViewIntent() {
    if (mPaths.size() == 0) return null;
    Intent viewIntent = new Intent(Intent.ACTION_VIEW);
    String filePath = mPaths.get(0);
    Uri mediaUri = mMediaUris.get(filePath);
    Uri uri =  mediaUri != null ? mediaUri :
    //Uri.parse(ContentResolver.SCHEME_FILE + "://" + filePath);
    Uri uri = FileProvider.getUriForFile(getContext(), getPackageName() + ".provider",
                new file(filePath));
    viewIntent.setDataAndTypeAndNormalize(uri, mMimeTypes.get(filePath));
    viewIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
    return viewIntent;
}
```

### 三、兼容方案 - VmPolicy

&emsp;在application的onCreate()方法中添加下面代码：
```java
import android.os.StrictMode;
@Override
public void onCreate() {
    super.onCreate();
    StrictMode.VmPolicy.Builder builder = new StrictMode.VmPolicy.Builder();
    StrictMode.setVmPolicy(builder.build());
}
```

### 四、FileProvider概述

#### 4.1 content uri的优点

&emsp;content uri可以临时获取读写权限，创建包含content uri的intent，可以调用Intent.setFlags()添加权限，该权限在对方app退出后失效。

&emsp;相比之下，使用file://Uri时只能通过修改文件系统的权限来实现访问控制，这样的话访问控制是它对所有app都生效的，即存在一定风险。

#### 4.2 定义FileProvider

&emsp;在AndroidManifest.xml的<application>节点中添加<provider>

```xml
<manifest>
    ...
    <application>
        ...
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.mydomain.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            ...
        </provider>
        ...
    </application>
</manifest>
```

&emsp;<1>android:authorities是用来标识provider的唯一标识，在同一部手机上一个”authority”串只能被一个app使用，冲突的话会导致app无法安装。我们可以利用manifest placeholders来保证authority的唯一性。

&emsp;<2>android:exported必须设置成false，否则运行时会报错java.lang.SecurityException:Provider must not be exported。

&emsp;<3>android:grantUriPermissions用来控制共享文件的访问权限，也可以在java代码中设置。

#### 4.3 指定可用文件

&emsp;在res/xml/provider_paths.xml文件中添加path节点：

```xml
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <files-path name="my_images" path="images/"/>
    ...
</paths>
```

&emsp;其中，<paths>节点可以是以下五种子节点：

```xml
<1><files-path name="name" path="path" />
  -->Context.getFilesDir()
<2><cache-path name="name" path="path" />
  -->Context.getCacheDir()
<3><external-path name="name" path="path" />
  -->Environment.getExternalStorageDirectory()
<4><external-files-path name="name" path="path" />
  -->Context.getExternalFilesDir(String)
<5><external-cache-path name="name" path="path" />
  -->Context.getExternalCacheDir()
```

#### 4.4 生成Content Uri

```java
File imagePath = new File(Context.getFilesDir(), "images");
File newFile = new File(imagePath, "default_image.jpg");
Uri contentUri = getUriForFile(getContext(), "com.mydomain.fileprovider", newFile);
```

#### 4.5 申请Uri临时权限

&emsp;<1>调用Context.grantUriPermission(package, Uri, mode_flags)。这种设置权限方式只有在调用Context.revokeUriPermission()方法或重启后才会失效。

&emsp;<2>调用Intent.setData()/setClipData()将content uri传入Intent。

&emsp;<3>调用Intent.setFlags()申请权限。

&emsp;<4>发送Intent给其它app，更多情况下会调用Intent.setResult()方法返回uri。

&emsp;&emsp;注：权限失效的时机是，接收Intent的Activity所在的stack销毁时。
