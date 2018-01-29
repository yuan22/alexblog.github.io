&emsp;在前面的博文中，我们介绍到，Android中进程会想PowerMS申请/释放WakeLock。而PowerMS收到申请/释放请求时，均会调用updatePowerStateLocked方法来更新电源状态。下面来分析下它的流程。

```java
protected void updatePowerStateLocked() {
  if (!mSystemReady || mDirty == 0) {
    return;
  }
  ......
  try {
    // 更新基本状态。
    updateIsPoweredLocked(mDirty);
    updateStayOnLocked(mDirty);
    updateScreenBrightnessBoostLocked(mDirty);
    // 更新wakelock和用户活动
    final long now = SystemClock.uptimeMillis();
    int dirtyPhase2 = 0;
    for (;;) {
      int dirtyPhase1 = mDirty;
      dirtyPhase2 |= dirtyPhase1;
      mDirty = 0;
      updateWakeLockSummaryLocked(dirtyPhase1);
      updateUserActivitySummaryLocked(now, dirtyPhase1);
      if (!updateWakefulnessLocked(dirtyPhase1)) {
        break;
      }
    }
    // 更新disply power状态。
    boolean displayBecameReady = updateDisplayPowerStateLocked(dirtyPhase2);
    // 更新dream状态。
    updateDreamLocked(dirtyPhase2, displayBecameReady);
    // 必要时发送通知。
    finishWakefulnessChangeIfNeededLocked();
    // 更新suspend block。
    updateSuspendBlockerLocked();
  } finally {
    ......
  }
}

```

### 一、更新基本状态信息

#### 1.1 PowerMS类的updateIsPoweredLocked方法

```java
private void updateIsPoweredLocked(int dirty) {
  if ((dirty & DIRTY_BATTERY_STATE) != 0) {
    // 记录过去的状态。
    final boolean wasPowered = mIsPowered;
    final int oldPlugType = mPlugType;
    final boolean oldLevelLow = mBatteryLevelLow;
    // 是否在充电。
mIsPowered =
mBatteryManagerInternal.isPowered(BatteryManager.BATTERY_PLUGGED_ANY);
    // 充电类型。
    mPlugType = mBatteryManagerInternal.getPlugType();
    // 当前充电电量。
    mBatteryLevel = mBatteryManagerInternal.getBatteryLevel();
    // 是否低电量。
    mBatteryLevelLow = mBatteryManagerInternal.getBatteryLevelLow();
    // 充电状态或充电类型是否改变。
    if (wasPowered != mIsPowered || oldPlugType != mPlugType) {
      mDirty |= DIRTY_IS_POWERED;
      // 无线充电相关。
      final boolean dockedOnWirelessCharger =
        mWirelessChargerDetector.update(
          mIsPowered, mPlugType, mBatteryLevel);
      final long now = SystemClock.uptimeMillis();
      // 插拔充电器或USB是否唤醒屏幕。
      if (shouldWakeUpWhenPluggedOrUnpluggedLocked(wasPowered, oldPlugType,
          dockedOnWirelessCharger)) {
        wakeUpNoUpdateLocked(now, "android.server.power:POWER",
 Process.SYSTEM_UID,
            mContext.getOpPackageName(), Process.SYSTEM_UID);
      }
      // 出发用户活动，修改PowerMS中记录用户活动事件的时间，同时通知BatteryStatsService等。
      userActivityNoUpdateLocked(
          now, PowerManager.USER_ACTIVITY_EVENT_OTHER, 0,
 Process.SYSTEM_UID);
      // 无线充电相关通知。
      if (dockedOnWirelessCharger) {
        mNotifier.onWirelessChargingStarted();
      }
    }
    if (wasPowered != mIsPowered || oldLevelLow != mBatteryLevelLow) {
      if (oldLevelLow != mBatteryLevelLow && !mBatteryLevelLow) {
        mAutoLowPowerModeSnoozing = false;
      }
      // 更新低电模式相关操作。
      updateLowPowerModeLocked();
    }
  }
}

// 更新低电模式操作
private void updateLowPowerModeLocked() {
  if ((mIsPowered || !mBatteryLevelLow && !mBootCompleted) &&
 mLowPowerModeSetting) {
    // 更新数据库，关闭低电模式。
    Settings.Global.putInt(mContext.getContentResolver(),
        Settings.Global.LOW_POWER_MODE, 0);
    mLowPowerModeSetting = false;
  }
  // 是否进入自动省电模式。
  final boolean autoLowPowerModeEnabled = !mIsPowered &&
 mAutoLowPowerModeConfigured
      && !mAutoLowPowerModeSnoozing && mBatteryLevelLow;
  final boolean lowPowerModeEnabled = mLowPowerModeSetting ||
 autoLowPowerModeEnabled;
  // 是否低电模式。
  if (mLowPowerModeEnabled != lowPowerModeEnabled) {
    mLowPowerModeEnabled = lowPowerModeEnabled;
    // 调用底层动态库的powerHint函数。
    powerHintInternal(POWER_HINT_LOW_POWER, lowPowerModeEnabled ? 1 : 0);
    postAfterBootCompleted(new Runnable() {
      @Override
      public void run() {
        Intent intent =
new Intent(PowerManager.ACTION_POWER_SAVE_MODE_CHANGING)
            .putExtra(PowerManager.EXTRA_POWER_SAVE_MODE, mLowPowerModeEnabled)
            .addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        // 发送低电模式充电广播。
        mContext.sendBroadcast(intent);
        ArrayList<PowerManagerInternal.LowPowerModeListener> listeners;
        // registerLowPowerModeObserver接口。
        synchronized (mLock) {
          listeners =
new ArrayList<PowerManagerInternal.LowPowerModeListener>(
              mLowPowerModeListeners);
        }
        for (int i=0; i<listeners.size(); i++) {
          // 通知其它进程低电模式发生改变。
          listeners.get(i).onLowPowerModeChanged(lowPowerModeEnabled);
        }
        // 发送CHANGED广播。
        intent = new Intent(PowerManager.ACTION_POWER_SAVE_MODE_CHANGED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        mContext.sendBroadcast(intent);
        // 发送内部版本请求签名权限。
        mContext.sendBroadcastAsUser(new Intent(
            PowerManager.ACTION_POWER_SAVE_MODE_CHANGED_INTERNAL), UserHandle.ALL,
            Manifest.permission.DEVICE_POWER);
      }
    });
  }
}

```

