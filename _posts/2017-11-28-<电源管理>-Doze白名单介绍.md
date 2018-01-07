### 一、三种情况处理

#### 1.1 System-excidle (system app) : used for doze

```
    ArrayMap<String, Interger> mPowerSaveWhitelistAppsExceptIdle
    String: packageName
    Interger: app Uid
    From etc/permissions/platform.xml tag: allow-in-power-save-except-idle (SystemConfig.mAllowInPowerSaveExceptIdle)
```

#### 1.2 System (system app) : used for doze & app standby

```
    ArrayMap<String, Interger> mPowerSaveWhitelistApps
    From etc/permissions/platform.xml tag : allow-in-power-save (SystemConfig.mAllowInPowerSave)
```


#### 1.3 User :  used for doze & app standby

```
    ArrayMap<String,Interger> mPowerSaveWhitelistUserApps
    From data/system/deviceidle.xml tag: wl
    The app uid’s array will be set to PowerMS as wakelock whitelist
```


### 二、Doze白名单

#### 2.1 onStart方法-Doze白名单服务

```java
    public void onStart() {
        final PackageManager pm = getContext().getPackageManager();
        synchronized (this) {
        // 默认为false,需要GCM相关服务支持；true表示打开自动省电模式。
        mLightEnabled = mDeepEnabled = getContext().getResources().getBoolean(
            com.android.internal.R.bool.config_enableAutoPowerModes);
        SystemConfig sysConfig = SystemConfig.getInstance();
        ArraySet<String> allowPowerExceptIdle = sysConfig.getAllowInPowerSaveExceptIdle();
        for (int i=0; i < allowPowerExceptIdle.size(); i++) {
            String pkg = allowPowerExceptIdle.valueAt(i);
            try {
            ApplicationInfo ai = pm.getApplicationInfo(pkg,
                PackageManager.MATCH_SYSTEM_ONLY);
            int appid = UserHandle.getAppId(ai.uid);
            mPowerSaveWhitelistAppsExceptIdle.put(ai.packageName, appid);
            mPowerSaveWhitelistSystemAppIdsExceptIdle.put(appid, true);
            } catch (PackageManager.NameNotFoundException e) {
            }
        }
        ArraySet<String> allowPower = sysConfig.getAllowInPowerSave();
        for (int i=0; i< allowPower.size(); i++) {
            String pkg = allowPower.valueAt(i);
            try {
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
        mConstants = new Constants(mHandler, getContext().getContentResolver());
        readConfigFileLocked();
        updateWhitelistAppIdsLocked();
        mNetworkConnected = true;
        mScreenOn = true;
        // 假定正在充电；电量下降时更新状态。
        mCharging = true;
        mState = STATE_ACTIVE;
        mLightState = LIGHT_STATE_ACTIVE;
        mInactiveTimeout = mConstants.INACTIVE_TIMEOUT;
        }
        mBinderService = new BinderService();
        publishBinderService(Context.DEVICE_IDLE_CONTROLLER, mBinderService);
        publishLocalService(LocalService.class, new LocalService());
    }
```


#### 2.2 BinderService - 处理Doze白名单

```java
    private final class BinderService extends IDeviceIdleController.Stub {
        @Override public void addPowerSaveWhitelistApp(String name) {
        getContext().enforceCallingOrSelfPermission(android.Manifest.permission.DEVICE_POWER,
            null);
        long ident = Binder.clearCallingIdentity();
        try {
            addPowerSaveWhitelistAppInternal(name);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
        }
        @Override public void removePowerSaveWhitelistApp(String name) {
        getContext().enforceCallingOrSelfPermission(android.Manifest.permission.DEVICE_POWER,
            null);
        long ident = Binder.clearCallingIdentity();
        try {
            removePowerSaveWhitelistAppInternal(name);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
        }
        @Override public String[] getSystemPowerWhitelistExceptIdle() {
        return getSystemPowerWhitelistExceptIdleInternal();
        }
        ......
    }
```

#### 2.3 addPowerSaveWhitelistAppInternal方法-将其加入到Doze白名单

```java
    public boolean addPowerSaveWhitelistAppInternal(String name) {
        synchronized (this) {
        try {
            ApplicationInfo ai = getContext().getPackageManager().getApplicationInfo(name,
                PackageManager.MATCH_UNINSTALLED_PACKAGES);
            if (mPowerSaveWhitelistUserApps.put(name, UserHandle.getAppId(ai.uid)) == null) {
            reportPowerSaveWhitelistChangedLocked();
            updateWhitelistAppIdsLocked();
            writeConfigFileLocked();
            }
            return true;
        } catch (PackageManager.NameNotFoundException e) {
            return false;
        }
        }
    }
```


#### 2.4 removePowerSaveWhitelistAppInternal方法-从Doze白名单中移除

```java
    public boolean removePowerSaveWhitelistAppInternal(String name) {
        synchronized (this) {
        if (mPowerSaveWhitelistUserApps.remove(name) != null) {
            reportPowerSaveWhitelistChangedLocked();
            updateWhitelistAppIdsLocked();
            writeConfigFileLocked();
            return true;
        }
        }
        return false;
    }
```


#### 2.5 reportPowerSaveWhitelistChangedLocked方法-发送Doze白名单变化广播

```java
    private void reportPowerSaveWhitelistChangedLocked() {
        Intent intent = new Intent(PowerManager.ACTION_POWER_SAVE_WHITELIST_CHANGED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        getContext().sendBroadcastAsUser(intent, UserHandle.SYSTEM);
    }
```


#### 2.6 updateWhitelistAppIdsLocked方法-更新Doze白名单信息

```java
    private void updateWhitelistAppIdsLocked() {
        mPowerSaveWhitelistExceptIdleAppIdArray = buildAppIdArray(mPowerSaveWhitelistAppsExceptIdle,
            mPowerSaveWhitelistUserApps, mPowerSaveWhitelistExceptIdleAppIds);
        mPowerSaveWhitelistAllAppIdArray = buildAppIdArray(mPowerSaveWhitelistApps,
            mPowerSaveWhitelistUserApps, mPowerSaveWhitelistAllAppIds);
        mPowerSaveWhitelistUserAppIdArray = buildAppIdArray(null,
            mPowerSaveWhitelistUserApps, mPowerSaveWhitelistUserAppIds);
        if (mLocalPowerManager != null) {
        if (DEBUG) {
            Slog.d(TAG, "Setting wakelock whitelist to "
                + Arrays.toString(mPowerSaveWhitelistAllAppIdArray));
        }
        mLocalPowerManager.setDeviceIdleWhitelist(mPowerSaveWhitelistAllAppIdArray);
        }
        if (mLocalAlarmManager != null) {
        if (DEBUG) {
            Slog.d(TAG, "Setting alarm whitelist to "
                + Arrays.toString(mPowerSaveWhitelistUserAppIdArray));
        }
        mLocalAlarmManager.setDeviceIdleUserWhitelist(mPowerSaveWhitelistUserAppIdArray);
        }
    }
```


#### 2.7 writeConfigFileLocked方法-写入config文件

```java
    void writeConfigFileLocked() {
        mHandler.removeMessages(MSG_WRITE_CONFIG);
        mHandler.sendEmptyMessageDelayed(MSG_WRITE_CONFIG, 5000);
    }
```



