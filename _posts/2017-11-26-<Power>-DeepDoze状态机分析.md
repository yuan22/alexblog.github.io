### 一、ACTIVE -> INACTIVE -> IDLE_PENDING状态切换

状态机：
![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2017-11-26-deep-doze-mode/doze_state_n_deep.bmp)

时序图：

![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2017-11-26-deep-doze-mode/active_inactive_idlepending.bmp)

#### 1.1 注册接收电池状态变化广播ACTION_BATTERY_CHANGED

```java
    private final BroadcastReceiver mReceiver = new BroadcastReceiver() {
    @Override public void onReceive(Context context, Intent intent) {
        switch (intent.getAction()) {
        ......
        case Intent.ACTION_BATTERY_CHANGED: {
            synchronized (DeviceIdleController.this) {
            int plugged = intent.getIntExtra("plugged", 0);
            updateChargingLocked(plugged != 0);
            }
        } break;
        ......
        }
    }
    };
```

#### 1.2 调用updateChargingLocked方法更新充电状态

```java
    void updateChargingLocked(boolean charging) {
        if (DEBUG) Slog.i(TAG, "updateChargingLocked: charging=" + charging);
        // 从充电->不充电状态
        if (!charging && mCharging) {
        // 将当前充电状态置为false
        mCharging = false;
        // 用于dumpsys调试时强制写入的状态
        if (!mForceIdle) {
            becomeInactiveIfAppropriateLocked();
        }
        } else if (charging) {
        mCharging = charging;
        if (!mForceIdle) {
            becomeActiveLocked("charging", Process.myUid());
        }
        }
    }
```

#### 1.3 调用becomeInactiveIfAppropriateLocked方法判断是否允许设备进入INACTIVE状态

```java
    void becomeInactiveIfAppropriateLocked() {
        // 灭屏&未充电，或强制空闲
        if ((!mScreenOn && !mCharging) || mForceIdle) {
        // 关闭屏幕，将变成inactive状态，开始等待最终是否会进入idle状态
        // mState为ACTIVE状态，mDeepEnabled从config文件读取config_enableAutoPowerModes字段，默认为false
        if (mState == STATE_ACTIVE && mDeepEnabled) {
            mState = STATE_INACTIVE;
            if (DEBUG) Slog.d(TAG, "Moved from STATE_ACTIVE to STATE_INACTIVE");
            resetIdleManagementLocked();
            scheduleAlarmLocked(mInactiveTimeout, false);
            EventLogTags.writeDeviceIdle(mState, "no activity");
        }
        if (mLightState == LIGHT_STATE_ACTIVE && mLightEnabled) {
            mLightState = LIGHT_STATE_INACTIVE;
            if (DEBUG) Slog.d(TAG, "Moved from LIGHT_STATE_ACTIVE to LIGHT_STATE_INACTIVE");
            resetLightIdleManagementLocked();
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

#### 1.4 屏幕状态变化监听器DisplayListener

```java
    private final DisplayManager.DisplayListener mDisplayListener
        = new DisplayManager.DisplayListener() {
        @Override public void onDisplayAdded(int displayId) {
        }
        @Override public void onDisplayRemoved(int displayId) {
        }
        @Override public void onDisplayChanged(int displayId) {
        if (displayId == Display.DEFAULT_DISPLAY) {
            synchronized (DeviceIdleController.this) {
            updateDisplayLocked();
            }
        }
        }
    };
```

#### 1.5 调用updateDisplayLocked方法更新显示器状态

```java
    void updateDisplayLocked() {
        mCurDisplay = mDisplayManager.getDisplay(Display.DEFAULT_DISPLAY);
        // 考虑到显示器显示某些东西的情况，因为如果显示任何东西都需频繁去更新，所以不允许进入深度休眠
        boolean screenOn = mCurDisplay.getState() == Display.STATE_ON;
        // 亮屏->灭屏
        if (!screenOn && mScreenOn) {
        mScreenOn = false;
        if (!mForceIdle) {
            becomeInactiveIfAppropriateLocked();
        }
        } else if (screenOn) {
        mScreenOn = true;
        if (!mForceIdle) {
            becomeActiveLocked("screen", Process.myUid());
        }
        }
    }
