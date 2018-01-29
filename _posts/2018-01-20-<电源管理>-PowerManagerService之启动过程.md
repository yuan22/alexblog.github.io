
&emsp;无论是前面介绍过的Doze省电模式还是耗流算法，都离不开Framework中一个系统核心服务：PowerManagerService。其在Android电源管理中的重要性，想必不用我介绍了吧。从本篇文章开始，我将用一个系列，来简单讲述下自己的理解。为避免造成其与另一核心服务PackageManagerService的混淆，本系列中将采用缩写PowerMS来代替PowerManagerService。

&emsp;本文主要介绍PowerMS的启动过程。

### 一、SystemServer启动

&emsp;从Systemserver.java中的代码我们能看到，先利用startService通过反射方式调用PowerMS的构造函数，紧接着调用onStart函数，再调用systemReady方法。

```java
private void startBootstrapServices() {
...
mPowerManagerService =
mSystemServiceManager.startService(PowerManagerService.class);
  ...
}

private void startOtherServices() {
  ...
  try {
    mPowerManagerService.systemReady(mActivityManagerService.getAppOpsService());
  } catch (Throwable e) {
    reportWtf("making Power Manager Service ready", e);
  }
  ...
}
```

### 二、PowerMS 构造函数

```java
（PowerManagerService.java）
public PowerManagerService(Context context) {
  super(context);
  mContext = context;
  // ServiceThread 继承自HandlerThread,专门针对系统服务定义的
  mHandlerThread = new ServiceThread(TAG,
Process.THREAD_PRIORITY_DISPLAY, false /*allowIo*/);
  mHandlerThread.start();
  mHandler = new PowerManagerHandler(mHandlerThread.getLooper());
  synchronized (mLock) {
    // 创建一些锁对象，同构acquire和release修改引用数
mWakeLockSuspendBlocker =
createSuspendBlockerLocked("PowerManagerService.WakeLocks");
mDisplaySuspendBlocker =
createSuspendBlockerLocked("PowerManagerService.Display");
    mDisplaySuspendBlocker.acquire();
    mHoldingDisplaySuspendBlocker = true;
    mHalAutoSuspendModeEnabled = false;
    mHalInteractiveModeEnabled = true;
    mWakefulness = WAKEFULNESS_AWAKE;
    nativeInit();
    nativeSetAutoSuspend(false);
    nativeSetInteractive(true);
    nativeSetFeature(POWER_FEATURE_DOUBLE_TAP_TO_WAKE, 0);
  }
}
```

&emsp;从代码中我们能看到，其构造函数还是比较简单的。下面开始陆续详细分析com_android_server_power_PowerManagerService.cpp中所调用的几个native函数。

#### 2.1 nativeInit函数

```c++
static void nativeInit(JNIEnv* env, jobject obj) {
  // 创建一个全局引用对象,引用PMS
  gPowerManagerServiceObj = env->NewGlobalRef(obj);
  // 利用hw_get_module加载底层动态库,具体实现依赖与disym函数
  status_t err = hw_get_module(POWER_HARDWARE_MODULE_ID,
      (hw_module_t const**)&gPowerModule);
  if (!err) {
    // 调用底层动态库init函数
    gPowerModule->init(gPowerModule);
  } else {
    ALOGE("Couldn't load %s module (%s)", POWER_HARDWARE_MODULE_ID, strerror(-err));
  }
}
```

#### 2.2 nativeSetAutoSuspend函数

```c++
static void nativeSetAutoSuspend(JNIEnv* /* env */, jclass /* clazz */, jboolean enable) {
  if (enable) {
    ALOGD_IF_SLOW(100, "Excessive delay in autosuspend_enable() while turning screen off");
    autosuspend_enable();
  } else {
    ALOGD_IF_SLOW(100, "Excessive delay in autosuspend_disable() while turning screen on");
    // 初始时调用autosuspend_disable函数，定义于system/core/libsuspend/autosuspend.c中
    // 最终将调用autosuspend_earlysuspend_disable函数，就是将底层的置为pwr_state_on状态
    autosuspend_disable();
  }
}
```

#### 2.3 nativeSetInteractive函数

