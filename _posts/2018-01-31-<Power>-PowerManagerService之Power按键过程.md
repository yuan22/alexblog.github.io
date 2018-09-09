&emsp;本文只介绍Power键相关内容，如果对输入系统感兴趣请自行查阅InputManagerService相关资料。先来看看PhoneWindowManager中interceptKeyBeforeQueueing方法：

```java
PhoneWindowManager.java::interceptKeyBeforeQueueing()方法
public int interceptKeyBeforeQueueing(KeyEvent event, int policyFlags) {
  // 是否开机完成。
  if (!mSystemBooted) {
    return 0;
  }
  // 屏幕是否点亮。
  final boolean interactive = (policyFlags & FLAG_INTERACTIVE) != 0;
  final boolean down = event.getAction() == KeyEvent.ACTION_DOWN;
  final boolean canceled = event.isCanceled();
  // 按键编码。
  final int keyCode = event.getKeyCode();
  ......
  switch (keyCode) {
    ......
    case KeyEvent.KEYCODE_POWER: {
      result &= ~ACTION_PASS_TO_USER;
      isWakeKey = false;
      if (down) {
        interceptPowerKeyDown(event, interactive);
      } else {
        interceptPowerKeyUp(event, interactive, canceled);
      }
      break;
    }
    ......
  }
  ......
  if (isWakeKey) {
wakeUp(event.getEventTime(), mAllowTheaterModeWakeFromKey,
 "android.policy:KEY");
  }
  return result;
}
```

&emsp;接下来再看看interceptPowerKeyDown（按下）和interceptPowerKeyUp（松开）姐妹方法。

### 一、interceptPowerKeyDown方法

```java
private void interceptPowerKeyDown(KeyEvent event, boolean interactive) {
  if (!mPowerKeyWakeLock.isHeld()) {
    mPowerKeyWakeLock.acquire();
  }
  if (mPowerKeyPressCounter != 0) {
    mHandler.removeMessages(MSG_POWER_DELAYED_PRESS);
  }
  boolean panic = mImmersiveModeConfirmation.onPowerKeyDown(interactive,
      SystemClock.elapsedRealtime(), isImmersiveMode(mLastSystemUiFlags),
      isNavBarEmpty(mLastSystemUiFlags));
  if (panic) {
    mHandler.post(mHiddenNavPanic);
  }
  // 截屏。
  if (interactive && !mScreenshotChordPowerKeyTriggered
      && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
    mScreenshotChordPowerKeyTriggered = true;
    mScreenshotChordPowerKeyTime = event.getDownTime();
    interceptScreenshotChord();
  }
  TelecomManager telecomManager = getTelecommService();
  boolean hungUp = false;
  if (telecomManager != null) {
    if (telecomManager.isRinging()) {
      // 来电静音。
      telecomManager.silenceRinger();
    } else if ((mIncallPowerBehavior
        & Settings.Secure.INCALL_POWER_BUTTON_BEHAVIOR_HANGUP) != 0
        && telecomManager.isInCall() && interactive) {
      // 来电拒接。
      hungUp = telecomManager.endCall();
    }
  }
  GestureLauncherService gestureService = LocalServices.getService(
      GestureLauncherService.class);
  boolean gesturedServiceIntercepted = false;
  if (gestureService != null) {
    // 手势服务。
    gesturedServiceIntercepted = gestureService.interceptPowerKeyDown(event, interactive,
        mTmpBoolean);
    if (mTmpBoolean.value && mGoingToSleep) {
      mCameraGestureTriggeredDuringGoingToSleep = true;
    }
  }
  mPowerKeyHandled = hungUp || mScreenshotChordVolumeDownKeyTriggered
      || mScreenshotChordVolumeUpKeyTriggered || gesturedServiceIntercepted;
  if (!mPowerKeyHandled) {
    if (interactive) {
      // 长按。
      if (hasLongPressOnPowerBehavior()) {
        Message msg = mHandler.obtainMessage(MSG_POWER_LONG_PRESS);
        msg.setAsynchronous(true);
        mHandler.sendMessageDelayed(msg,
            ViewConfiguration.get(mContext).getDeviceGlobalActionKeyTimeout());
      }
    } else {
      // 唤醒系统。
      wakeUpFromPowerKey(event.getDownTime());
      // 熄屏长按。
      if (mSupportLongPressPowerWhenNonInteractive &&
hasLongPressOnPowerBehavior()) {
        Message msg = mHandler.obtainMessage(MSG_POWER_LONG_PRESS);
        msg.setAsynchronous(true);
        mHandler.sendMessageDelayed(msg,
            ViewConfiguration.get(mContext).getDeviceGlobalActionKeyTimeout());
        mBeganFromNonInteractive = true;
      } else {
        final int maxCount = getMaxMultiPressPowerCount();
        if (maxCount <= 1) {
          mPowerKeyHandled = true;
        } else {
          mBeganFromNonInteractive = true;
        }
      }
    }
  }
}
```

