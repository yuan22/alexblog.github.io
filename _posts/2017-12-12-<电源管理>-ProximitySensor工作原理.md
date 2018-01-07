

### 1、DisplayPowerController构造函数

```java
        public DisplayPowerController(Context context,
                DisplayPowerCallbacks callbacks, Handler handler,
                SensorManager sensorManager, DisplayBlanker blanker) {
            mHandler = new DisplayControllerHandler(handler.getLooper());
            mCallbacks = callbacks;
            // 获取电池统计服务
            mBatteryStats = BatteryStatsService.getService();
            // 初始化传感器
            mSensorManager = sensorManager;
            // 窗口管理策略
            mWindowManagerPolicy = LocalServices.getService(WindowManagerPolicy.class);
            // 显示屏
            mBlanker = blanker;
            mContext = context;
            final Resources resources = context.getResources();
            // 最低屏幕亮度：10
            final int screenBrightnessSettingMinimum = clampAbsoluteBrightness(resources.getInteger(
                    com.android.internal.R.integer.config_screenBrightnessSettingMinimum));
            // Doze状态最低屏幕亮度：1
            mScreenBrightnessDozeConfig = clampAbsoluteBrightness(resources.getInteger(
                    com.android.internal.R.integer.config_screenBrightnessDoze));
            // Dim状态屏幕亮度：10
            mScreenBrightnessDimConfig = clampAbsoluteBrightness(resources.getInteger(
                    com.android.internal.R.integer.config_screenBrightnessDim));
            // 暗间中最低屏幕亮度：1
            mScreenBrightnessDarkConfig = clampAbsoluteBrightness(resources.getInteger(
                    com.android.internal.R.integer.config_screenBrightnessDark));
            if (mScreenBrightnessDarkConfig > mScreenBrightnessDimConfig) {
                Slog.w(TAG, "Expected config_screenBrightnessDark ("
                        + mScreenBrightnessDarkConfig + ") to be less than or equal to "
                        + "config_screenBrightnessDim (" + mScreenBrightnessDimConfig + ").");
            }
            if (mScreenBrightnessDarkConfig > mScreenBrightnessDimConfig) {
                Slog.w(TAG, "Expected config_screenBrightnessDark ("
                        + mScreenBrightnessDarkConfig + ") to be less than or equal to "
                        + "config_screenBrightnessSettingMinimum ("
                        + screenBrightnessSettingMinimum + ").");
            }
            // 取Min、Dim、Dark中最小值
            int screenBrightnessRangeMinimum = Math.min(Math.min(
                    screenBrightnessSettingMinimum, mScreenBrightnessDimConfig),
                    mScreenBrightnessDarkConfig);
            // 亮度最大值：255
            mScreenBrightnessRangeMaximum = PowerManager.BRIGHTNESS_ON;
            // 软件自动调节亮度：false
            mUseSoftwareAutoBrightnessConfig = resources.getBoolean(
                    com.android.internal.R.bool.config_automatic_brightness_available);
            // Doze模式自动调节亮度：false
            mAllowAutoBrightnessWhileDozingConfig = resources.getBoolean(
                    com.android.internal.R.bool.config_allowAutoBrightnessWhileDozing);
            // 快速动画亮度每秒缓变率：200
            mBrightnessRampRateFast = resources.getInteger(
                    com.android.internal.R.integer.config_brightness_ramp_rate_fast);
            // 慢速动画亮度每秒缓变率：40
            mBrightnessRampRateSlow = resources.getInteger(
                    com.android.internal.R.integer.config_brightness_ramp_rate_slow);
            // 亮度传感器事件传输速率：250ms
            int lightSensorRate = resources.getInteger(
                    com.android.internal.R.integer.config_autoBrightnessLightSensorRate);
            // 设备退出Doze模式时用于获取首次亮度，该值为0即禁用该功能
            int initialLightSensorRate = resources.getInteger(
                    com.android.internal.R.integer.config_autoBrightnessInitialLightSensorRate);
            if (initialLightSensorRate == -1) {
              initialLightSensorRate = lightSensorRate;
            } else if (initialLightSensorRate > lightSensorRate) {
              Slog.w(TAG, "Expected config_autoBrightnessInitialLightSensorRate ("
                      + initialLightSensorRate + ") to be less than or equal to "
                      + "config_autoBrightnessLightSensorRate (" + lightSensorRate + ").");
            }
            // 亮光度抖动值：8000ms
            long brighteningLightDebounce = resources.getInteger(
                    com.android.internal.R.integer.config_autoBrightnessBrighteningLightDebounce);
            // 暗光度抖动值：4000ms
            long darkeningLightDebounce = resources.getInteger(
                    com.android.internal.R.integer.config_autoBrightnessDarkeningLightDebounce);
            // 默认为true，计算发现光度变化后立即重置屏幕亮度；否则会很慢根据光度变化屏幕亮度（适用于手表）
            boolean autoBrightnessResetAmbientLuxAfterWarmUp = resources.getBoolean(
                    com.android.internal.R.bool.config_autoBrightnessResetAmbientLuxAfterWarmUp);
            // 毫秒计算亮度，10000
            int ambientLightHorizon = resources.getInteger(
                    com.android.internal.R.integer.config_autoBrightnessAmbientLightHorizon);
            // 自动调整gamma的范围：300%
            float autoBrightnessAdjustmentMaxGamma = resources.getFraction(
                    com.android.internal.R.fraction.config_autoBrightnessAdjustmentMaxGamma,
                    1, 1);
            if (mUseSoftwareAutoBrightnessConfig) {
                // 自动亮度级别
                int[] lux = resources.getIntArray(
                        com.android.internal.R.array.config_autoBrightnessLevels);
                // 自动亮度lcd背光灯亮度
                int[] screenBrightness = resources.getIntArray(
                        com.android.internal.R.array.config_autoBrightnessLcdBacklightValues);
                // 光感预估时间：0
                int lightSensorWarmUpTimeConfig = resources.getInteger(
                        com.android.internal.R.integer.config_lightSensorWarmupTime);
                // doze状态允许自动调节亮度值的规模：100%
                final float dozeScaleFactor = resources.getFraction(
                        com.android.internal.R.fraction.config_screenAutoBrightnessDozeScaleFactor,
                        1, 1);
                // 亮度滞后level配置
                int[] brightHysteresisLevels = resources.getIntArray(
                        com.android.internal.R.array.config_dynamicHysteresisBrightLevels);
                int[] darkHysteresisLevels = resources.getIntArray(
                        com.android.internal.R.array.config_dynamicHysteresisDarkLevels);
                int[] luxHysteresisLevels = resources.getIntArray(
                        com.android.internal.R.array.config_dynamicHysteresisLuxLevels);
                // Doze模式亮度level配置
                int[] dozeSensorLuxLevels = resources.getIntArray(
                        com.android.internal.R.array.config_dozeSensorLuxLevels);
                int[] dozeBrightnessBacklightValues = resources.getIntArray(
                        com.android.internal.R.array.config_dozeBrightnessBacklightValues);
                boolean useNewSensorSamplesForDoze = resources.getBoolean(
                        com.android.internal.R.bool.config_useNewSensorSamplesForDoze);
                mUseActiveDozeLightSensorConfig = resources.getBoolean(
                        com.android.internal.R.bool.config_allowAutoBrightnessActiveDozeLightSensor);
                LuxLevels luxLevels = new LuxLevels(brightHysteresisLevels, darkHysteresisLevels,
                        luxHysteresisLevels, dozeSensorLuxLevels, dozeBrightnessBacklightValues);
                Spline screenAutoBrightnessSpline = createAutoBrightnessSpline(lux, screenBrightness);
                if (screenAutoBrightnessSpline == null) {
                    Slog.e(TAG, "Error in config.xml.  config_autoBrightnessLcdBacklightValues "
                            + "(size " + screenBrightness.length + ") "
                            + "must be monotic and have exactly one more entry than "
                            + "config_autoBrightnessLevels (size " + lux.length + ") "
                            + "which must be strictly increasing.  "
                            + "Auto-brightness will be disabled.");
                    mUseSoftwareAutoBrightnessConfig = false;
                } else {
                    int bottom = clampAbsoluteBrightness(screenBrightness[0]);
                    if (mScreenBrightnessDarkConfig > bottom) {
                        Slog.w(TAG, "config_screenBrightnessDark (" + mScreenBrightnessDarkConfig
                                + ") should be less than or equal to the first value of "
                                + "config_autoBrightnessLcdBacklightValues ("
                                + bottom + ").");
                    }
                    if (bottom < screenBrightnessRangeMinimum) {
                        screenBrightnessRangeMinimum = bottom;
                    }
                    mAutomaticBrightnessController = new AutomaticBrightnessController(this,
                            handler.getLooper(), sensorManager, screenAutoBrightnessSpline,
                            lightSensorWarmUpTimeConfig, screenBrightnessRangeMinimum,
                            mScreenBrightnessRangeMaximum, dozeScaleFactor, lightSensorRate,
                            initialLightSensorRate, brighteningLightDebounce, darkeningLightDebounce,
                            autoBrightnessResetAmbientLuxAfterWarmUp, ambientLightHorizon,
                            autoBrightnessAdjustmentMaxGamma, mUseActiveDozeLightSensorConfig,
                            useNewSensorSamplesForDoze, luxLevels);
                }
            }
            mScreenBrightnessRangeMinimum = screenBrightnessRangeMinimum;
            // 默认为false,屏幕不会淡出
            mColorFadeFadesConfig = resources.getBoolean(
                    com.android.internal.R.bool.config_animateScreenLights);
            // 初始化接近传感器
            if (!DEBUG_PRETEND_PROXIMITY_SENSOR_ABSENT) {
                mProximitySensor = mSensorManager.getDefaultSensor(Sensor.TYPE_PROXIMITY);
                if (mProximitySensor != null) {
                    mProximityThreshold = Math.min(mProximitySensor.getMaximumRange(),
                            TYPICAL_PROXIMITY_THRESHOLD);
                }
            }

        }
```