```c++
static void nativeSetInteractive(JNIEnv* /* env */, jclass /* clazz */, jboolean enable) {
  if (gPowerModule) {
    if (enable) {
      ALOGD_IF_SLOW(20, "Excessive delay in setInteractive(true) while turning screen on");
      // 初始时调用动态库的setInteractive函数
      gPowerModule->setInteractive(gPowerModule, true);
    } else {
      ALOGD_IF_SLOW(20, "Excessive delay in setInteractive(false) while turning screen off");
      gPowerModule->setInteractive(gPowerModule, false);
    }
  }
}
```

#### 2.4 nativeSetFeature函数

```c++
static void nativeSetFeature(JNIEnv *env, jclass clazz, jint featureId, jint data) {
  int data_param = data;
  if (gPowerModule && gPowerModule->setFeature) {
    // 初始时，调用动态库的setFeature函数，将POWER_FEATURE_DOUBLE_TAP_TO_WAKE置为0
    gPowerModule->setFeature(gPowerModule, (feature_t)featureId, data_param);
  }
}
```

### 三、PowerMS onStart方法

&emsp;紧接着来看看PowerMS的onStart函数：

```java
（PowerManagerService.java）
public void onStart() {
  publishBinderService(Context.POWER_SERVICE, new BinderService());
  publishLocalService(PowerManagerInternal.class, new LocalService());
  // PMS实现了Watchdog.Monitor接口，将PMS加入到Watchdog的mMonitorChecker中
  Watchdog.getInstance().addMonitor(this);
  // 将PMS的ServiceThread对应的handler传入wathcdog中，watchdog将利用该handler构造一个Handler
  Watchdog.getInstance().addThread(mHandler);
}
```

&emsp;从上面的代码能了解到，onStart函数主要包括发布Binder服务，发布Local服务，完成Watchdog三类工作。

#### 3.1 SystemService类的publishBinderService方法

```java
// PowerMS中的BinderService继承自IPowerManager.Stub，实现了IBinder接口
// name为PowerMS的名称"power"
protected final void publishBinderService(String name, IBinder service) {
  publishBinderService(name, service, false);
}
protected final void publishBinderService(String name, IBinder service,
    boolean allowIsolated) {
  // 调用ServiceManager接口，利用Binder通信向ServiceManager进程注册服务
  ServiceManager.addService(name, service, allowIsolated);
}
```

#### 3.2 SystemService类的publishLocalService方法

```java
// 参数中的LocalServices继承自PowerManagerInternal类
protected final <T> void publishLocalService(Class<T> type, T service) {
  LocalServices.addService(type, service);
}

// LocalServices类的addService方法
public static <T> void addService(Class<T> type, T service) {
  synchronized (sLocalServiceObjects) {
    if (sLocalServiceObjects.containsKey(type)) {
      throw new IllegalStateException("Overriding service registration");
    }
    sLocalServiceObjects.put(type, service);
  }
}
```

#### 3.3 Watchdog类的addMonitor方法

```java
（Watchdog.java）
public void addMonitor(Monitor monitor) {
  synchronized (this) {
    if (isAlive()) {
      throw new RuntimeException("Monitors can't be added once the Watchdog is running");
    }
    mMonitorChecker.addMonitor(monitor);
  }
}
```

```java
//Watchdog类的HandlerChecker方法
public void scheduleCheckLocked() {
  // PowerMS对应的HandlerChecker没有monitor，因此mMonitors.size()恒等于零
  // 调用MessageQueue的isPolling函数，判断是否处于polling状态
  // 当MessageQueue native层looper处于等待状态，即没有事件要处理时，isPolling返回true
  if (mMonitors.size() == 0 && mHandler.getLooper().getQueue().isPolling()) {
    mCompleted = true
    return;
  }
  if (!mCompleted) {
    return;
  }
  mCompleted = false;
  mCurrentMonitor = null;
  // 向PowerMS的handler发送一个HandlerChecker类型的runnable事件
  mHandler.postAtFrontOfQueue(this);
}

public void run() {
  final int size = mMonitors.size();
  // PowerMS对应的HandlerChecker的mMonitors.size为0，跳过
  for (int i = 0 ; i < size ; i++) {
    synchronized (Watchdog.this) {
      mCurrentMonitor = mMonitors.get(i);
    }
    mCurrentMonitor.monitor();
  }
  // 只要PowerMS的mHandler在规定事件内，执行了上文传入的runnable事件，就说明没有阻塞，PowerMS是正常的
  synchronized (Watchdog.this) {
    mCompleted = true;
    mCurrentMonitor = null;
  }
}
```