#### 1.1 hasLongPressOnPowerBehavior方法

```java
//是否支持长按行为
private boolean hasLongPressOnPowerBehavior() {
return getResolvedLongPressOnPowerBehavior() !=
LONG_PRESS_POWER_NOTHING;
}

private int getResolvedLongPressOnPowerBehavior() {
  if (FactoryTest.isLongPressOnPowerOffEnabled()) {
    return LONG_PRESS_POWER_SHUT_OFF_NO_CONFIRM;
  }
  return mLongPressOnPowerBehavior;
}

public void init(Context context, IWindowManager windowManager,
    WindowManagerFuncs windowManagerFuncs) {
  ......
  mLongPressOnPowerBehavior = mContext.getResources().getInteger(
      com.android.internal.R.integer.config_longPressOnPowerBehavior);
  ......
}

```

#### 1.2 MSG_POWER_LONG_PRESS消息处理

```java
// 处理MSG_POWER_LONG_PRESS消息
private class PolicyHandler extends Handler {
  @Override
  public void handleMessage(Message msg) {
    switch (msg.what) {
      ......
      case MSG_POWER_LONG_PRESS:
        powerLongPress();
      ......
    }
  }
}

private void powerLongPress() {
  final int behavior = getResolvedLongPressOnPowerBehavior();
  switch (behavior) {
  case LONG_PRESS_POWER_NOTHING:
    break;
  case LONG_PRESS_POWER_GLOBAL_ACTIONS:
    mPowerKeyHandled = true;
    // performHapticFeedbackLw震动反馈。
    if (!performHapticFeedbackLw(null, HapticFeedbackConstants.LONG_PRESS, false)) {
      performAuditoryFeedbackForAccessibilityIfNeed();
    }
    // 关机、重启对话框。
    showGlobalActionsInternal();
    break;
  ......
  }
}
```

#### 1.3 wakeUpFromPowerKey方法

```java
// 灭屏按Power键调用wakeUpFromPowerKey方法
private void wakeUpFromPowerKey(long eventTime) {
  wakeUp(eventTime, mAllowTheaterModeWakeFromPowerKey,
"android.policy:POWER");
}

private boolean wakeUp(long wakeTime, boolean wakeInTheaterMode, String reason) {
  // 取数据库中THEATER_MODE_ON值。
  final boolean theaterModeEnabled = isTheaterModeEnabled();
  // 按power键返回false。
  if (!wakeInTheaterMode && theaterModeEnabled) {
    return false;
  }
  // 唤醒时退出剧院模式
  if (theaterModeEnabled) {
    Settings.Global.putInt(mContext.getContentResolver(),
        Settings.Global.THEATER_MODE_ON, 0);
  }
  mPowerManager.wakeUp(wakeTime, reason);
  return true;
}
```

```java
PowerManager.java
public void wakeUp(long time, String reason) {
  try {
    mService.wakeUp(time, reason, mContext.getOpPackageName());
  } catch (RemoteException e) {
    throw e.rethrowFromSystemServer();
  }
}

```