#### 1.2 PowerMS类的updateStayOnLocked方法

```java
private void updateStayOnLocked(int dirty) {
  // 电源状态或电源设置发生变化。
  if ((dirty & (DIRTY_BATTERY_STATE | DIRTY_SETTINGS)) != 0) {
    final boolean wasStayOn = mStayOn;
    // 判断插入充电器是否亮屏。
    if (mStayOnWhilePluggedInSetting != 0
        && !isMaximumScreenOffTimeoutFromDeviceAdminEnforcedLocked()) {
      mStayOn =
        mBatteryManagerInternal.isPowered(mStayOnWhilePluggedInSetting);
    } else {
      mStayOn = false;
    }
    if (mStayOn != wasStayOn) {
      mDirty |= DIRTY_STAY_ON;
    }
  }
}
```

#### 1.3 PowerMS类的updateScreenBrightnessBoostLocked方法

```java
private void updateScreenBrightnessBoostLocked(int dirty) {
  if ((dirty & DIRTY_SCREEN_BRIGHTNESS_BOOST) != 0) {
    if (mScreenBrightnessBoostInProgress) {
      // 移除旧的超时事件。
      final long now = SystemClock.uptimeMillis();
      mHandler.removeMessages(MSG_SCREEN_BRIGHTNESS_BOOST_TIMEOUT);
      if (mLastScreenBrightnessBoostTime > mLastSleepTime) {
        final long boostTimeout = mLastScreenBrightnessBoostTime +
            SCREEN_BRIGHTNESS_BOOST_TIMEOUT;
        if (boostTimeout > now) {
          Message msg =
 mHandler.obtainMessage(MSG_SCREEN_BRIGHTNESS_BOOST_TIMEOUT);
          msg.setAsynchronous(true);
          // 屏幕离开最大亮度状态时该事件将被发送。
          mHandler.sendMessageAtTime(msg, boostTimeout);
          return;
        }
      }
      mScreenBrightnessBoostInProgress = false;
      mNotifier.onScreenBrightnessBoostChanged();
      userActivityNoUpdateLocked(now,
          PowerManager.USER_ACTIVITY_EVENT_OTHER, 0, Process.SYSTEM_UID);
    }
  }
}
```