```java
// PowerMS的monitor方法
public void monitor() {
  // Grab and release lock for watchdog monitor to detect deadlocks.
  synchronized (mLock) {
  }
}
```

#### 3.4 addThread

```java
public void addThread(Handler thread) {
  addThread(thread, DEFAULT_TIMEOUT);
}
public void addThread(Handler thread, long timeoutMillis) {
  synchronized (this) {
    if (isAlive()) {
      throw new RuntimeException("Threads can't be added once the Watchdog is running");
    }
    final String name = thread.getLooper().getThread().getName();
    mHandlerCheckers.add(new HandlerChecker(thread, name, timeoutMillis));
  }
}
```

### 四、PowerMS类的systemReady方法

&emsp;systemReady方法主要是获取部分成员变量，注册部分广播用于接收对象、读取配置。其中最关键于监测到变化时调用的updatePowerStateLocked方法更新一些状态，后续将会讲解。

```java
public void systemReady(IAppOpsService appOps) {
  synchronized (mLock) {
    mSystemReady = true;
    mAppOps = appOps;
    // Doze相关
    mDreamManager = getLocalService(DreamManagerInternal.class);
    // 显示管理相关
    mDisplayManagerInternal = getLocalService(DisplayManagerInternal.class);
    // window管理相关
    mPolicy = getLocalService(WindowManagerPolicy.class);
    // 电源管理相关
    mBatteryManagerInternal = getLocalService(BatteryManagerInternal.class);
    // 获取最小、最大、默认屏幕亮度属性
PowerManager pm =
(PowerManager) mContext.getSystemService(Context.POWER_SERVICE);
    mScreenBrightnessSettingMinimum = pm.getMinimumScreenBrightnessSetting();
    mScreenBrightnessSettingMaximum = pm.getMaximumScreenBrightnessSetting();
    mScreenBrightnessSettingDefault = pm.getDefaultScreenBrightnessSetting();
    // Sensor相关
SensorManager sensorManager =
new SystemSensorManager(mContext, mHandler.getLooper());
    // 系统电量统计相关
    mBatteryStats = BatteryStatsService.getService();
    mNotifier = new Notifier(Looper.getMainLooper(), mContext, mBatteryStats,
        mAppOps, createSuspendBlockerLocked("PowerManagerService.Broadcasts"),
        mPolicy);
    // 无线充电检测相关
    mWirelessChargerDetector = new WirelessChargerDetector(sensorManager,
        createSuspendBlockerLocked("PowerManagerService.WirelessChargerDetector"),
        mHandler);
    mSettingsObserver = new SettingsObserver(mHandler);
    // 指示灯相关
    mLightsManager = getLocalService(LightsManager.class);
    mAttentionLight = mLightsManager.getLight(LightsManager.LIGHT_ID_ATTENTION);

    // 调用DisplayManagerService内部的LocalService的函数，创建出DisplayBlanker和DisplayPowerController
    mDisplayManagerInternal.initPowerManagement(
        mDisplayPowerCallbacks, mHandler, sensorManager);
    // 注册一系列系统其它广播
    ...
    // 监听Settings数据库字段变化
    ...
    // VR相关
    IVrManager vrManager =
        (IVrManager) getBinderService(VrManagerService.VR_MANAGER_BINDER_SERVICE);
    try {
      vrManager.registerListener(mVrStateCallbacks);
    } catch (RemoteException e) {
      Slog.e(TAG, "Failed to register VR mode state listener: " + e);
    }
    // 从资源文件中读取配置信息
    readConfigurationLocked();
    // 读取数据库字段，保存到本地变量中
    updateSettingsLocked();
    mDirty |= DIRTY_BATTERY_STATE;
    // 更新全局电源状态
    updatePowerStateLocked();
  }
}
```
