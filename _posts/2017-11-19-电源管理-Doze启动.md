
### 一、SystemServer启动DeviceIdleController服务

```java
    private void startOtherServices() {
        ...
        //　非工厂模式，且非核心服务未被禁用时执行
        if (!disableNonCoreService) {
            ...
            traceBeginAndSlog("StartDeviceIdleController");
            // 通过反射方式创建DeviceIdleController服务
            mSystemServiceManager.startService(DeviceIdleController.class);
            traceEnd();
            ...
        }
    }
```

### 二、DeviceIdleController的启动

#### 2.1 构造函数

```java
    public DeviceIdleController(Context context) {
        //　继承SystemService
        super(context);
        //　在/data/system/目录下创建deviceidle.xml文件
        mConfigFile = new AtomicFile(new File(getSystemDir(), "deviceidle.xml"));
        //　创建一个handler后台运行，用于接收状态切换时所发送的信息，并据此做出相应的行为变化，详见后续状态机分析
        mHandler = new MyHandler(BackgroundThread.getHandler().getLooper());
    }
```


#### 2.2 onStart方法

```java
    @Override
    public void onStart() {
        final PackageManager pm = getContext().getPackageManager();
        synchronized (this) {
            // 从config.xml中读取config_enableAutoPowerModes的值，默认为false
            mLightEnabled = mDeepEnabled = getContext().getResources().getBoolean(
                    com.android.internal.R.bool.config_enableAutoPowerModes);
            SystemConfig sysConfig = SystemConfig.getInstance();
            // 从/system/etc/permissions/platform.xml文件中读取PowerSave模式应用白名单（非Device Idle模式）
            ArraySet<String> allowPowerExceptIdle = sysConfig.getAllowInPowerSaveExceptIdle();
            for (int i=0; i<allowPowerExceptIdle.size(); i++) {
                String pkg = allowPowerExceptIdle.valueAt(i);
                try {
                    // 列出白名单中系统应用包名及uid
                    ApplicationInfo ai = pm.getApplicationInfo(pkg,
                            PackageManager.MATCH_SYSTEM_ONLY);
                    int appid = UserHandle.getAppId(ai.uid);
                    mPowerSaveWhitelistAppsExceptIdle.put(ai.packageName, appid);
                    mPowerSaveWhitelistSystemAppIdsExceptIdle.put(appid, true);
                } catch (PackageManager.NameNotFoundException e) {
                }
            }
            // 从/system/etc/permiisons/platform.xml文件中读取PowerSave模式应用白名单
            ArraySet<String> allowPower = sysConfig.getAllowInPowerSave();
            for (int i=0; i<allowPower.size(); i++) {
                String pkg = allowPower.valueAt(i);
                try {
                    // 列出白名单中系统应用包名及uid，其中部分应用既在PowerSaveExceptIdle白名单中也在PowerSave白名单中。
                    ApplicationInfo ai = pm.getApplicationInfo(pkg,
                            PackageManager.MATCH_SYSTEM_ONLY);
                    int appid = UserHandle.getAppId(ai.uid);
                    mPowerSaveWhitelistAppsExceptIdle.put(ai.packageName, appid);
                    mPowerSaveWhitelistSystemAppIdsExceptIdle.put(appid, true);
                    mPowerSaveWhitelistApps.put(ai.packageName, appid);
                    mPowerSaveWhitelistSystemAppIds.put(appid, true);
                } catch (PackageManager.NameNotFoundException e) {
                }
            }
            // 监控数据库变化
            mConstants = new Constants(mHandler, getContext().getContentResolver());
            // 读取PowerSave用户应用白名单列表
            readConfigFileLocked();
            // 更新PowerSave应用白名单列表
            updateWhitelistAppIdsLocked();

            // 初始化network，亮屏、充电、指示灯状态等为true
            mNetworkConnected = true;
            mScreenOn = true;
            mCharging = true;
            mState = STATE_ACTIVE;
            mLightState = LIGHT_STATE_ACTIVE;
            mInactiveTimeout = mConstants.INACTIVE_TIMEOUT;
        }
        // 发布BinderService和LocalService
        mBinderService = new BinderService();
        publishBinderService(Context.DEVICE_IDLE_CONTROLLER, mBinderService);
        publishLocalService(LocalService.class, new LocalService());
    }
```

```java
    private void updateWhitelistAppIdsLocked() {
        mPowerSaveWhitelistExceptIdleAppIdArray = buildAppIdArray(mPowerSaveWhitelistAppsExceptIdle,
                mPowerSaveWhitelistUserApps, mPowerSaveWhitelistExceptIdleAppIds);
        mPowerSaveWhitelistAllAppIdArray = buildAppIdArray(mPowerSaveWhitelistApps,
                mPowerSaveWhitelistUserApps, mPowerSaveWhitelistAllAppIds);
        mPowerSaveWhitelistUserAppIdArray = buildAppIdArray(null,
                mPowerSaveWhitelistUserApps, mPowerSaveWhitelistUserAppIds);
        if (mLocalActivityManager != null) {
            // 设置activity白名单
            mLocalActivityManager.setDeviceIdleWhitelist(mPowerSaveWhitelistAllAppIdArray);
        }
        if (mLocalPowerManager != null) {
            // 设置wakelock白名单
            mLocalPowerManager.setDeviceIdleWhitelist(mPowerSaveWhitelistAllAppIdArray);
        }
        if (mLocalAlarmManager != null) {
            // 设置alarm白名单
            mLocalAlarmManager.setDeviceIdleUserWhitelist(mPowerSaveWhitelistUserAppIdArray);
        }
    }
```

#### 2.3 onBootPhase方法