#### 1.4 PowerMS类的BinderService方法

```java

private void boostScreenBrightnessInternal(long eventTime, int uid) {
  synchronized (mLock) {
    if (!mSystemReady || mWakefulness == WAKEFULNESS_ASLEEP
        || eventTime < mLastScreenBrightnessBoostTime) {
      return;
    }
    // 记录事件到来的时间，即设备处于最亮屏幕状态起始时间。
    mLastScreenBrightnessBoostTime = eventTime;
    if (!mScreenBrightnessBoostInProgress) {
      mScreenBrightnessBoostInProgress = true;
      mNotifier.onScreenBrightnessBoostChanged();
    }
    // 修改mDirty值，表示最大屏幕亮度的状态发生变化。
    mDirty |= DIRTY_SCREEN_BRIGHTNESS_BOOST;
    userActivityNoUpdateLocked(eventTime,
        PowerManager.USER_ACTIVITY_EVENT_OTHER, 0, uid);
    // 更新电源状态信息。
    updatePowerStateLocked();
  }
}
```

### 二、更新WakeLock和用户活动

#### 2.1 PowerMS类的updateWakeLockSummaryLocked方法

```java
private void updateWakeLockSummaryLocked(int dirty) {
  // PowerMS持有的WakeLock发生变化或唤醒状态发生变化时，才重新进行更新mWakeLockSummary。
  if ((dirty & (DIRTY_WAKE_LOCKS | DIRTY_WAKEFULNESS)) != 0) {
    mWakeLockSummary = 0;
    final int numWakeLocks = mWakeLocks.size();
    // level对应的wake lock信息。
    ......
    // 若不是dozing状态，移除相应的WakeLock标志位。
    if (mWakefulness != WAKEFULNESS_DOZING) {
      mWakeLockSummary &= ~(WAKE_LOCK_DOZE | WAKE_LOCK_DRAW);
    }
    // asleep或有dozing的WakeLock标志位。
    if (mWakefulness == WAKEFULNESS_ASLEEP
        || (mWakeLockSummary & WAKE_LOCK_DOZE) != 0) {
      mWakeLockSummary &= ~(WAKE_LOCK_SCREEN_BRIGHT | WAKE_LOCK_SCREEN_DIM
          | WAKE_LOCK_BUTTON_BRIGHT);
      // 休眠时传感器不监听。
      if (mWakefulness == WAKEFULNESS_ASLEEP) {
        mWakeLockSummary &= ~WAKE_LOCK_PROXIMITY_SCREEN_OFF;
      }
    }
if ((mWakeLockSummary & (WAKE_LOCK_SCREEN_BRIGHT |
 WAKE_LOCK_SCREEN_DIM)) != 0) {
      if (mWakefulness == WAKEFULNESS_AWAKE) {
        mWakeLockSummary |= WAKE_LOCK_CPU | WAKE_LOCK_STAY_AWAKE;
      } else if (mWakefulness == WAKEFULNESS_DREAMING) {
        mWakeLockSummary |= WAKE_LOCK_CPU;
      }
    }
    if ((mWakeLockSummary & WAKE_LOCK_DRAW) != 0) {
      mWakeLockSummary |= WAKE_LOCK_CPU;
    }
  }
}

```

#### 2.2 PowerMS类的updateUserActivitySummaryLocked方法