```

#### 1.6 调用resetIdleManagementLocked方法重置之前设置过的alarm等状态

```java
    void resetIdleManagementLocked() {
        mNextIdlePendingDelay = 0;
        mNextIdleDelay = 0;
        mNextLightIdleDelay = 0;
        cancelAlarmLocked();
        cancelSensingTimeoutAlarmLocked();
        cancelLocatingLocked();
        stopMonitoringMotionLocked();
        mAnyMotionDetector.stop();
    }
```

#### 1.7 调用scheduleAlarmLocked方法进行状态切换

```java
    void scheduleAlarmLocked(long delay, boolean idleUntil) {
        if (mMotionSensor == null) {
        // 如果设备没有运动传感器，则不安排时钟，因为无法确定设备是否移动
        // 这有效关闭了设备空闲的正常执行，尽管仍然可能假装像关闭闹钟一样去拨动它
        return;
        }
        mNextAlarmTime = SystemClock.elapsedRealtime() + delay;
        // 设置AlarmListener监听器，调用stepIdleStateLocked方法做状态切换
        if (idleUntil) {
        mAlarmManager.setIdleUntil(AlarmManager.ELAPSED_REALTIME_WAKEUP,
            mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        } else {
        mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
            mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        }
    }
```

#### 1.8 深度休眠监听器

```java
    private final AlarmManager.OnAlarmListener mDeepAlarmListener
        = new AlarmManager.OnAlarmListener() {
        @Override
        public void onAlarm() {
        synchronized (DeviceIdleController.this) {
            stepIdleStateLocked("s:alarm");
        }
        }
    };
```

#### 1.9 调用stepIdleStateLocked处理STATE_INACTIVE->STATE_IDLE_PENDING状态切换

```java
    void stepIdleStateLocked(String reason) {
        ......
        switch (mState) {
        case STATE_INACTIVE:
            // 现在不活跃，开始寻找更多休眠、运动信息
            startMonitoringMotionLocked();
            // 30 * 60 * 1000L: 30分钟
            scheduleAlarmLocked(mConstants.IDLE_AFTER_INACTIVE_TIMEOUT, false);
            // 重置即将到来的空闲
            // 5 * 60 * 1000L: 5分钟
            mNextIdlePendingDelay = mConstants.IDLE_PENDING_TIMEOUT;
            // 60 * 60 * 1000L: 60分钟
            mNextIdleDelay = mConstants.IDLE_TIMEOUT;
            mState = STATE_IDLE_PENDING;
            if (DEBUG) Slog.d(TAG, "Moved from STATE_INACTIVE to STATE_IDLE_PENDING.");
            EventLogTags.writeDeviceIdle(mState, reason);
            break;
        ......
    }
```

```
    long idleAfterInactiveTimeout = (mHasWatch ? 15 : 30) * 60 * 1000L;
    IDLE_AFTER_INACTIVE_TIMEOUT = mParser.getLong(KEY_IDLE_AFTER_INACTIVE_TIMEOUT,!COMPRESS_TIME ? idleAfterInactiveTimeout : (idleAfterInactiveTimeout / 10));

    IDLE_PENDING_TIMEOUT = mParser.getLong(KEY_IDLE_PENDING_TIMEOUT, !COMPRESS_TIME ? 5 * 60 * 1000L : 30 * 1000L);

    IDLE_TIMEOUT = mParser.getLong(KEY_IDLE_TIMEOUT, !COMPRESS_TIME ? 60 * 60 * 1000L : 6 * 60 * 1000L);
```

#### 1.10 调用startMonitoringMotionLocked方法注册motion传感器获取运动信息

```java
    void startMonitoringMotionLocked() {
        if (DEBUG) Slog.d(TAG, "startMonitoringMotionLocked()");
        // 注册sensor监听器
        if (mMotionSensor != null && !mMotionListener.active) {
        mMotionListener.registerLocked();
        }
    }

    public boolean registerLocked() {
        boolean success;
        if (mMotionSensor.getReportingMode() == Sensor.REPORTING_MODE_ONE_SHOT) {
        success = mSensorManager.requestTriggerSensor(mMotionListener, mMotionSensor);
        } else {
        success = mSensorManager.registerListener(
            mMotionListener, mMotionSensor, SensorManager.SENSOR_DELAY_NORMAL);
        }
        if (success) {
        active = true;
        } else {
        Slog.e(TAG, "Unable to register for " + mMotionSensor);
        }
        return success;
    }
```


#### 1.11 调用scheduleAlarmLocked方法进行状态切换

```java
    void scheduleAlarmLocked(long delay, boolean idleUntil) {
        if (DEBUG) Slog.d(TAG, "scheduleAlarmLocked(" + delay + ", " + idleUntil + ")");
        if (mMotionSensor == null) {
        // 如果设备没有运动传感器，则不安排时钟，因为无法确定设备是否移动
        // 这有效关闭了设备空闲的正常执行，尽管仍然可能假装像关闭闹钟一样去拨动它
        return;
        }
        mNextAlarmTime = SystemClock.elapsedRealtime() + delay;
        // 设置AlarmListener监听器，调用stepIdleStateLocked方法做状态切换
        if (idleUntil) {
        mAlarmManager.setIdleUntil(AlarmManager.ELAPSED_REALTIME_WAKEUP,
            mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        } else {
        mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
            mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        }
    }
```

### 二、IDLE_PENDING -> SENSING -> LOCATING状态切换

时序图：

![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2017-11-26-deep-doze-mode/idlepending_sensing.bmp)

![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2017-11-26-deep-doze-mode/sensing_locating.bmp)

#### 2.1 stepIdleStateLocked

```java
    void stepIdleStateLocked(String reason) {
        ......
        switch (mState) {
        ......
        case STATE_IDLE_PENDING:
            mState = STATE_SENSING;
            if (DEBUG) Slog.d(TAG, "Moved from STATE_IDLE_PENDING to STATE_SENSING.");
            EventLogTags.writeDeviceIdle(mState, reason);
            // 4 * 60 * 1000L: 4分钟
            scheduleSensingTimeoutAlarmLocked(mConstants.SENSING_TIMEOUT);
            cancelLocatingLocked();
            mNotMoving = false;
            mLocated = false;
            mLastGenericLocation = null;
            mLastGpsLocation = null;
            mAnyMotionDetector.checkForAnyMotion();
            break;
        ......
        }
    }
```

#### 2.2 scheduleSensingTimeoutAlarmLocked方法设置Alarm监听器

```java
    void scheduleSensingTimeoutAlarmLocked(long delay) {
        if (DEBUG) Slog.d(TAG, "scheduleSensingAlarmLocked(" + delay + ")");
        mNextSensingTimeoutAlarmTime = SystemClock.elapsedRealtime() + delay;
        mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP, mNextSensingTimeoutAlarmTime,
        "DeviceIdleController.sensing", mSensingTimeoutAlarmListener, mHandler);
    }
```

#### 2.3 cancelLocatingLocked方法取消位置监听

```java
    void cancelLocatingLocked() {
        if (mLocating) {
        // STATE_SENSING下注册mGenericLocationListener、mGpsLocationListener监听器
        mLocationManager.removeUpdates(mGenericLocationListener);
        mLocationManager.removeUpdates(mGpsLocationListener);
        mLocating = false;
        }
    }
```

#### 2.4 checkForAnyMotion方法确定anymotion状态

```java
    public void checkForAnyMotion() {
        if (mState != STATE_ACTIVE) {
        synchronized (mLock) {
            mState = STATE_ACTIVE;
            if (DEBUG) {
            Slog.d(TAG, "Moved from STATE_INACTIVE to STATE_ACTIVE.");
            }
            mCurrentGravityVector = null;
            mPreviousGravityVector = null;
            mWakeLock.acquire();
            Message wakelockTimeoutMsg = Message.obtain(mHandler, mWakelockTimeout);
            mHandler.sendMessageDelayed(wakelockTimeoutMsg, WAKELOCK_TIMEOUT_MILLIS);
            mWakelockTimeoutIsActive = true;
            startOrientationMeasurementLocked();
        }
        }
    }
```

#### 2.5 startOrientationMeasurementLocked进行行为检测

```java
    private void startOrientationMeasurementLocked() {
        if (DEBUG) Slog.d(TAG, "startOrientationMeasurementLocked: mMeasurementInProgress=" +
        mMeasurementInProgress + ", (mAccelSensor != null)=" + (mAccelSensor != null));
        if (!mMeasurementInProgress && mAccelSensor != null) {
        // 注册加速度传感器
        if (mSensorManager.registerListener(mListener, mAccelSensor,
            SAMPLING_INTERVAL_MILLIS * 1000)) {
            mMeasurementInProgress = true;
            mRunningStats.reset();
        }
        Message measurementTimeoutMsg = Message.obtain(mHandler, mMeasurementTimeout);
        mHandler.sendMessageDelayed(measurementTimeoutMsg, ACCELEROMETER_DATA_TIMEOUT_MILLIS);
        mMeasurementTimeoutIsActive = true;
        }
    }
```

#### 2.6 mMeasurementTimeout回调返回anymotion结果至DeviceIdleContrroller

```java
    private final Runnable mMeasurementTimeout = new Runnable() {
        @Override
        public void run() {
        int status = RESULT_UNKNOWN;
        synchronized (mLock) {
            if (mMeasurementTimeoutIsActive == true) {
            mMeasurementTimeoutIsActive = false;
            if (DEBUG) Slog.i(TAG, "mMeasurementTimeout. Failed to collect sufficient accel " +
                  "data within " + ACCELEROMETER_DATA_TIMEOUT_MILLIS + " ms. Stopping " +
                  "orientation measurement.");
            status = stopOrientationMeasurementLocked();
            if (status != RESULT_UNKNOWN) {
                mHandler.removeCallbacks(mWakelockTimeout);
                mWakelockTimeoutIsActive = false;
                mCallback.onAnyMotionResult(status);
            }
            }
        }
        }
    };
```

#### 2.7 onAnyMotionResult方法

```java
    public void onAnyMotionResult(int result) {
        if (DEBUG) Slog.d(TAG, "onAnyMotionResult(" + result + ")");
        if (result != AnyMotionDetector.RESULT_UNKNOWN) {
        synchronized (this) {
            cancelSensingTimeoutAlarmLocked();
        }
        }
        if ((result == AnyMotionDetector.RESULT_MOVED) ||
        (result == AnyMotionDetector.RESULT_UNKNOWN)) {
        synchronized (this) {
            handleMotionDetectedLocked(mConstants.INACTIVE_TIMEOUT, "non_stationary");
        }
        } else if (result == AnyMotionDetector.RESULT_STATIONARY) {
        if (mState == STATE_SENSING) {
            // 如果当前处于SENSING状态，则调用stepIdleStateLocked切换至LOCATING状态
            synchronized (this) {
            mNotMoving = true;
            stepIdleStateLocked("s:stationary");
            }
        }
        ......
        }
    }
```

#### 2.8 stepIdleStateLocked

```java
    void stepIdleStateLocked(String reason) {
        ......
        switch (mState) {
        ......
        case STATE_SENSING:
            cancelSensingTimeoutAlarmLocked();
            mState = STATE_LOCATING;
            if (DEBUG) Slog.d(TAG, "Moved from STATE_SENSING to STATE_LOCATING.");
            EventLogTags.writeDeviceIdle(mState, reason);
            // 30 * 1000L: 30s
            scheduleAlarmLocked(mConstants.LOCATING_TIMEOUT, false);
            if (mLocationManager != null
                && mLocationManager.getProvider(LocationManager.NETWORK_PROVIDER) != null) {
            // 精度为ACCURACY_FINE的定位服务
            mLocationManager.requestLocationUpdates(mLocationRequest,
                mGenericLocationListener, mHandler.getLooper());
            mLocating = true;
            } else {
            mHasNetworkLocation = false;
            }
            if (mLocationManager != null
                && mLocationManager.getProvider(LocationManager.GPS_PROVIDER) != null) {
            mHasGps = true;
            mLocationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 1000, 5,
                mGpsLocationListener, mHandler.getLooper());
            mLocating = true;
            } else {
            mHasGps = false;
            }
            if (mLocating) {
            break;
            }
        ......
        }
    }
```

#### 2.9 cancelSensingTimeoutAlarmLocked方法取消Alarm监听

```java
    void cancelSensingTimeoutAlarmLocked() {
        if (mNextSensingTimeoutAlarmTime != 0) {
        mNextSensingTimeoutAlarmTime = 0;
        mAlarmManager.cancel(mSensingTimeoutAlarmListener);
        }
    }
```

#### 2.10 scheduleAlarmLocked方法在有motion sensor的前提下设置DeepAlarm监听器

```java
    void scheduleAlarmLocked(long delay, boolean idleUntil) {
        if (DEBUG) Slog.d(TAG, "scheduleAlarmLocked(" + delay + ", " + idleUntil + ")");
        if (mMotionSensor == null) {
        // 如果设备没有运动传感器，则不安排时钟，因为无法确定设备是否移动
        // 这有效关闭了设备空闲的正常执行，尽管仍然可能假装像关闭闹钟一样去拨动它
        return;
        }
        mNextAlarmTime = SystemClock.elapsedRealtime() + delay;
        // 设置AlarmListener监听器，调用stepIdleStateLocked方法做状态切换
        if (idleUntil) {
        mAlarmManager.setIdleUntil(AlarmManager.ELAPSED_REALTIME_WAKEUP,
            mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        } else {
        mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
            mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        }
    }
```

#### 2.11 mGenericLocationListener位置监听器

```java
    private final LocationListener mGenericLocationListener = new LocationListener() {
        @Override
        public void onLocationChanged(Location location) {
        synchronized (DeviceIdleController.this) {
            receivedGenericLocationLocked(location);
        }
        }
        ......
    };
```

#### 2.12 receivedGenericLocationLocked方法监听L精度为OCATION_ACCURACY的定位服务

```java
    void receivedGenericLocationLocked(Location location) {
        if (mState != STATE_LOCATING) {
        cancelLocatingLocked();
        return;
        }
        if (DEBUG) Slog.d(TAG, "Generic location: " + location);
        mLastGenericLocation = new Location(location);
        if (location.getAccuracy() > mConstants.LOCATION_ACCURACY && mHasGps) {
        return;
        }
        mLocated = true;
        if (mNotMoving) {
        stepIdleStateLocked("s:location");
        }
    }
```

#### 2.13 mGpsLocationListener方法GPS位置监听

```java
    private final LocationListener mGpsLocationListener = new LocationListener() {
        @Override
        public void onLocationChanged(Location location) {
        synchronized (DeviceIdleController.this) {
            receivedGpsLocationLocked(location);
        }
        }
        ......
    };
```

#### 2.14 receivedGpsLocationLocked方法接收GPS位置信息

```java
    void receivedGpsLocationLocked(Location location) {
        if (mState != STATE_LOCATING) {
        cancelLocatingLocked();
        return;
        }
        if (DEBUG) Slog.d(TAG, "GPS location: " + location);
        mLastGpsLocation = new Location(location);
        if (location.getAccuracy() > mConstants.LOCATION_ACCURACY) {
        return;
        }
        mLocated = true;
        if (mNotMoving) {
        stepIdleStateLocked("s:gps");
        }
    }
```

### 三、LOCATING -> IDLE_MAINTENANCE --> IDLE --> IDLE_MAINTENANCE状态切换

时序图：

![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2017-11-26-deep-doze-mode/location_idlemaintenance_idle_idlemaintenance.bmp)

#### 3.1 stepIdleStateLocked

```java
    void stepIdleStateLocked(String reason) {
        ......
        switch (mState) {
        ......
        case STATE_LOCATING:
            cancelAlarmLocked();
            cancelLocatingLocked();
            mAnyMotionDetector.stop();
        ......
        }
    }
```

#### 3.2 cancelAlarmLocked方法取消DeepAlarm监听

```java
    void cancelAlarmLocked() {
        if (mNextAlarmTime != 0) {
        mNextAlarmTime = 0;
        mAlarmManager.cancel(mDeepAlarmListener);
        }
    }
```

#### 3.3 cancelLocatingLocked方法取消定位监听（Generic、Gps）

```java
    void cancelLocatingLocked() {
        if (mLocating) {
        // STATE_SENSING下注册mGenericLocationListener、mGpsLocationListener监听器
        mLocationManager.removeUpdates(mGenericLocationListener);
        mLocationManager.removeUpdates(mGpsLocationListener);
        mLocating = false;
        }
    }
```

#### 3.4 AnyMotionDetector函数stop方法回调onAnyMotionResult方法

```java
    public void stop() {
        synchronized (mLock) {
        if (mState == STATE_ACTIVE) {
            mState = STATE_INACTIVE;
            if (DEBUG) Slog.d(TAG, "Moved from STATE_ACTIVE to STATE_INACTIVE.");
        }
        mHandler.removeCallbacks(mMeasurementTimeout);
        mHandler.removeCallbacks(mSensorRestart);
        mMeasurementTimeoutIsActive = false;
        mSensorRestartIsActive = false;
        if (mMeasurementInProgress) {
            mMeasurementInProgress = false;
            mSensorManager.unregisterListener(mListener);
        }
        mCurrentGravityVector = null;
        mPreviousGravityVector = null;
        if (mWakeLock.isHeld()) {
            mHandler.removeCallbacks(mWakelockTimeout);
            mWakelockTimeoutIsActive = false;
            mWakeLock.release();
        }
        }
    }
```

#### 3.5 onAnyMotionResult方法调用stepIdleStateLocked切换状态

```java
    public void onAnyMotionResult(int result) {
        ......
        if ((result == AnyMotionDetector.RESULT_MOVED) ||
        (result == AnyMotionDetector.RESULT_UNKNOWN)) {
        synchronized (this) {
            handleMotionDetectedLocked(mConstants.INACTIVE_TIMEOUT, "non_stationary");
        }
        } else if (result == AnyMotionDetector.RESULT_STATIONARY) {
        if (mState == STATE_SENSING) {
            ......
        } else if (mState == STATE_LOCATING) {
            // 如果当前处于LOCATING状态，则通过mLocated来决定是否切换状态
            synchronized (this) {
            mNotMoving = true;
            if (mLocated) {
                stepIdleStateLocked("s:stationary");
            }
            }
        }
        }
    }
```

#### 3.6 stepIdleStateLocked方法切换至SDLE_MAINTENANCE状态

```java
    void stepIdleStateLocked(String reason) {
        ......
        switch (mState) {
        ......
        case STATE_IDLE_MAINTENANCE:
            scheduleAlarmLocked(mNextIdleDelay, true);
            if (DEBUG) Slog.d(TAG, "Moved to STATE_IDLE. Next alarm in " + mNextIdleDelay +
                " ms.");
            // 60分钟乘以2
            mNextIdleDelay = (long)(mNextIdleDelay * mConstants.IDLE_FACTOR);
            if (DEBUG) Slog.d(TAG, "Setting mNextIdleDelay = " + mNextIdleDelay);
            // 同6小时比较取小的
            mNextIdleDelay = Math.min(mNextIdleDelay, mConstants.MAX_IDLE_TIMEOUT);
            if (mNextIdleDelay < mConstants.IDLE_TIMEOUT) {
            mNextIdleDelay = mConstants.IDLE_TIMEOUT;
            }
            mState = STATE_IDLE;
            if (mLightState != LIGHT_STATE_OVERRIDE) {
            mLightState = LIGHT_STATE_OVERRIDE;
            cancelLightAlarmLocked();
            }
            EventLogTags.writeDeviceIdle(mState, reason);
            addEvent(EVENT_DEEP_IDLE);
            mGoingIdleWakeLock.acquire();
            mHandler.sendEmptyMessage(MSG_REPORT_IDLE_ON);
            break;
        ......
        }
    }
```

#### 3.7 scheduleAlarmLocked方法设置Alarm监听直至进入IDLE模式

```java
    void scheduleAlarmLocked(long delay, boolean idleUntil) {
        if (DEBUG) Slog.d(TAG, "scheduleAlarmLocked(" + delay + ", " + idleUntil + ")");
        if (mMotionSensor == null) {
        // 如果设备没有运动传感器，则不安排时钟，因为无法确定设备是否移动
        // 这有效关闭了设备空闲的正常执行，尽管仍然可能假装像关闭闹钟一样去拨动它
        return;
        }
        mNextAlarmTime = SystemClock.elapsedRealtime() + delay;
        // 设置AlarmListener监听器，调用stepIdleStateLocked方法做状态切换
        if (idleUntil) {
        mAlarmManager.setIdleUntil(AlarmManager.ELAPSED_REALTIME_WAKEUP,
            mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        } else {
        mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
            mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        }
    }
```


#### 3.8 cancelLightAlarmLocked方法取消LightAlarm监听

```java
    void cancelLightAlarmLocked() {
        if (mNextLightAlarmTime != 0) {
        mNextLightAlarmTime = 0;
        mAlarmManager.cancel(mLightAlarmListener);
        }
    }

    private final AlarmManager.OnAlarmListener mLightAlarmListener
        = new AlarmManager.OnAlarmListener() {
        @Override
        public void onAlarm() {
        synchronized (DeviceIdleController.this) {
            stepLightIdleStateLocked("s:alarm");
        }
        }
    };
```

#### 3.9 发送MSG_REPORT_IDLE_ON消息，退出LightIdle模式，进入DeepIdle模式

```java
    final class MyHandler extends Handler {
        ......
        @Override public void handleMessage(Message msg) {
        if (DEBUG) Slog.d(TAG, "handleMessage(" + msg.what + ")");
        switch (msg.what) {
            ......
            case MSG_REPORT_IDLE_ON:
            case MSG_REPORT_IDLE_ON_LIGHT: {
            EventLogTags.writeDeviceIdleOnStart();
            final boolean deepChanged;
            final boolean lightChanged;
            if (msg.what == MSG_REPORT_IDLE_ON) {
                deepChanged = mLocalPowerManager.setDeviceIdleMode(true);
                lightChanged = mLocalPowerManager.setLightDeviceIdleMode(false);
            } else {
                deepChanged = mLocalPowerManager.setDeviceIdleMode(false);
                lightChanged = mLocalPowerManager.setLightDeviceIdleMode(true);
            }
            try {
                mNetworkPolicyManager.setDeviceIdleMode(true);
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
            mGoingIdleWakeLock.release();
            } break;
            ......
        }
        }
    }
```

#### 3.10 stepIdleStateLocked方法切换至IDLE_MAINTENANCE状态

```java
    void stepIdleStateLocked(String reason) {
        ......
        switch (mState) {
        ......
        case STATE_IDLE:
            mActiveIdleOpCount = 1;
            mActiveIdleWakeLock.acquire();
            scheduleAlarmLocked(mNextIdlePendingDelay, false);
            if (DEBUG) Slog.d(TAG, "Moved from STATE_IDLE to STATE_IDLE_MAINTENANCE. " +
                "Next alarm in " + mNextIdlePendingDelay + " ms.");
            mMaintenanceStartTime = SystemClock.elapsedRealtime();
            mNextIdlePendingDelay = Math.min(mConstants.MAX_IDLE_PENDING_TIMEOUT,
                (long)(mNextIdlePendingDelay * mConstants.IDLE_PENDING_FACTOR));
            if (mNextIdlePendingDelay < mConstants.IDLE_PENDING_TIMEOUT) {
            mNextIdlePendingDelay = mConstants.IDLE_PENDING_TIMEOUT;
            }
            mState = STATE_IDLE_MAINTENANCE;
            EventLogTags.writeDeviceIdle(mState, reason);
            addEvent(EVENT_DEEP_MAINTENANCE);
            mHandler.sendEmptyMessage(MSG_REPORT_IDLE_OFF);
            break;
        }
    }
```

#### 3.11 发送MSG_REPORT_IDLE_OFF消息退出Idle模式

```java
    final class MyHandler extends Handler {
        ......
        @Override public void handleMessage(Message msg) {
        if (DEBUG) Slog.d(TAG, "handleMessage(" + msg.what + ")");
        switch (msg.what) {
            ......
            case MSG_REPORT_IDLE_OFF: {
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
    }
```