### 2、DisplayControllerHandler方法

```java
        private final class DisplayControllerHandler extends Handler {
            public DisplayControllerHandler(Looper looper) {
                super(looper, null, true /*async*/);
            }
            @Override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case MSG_UPDATE_POWER_STATE:
                        updatePowerState();
                        break;
                    case MSG_PROXIMITY_SENSOR_DEBOUNCED:
                        debounceProximitySensor();
                        break;
                    case MSG_SCREEN_ON_UNBLOCKED:
                        if (mPendingScreenOnUnblocker == msg.obj) {
                            unblockScreenOn();
                            updatePowerState();
                        }
                        break;
                }
            }
        }
```

### 3、updatePowerState方法更新电源状态

```java
        private void updatePowerState() {
            // 更新电源状态
            final boolean mustNotify;
            boolean mustInitialize = false;
            boolean autoBrightnessAdjustmentChanged = false;
            synchronized (mLock) {
                mPendingUpdatePowerStateLocked = false;
                if (mPendingRequestLocked == null) {
                    return
                }
                if (mPowerRequest == null) {
                    mPowerRequest = new DisplayPowerRequest(mPendingRequestLocked);
                    mWaitingForNegativeProximity = mPendingWaitForNegativeProximityLocked;
                    mPendingWaitForNegativeProximityLocked = false;
                    mPendingRequestChangedLocked = false;
                    mustInitialize = true;
                } else if (mPendingRequestChangedLocked) {
                    autoBrightnessAdjustmentChanged = (mPowerRequest.screenAutoBrightnessAdjustment
                            != mPendingRequestLocked.screenAutoBrightnessAdjustment);
                    mPowerRequest.copyFrom(mPendingRequestLocked);
                    mWaitingForNegativeProximity |= mPendingWaitForNegativeProximityLocked;
                    mPendingWaitForNegativeProximityLocked = false;
                    mPendingRequestChangedLocked = false;
                    mDisplayReadyLocked = false;
                }
                mustNotify = !mDisplayReadyLocked;
            }
            // 第一次电源状态变化时要进行一些初始化处理
            if (mustInitialize) {
                initialize();
            }
            // 使用下述测量计算基本显示状态。可能根据其它因素覆盖下述信息。
            int state;
            int brightness = PowerManager.BRIGHTNESS_DEFAULT;
            boolean performScreenOffTransition = false;
            switch (mPowerRequest.policy) {
                case DisplayPowerRequest.POLICY_OFF:
                    state = Display.STATE_OFF;
                    performScreenOffTransition = true;
                    break;
                case DisplayPowerRequest.POLICY_DOZE:
                    if (mPowerRequest.dozeScreenState != Display.STATE_UNKNOWN) {
                        state = mPowerRequest.dozeScreenState;
                    } else {
                        state = Display.STATE_DOZE;
                    }
                    // mAllowAutoBrightnessWhileDozingConfig默认为false，取反进入分支
                    if (!mAllowAutoBrightnessWhileDozingConfig) {
                        brightness = mPowerRequest.dozeScreenBrightness;
                    }
                    break;
                case DisplayPowerRequest.POLICY_VR:
                    state = Display.STATE_VR;
                    break;
                case DisplayPowerRequest.POLICY_DIM:
                case DisplayPowerRequest.POLICY_BRIGHT:
                default:
                    state = Display.STATE_ON;
                    break;
            }
            assert(state != Display.STATE_UNKNOWN);
            // 应用接近传感器
            if (mProximitySensor != null) {
                if (mPowerRequest.useProximitySensor && state != Display.STATE_OFF) {
                    setProximitySensorEnabled(true);
                    if (!mScreenOffBecauseOfProximity
                            && mProximity == PROXIMITY_POSITIVE) {
                        mScreenOffBecauseOfProximity = true;
                        sendOnProximityPositiveWithWakelock();
                    }
                } else if (mWaitingForNegativeProximity
                        && mScreenOffBecauseOfProximity
                        && mProximity == PROXIMITY_POSITIVE
                        && state != Display.STATE_OFF) {
                    setProximitySensorEnabled(true);
                } else {
                    setProximitySensorEnabled(false);
                    mWaitingForNegativeProximity = false;
                }
                if (mScreenOffBecauseOfProximity
                        && mProximity != PROXIMITY_POSITIVE) {
                    mScreenOffBecauseOfProximity = false;
                    sendOnProximityNegativeWithWakelock();
                }
            } else {
                mWaitingForNegativeProximity = false;
            }
            if (mScreenOffBecauseOfProximity) {
                state = Display.STATE_OFF;
            }
            // 屏幕活动状态变更，其过度可能会延迟，所以最终使用实际状态而非所需状态。
            final int oldState = mPowerState.getScreenState();
            animateScreenStateChange(state, performScreenOffTransition);
            state = mPowerState.getScreenState();
            // 屏幕关闭时设置亮度为0
            if (state == Display.STATE_OFF) {
                brightness = PowerManager.BRIGHTNESS_OFF;
            }
            // 自动调节亮度配置
            boolean autoBrightnessEnabled = false;
            if (mAutomaticBrightnessController != null) {
                final boolean autoBrightnessEnabledInDoze = mAllowAutoBrightnessWhileDozingConfig
                        && (state == Display.STATE_DOZE || state == Display.STATE_DOZE_SUSPEND);
                autoBrightnessEnabled = (mPowerRequest.useAutoBrightness
                        && (state == Display.STATE_ON || autoBrightnessEnabledInDoze)
                        || mUseActiveDozeLightSensorConfig && autoBrightnessEnabledInDoze)
                        && brightness < 0;
                final boolean userInitiatedChange = autoBrightnessAdjustmentChanged
                        && mPowerRequest.brightnessSetByUser;
                mAutomaticBrightnessController.configure(autoBrightnessEnabled,
                        mPowerRequest.screenAutoBrightnessAdjustment, state != Display.STATE_ON,
                        userInitiatedChange);
            }
            // 在配置自动亮度后应用亮度提升，以至于在临时状态中无法禁用光感。
            // 当提升结束后能毫无延迟的恢复自动亮度行为
            if (mPowerRequest.boostScreenBrightness
                    && brightness != PowerManager.BRIGHTNESS_OFF) {
                brightness = PowerManager.BRIGHTNESS_ON;
            }
            // 申请自动亮度调节
            boolean slowChange = false;
            if (brightness < 0) {
                if (autoBrightnessEnabled) {
                    brightness = mAutomaticBrightnessController.getAutomaticScreenBrightness();
                }
                if (brightness >= 0) {
                    // 缓慢自动调节亮度
                    brightness = clampScreenBrightness(brightness);
                    if (mAppliedAutoBrightness && !autoBrightnessAdjustmentChanged) {
                        slowChange = true;
                    }
                    mAppliedAutoBrightness = true;
                } else {
                    mAppliedAutoBrightness = false;
                }
            } else {
                mAppliedAutoBrightness = false;
            }
            // Doze状态使用默认亮度，除非被修改或者收集到传感器信息
            if (state == Display.STATE_DOZE || state == Display.STATE_DOZE_SUSPEND) {
                if (brightness < 0) {
                    brightness = mScreenBrightnessDozeConfig;
                } else if (mUseActiveDozeLightSensorConfig) {
                    brightness = Math.min(brightness, mScreenBrightnessDozeConfig);
                    if (DEBUG) {
                        Slog.d(TAG, "updatePowerState: ALS-based doze brightness: " + brightness);
                    }
                }
            }
            // 手动调节亮度
            if (brightness < 0) {
                brightness = clampScreenBrightness(mPowerRequest.screenBrightness);
            }
            // Dim状态亮度
            if (mPowerRequest.policy == DisplayPowerRequest.POLICY_DIM) {
                if (brightness > mScreenBrightnessRangeMinimum) {
                    brightness = Math.max(Math.min(brightness - SCREEN_DIM_MINIMUM_REDUCTION,
                            mScreenBrightnessDimConfig), mScreenBrightnessRangeMinimum);
                }
                if (!mAppliedDimming) {
                    slowChange = false;
                }
                mAppliedDimming = true;
            } else if (mAppliedDimming) {
                slowChange = false;
                mAppliedDimming = false;
            }
            // 如果进入低电量模式，高于最小阀值的前提下亮度减半
            if (mPowerRequest.lowPowerMode) {
                if (brightness > mScreenBrightnessRangeMinimum) {
                    brightness = Math.max(brightness / 2, mScreenBrightnessRangeMinimum);
                }
                if (!mAppliedLowPower) {
                    slowChange = false;
                }
                mAppliedLowPower = true;
            } else if (mAppliedLowPower) {
                slowChange = false;
                mAppliedLowPower = false;
            }
            // 亮屏/Doze状态调整亮度。灭屏或者VR模式跳过。
            if (!mPendingScreenOff) {
                boolean wasOrWillBeInVr = (state == Display.STATE_VR || oldState == Display.STATE_VR);
                if ((state == Display.STATE_ON || state == Display.STATE_DOZE) && !wasOrWillBeInVr) {
                    animateScreenBrightness(brightness,
                            slowChange ? mBrightnessRampRateSlow : mBrightnessRampRateFast);
                } else {
                    animateScreenBrightness(brightness, 0);
                }
            }
            // 决定显示器是否准备好在新的请求状态下使用。
            // 不能等待亮度斜坡动画完成才报告，因为我们只需确保屏幕在正确的电源状态下，即使它继续收敛所需亮度
            final boolean ready = mPendingScreenOnUnblocker == null
                    && !mColorFadeOnAnimator.isStarted()
                    && !mColorFadeOffAnimator.isStarted()
                    && mPowerState.waitUntilClean(mCleanListener);
            final boolean finished = ready
                    && !mScreenBrightnessRampAnimator.isAnimating();
            // 亮屏通知策略
            if (ready && state != Display.STATE_OFF
                    && mReportedScreenStateToPolicy == REPORTED_TO_POLICY_SCREEN_TURNING_ON) {
                mReportedScreenStateToPolicy = REPORTED_TO_POLICY_SCREEN_ON;
                mWindowManagerPolicy.screenTurnedOn();
            }
            // 如果活动未完成依旧持有wakelock
            if (!finished && !mUnfinishedBusiness) {
                if (DEBUG) {
                    Slog.d(TAG, "Unfinished business...");
                }
                mCallbacks.acquireSuspendBlocker();
                mUnfinishedBusiness = true;
            }
            // 准备好通知PowerManager
            if (ready && mustNotify) {
                // 发送改变的状态
                synchronized (mLock) {
                    if (!mPendingRequestChangedLocked) {
                        mDisplayReadyLocked = true;

                        if (DEBUG) {
                            Slog.d(TAG, "Display ready!");
                        }
                    }
                }
                sendOnStateChangedWithWakelock();
            }
            // 没有未完成活动时释放wakelock
            if (finished && mUnfinishedBusiness) {
                if (DEBUG) {
                    Slog.d(TAG, "Finished business...");
                }
                mUnfinishedBusiness = false;
                mCallbacks.releaseSuspendBlocker();
            }
        }
```