```java
// 根据用户最后活动决定当前屏幕状态。
private void updateUserActivitySummaryLocked(long now, int dirty) {
  if ((dirty & (DIRTY_WAKE_LOCKS | DIRTY_USER_ACTIVITY
      | DIRTY_WAKEFULNESS | DIRTY_SETTINGS)) != 0) {
    mHandler.removeMessages(MSG_USER_ACTIVITY_TIMEOUT);
    long nextTimeout = 0;
    if (mWakefulness == WAKEFULNESS_AWAKE
        || mWakefulness == WAKEFULNESS_DREAMING
        || mWakefulness == WAKEFULNESS_DOZING) {
      // 获取进入休眠状态的时间。
      final int sleepTimeout = getSleepTimeoutLocked();
      // 获取灭屏时间。
      final int screenOffTimeout = getScreenOffTimeoutLocked(sleepTimeout);
      // 获取屏幕变暗时间。
      final int screenDimDuration =
  getScreenDimDurationLocked(screenOffTimeout);
      final boolean userInactiveOverride =
mUserInactiveOverrideFromWindowManager;
      mUserActivitySummary = 0;
      // 唤醒状态下发生过用户事件。
      if (mLastUserActivityTime >= mLastWakeTime) {
        // 重新计算出屏幕变暗的时间。
        nextTimeout = mLastUserActivityTime
            + screenOffTimeout - screenDimDuration;
        // 没到屏幕变暗时间。
        if (now < nextTimeout) {
          mUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
        } else {
          nextTimeout = mLastUserActivityTime + screenOffTimeout;
          // 没到灭屏时间。
          if (now < nextTimeout) {
            mUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
          }
        }
      }
      if (mUserActivitySummary == 0
          && mLastUserActivityTimeNoChangeLights >= mLastWakeTime) {
        // 计算下次灭屏时间。
        nextTimeout = mLastUserActivityTimeNoChangeLights + screenOffTimeout;
        // 没到灭屏时间。
        if (now < nextTimeout) {
          if (mDisplayPowerRequest.policy ==
DisplayPowerRequest.POLICY_BRIGHT) {
            mUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
          } else if (mDisplayPowerRequest.policy ==
              DisplayPowerRequest.POLICY_DIM) {
            mUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
          }
        }
      }
      if (mUserActivitySummary == 0) {
        if (sleepTimeout >= 0) {
          // 计算用户活动的最后时间。
          final long anyUserActivity = Math.max(mLastUserActivityTime,
              mLastUserActivityTimeNoChangeLights);
          // 唤醒状态下进行用户活动才会重新更新休眠时间。
          if (anyUserActivity >= mLastWakeTime) {
            nextTimeout = anyUserActivity + sleepTimeout;
            if (now < nextTimeout) {
              mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
            }
          }
        } else {
          mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
          nextTimeout = -1;
        }
      }
      if (mUserActivitySummary != USER_ACTIVITY_SCREEN_DREAM &&
 userInactiveOverride) {
        if ((mUserActivitySummary &
            (USER_ACTIVITY_SCREEN_BRIGHT | USER_ACTIVITY_SCREEN_DIM)) != 0) {
          if (nextTimeout >= now && mOverriddenTimeout == -1) {
            mOverriddenTimeout = nextTimeout;
          }
        }
        mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
        nextTimeout = -1;
      }
      if (mUserActivitySummary != 0 && nextTimeout >= 0) {
        Message msg = mHandler.obtainMessage(MSG_USER_ACTIVITY_TIMEOUT);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, nextTimeout);
      }
    } else {
      mUserActivitySummary = 0;
    }
  }
}
```

#### 2.3 PowerMS类的updateWakefulnessLocked方法

```java
private boolean updateWakefulnessLocked(int dirty) {
  boolean changed = false;
  if ((dirty & (DIRTY_WAKE_LOCKS | DIRTY_USER_ACTIVITY | DIRTY_BOOT_COMPLETED
      | DIRTY_WAKEFULNESS | DIRTY_STAY_ON | DIRTY_PROXIMITY_POSITIVE
      | DIRTY_DOCK_STATE)) != 0) {
    if (mWakefulness == WAKEFULNESS_AWAKE && isItBedTimeYetLocked()) {
      final long time = SystemClock.uptimeMillis();
      // 判断是否进入dream状态。
      if (shouldNapAtBedTimeLocked()) {
        changed = napNoUpdateLocked(time, Process.SYSTEM_UID);
      } else {
        changed = goToSleepNoUpdateLocked(time,
            PowerManager.GO_TO_SLEEP_REASON_TIMEOUT, 0, Process.SYSTEM_UID);
      }
    }
  }
  return changed;
}

private boolean isItBedTimeYetLocked() {
  return mBootCompleted && !isBeingKeptAwakeLocked();
}

private boolean isBeingKeptAwakeLocked() {
  return mStayOn
      || mProximityPositive
      || (mWakeLockSummary & WAKE_LOCK_STAY_AWAKE) != 0
      || (mUserActivitySummary & (USER_ACTIVITY_SCREEN_BRIGHT
          | USER_ACTIVITY_SCREEN_DIM)) != 0
      || mScreenBrightnessBoostInProgress;
}
```

### 三、更新Display Power State

#### 3.1 PowerMS类的updateDisplayPowerStateLocked方法

