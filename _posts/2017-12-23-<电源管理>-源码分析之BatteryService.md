
### 一、启动过程

#### 1.1 SystemServer启动BatteryService

```java
        private void startCoreServices() {
            mSystemServiceManager.startService(BatteryService.class);
        }
```

#### 1.2 BatteryService构造函数

```java
        public BatteryService(Context context) {
            super(context);
            mContext = context;
            mHandler = new Handler(true /*async*/);
            // 电源指示灯服务
            mLed = new Led(context, getLocalService(LightsManager.class));
            // 电量统计服务
            mBatteryStats = BatteryStatsService.getService();
            // 从ro.cutoff_voltage_mv属性计算截止电压(默认为3200mV)
            // 如果该属性小于0，则忽略关机逻辑
            mWeakChgCutoffVoltageMv = SystemProperties.getInt("ro.cutoff_voltage_mv", 0);
            if (mWeakChgCutoffVoltageMv > 2700)
               mVoltageNowFile = new File("/sys/class/power_supply/battery/voltage_now");
            // 严重低电量警告值:5%
            mCriticalBatteryLevel = mContext.getResources().getInteger(
                    com.android.internal.R.integer.config_criticalBatteryWarningLevel);
            // 低电量警告值:15%
            mLowBatteryWarningLevel = mContext.getResources().getInteger(
                    com.android.internal.R.integer.config_lowBatteryWarningLevel);
            // 低电量停止警告值:5%
            mLowBatteryCloseWarningLevel = mLowBatteryWarningLevel + mContext.getResources().getInteger(
                    com.android.internal.R.integer.config_lowBatteryCloseWarningBump);
            // 高温关机值:68度
            mShutdownBatteryTemperature = mContext.getResources().getInteger(
                    com.android.internal.R.integer.config_shutdownBatteryTemperature);
            // 如果invalid_charger开关存在，则关注不合法充电器信息
                if (new File("/sys/devices/virtual/switch/invalid_charger/state").exists()) {
                    UEventObserver invalidChargerObserver = new UEventObserver() {
                        @Override
                        public void onUEvent(UEvent event) {
                            final int invalidCharger = "1".equals(event.get("SWITCH_STATE")) ? 1 : 0;
                            synchronized (mLock) {
                                if (mInvalidCharger != invalidCharger) {
                                    mInvalidCharger = invalidCharger;
                                }
                            }
                        }
                    };
                    invalidChargerObserver.startObserving(
                            "DEVPATH=/devices/virtual/switch/invalid_charger");
                }
            }
```

#### 1.3 onStart方法

```java
        public void onStart() {
            // 获取电源属性服务
            IBinder b = ServiceManager.getService("batteryproperties");
            final IBatteryPropertiesRegistrar batteryPropertiesRegistrar =
                    IBatteryPropertiesRegistrar.Stub.asInterface(b);
            try {
                // 注册监听batteryProperties监听器
                batteryPropertiesRegistrar.registerListener(new BatteryListener());
            } catch (RemoteException e) {
            }
            mBinderService = new BinderService();
            // 将其注册到ServiceManager进程中
            publishBinderService("battery", mBinderService);
            publishLocalService(BatteryManagerInternal.class, new LocalService());
        }
```

#### 1.4 BatteryListener

```java
        private final class BatteryListener extends IBatteryPropertiesListener.Stub {
            @Override public void batteryPropertiesChanged(BatteryProperties props) {
                final long identity = Binder.clearCallingIdentity();
                try {
                    // 电源属性值变化后，回调update方法
                    BatteryService.this.update(props);
                } finally {
                    Binder.restoreCallingIdentity(identity);
                }
           }
        }
```

```java
        private void update(BatteryProperties props) {
            synchronized (mLock) {
                if (!mUpdatesStopped) {
                    mBatteryProps = props;
                    // 更新新的电源相关信息
                    processValuesLocked(false);
                } else {
                    mLastBatteryProps.set(props);
                }
            }
        }
```

### 二、onBootPhase

#### 2.1 onBootPhase方法

