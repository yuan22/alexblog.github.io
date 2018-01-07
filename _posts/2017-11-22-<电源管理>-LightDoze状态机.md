### 一、Light Doze状态机

![](https://xiezeyangcn.github.io/xzyblog.github.io/assets/images/2017-11-22-light-doze-mode/doze_state_n_light.bmp)


### 二、Light Doze状态切换

#### 2.1 调用becomeInactiveIfAppropriateLocked方法判断是否允许设备进入LIGHT_INACTIVE状态

```java
    void becomeInactiveIfAppropriateLocked() {
        // 灭屏&未充电，或强制空闲
        if ((!mScreenOn && !mCharging) || mForceIdle) {
        // 关闭屏幕，将变成inactive状态，开始等待最终是否会进入idle状态
        // mState为ACTIVE状态，mDeepEnabled从config文件读取config_enableAutoPowerModes字段，默认为false，GCM置为true.
        if (mState == STATE_ACTIVE && mDeepEnabled) {
            mState = STATE_INACTIVE;
            if (DEBUG) Slog.d(TAG, "Moved from STATE_ACTIVE to STATE_INACTIVE");
            resetIdleManagementLocked();
            scheduleAlarmLocked(mInactiveTimeout, false);
            EventLogTags.writeDeviceIdle(mState, "no activity");
        }
        // mLightEnabled值同mDeepEnabled取值，mLightState为LIGHT_ACTIVE状态
        if (mLightState == LIGHT_STATE_ACTIVE && mLightEnabled) {
            mLightState = LIGHT_STATE_INACTIVE;
            if (DEBUG) Slog.d(TAG, "Moved from LIGHT_STATE_ACTIVE to LIGHT_STATE_INACTIVE");
            resetLightIdleManagementLocked();
            // 默认5分钟
            scheduleLightAlarmLocked(mConstants.LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT);
            EventLogTags.writeDeviceIdleLight(mLightState, "no activity");
        }
        }
    }
```

&emsp;&emsp;关于config_enableAutoPowerModes字段，在GMS包中有做overlay处理，以便通过FCM推送：
```xml
    gms/products/gms_overlay/frameworks/base/core/res/res/values/config.xml
    <bool name="config_enableAutoPowerModes">true</bool>
```

#### 2.2 调用resetLightIdleManagementLocked方法重置之前设置过的轻度alarm等状态

```java
    void resetLightIdleManagementLocked() {
        // 取消轻度Alarm唤醒
        cancelLightAlarmLocked();
    }
```

#### 2.3 调用scheduleLightAlarmLocked方法进行状态切换

```java
    void scheduleLightAlarmLocked(long delay) {
        if (DEBUG) Slog.d(TAG, "scheduleLightAlarmLocked(" + delay + ")");
        mNextLightAlarmTime = SystemClock.elapsedRealtime() + delay;
        // 设置LightAlarmListener监听器
        mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
            mNextLightAlarmTime, "DeviceIdleController.light", mLightAlarmListener, mHandler);
    }
```

#### 2.4 轻度休眠监听器LightAlarmListener

```java
    private final AlarmManager.OnAlarmListener mLightAlarmListener
        = new AlarmManager.OnAlarmListener() {
        @Override
        public void onAlarm() {
        synchronized (DeviceIdleController.this) {
            // 状态切换操作
            stepLightIdleStateLocked("s:alarm");
        }
        }
    };
```

#### 2.5 调用stepLightIdleStateLocked处理状态切换(LIGHT_STATE_INACTIVE -> LIGHT_STATE_PRE_IDLE)

```java
    void stepLightIdleStateLocked(String reason) {
        // 如果进入深度休眠，则不予处理轻度休眠操作。
        if (mLightState == LIGHT_STATE_OVERRIDE) {
        return;
        }
        ......
        switch (mLightState) {
        case LIGHT_STATE_INACTIVE:
            // LIGHT_IDLE_MAINTENANCE最小预估时间：1 * 60 * 1000（1分钟）
            mCurIdleBudget = mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET;
            // 重置即将到来的空闲延迟时间：默认5 * 60 * 1000（5分钟）
            mNextLightIdleDelay = mConstants.LIGHT_IDLE_TIMEOUT;
            mMaintenanceStartTime = 0;
            if (!isOpsInactiveLocked()) {
            // 在首次idle状态之前要做一些处理.
            mLightState = LIGHT_STATE_PRE_IDLE;
            EventLogTags.writeDeviceIdleLight(mLightState, reason);
            // LIGHT_PRE_IDLE默认时间：10 * 60 * 1000（10分钟）
            scheduleLightAlarmLocked(mConstants.LIGHT_PRE_IDLE_TIMEOUT);
            break;
            }
        ......
        }
    }
```

```xml
    LIGHT_IDLE_MAINTENANCE_MIN_BUDGET = mParser.getLong( KEY_LIGHT_IDLE_MAINTENANCE_MIN_BUDGET, !COMPRESS_TIME ? 1 * 60 * 1000L : 15 * 1000L);
    LIGHT_IDLE_TIMEOUT = mParser.getLong(KEY_LIGHT_IDLE_TIMEOUT, !COMPRESS_TIME ? 5 * 60 * 1000L : 15 * 1000L);
    LIGHT_PRE_IDLE_TIMEOUT = mParser.getLong(KEY_LIGHT_PRE_IDLE_TIMEOUT, !COMPRESS_TIME ? 10 * 60 * 1000L : 30 * 1000L);
```

#### 2.6 调用stepLightIdleStateLocked处理状态切换(LIGHT_STATE_PRE_IDLE -> LIGHT_STATE_IDLE)

```java
    void stepLightIdleStateLocked(String reason) {
        ......
        switch (mLightState) {
        ......
        case LIGHT_STATE_PRE_IDLE:
            // mMaintenanceStartTime为空不进入
            .....
            mMaintenanceStartTime = 0;
            // 5 * 60 * 1000 * (2 ^ n) 
            scheduleLightAlarmLocked(mNextLightIdleDelay);
            // min(15 * 60 * 1000, 5 * 60 * 1000 * (2 ^ n) )
            mNextLightIdleDelay = Math.min(mConstants.LIGHT_MAX_IDLE_TIMEOUT,
                (long)(mNextLightIdleDelay * mConstants.LIGHT_IDLE_FACTOR));
            // 确保mNextLightIdleDelay始终大于默认值
            ......
            if (DEBUG) Slog.d(TAG, "Moved to LIGHT_STATE_IDLE.");
            mLightState = LIGHT_STATE_IDLE;
            EventLogTags.writeDeviceIdleLight(mLightState, reason);
            addEvent(EVENT_LIGHT_IDLE);
            // 持有PARTIAL_WAKE_LOCK锁（Cpu唤醒，屏幕/键盘关）
            mGoingIdleWakeLock.acquire();
            // 发送MSG_REPORT_IDLE_ON_LIGHT，准备进入轻度IDLE模式
            mHandler.sendEmptyMessage(MSG_REPORT_IDLE_ON_LIGHT);
            break;
        ......
        }
    }
```

#### 2.7 发送MSG_REPORT_IDLE_ON_LIGHT消息，准备进入LIGHT_IDLE模式

```java
    @Override
    public void handleMessage(Message msg) {
        if (DEBUG) Slog.d(TAG, "handleMessage(" + msg.what + ")");
        switch (msg.what) {
        ......
        case MSG_REPORT_IDLE_ON_LIGHT: {
            // 持有mGoingIdleWakeLock
            EventLogTags.writeDeviceIdleOnStart();
            final boolean deepChanged;
            final boolean lightChanged;
            if (msg.what == MSG_REPORT_IDLE_ON) {
            deepChanged = mLocalPowerManager.setDeviceIdleMode(true);
            lightChanged = mLocalPowerManager.setLightDeviceIdleMode(false);
            // 打开轻度休眠，关闭深度休眠
            } else {
            deepChanged = mLocalPowerManager.setDeviceIdleMode(false);
            lightChanged = mLocalPowerManager.setLightDeviceIdleMode(true);
            }
            try {
            // 网络服务处于休眠模式
            mNetworkPolicyManager.setDeviceIdleMode(true);
            // 电池统计服务处于轻度休眠模式
            mBatteryStats.noteDeviceIdleMode(msg.what == MSG_REPORT_IDLE_ON
                ? BatteryStats.DEVICE_IDLE_MODE_DEEP
                : BatteryStats.DEVICE_IDLE_MODE_LIGHT, null, Process.myUid());
            } catch (RemoteException e) {
            }
            if (deepChanged) {
            getContext().sendBroadcastAsUser(mIdleIntent, UserHandle.ALL);
            }
            if (lightChanged) {
            getContext().sendBroadcastAsUser(mLightIdleIntent, UserHandle.ALL);
            }
            EventLogTags.writeDeviceIdleOnComplete();
            // 释放mGoingIdleWakeLock
            mGoingIdleWakeLock.release();
        } break;
        ......
        }
    }
```

#### 2.8 调用stepLightIdleStateLocked处理状态切换(LIGHT_STATE_PRE_IDLE -> LIGHT_STATE_WAITING_FOR_NETWORK)

```java
    void stepLightIdleStateLocked(String reason) {
        ......
        switch (mLightState) {
        ......
        case LIGHT_STATE_PRE_IDLE:
            if (mNetworkConnected || mLightState == LIGHT_STATE_WAITING_FOR_NETWORK) {
            ......
            } else {
            // We'd like to do maintenance, but currently don't have network
            //　想继续保持，但是没有网络，尝试等待直至网络重连，直到超过一个idle周期放弃。
            // min(15 * 60 * 1000, 5 * 60 * 1000 * (2 ^ n) )
            scheduleLightAlarmLocked(mNextLightIdleDelay);
            if (DEBUG) Slog.d(TAG, "Moved to LIGHT_WAITING_FOR_NETWORK.");
            mLightState = LIGHT_STATE_WAITING_FOR_NETWORK;
            EventLogTags.writeDeviceIdleLight(mLightState, reason);
            }
        ......
        }
    }
```

#### 2.9 调用stepLightIdleStateLocked处理状态切换(LIGHT_STATE_WAITING_FOR_NETWORK -> LIGHT_STATE_IDLE_MAINTENANCE)

```java
    void stepLightIdleStateLocked(String reason) {
        ......
        switch (mLightState) {
        ......
        case LIGHT_STATE_WAITING_FOR_NETWORK:
            if (mNetworkConnected || mLightState == LIGHT_STATE_WAITING_FOR_NETWORK) {
            mActiveIdleOpCount = 1;
            mActiveIdleWakeLock.acquire();
            mMaintenanceStartTime = SystemClock.elapsedRealtime();
            // Maintenance状态最小预估时间１分钟，最长预估时间５分钟
            ......
            // 取值1 - 5分钟
            scheduleLightAlarmLocked(mCurIdleBudget);
            if (DEBUG) Slog.d(TAG,
                "Moved from LIGHT_STATE_IDLE to LIGHT_STATE_IDLE_MAINTENANCE.");
            mLightState = LIGHT_STATE_IDLE_MAINTENANCE;
            EventLogTags.writeDeviceIdleLight(mLightState, reason);
            addEvent(EVENT_LIGHT_MAINTENANCE);
            // 发送MSG_REPORT_IDLE_OFF消息，退出深度IDLE模式
            mHandler.sendEmptyMessage(MSG_REPORT_IDLE_OFF);
            } else {
            ......
            }
        ......
        }
    }
```

#### 2.10 发送MSG_REPORT_IDLE_OFF消息，准备退出LIGHT_IDLE模式

```java
    @Override
    public void handleMessage(Message msg) {
        if (DEBUG) Slog.d(TAG, "handleMessage(" + msg.what + ")");
        switch (msg.what) {
        ......
        case MSG_REPORT_IDLE_OFF: {
            // 持有mActiveIdleWakeLock
            EventLogTags.writeDeviceIdleOffStart("unknown");
            final boolean deepChanged = mLocalPowerManager.setDeviceIdleMode(false);
            final boolean lightChanged = mLocalPowerManager.setLightDeviceIdleMode(false);
            try {
            mNetworkPolicyManager.setDeviceIdleMode(false);
            mBatteryStats.noteDeviceIdleMode(BatteryStats.DEVICE_IDLE_MODE_OFF,
                null, Process.myUid());
            } catch (RemoteException e) {
            }
            if (deepChanged) {
            incActiveIdleOps();
            getContext().sendOrderedBroadcastAsUser(mIdleIntent, UserHandle.ALL,
                null, mIdleStartedDoneReceiver, null, 0, null, null);
            }
            if (lightChanged) {
            incActiveIdleOps();
            getContext().sendOrderedBroadcastAsUser(mLightIdleIntent, UserHandle.ALL,
                null, mIdleStartedDoneReceiver, null, 0, null, null);
            }
            decActiveIdleOps();
            EventLogTags.writeDeviceIdleOffComplete();
        } break;
        ......
        }
    }
```

#### 2.11 调用decActiveIdleOps方法退出Maintenance状态

```java
    void decActiveIdleOps() {
        synchronized (this) {
        mActiveIdleOpCount--;
        if (mActiveIdleOpCount <= 0) {
            exitMaintenanceEarlyIfNeededLocked();
            mActiveIdleWakeLock.release();
        }
        }
    }
```

#### 2.12 调用exitMaintenanceEarlyIfNeededLocked方法处理退出Maintenance一系列操作

```java
    void exitMaintenanceEarlyIfNeededLocked() {
        if (mState == STATE_IDLE_MAINTENANCE || mLightState == LIGHT_STATE_IDLE_MAINTENANCE
            || mLightState == LIGHT_STATE_PRE_IDLE) {
        if (isOpsInactiveLocked()) {
            final long now = SystemClock.elapsedRealtime();
            if (DEBUG) {
            StringBuilder sb = new StringBuilder();
            sb.append("Exit: start=");
            TimeUtils.formatDuration(mMaintenanceStartTime, sb);
            sb.append(" now=");
            TimeUtils.formatDuration(now, sb);
            Slog.d(TAG, sb.toString());
            }
            if (mState == STATE_IDLE_MAINTENANCE) {
            stepIdleStateLocked("s:early");
            } else if (mLightState == LIGHT_STATE_PRE_IDLE) {
            stepLightIdleStateLocked("s:predone");
            } else {
            stepLightIdleStateLocked("s:early");
            }
        }
        }
    }
```

#### 2.13 调用stepLightIdleStateLocked处理状态切换(LIGHT_STATE_IDLE_MAINTENANCE -> LIGHT_STATE_IDLE)

```java
    void stepLightIdleStateLocked(String reason) {
        ......
        switch (mLightState) {
        ......
        case LIGHT_STATE_IDLE_MAINTENANCE:
            // mMaintenanceStartTime不为空时调整最小预估时间
            if (mMaintenanceStartTime != 0) {
            long duration = SystemClock.elapsedRealtime() - mMaintenanceStartTime;
            if (duration < mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET) {
                mCurIdleBudget += (mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET-duration);
            } else {
                mCurIdleBudget -= (duration-mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET);
            }
            }
            mMaintenanceStartTime = 0;
            // 5 * 60 * 1000 * (2 ^ n) 
            scheduleLightAlarmLocked(mNextLightIdleDelay);
            // min(15 * 60 * 1000, 5 * 60 * 1000 * (2 ^ n) )
            mNextLightIdleDelay = Math.min(mConstants.LIGHT_MAX_IDLE_TIMEOUT,
                (long)(mNextLightIdleDelay * mConstants.LIGHT_IDLE_FACTOR));
            // 确保mNextLightIdleDelay始终大于默认值
            ......
            if (DEBUG) Slog.d(TAG, "Moved to LIGHT_STATE_IDLE.");
            mLightState = LIGHT_STATE_IDLE;
            EventLogTags.writeDeviceIdleLight(mLightState, reason);
            addEvent(EVENT_LIGHT_IDLE);
            // 持有PARTIAL_WAKE_LOCK锁（Cpu唤醒，屏幕/键盘关）
            mGoingIdleWakeLock.acquire();
            // 发送MSG_REPORT_IDLE_ON_LIGHT，准备进入轻度IDLE模式
            mHandler.sendEmptyMessage(MSG_REPORT_IDLE_ON_LIGHT);
            break;
        ......
        }
    }
```