```java
private boolean updateDisplayPowerStateLocked(int dirty) {
  final boolean oldDisplayReady = mDisplayReady;
  if ((dirty & (DIRTY_WAKE_LOCKS | DIRTY_USER_ACTIVITY | DIRTY_WAKEFULNESS
      | DIRTY_ACTUAL_DISPLAY_POWER_STATE_UPDATED | DIRTY_BOOT_COMPLETED
      | DIRTY_SETTINGS | DIRTY_SCREEN_BRIGHTNESS_BOOST)) != 0) {
    mDisplayPowerRequest.policy = getDesiredScreenPolicyLocked();
    boolean brightnessSetByUser = true;
    int screenBrightness = mScreenBrightnessSettingDefault;
    float screenAutoBrightnessAdjustment = 0.0f;
    boolean autoBrightness = (mScreenBrightnessModeSetting ==
        Settings.System.SCREEN_BRIGHTNESS_MODE_AUTOMATIC);
    // 屏幕亮度。
    ......
    // 更新mDisplayPowerRequest参数。
    .......
mDisplayReady =
 mDisplayManagerInternal.requestPowerState(mDisplayPowerRequest,
        mRequestWaitForNegativeProximity);
    mRequestWaitForNegativeProximity = false;
  }
  return mDisplayReady && !oldDisplayReady;
}

```

#### 3.2 DisplayManager类的requestPowerState方法

```java
public boolean requestPowerState(DisplayPowerRequest request,
    boolean waitForNegativeProximity) {
  synchronized (mLock) {
    boolean changed = false;
    // 传感器检测距离。
    if (waitForNegativeProximity
        && !mPendingWaitForNegativeProximityLocked) {
      mPendingWaitForNegativeProximityLocked = true;
      changed = true;
    }
    if (mPendingRequestLocked == null) {
      mPendingRequestLocked = new DisplayPowerRequest(request);
      changed = true;
    } else if (!mPendingRequestLocked.equals(request)) {
      mPendingRequestLocked.copyFrom(request);
      changed = true;
    }
    if (changed) {
      mDisplayReadyLocked = false;
    }
    if (changed && !mPendingRequestChangedLocked) {
      mPendingRequestChangedLocked = true;
      sendUpdatePowerStateLocked();
    }
    return mDisplayReadyLocked;
  }
}
```

### 四、更新Dream State

#### 4.1 PowerMS类的updateDreamLocked方法

```java
private void updateDreamLocked(int dirty, boolean displayBecameReady) {
  if ((dirty & (DIRTY_WAKEFULNESS
      | DIRTY_USER_ACTIVITY
      | DIRTY_WAKE_LOCKS
      | DIRTY_BOOT_COMPLETED
      | DIRTY_SETTINGS
      | DIRTY_IS_POWERED
      | DIRTY_STAY_ON
      | DIRTY_PROXIMITY_POSITIVE
      | DIRTY_BATTERY_STATE)) != 0 || displayBecameReady) {
    if (mDisplayReady) {
      scheduleSandmanLocked();
    }
  }
}

private void scheduleSandmanLocked() {
  if (!mSandmanScheduled) {
    mSandmanScheduled = true;
    Message msg = mHandler.obtainMessage(MSG_SANDMAN);
    msg.setAsynchronous(true);
    mHandler.sendMessage(msg);
  }
}
```