![](https://xiezeyangcn.github.io/xzyblog.github.io/assets/images/2017-12-12-proximity-sensor/display_proximity_sensor_state.bmp)

&emsp;<1>DisplayPowerController构造函数中初始化SensorManager和ProximitySensor

&emsp;<2>调用updatePowerState方法更新电源状态

&emsp;<3>判断ProximitySensor是否存在，存在走<4>，否则结束

&emsp;<4>case1:判断ProximitySensor是否使用，且处于非灭屏状态，是则走<11>，否则走<7>

&emsp;<5>case2:判断ProximitySensor处于Positive状态，且没有因为ProximitySensor而灭屏，则走<6>

&emsp;<6>处理ProximitySensor接近事件，灭屏操作

&emsp;<7>case3:ProximitySensor等待亮屏，且之前因ProximitySensor灭屏，另外ProximitySensor状态为Positive，且未灭屏，则走<11>，否则走<8>

&emsp;<8>判断ProximitySensor是否在使用，是则走<9>，否则走<12>

&emsp;<9>判断是否因ProximitySensor导致灭屏，是则根据<4>和<8>知走至<9>时display为OFF，而因为ProximitySensor导致的灭屏display保持ON,足以证明ProximitySensor出现异常，则执行<10>,否则走<11>

&emsp;<10>将ProximitySensor置为PROXIMITY_UNKNOWN

&emsp;<11>注册ProximitySensor监听器

&emsp;<12>取消ProximitySensor监听器

&emsp;<13>case4:有因为ProximitySensor导致的灭屏状态，且ProximitySensor的状态为非Positive，是则执行<14>

&emsp;<14>处理ProximitySensor远离事件


### 4、initialize方法进行一些初始化操作

```java
        private void initialize() {
            // 初始化默认显示器的电源状态对象，以后可能独立管理多显示器
            mPowerState = new DisplayPowerState(mBlanker,
                    new ColorFade(Display.DEFAULT_DISPLAY));
            mColorFadeOnAnimator = ObjectAnimator.ofFloat(
                    mPowerState, DisplayPowerState.COLOR_FADE_LEVEL, 0.0f, 1.0f);
            mColorFadeOnAnimator.setDuration(COLOR_FADE_ON_ANIMATION_DURATION_MILLIS);
            mColorFadeOnAnimator.addListener(mAnimatorListener);

            mColorFadeOffAnimator = ObjectAnimator.ofFloat(
                    mPowerState, DisplayPowerState.COLOR_FADE_LEVEL, 1.0f, 0.0f);
            mColorFadeOffAnimator.setDuration(COLOR_FADE_OFF_ANIMATION_DURATION_MILLIS);
            mColorFadeOffAnimator.addListener(mAnimatorListener);

            mScreenBrightnessRampAnimator = new RampAnimator<DisplayPowerState>(
                    mPowerState, DisplayPowerState.SCREEN_BRIGHTNESS);
            mScreenBrightnessRampAnimator.setListener(mRampAnimatorListener);
            // 电池统计服务初始化屏幕状态
            try {
                mBatteryStats.noteScreenState(mPowerState.getScreenState());
                mBatteryStats.noteScreenBrightness(mPowerState.getScreenBrightness());
            } catch (RemoteException ex) {
                // same process
            }
        }
```

### 5、setProximitySensorEnabled方法注册、取消接近传感器监听器

```java
        private void setProximitySensorEnabled(boolean enable) {
            if (enable) {
                if (!mProximitySensorEnabled) {
                    // 注册接近传感器监听，清空初始化接近传感器状态
                    mProximitySensorEnabled = true;
                    mSensorManager.registerListener(mProximitySensorListener, mProximitySensor,
                            SensorManager.SENSOR_DELAY_NORMAL, mHandler);
                }
            } else {
                if (mProximitySensorEnabled) {
                    // 取消接近传感器监听，清除接近传感器状态以便下次使用
                    mProximitySensorEnabled = false;
                    mProximity = PROXIMITY_UNKNOWN;
                    mPendingProximity = PROXIMITY_UNKNOWN;
                    // 取消接近传感器抖动消息
                    mHandler.removeMessages(MSG_PROXIMITY_SENSOR_DEBOUNCED);
                    mSensorManager.unregisterListener(mProximitySensorListener);
                    // 释放电源锁
                    clearPendingProximityDebounceTime();
                }
            }
        }
```

### 6、mProximitySensorListener接近传感器监听器

```java
        private final SensorEventListener mProximitySensorListener = new SensorEventListener() {
            @Override
            public void onSensorChanged(SensorEvent event) {
                if (mProximitySensorEnabled) {
                    final long time = SystemClock.uptimeMillis();
                    // 获取上报的距离
                    final float distance = event.values[0];
                    // 检测距离是否在0-阀值之间，阀值为1.0f
                    boolean positive = distance >= 0.0f && distance < mProximityThreshold;
                    handleProximitySensorEvent(time, positive);
                }
            }
            @Override
            public void onAccuracyChanged(Sensor sensor, int accuracy) {
                // Not used.
            }
        };
```

### 7、handleProximitySensorEvent方法处理接近、远离传感器行为

```java
        private void handleProximitySensorEvent(long time, boolean positive) {
            if (mProximitySensorEnabled) {
                if (mPendingProximity == PROXIMITY_NEGATIVE && !positive) {
                    return; // no change
                }
                if (mPendingProximity == PROXIMITY_POSITIVE && positive) {
                    return; // no change
                }
                // 接近传感器防抖处理，获取电源锁
                mHandler.removeMessages(MSG_PROXIMITY_SENSOR_DEBOUNCED);
                if (positive) {
                    mPendingProximity = PROXIMITY_POSITIVE;
                    setPendingProximityDebounceTime(
                            time + PROXIMITY_SENSOR_POSITIVE_DEBOUNCE_DELAY);
                } else {
                    mPendingProximity = PROXIMITY_NEGATIVE;
                    setPendingProximityDebounceTime(
                            time + PROXIMITY_SENSOR_NEGATIVE_DEBOUNCE_DELAY);
                }
                // 校准接近传感器
                debounceProximitySensor();
            }
        }
```

### 8、debounceProximitySensor校准接近传感器

```java
        private void debounceProximitySensor() {
            if (mProximitySensorEnabled
                    && mPendingProximity != PROXIMITY_UNKNOWN
                    && mPendingProximityDebounceTime >= 0) {
                final long now = SystemClock.uptimeMillis();
                if (mPendingProximityDebounceTime <= now) {
                    // 通过校准获取接近传感器状态
                    mProximity = mPendingProximity;
                    // 更新电源状态
                    updatePowerState();
                    // 释放电源锁
                    clearPendingProximityDebounceTime();
                } else {
                    // 校准未结束，发送消息继续校准
                    Message msg = mHandler.obtainMessage(MSG_PROXIMITY_SENSOR_DEBOUNCED);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtTime(msg, mPendingProximityDebounceTime);
                }
            }
        }
```