```java
//PowerMS类的wakeup方法
public void wakeUp(long eventTime, String reason, String opPackageName) {
  ......
  try {
    wakeUpInternal(eventTime, reason, uid, opPackageName, uid);
  } finally {
    Binder.restoreCallingIdentity(ident);
  }
}

private void wakeUpInternal(long eventTime, String reason, int uid, String opPackageName,
    int opUid) {
  synchronized (mLock) {
    if (wakeUpNoUpdateLocked(eventTime, reason, uid, opPackageName, opUid)) {
      updatePowerStateLocked();
    }
  }
}
```

### 二、interceptPowerKeyUp方法

```java
private void interceptPowerKeyUp(KeyEvent event, boolean interactive, boolean canceled) {
  final boolean handled = canceled || mPowerKeyHandled;
  mScreenshotChordPowerKeyTriggered = false;
  // 退出截屏
  cancelPendingScreenshotChordAction();
  // 取消MSG_POWER_LONG_PRESS事件
  cancelPendingPowerKeyAction();
  // 短按Power键
  if (!handled) {
    mPowerKeyPressCounter += 1;
    final int maxCount = getMaxMultiPressPowerCount();
    final long eventTime = event.getDownTime();
    if (mPowerKeyPressCounter < maxCount) {
      // 每次按下发送MSG_POWER_DELAYED_PRESS消息
      Message msg = mHandler.obtainMessage(MSG_POWER_DELAYED_PRESS,
          interactive ? 1 : 0, mPowerKeyPressCounter, eventTime);
      msg.setAsynchronous(true);
      mHandler.sendMessageDelayed(msg, ViewConfiguration.getDoubleTapTimeout());
      return;
    }
    powerPress(eventTime, interactive, mPowerKeyPressCounter);
  }
  finishPowerKeyPress();
}
```

#### 2.1 powerPress方法

```java
private void powerPress(long eventTime, boolean interactive, int count) {
  if (mScreenOnEarly && !mScreenOnFully) {
    ......
    return;
  }
  if (count == 2) {
    powerMultiPressAction(eventTime, interactive, mDoublePressOnPowerBehavior);
  } else if (count == 3) {
    powerMultiPressAction(eventTime, interactive, mTriplePressOnPowerBehavior);
  } else if (interactive && !mBeganFromNonInteractive) {
    switch (mShortPressOnPowerBehavior) {
      case SHORT_PRESS_POWER_NOTHING:
        break;
      case SHORT_PRESS_POWER_GO_TO_SLEEP:
        mPowerManager.goToSleep(eventTime,
            PowerManager.GO_TO_SLEEP_REASON_POWER_BUTTON, 0);
        break;
      ......
    }
  }
}
```

#### 2.2 goToSleep方法

```java
// PowerManager
public void goToSleep(long time, int reason, int flags) {
  try {
    mService.goToSleep(time, reason, flags);
  } catch (RemoteException e) {
    throw e.rethrowFromSystemServer();
  }
}
```

```java
// PowerMS
public void goToSleep(long eventTime, int reason, int flags) {
  if (eventTime > SystemClock.uptimeMillis()) {
    throw new IllegalArgumentException("event time must not be in the future");
  }
  mContext.enforceCallingOrSelfPermission(
      android.Manifest.permission.DEVICE_POWER, null);
  final int uid = Binder.getCallingUid();
  final long ident = Binder.clearCallingIdentity();
  try {
    goToSleepInternal(eventTime, reason, flags, uid);
  } finally {
    Binder.restoreCallingIdentity(ident);
  }
}

private void goToSleepInternal(long eventTime, int reason, int flags, int uid) {
  synchronized (mLock) {
    if (goToSleepNoUpdateLocked(eventTime, reason, flags, uid)) {
      updatePowerStateLocked();
    }
  }
}
```

#### 2.3 finishPowerKeyPress方法

```java
private void finishPowerKeyPress() {
  mBeganFromNonInteractive = false;
  mPowerKeyPressCounter = 0;
  if (mPowerKeyWakeLock.isHeld()) {
    mPowerKeyWakeLock.release();
  }
}
```