```java
private void handleSandman() {
  final boolean startDreaming;
  final int wakefulness;
  synchronized (mLock) {
    mSandmanScheduled = false;
    wakefulness = mWakefulness;
if (mSandmanSummoned && mDisplayReady) {
  // 判断设备是否可以进入Dream/Doze状态。
      startDreaming = canDreamLocked() || canDozeLocked();
      mSandmanSummoned = false;
    } else {
      startDreaming = false;
    }
  }
  final boolean isDreaming;
  if (mDreamManager != null) {
if (startDreaming) {
// 结束旧梦
      mDreamManager.stopDream(false /*immediate*/);
      // 开始新梦
      mDreamManager.startDream(wakefulness == WAKEFULNESS_DOZING);
    }
    isDreaming = mDreamManager.isDreaming();
  } else {
    isDreaming = false;
  }
  synchronized (mLock) {
    if (startDreaming && isDreaming) {
      mBatteryLevelWhenDreamStarted = mBatteryLevel;
      if (wakefulness == WAKEFULNESS_DOZING) {
        Slog.i(TAG, "Dozing...");
      } else {
        Slog.i(TAG, "Dreaming...");
      }
    }
    if (mSandmanSummoned || mWakefulness != wakefulness) {
      return;
    }
    if (wakefulness == WAKEFULNESS_DREAMING) {
      if (isDreaming && canDreamLocked()) {
        if (mDreamsBatteryLevelDrainCutoffConfig >= 0
            && mBatteryLevel < mBatteryLevelWhenDreamStarted
                - mDreamsBatteryLevelDrainCutoffConfig
            && !isBeingKeptAwakeLocked()) {
          ......
        } else {
          return;
        }
      }
      if (isItBedTimeYetLocked()) {
        // 休眠。
        goToSleepNoUpdateLocked(SystemClock.uptimeMillis(),
            PowerManager.GO_TO_SLEEP_REASON_TIMEOUT, 0, Process.SYSTEM_UID);
        updatePowerStateLocked();
      } else {
        // 唤醒。
        wakeUpNoUpdateLocked(SystemClock.uptimeMillis(), "android.server.power:DREAM",
            Process.SYSTEM_UID, mContext.getOpPackageName(),
Process.SYSTEM_UID);
        updatePowerStateLocked();
      }
    } else if (wakefulness == WAKEFULNESS_DOZING) {
      if (isDreaming) {
        return;
      }
      reallyGoToSleepNoUpdateLocked(SystemClock.uptimeMillis(), Process.SYSTEM_UID);
      updatePowerStateLocked();
    }
  }
  if (isDreaming) {
    mDreamManager.stopDream(false /*immediate*/);
  }
}

private boolean canDreamLocked() {
  if (mWakefulness != WAKEFULNESS_DREAMING
      // 设备支持Dreaming。
      || !mDreamsSupportedConfig
      // 设置开启Dreaming开关。
      || !mDreamsEnabledSetting
      // 屏幕熄灭。
      || !mDisplayPowerRequest.isBrightOrDim()
      || (mUserActivitySummary & (USER_ACTIVITY_SCREEN_BRIGHT
          | USER_ACTIVITY_SCREEN_DIM | USER_ACTIVITY_SCREEN_DREAM)) == 0
      || !mBootCompleted) {
    return false;
  }
  // 非唤醒状态。
  if (!isBeingKeptAwakeLocked()) {
    // 未充电，未设置电影选项，不进入Dreaming。
    if (!mIsPowered && !mDreamsEnabledOnBatteryConfig) {
      return false;
}
// 没充电，电池电量过低，不进入Dreaming。
    if (!mIsPowered
        && mDreamsBatteryLevelMinimumWhenNotPoweredConfig >= 0
        && mBatteryLevel < mDreamsBatteryLevelMinimumWhenNotPoweredConfig) {
      return false;
}
// 充电，电池电量过低，不进入Dreaming。
    if (mIsPowered
        && mDreamsBatteryLevelMinimumWhenPoweredConfig >= 0
        && mBatteryLevel < mDreamsBatteryLevelMinimumWhenPoweredConfig) {
      return false;
    }
  }
  return true;
}

```

```java
private boolean canDozeLocked() {
  return mWakefulness == WAKEFULNESS_DOZING;
}

```

#### 4.2 DreamManagerService类

```java
private final class LocalService extends DreamManagerInternal {
  public void startDream(boolean doze) {
    startDreamInternal(doze);
  }
  ......
}

private void startDreamInternal(boolean doze) {
  final int userId = ActivityManager.getCurrentUser();
  final ComponentName dream = chooseDreamForUser(doze, userId);
  if (dream != null) {
    synchronized (mLock) {
      startDreamLocked(dream, false /*isTest*/, doze, userId);
    }
  }
}

private void startDreamLocked(final ComponentName name,
    final boolean isTest, final boolean canDoze, final int userId) {
  if (Objects.equal(mCurrentDreamName, name)
      && mCurrentDreamIsTest == isTest
      && mCurrentDreamCanDoze == canDoze
      && mCurrentDreamUserId == userId) {
    return;
  }
  // 停止当前屏保。
  stopDreamLocked(true /*immediate*/);
  final Binder newToken = new Binder();
  mCurrentDreamToken = newToken;
  mCurrentDreamName = name;
  mCurrentDreamIsTest = isTest;
  mCurrentDreamCanDoze = canDoze;
  mCurrentDreamUserId = userId;
  PowerManager.WakeLock wakeLock = mPowerManager
      .newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "startDream");
  mHandler.post(wakeLock.wrap(
      () -> mController.startDream(newToken, name, isTest, canDoze, userId, wakeLock)));
}
```