```java
        public void onBootPhase(int phase) {
            if (phase == PHASE_ACTIVITY_MANAGER_READY) {
                // 检查当前电源状况，安全显示关机对话框
                synchronized (mLock) {
                    ContentObserver obs = new ContentObserver(mHandler) {
                        @Override
                        public void onChange(boolean selfChange) {
                            synchronized (mLock) {
                                updateBatteryWarningLevelLocked();
                            }
                        }
                    };
                    final ContentResolver resolver = mContext.getContentResolver();
                    resolver.registerContentObserver(Settings.Global.getUriFor(
                            Settings.Global.LOW_POWER_MODE_TRIGGER_LEVEL),
                            false, obs, UserHandle.USER_ALL);
                    updateBatteryWarningLevelLocked();
                }
            }
        }
```

#### 2.2 updateBatteryWarningLevelLocked方法

```java
        // 更新电量警告值，若用户未改变电量值则为默认值5%，若用户设置值小于严重警告值则取严重警告值5%
        // 否则关闭低电量警告为用户设置的低电量警告值+5%
        private void updateBatteryWarningLevelLocked() {
            final ContentResolver resolver = mContext.getContentResolver();
            int defWarnLevel = mContext.getResources().getInteger(
                    com.android.internal.R.integer.config_lowBatteryWarningLevel);
            mLowBatteryWarningLevel = Settings.Global.getInt(resolver,
                    Settings.Global.LOW_POWER_MODE_TRIGGER_LEVEL, defWarnLevel);
            if (mLowBatteryWarningLevel == 0) {
                mLowBatteryWarningLevel = defWarnLevel;
            }
            if (mLowBatteryWarningLevel < mCriticalBatteryLevel) {
                mLowBatteryWarningLevel = mCriticalBatteryLevel;
            }
            mLowBatteryCloseWarningLevel = mLowBatteryWarningLevel + mContext.getResources().getInteger(
                    com.android.internal.R.integer.config_lowBatteryCloseWarningBump);
            processValuesLocked(true);
        }
```

### 三、processValuesLocked

#### 3.1 processValuesLocked方法