```java
    @Override
    public void onBootPhase(int phase) {
        // PHASE_SYSTEM_SERVICES_READY：核心服务可被调用时才会进入
        if (phase == PHASE_SYSTEM_SERVICES_READY) {
            synchronized (this) {
                // 初始化...
                mAlarmManager = (AlarmManager) getContext().getSystemService(Context.ALARM_SERVICE);
                mBatteryStats = BatteryStatsService.getService();
                mLocalActivityManager = getLocalService(ActivityManagerInternal.class);
                mLocalPowerManager = getLocalService(PowerManagerInternal.class);
                mPowerManager = getContext().getSystemService(PowerManager.class);
                // 创建非计数唤醒锁deviceidle_maint
                mActiveIdleWakeLock = mPowerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
                        "deviceidle_maint");
                mActiveIdleWakeLock.setReferenceCounted(false);
                // 创建计数唤醒锁deviceidle_going_idle
                mGoingIdleWakeLock = mPowerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
                        "deviceidle_going_idle");
                mGoingIdleWakeLock.setReferenceCounted(true);
                mConnectivityService = (ConnectivityService)ServiceManager.getService(
                        Context.CONNECTIVITY_SERVICE);
                mLocalAlarmManager = getLocalService(AlarmManagerService.LocalService.class);
                mNetworkPolicyManager = INetworkPolicyManager.Stub.asInterface(
                        ServiceManager.getService(Context.NETWORK_POLICY_SERVICE));
                mSensorManager = (SensorManager) getContext().getSystemService(Context.SENSOR_SERVICE);
                // 默认为0，即表明Any Motion传感器不可用
                int sigMotionSensorId = getContext().getResources().getInteger(
                        com.android.internal.R.integer.config_autoPowerModeAnyMotionSensor);
                if (sigMotionSensorId > 0) {
                    mMotionSensor = mSensorManager.getDefaultSensor(sigMotionSensorId, true);
                }
                // 默认为false,即不支持使用手腕倾斜手势传感器
                if (mMotionSensor == null && getContext().getResources().getBoolean(
                        com.android.internal.R.bool.config_autoPowerModePreferWristTilt)) {
                    mMotionSensor = mSensorManager.getDefaultSensor(
                            Sensor.TYPE_WRIST_TILT_GESTURE, true);
                }
                // 启用重要运动触发传感器
                if (mMotionSensor == null) {
                    mMotionSensor = mSensorManager.getDefaultSensor(
                            Sensor.TYPE_SIGNIFICANT_MOTION, true);
                }
                // 默认为true，即默认在设备进入idle模式时获取高精度定位信息
                if (getContext().getResources().getBoolean(
                        com.android.internal.R.bool.config_autoPowerModePrefetchLocation)) {
                    mLocationManager = (LocationManager) getContext().getSystemService(
                            Context.LOCATION_SERVICE);
                    mLocationRequest = new LocationRequest()
                        .setQuality(LocationRequest.ACCURACY_FINE)
                        .setInterval(0)
                        .setFastestInterval(0)
                        .setNumUpdates(1);
                }
                // Any Motion检测的角度变化阈值，默认为200/100f
                float angleThreshold = getContext().getResources().getInteger(
                        com.android.internal.R.integer.config_autoPowerModeThresholdAngle) / 100f;
                mAnyMotionDetector = new AnyMotionDetector(
                        (PowerManager) getContext().getSystemService(Context.POWER_SERVICE),
                        mHandler, mSensorManager, this, angleThreshold);
                // 注册idle/lightidle监听事件
                mIdleIntent = new Intent(PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED);
                mIdleIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND);
                mLightIdleIntent = new Intent(PowerManager.ACTION_LIGHT_DEVICE_IDLE_MODE_CHANGED);
                mLightIdleIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND);
                // 电量变化广播
                IntentFilter filter = new IntentFilter();
                filter.addAction(Intent.ACTION_BATTERY_CHANGED);
                getContext().registerReceiver(mReceiver, filter);
                // 卸载应用广播
                filter = new IntentFilter();
                filter.addAction(Intent.ACTION_PACKAGE_REMOVED);
                filter.addDataScheme("package");
                getContext().registerReceiver(mReceiver, filter);
                // 联网广播
                filter = new IntentFilter();
                filter.addAction(ConnectivityManager.CONNECTIVITY_ACTION);
                getContext().registerReceiver(mReceiver, filter);
                // 亮灭屏广播
                filter = new IntentFilter();
                filter.addAction(Intent.ACTION_SCREEN_OFF);
                filter.addAction(Intent.ACTION_SCREEN_ON);
                getContext().registerReceiver(mInteractivityReceiver, filter);
                // 设置activity,wakelock, alarm应用白名单
                mLocalActivityManager.setDeviceIdleWhitelist(mPowerSaveWhitelistAllAppIdArray);
                mLocalPowerManager.setDeviceIdleWhitelist(mPowerSaveWhitelistAllAppIdArray);
                mLocalAlarmManager.setDeviceIdleUserWhitelist(mPowerSaveWhitelistUserAppIdArray);
                // 监听屏幕状态相关信息
                updateInteractivityLocked();
            }
            // 更新连接状态相关信息
            updateConnectivityState(null);
        }
    }
```

```java
    void updateInteractivityLocked() {
        boolean screenOn = mPowerManager.isInteractive();
        if (!screenOn && mScreenOn) {
            // 灭屏
            mScreenOn = false;
            if (!mForceIdle) {
                // 进入doze模式
                becomeInactiveIfAppropriateLocked();
            }
        } else if (screenOn) {
            // 亮屏
            mScreenOn = true;
            if (!mForceIdle) {
                // 退出doze模式
                becomeActiveLocked("screen", Process.myUid());
            }
        }
    }
```