#### 4.3 DreamController类

```java
public void startDream(Binder token, ComponentName name,
boolean isTest, boolean canDoze, int userId, PowerManager.WakeLock wakeLock) {
  // 移除当前屏保
  stopDream(true /*immediate*/);
  ......
  try {
    mContext.sendBroadcastAsUser(mCloseNotificationShadeIntent, UserHandle.ALL);
    mCurrentDream = new DreamRecord(token, name, isTest, canDoze, userId, wakeLock);
    mDreamStartTime = SystemClock.elapsedRealtime();
    MetricsLogger.visible(mContext,
        mCurrentDream.mCanDoze ? MetricsEvent.DOZING :
     MetricsEvent.DREAMING);
    try {
      mIWindowManager.addWindowToken(token, WindowManager.LayoutParams.TYPE_DREAM);
    } catch (RemoteException ex) {
      stopDream(true /*immediate*/);
      return;
    }
    Intent intent = new Intent(DreamService.SERVICE_INTERFACE);
    intent.setComponent(name);
    intent.addFlags(Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
try {
  // 启动屏保服务。
      if (!mContext.bindServiceAsUser(intent, mCurrentDream,
          Context.BIND_AUTO_CREATE | Context.BIND_FOREGROUND_SERVICE,
          new UserHandle(userId))) {
        stopDream(true /*immediate*/);
        return;
      }
    } catch (SecurityException ex) {
      stopDream(true /*immediate*/);
      return;
    }
    mCurrentDream.mBound = true;
    mHandler.postDelayed(mStopUnconnectedDreamRunnable, DREAM_CONNECTION_TIMEOUT);
  } finally {
    ......
  }
}
```

### 五、更新Suspend Blocker

```java
PowerManagerService.java::updateSuspendBlockerLocked()方法
private void updateSuspendBlockerLocked() {
  // 是否有CPU的wakelock决定CPU是否唤醒。
  final boolean needWakeLockSuspendBlocker = ((mWakeLockSummary &
WAKE_LOCK_CPU) != 0);
  // 根据屏幕状态决定是否需持有屏幕锁。
  final boolean needDisplaySuspendBlocker =
needDisplaySuspendBlockerLocked();
  // 屏幕若无需自动开启，即可自动熄灭。
  final boolean autoSuspend = !needDisplaySuspendBlocker;
  final boolean interactive = mDisplayPowerRequest.isBrightOrDim();
  if (!autoSuspend && mDecoupleHalAutoSuspendModeFromDisplayConfig) {
    // native函数调用底层autoSuspend_disable。
    setHalAutoSuspendModeLocked(false);
  }
  // 若需要获取CPU\屏幕锁。
  if (needWakeLockSuspendBlocker && !mHoldingWakeLockSuspendBlocker) {
    mWakeLockSuspendBlocker.acquire();
    mHoldingWakeLockSuspendBlocker = true;
  }
  if (needDisplaySuspendBlocker && !mHoldingDisplaySuspendBlocker) {
    mDisplaySuspendBlocker.acquire();
    mHoldingDisplaySuspendBlocker = true;
  }
  if (mDecoupleHalInteractiveModeFromDisplayConfig) {
    if (interactive || mDisplayReady) {
      // 调用底层动态库的setHalInteractive函数决定系统是否可交互。
      setHalInteractiveModeLocked(interactive);
    }
  }
  // 若不需要释放CPU\屏幕锁。
  if (!needWakeLockSuspendBlocker && mHoldingWakeLockSuspendBlocker) {
    mWakeLockSuspendBlocker.release();
    mHoldingWakeLockSuspendBlocker = false;
  }
  if (!needDisplaySuspendBlocker && mHoldingDisplaySuspendBlocker) {
    mDisplaySuspendBlocker.release();
    mHoldingDisplaySuspendBlocker = false;
  }
  // 若需要设置自动休眠模式。
  if (autoSuspend && mDecoupleHalAutoSuspendModeFromDisplayConfig) {
    setHalAutoSuspendModeLocked(true);
  }
}
```