```java
        private void processValuesLocked(boolean force) {
            boolean logOutlier = false;
            long dischargeDuration = 0;
            mBatteryLevelCritical = (mBatteryProps.batteryLevel <= mCriticalBatteryLevel);
            // 电源充电
            if (mBatteryProps.chargerAcOnline) {
                mPlugType = BatteryManager.BATTERY_PLUGGED_AC;
            // USB充电
            } else if (mBatteryProps.chargerUsbOnline) {
                mPlugType = BatteryManager.BATTERY_PLUGGED_USB;
            // 无线充电
            } else if (mBatteryProps.chargerWirelessOnline) {
                mPlugType = BatteryManager.BATTERY_PLUGGED_WIRELESS;
            } else {
            // 未充电
                mPlugType = BATTERY_PLUGGED_NONE;
            }
            // 追踪统计当前电量值
            try {
                mBatteryStats.setBatteryState(mBatteryProps.batteryStatus, mBatteryProps.batteryHealth,
                        mPlugType, mBatteryProps.batteryLevel, mBatteryProps.batteryTemperature,
                        mBatteryProps.batteryVoltage, mBatteryProps.batteryChargeCounter);
            } catch (RemoteException e) {
            }
            // 如果电量值为0，插入充电器且截止电压不合法，则调用弱充电关机进程。
            if ((mBatteryProps.batteryLevel == 0)
                     && (mWeakChgSocCheckStarted == 0)
                     && (mWeakChgCutoffVoltageMv > 0)
                     && (mPlugType != BATTERY_PLUGGED_NONE)) {
                     mWeakChgSocCheckStarted = 1;
                     mHandler.removeCallbacks(runnable);
                     mHandler.postDelayed(runnable, mVbattSamplingIntervalMsec);
            }
            //如果电量值小于严重低电量警告值，且尚未充电，则可正常关闭。且在显示关机对话框前需正常开机
            shutdownIfNoPowerLocked();
            // 如果电池温度过高（比如高于68摄氏度）,可正常关机。且在显示关机对话框前需正常开机
            shutdownIfOverTempLocked();
            if (force || (mBatteryProps.batteryStatus != mLastBatteryStatus ||
                    mBatteryProps.batteryHealth != mLastBatteryHealth ||
                    mBatteryProps.batteryPresent != mLastBatteryPresent ||
                    mBatteryProps.batteryLevel != mLastBatteryLevel ||
                    mPlugType != mLastPlugType ||
                    mBatteryProps.batteryVoltage != mLastBatteryVoltage ||
                    mBatteryProps.batteryTemperature != mLastBatteryTemperature ||
                    mBatteryProps.maxChargingCurrent != mLastMaxChargingCurrent ||
                    mBatteryProps.maxChargingVoltage != mLastMaxChargingVoltage ||
                    mBatteryProps.batteryChargeCounter != mLastChargeCounter ||
                    mInvalidCharger != mLastInvalidCharger)) {

                if (mPlugType != mLastPlugType) {
                    if (mLastPlugType == BATTERY_PLUGGED_NONE) {
                        // 未充电->充电情况
                        // 除非至少一次停充或电量值变化，否则无数据
                        if (mDischargeStartTime != 0 && mDischargeStartLevel != mBatteryProps.batteryLevel) {
                            dischargeDuration = SystemClock.elapsedRealtime() - mDischargeStartTime;
                            logOutlier = true;
                            EventLog.writeEvent(EventLogTags.BATTERY_DISCHARGE, dischargeDuration,
                                    mDischargeStartLevel, mBatteryProps.batteryLevel);
                            // 在再次记录电量数据前确保发生过一次停充事件
                            mDischargeStartTime = 0;
                        }
                    } else if (mPlugType == BATTERY_PLUGGED_NONE) {
                        // 充电->停充或只充电情况
                        mDischargeStartTime = SystemClock.elapsedRealtime();
                        mDischargeStartLevel = mBatteryProps.batteryLevel;
                    }
                }
                if (mBatteryProps.batteryStatus != mLastBatteryStatus ||
                        mBatteryProps.batteryHealth != mLastBatteryHealth ||
                        mBatteryProps.batteryPresent != mLastBatteryPresent ||
                        mPlugType != mLastPlugType) {
                    EventLog.writeEvent(EventLogTags.BATTERY_STATUS,
                            mBatteryProps.batteryStatus, mBatteryProps.batteryHealth, mBatteryProps.batteryPresent ? 1 : 0,
                            mPlugType, mBatteryProps.batteryTechnology);
                }
                if (mBatteryProps.batteryLevel != mLastBatteryLevel) {
                    // 仅电量值变化才执行此操作，即便电压或温度有变化
                    EventLog.writeEvent(EventLogTags.BATTERY_LEVEL,
                            mBatteryProps.batteryLevel, mBatteryProps.batteryVoltage, mBatteryProps.batteryTemperature);
                }
                if (mBatteryLevelCritical && !mLastBatteryLevelCritical &&
                        mPlugType == BATTERY_PLUGGED_NONE) {
                    // 如果电池损坏，需要确保能记录停充循环曲线
                    dischargeDuration = SystemClock.elapsedRealtime() - mDischargeStartTime;
                    logOutlier = true;
                }
                if (!mBatteryLevelLow) {
                    // 判断是否需要进入低电量模式(未充电且电量值小于低电量警告值)
                    if (mPlugType == BATTERY_PLUGGED_NONE
                            && mBatteryProps.batteryLevel <= mLowBatteryWarningLevel) {
                        mBatteryLevelLow = true;
                    }
                } else {
                    // 判断是否需要退出低电量模式(充电、电量值高于低电量警告值或关闭低电量警告值)
                    if (mPlugType != BATTERY_PLUGGED_NONE) {
                        mBatteryLevelLow = false;
                    } else if (mBatteryProps.batteryLevel >= mLowBatteryCloseWarningLevel)  {
                        mBatteryLevelLow = false;
                    } else if (force && mBatteryProps.batteryLevel >= mLowBatteryWarningLevel) {
                        // 如果被强制退出，则只会检查是否处于低电量警告范围
                        mBatteryLevelLow = false;
                    }
                }
                // 打包下列电源属性值及广播一起发送：ACTION_BATTERY_CHANGED;粘性广播
                sendIntentLocked();
                // 连接、断开充电器分别发送广播。由于标准intent无法唤醒任何应用，且有些应用可能需要基于此拥有智能行为
                if (mPlugType != 0 && mLastPlugType == 0) {
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            Intent statusIntent = new Intent(Intent.ACTION_POWER_CONNECTED);
                            statusIntent.setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
                            mContext.sendBroadcastAsUser(statusIntent, UserHandle.ALL);
                        }
                    });
                }
                else if (mPlugType == 0 && mLastPlugType != 0) {
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            Intent statusIntent = new Intent(Intent.ACTION_POWER_DISCONNECTED);
                            statusIntent.setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
                            mContext.sendBroadcastAsUser(statusIntent, UserHandle.ALL);
                        }
                    });
                }
                // 在以下情况下会发送ACTION_BATTERY_LOW广播：
                //插拔充电器，且电量值小于等于低电量警告值；或未插入充电器，电量值迅速掉入低电量警告值区域
                if (shouldSendBatteryLowLocked()) {
                    mSentLowBatteryBroadcast = true;
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            Intent statusIntent = new Intent(Intent.ACTION_BATTERY_LOW);
                            statusIntent.setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
                            mContext.sendBroadcastAsUser(statusIntent, UserHandle.ALL);
                        }
                    });
                } else if (mSentLowBatteryBroadcast && mLastBatteryLevel >= mLowBatteryCloseWarningLevel) {
                    mSentLowBatteryBroadcast = false;
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            Intent statusIntent = new Intent(Intent.ACTION_BATTERY_OKAY);
                            statusIntent.setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);
                            mContext.sendBroadcastAsUser(statusIntent, UserHandle.ALL);
                        }
                    });
                }
                // 更新电源led灯显示
                mLed.updateLightsLocked();
                // 发送广播后做此操作以便及时更新电源信息
                if (logOutlier && dischargeDuration != 0) {
                    logOutlierLocked(dischargeDuration);
                }
                mLastBatteryStatus = mBatteryProps.batteryStatus;
                mLastBatteryHealth = mBatteryProps.batteryHealth;
                mLastBatteryPresent = mBatteryProps.batteryPresent;
                mLastBatteryLevel = mBatteryProps.batteryLevel;
                mLastPlugType = mPlugType;
                mLastBatteryVoltage = mBatteryProps.batteryVoltage;
                mLastBatteryTemperature = mBatteryProps.batteryTemperature;
                mLastMaxChargingCurrent = mBatteryProps.maxChargingCurrent;
                mLastMaxChargingVoltage = mBatteryProps.maxChargingVoltage;
                mLastChargeCounter = mBatteryProps.batteryChargeCounter;
                mLastBatteryLevelCritical = mBatteryLevelCritical;
                mLastInvalidCharger = mInvalidCharger;
            }
        }
```

#### 3.2 updateLightsLocked方法

```java
        public void updateLightsLocked() {
            final int level = mBatteryProps.batteryLevel;
            final int status = mBatteryProps.batteryStatus;
            // 低电量时
            if (level < mLowBatteryWarningLevel) {
                if (status == BatteryManager.BATTERY_STATUS_CHARGING) {
                    // 电量不足红色指示灯
                    mBatteryLight.setColor(mBatteryLowARGB);
                } else {
                    // 电量不足且未充电，红色指示灯闪烁
                    mBatteryLight.setFlashing(mBatteryLowARGB, Light.LIGHT_FLASH_TIMED,
                            mBatteryLedOn, mBatteryLedOff);
                }
            } else if (status == BatteryManager.BATTERY_STATUS_CHARGING
                    || status == BatteryManager.BATTERY_STATUS_FULL) {
                //add by ICE2-98 zhangzhen 20170419
                if (status == BatteryManager.BATTERY_STATUS_FULL || level >= 100) {
                    // 充满电绿灯
                    mBatteryLight.setColor(mBatteryFullARGB);
                } else {
                    if (isHvdcpPresent()) {
                        // 连接hvdcp充电器闪烁橙色灯
                        mBatteryLight.setFlashing(mBatteryMediumARGB, Light.LIGHT_FLASH_TIMED,
                                mBatteryLedOn, mBatteryLedOn);
                    } else {
                        // 充电时中橙色
                        mBatteryLight.setColor(mBatteryMediumARGB);
                    }
                }
            } else {
                // 未充电且非低点不显示灯
                mBatteryLight.turnOff();
            }
        }
```
