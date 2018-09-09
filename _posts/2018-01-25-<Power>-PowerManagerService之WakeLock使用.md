&emsp;我们都知道Android系统中有那么一套机制用于控制空闲休眠忙时唤醒。而这套机制就是由PowerMS管理的WakeLock，接下来我们看看它是如何工作的。

### 一、创建WakeLock

#### 1.1 以RIL为例

```java
//  RIL.java的构造函数
public RIL(Context context, int preferredNetworkType,
    int cdmaSubscription, Integer instanceId) {
  ......
  PowerManager pm =
  (PowerManager)context.getSystemService(Context.POWER_SERVICE);
  // 获取WakeLock，第一个参数决定等级和Flag。
  mWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
RILJ_LOG_TAG);
  mWakeLock.setReferenceCounted(false);
  mAckWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
      RILJ_ACK_WAKELOCK_NAME);
  // 一次申请对应一次释放，设为false后，一次释放可对应所有的申请。
  mAckWakeLock.setReferenceCounted(false);
  mWakeLockTimeout =
      SystemProperties.getInt(TelephonyProperties.PROPERTY_WAKE_LOCK_TIMEOUT,
      DEFAULT_WAKE_LOCK_TIMEOUT_MS);
  mAckWakeLockTimeout = SystemProperties.getInt(
      TelephonyProperties.PROPERTY_WAKE_LOCK_TIMEOUT, DEFAULT_ACK_WAKE_LOCK_TIMEOUT_MS);
  mWakeLockCount = 0;
  ......
}
```

&emsp;很容易发现上述代码调用PowerManager的newWakeLock方法创建新的wakeloc。

```java
public WakeLock newWakeLock(int levelAndFlags, String tag) {
  // 检查参数有效性，即levelAndFlags必须对应于PowerManager中定义的WakeLock级别和flag。
  validateWakeLockParameters(levelAndFlags, tag);
  // PM内部类
  return new WakeLock(levelAndFlags, tag, mContext.getOpPackageName());
}
```

&emsp;下面看看PowerManager类中的介绍。

```java
//Wake lock level -- 7种
// CPU-ON,Screen-Off,Keyboard backlight-Off;按下Power键Screen-Off,CPU保持。
public static final int PARTIAL_WAKE_LOCK = 0x00000001;
// Screen-Dim,Keyboard backlight-Off;按下Power键Screen-Off,CPU-OFF。
// 大部分应用应使用FLAG_KEEP_SCREEN_ON替代它。
public static final int SCREEN_DIM_WAKE_LOCK = 0x00000006;
// Screen-Bright,Keyboard backlight-Off;按下Power键Screen-Off,CPU-OFF。
public static final int SCREEN_BRIGHT_WAKE_LOCK = 0x0000000a;
// Screen-Bright,Keyboard backlight-Bright;按下Power键creen-Off,CPU-OFF。
// 大部分应用应使用FLAG_KEEP_SCREEN_ON替代它。
public static final int FULL_WAKE_LOCK = 0x0000001a;
// 传感器感应到它物靠近设备时屏幕灭，拉开距离后屏幕又亮。
public static final int PROXIMITY_SCREEN_OFF_WAKE_LOCK = 0x00000020;
// Doze模式下Screen-Lowpower,CPU-Suspend;
public static final int DOZE_WAKE_LOCK = 0x00000040;
// Doze模式下让设备有足够时间Draw。
public static final int DRAW_WAKE_LOCK = 0x00000080;

// Wake lock flag
// 获取锁，Screen-On。
public static final int ACQUIRE_CAUSES_WAKEUP = 0x10000000;
// 释放锁，延长一小段时间后Screen-Off。
public static final int ON_AFTER_RELEASE = 0x20000000;
// 获取锁，打印Log日志事件
public static final int UNIMPORTANT_FOR_LOGGING = 0x40000000;
```

#### 1.2 WakeLock构造函数

```java
WakeLock(int flags, String tag, String packageName) {
  mFlags = flags;
  mTag = tag;
  mPackageName = packageName;
  // PMS作为Binder客户端监听对应进程是否死亡
  mToken = new Binder();
  mTraceName = "WakeLock (" + mTag + ")";
}
```

### 二、Acquire WakeLock

&emsp;进程创建wakelock后，需要将其发送给PowerMS，交其管理。这个过程即acquire wakelock。

#### 2.1 继续以RIL为例

```java
// RIL类的send方法
private void send(RILRequest rr) {
  Message msg;
  if (mSocket == null) {
    rr.onError(RADIO_NOT_AVAILABLE, null);
    rr.release();
    return;
  }
  msg = mSender.obtainMessage(EVENT_SEND, rr);
  acquireWakeLock(rr, FOR_WAKELOCK);
  msg.sendToTarget();
}

// RIL类在发送信息给RILSender之前会调用acquireWakeLock方法
private void acquireWakeLock(RILRequest rr, int wakeLockType) {
  synchronized(rr) {
    ......
    switch(wakeLockType) {
      case FOR_WAKELOCK:
        synchronized (mWakeLock) {
          mWakeLock.acquire();
          mWakeLockCount++;
          mWlSequenceNum++;
          ......
        }
        break;
      ......
    }
    rr.mWakeLockType = wakeLockType;
  }
}
```

#### 2.2 PowerManager类的WakeLock方法

```java
public void acquire() {
  synchronized (mToken) {
    acquireLocked();
  }
}

private void acquireLocked() {
  if (!mRefCounted || mCount++ == 0) {
    mHandler.removeCallbacks(mReleaser);
    Trace.asyncTraceBegin(Trace.TRACE_TAG_POWER, mTraceName, 0);
    try {
      mService.acquireWakeLock(mToken, mFlags, mTag, mPackageName, mWorkSource,
          mHistoryTag);
    } catch (RemoteException e) {
      throw e.rethrowFromSystemServer();
    }
    mHeld = true;
  }
}
```

#### 2.3 PowerMS类的BinderService方法

```java
public void acquireWakeLock(IBinder lock, int flags, String tag, String packageName,
    WorkSource ws, String historyTag) {
  ......
  final int uid = Binder.getCallingUid();
  final int pid = Binder.getCallingPid();
  final long ident = Binder.clearCallingIdentity();
  try {
    acquireWakeLockInternal(lock, flags, tag, packageName, ws, historyTag, uid, pid);
  } finally {
    Binder.restoreCallingIdentity(ident);
  }
}

private void acquireWakeLockInternal(IBinder lock, int flags, String tag, String packageName,
    WorkSource ws, String historyTag, int uid, int pid) {
  synchronized (mLock) {
    // PowerMS中内部类
    WakeLock wakeLock;
    // PMS中维持一个ArrayList，记录当前已申请的WakeLock
    // findWakeLockIndexLocked查找ArrayList，判断参数对应的WakeLock是否在之前被申请过。
    int index = findWakeLockIndexLocked(lock);
    boolean notifyAcquire;
    if (index >= 0) {
      // 如果index大于0，Acquire的是一个旧的WakeLock
      wakeLock = mWakeLocks.get(index);
      // 判断WakeLock对应的成员变量是否发生改变。
      if (!wakeLock.hasSameProperties(flags, tag, ws, uid, pid)) {
        notifyWakeLockChangingLocked(wakeLock, flags, tag, packageName,
            uid, pid, ws, historyTag);
        // 若wakelock属性发生变化，更新该属性。
        wakeLock.updateProperties(flags, tag, packageName, ws, historyTag, uid, pid);
      }
      notifyAcquire = false;
    } else {
      // 创建一个新的WakeLock。
      wakeLock = new WakeLock(lock, flags, tag, packageName, ws, historyTag, uid, pid);
      try {
        // 监控申请WakeLock的进程是否死亡。
        lock.linkToDeath(wakeLock, 0);
      } catch (RemoteException ex) {
        throw new IllegalArgumentException("Wake lock is already dead.");
      }
      // 添加到wakelock列表。
      mWakeLocks.add(wakeLock);
      // 根据Doze白名单更新wakelock的disable变量。
      setWakeLockDisabledStateLocked(wakeLock);
      qcNsrmPowExt.checkPmsBlockedWakelocks(uid, pid, flags, tag, wakeLock);
      notifyAcquire = true;
    }
    // 判断WakeLock是否有ACQUIRE_CAUSES_WAKEUP，必要时唤醒。
    applyWakeLockFlagsOnAcquireLocked(wakeLock, uid);
    mDirty |= DIRTY_WAKE_LOCKS;
    // 更新电源状态。
    updatePowerStateLocked();
    if (notifyAcquire) {
      // 通知wakelock发生变化，电量统计服务做相关统计。
      notifyWakeLockAcquiredLocked(wakeLock);
    }
  }
}
```

#### 2.4 PowerMS的WakeLock方法

```java
//检测客户端死亡
public void binderDied() {
  PowerManagerService.this.handleWakeLockDeath(this);
}

private void handleWakeLockDeath(WakeLock wakeLock) {
  synchronized (mLock) {
    int index = mWakeLocks.indexOf(wakeLock);
    if (index < 0) {
      return;
    }
    removeWakeLockLocked(wakeLock, index);
  }
}

private void removeWakeLockLocked(WakeLock wakeLock, int index) {
  mWakeLocks.remove(index);
  // 通知到BatteryStatsService。
  notifyWakeLockReleasedLocked(wakeLock);
  // 判断是否立即灭屏。
  applyWakeLockFlagsOnReleaseLocked(wakeLock);
  mDirty |= DIRTY_WAKE_LOCKS;
  // 锁移除后，更新电源状态。
  updatePowerStateLocked();
}

```

```java
// 特殊处理PARTIAL_WAKE_LOCK
private boolean setWakeLockDisabledStateLocked(WakeLock wakeLock) {
  if ((wakeLock.mFlags & PowerManager.WAKE_LOCK_LEVEL_MASK)
      == PowerManager.PARTIAL_WAKE_LOCK) {
    boolean disabled = false;
    // 设备处于Doze模式。
    if (mDeviceIdleMode) {
      final int appid = UserHandle.getAppId(wakeLock.mOwnerUid);
      // 判断是否为非系统应用。
      if (appid >= Process.FIRST_APPLICATION_UID &&
          // 白名单搜索。
          Arrays.binarySearch(mDeviceIdleWhitelist, appid) < 0 &&
          Arrays.binarySearch(mDeviceIdleTempWhitelist, appid) < 0 &&
          // 数字越大，对处理事件的时效性要求越低。
          mUidState.get(wakeLock.mOwnerUid,
              ActivityManager.PROCESS_STATE_CACHED_EMPTY)
              > ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE) {
        disabled = true;
      }
    }
    if (wakeLock.mDisabled != disabled) {
      wakeLock.mDisabled = disabled;
      return true;
    }
  }
  return false;
}
```

```java
// 处理flag
private void applyWakeLockFlagsOnAcquireLocked(WakeLock wakeLock, int uid) {
  if ((wakeLock.mFlags & PowerManager.ACQUIRE_CAUSES_WAKEUP) != 0
      && isScreenLock(wakeLock)) {
    String opPackageName;
    int opUid;
    if (wakeLock.mWorkSource != null && wakeLock.mWorkSource.getName(0) != null) {
      opPackageName = wakeLock.mWorkSource.getName(0);
      opUid = wakeLock.mWorkSource.get(0);
    } else {
      opPackageName = wakeLock.mPackageName;
      opUid = wakeLock.mWorkSource != null ? wakeLock.mWorkSource.get(0)
          : wakeLock.mOwnerUid;
    }
    wakeUpNoUpdateLocked(SystemClock.uptimeMillis(), wakeLock.mTag, opUid,
        opPackageName, opUid);
  }
}

private boolean wakeUpNoUpdateLocked(long eventTime, String reason, int reasonUid,
    String opPackageName, int opUid) {
  if (eventTime < mLastSleepTime || mWakefulness == WAKEFULNESS_AWAKE
      || !mBootCompleted || !mSystemReady) {
    return false;
  }
  try {
    mLastWakeTime = eventTime;
    // PowerMS根据mDirty的位信息管理电源状态，同时唤醒屏幕。
    setWakefulnessLocked(WAKEFULNESS_AWAKE, 0);
    // 通知电源统计服务
    mNotifier.onWakeUp(reason, reasonUid, opPackageName, opUid);
    userActivityNoUpdateLocked(
        eventTime, PowerManager.USER_ACTIVITY_EVENT_OTHER, 0, reasonUid);
  } finally {
    ......
  }
  return true;
}

private void setWakefulnessLocked(int wakefulness, int reason) {
  if (mWakefulness != wakefulness) {
    mWakefulness = wakefulness;
    mWakefulnessChanging = true;
    mDirty |= DIRTY_WAKEFULNESS;
    mNotifier.onWakefulnessChangeStarted(wakefulness, reason);
  }
}

public void onWakefulnessChangeStarted(final int wakefulness, int reason) {
  final boolean interactive = PowerManagerInternal.isInteractive(wakefulness);
  mHandler.post(new Runnable() {
    @Override
    public void run() {
      mActivityManagerInternal.onWakefulnessChanged(wakefulness);
    }
  });
  if (mInteractive != interactive) {
    if (mInteractiveChanging) {
      handleLateInteractiveChange();
    }
    mInputManagerInternal.setInteractive(interactive);
    mInputMethodManagerInternal.setInteractive(interactive);
    try {
      mBatteryStats.noteInteractive(interactive);
    } catch (RemoteException ex) { }
    mInteractive = interactive;
    mInteractiveChangeReason = reason;
    mInteractiveChanging = true;
    handleEarlyInteractiveChange();
  }
}

private void handleEarlyInteractiveChange() {
  synchronized (mLock) {
    if (mInteractive) {
      mHandler.post(new Runnable() {
        @Override
        public void run() {
          EventLog.writeEvent(EventLogTags.POWER_SCREEN_STATE, 1, 0, 0, 0);
          mPolicy.startedWakingUp();
        }
      });
      mPendingInteractiveState = INTERACTIVE_STATE_AWAKE;
      mPendingWakeUpBroadcast = true;
      updatePendingBroadcastLocked();
    } else {
      final int why = translateOffReason(mInteractiveChangeReason);
      mHandler.post(new Runnable() {
        @Override
        public void run() {
          mPolicy.startedGoingToSleep(why);
        }
      });
    }
  }
}

```

```java
private void applyWakeLockFlagsOnReleaseLocked(WakeLock wakeLock) {
  if ((wakeLock.mFlags & PowerManager.ON_AFTER_RELEASE) != 0
      && isScreenLock(wakeLock)) {
    userActivityNoUpdateLocked(SystemClock.uptimeMillis(),
        PowerManager.USER_ACTIVITY_EVENT_OTHER,
        PowerManager.USER_ACTIVITY_FLAG_NO_CHANGE_LIGHTS,
        wakeLock.mOwnerUid);
  }
}

private boolean userActivityNoUpdateLocked(long eventTime, int event, int flags, int uid) {
  if (eventTime < mLastSleepTime || eventTime < mLastWakeTime
      || !mBootCompleted || !mSystemReady) {
    return false;
  }
  try {
    if (eventTime > mLastInteractivePowerHintTime) {
      powerHintInternal(POWER_HINT_INTERACTION, 0);
      mLastInteractivePowerHintTime = eventTime;
    }
    mNotifier.onUserActivity(event, uid);
    if (mUserInactiveOverrideFromWindowManager) {
      mUserInactiveOverrideFromWindowManager = false;
      mOverriddenTimeout = -1;
    }
    if (mWakefulness == WAKEFULNESS_ASLEEP
        || mWakefulness == WAKEFULNESS_DOZING
        || (flags & PowerManager.USER_ACTIVITY_FLAG_INDIRECT) != 0) {
      return false;
    }
    if ((flags & PowerManager.USER_ACTIVITY_FLAG_NO_CHANGE_LIGHTS) != 0) {
      if (eventTime > mLastUserActivityTimeNoChangeLights
          && eventTime > mLastUserActivityTime) {
        mLastUserActivityTimeNoChangeLights = eventTime;
        mDirty |= DIRTY_USER_ACTIVITY;
        return true;
      }
    } else {
      if (eventTime > mLastUserActivityTime) {
        mLastUserActivityTime = eventTime;
        mDirty |= DIRTY_USER_ACTIVITY;
        return true;
      }
    }
  } finally {
    ......
  }
  return false;
}
```

### 三、释放WakeLock

#### 3.1 RIL类的processResponse方法

```java
private void processResponse (Parcel p) {
  int type;
  type = p.readInt();
  if (type == RESPONSE_UNSOLICITED || type == RESPONSE_UNSOLICITED_ACK_EXP) {
    processUnsolicited (p, type);
  } else if (type == RESPONSE_SOLICITED || type == RESPONSE_SOLICITED_ACK_EXP) {
    RILRequest rr = processSolicited (p, type);
    if (rr != null) {
      if (type == RESPONSE_SOLICITED) {
        decrementWakeLock(rr);
      }
      rr.release();
      return;
    }
  } else if (type == RESPONSE_SOLICITED_ACK) {
    int serial;
    serial = p.readInt();
    RILRequest rr;
    synchronized (mRequestList) {
      rr = mRequestList.get(serial);
    }
    if (rr == null) {
      Rlog.w(RILJ_LOG_TAG, "Unexpected solicited ack response! sn: " + serial);
    } else {
      decrementWakeLock(rr);
      if (RILJ_LOGD) {
        riljLog(rr.serialString() + " Ack < " + requestToString(rr.mRequest));
      }
    }
  }
}

private void decrementWakeLock(RILRequest rr) {
  synchronized(rr) {
    switch(rr.mWakeLockType) {
      case FOR_WAKELOCK:
        synchronized (mWakeLock) {
          if (mWakeLockCount > 1) {
            mWakeLockCount--;
          } else {
            mWakeLockCount = 0;
            mWakeLock.release();
          }
        }
        break;
      case FOR_ACK_WAKELOCK:
        break;
      case INVALID_WAKELOCK:
        break;
      default:
        Rlog.w(RILJ_LOG_TAG, "Decrementing Invalid Wakelock type " + rr.mWakeLockType);
    }
    rr.mWakeLockType = INVALID_WAKELOCK;
  }
}
```

#### 3.2 PowerManager类的WakeLock中定义的release方法

```java
public void release(int flags) {
  synchronized (mToken) {
    if (!mRefCounted || --mCount == 0) {
      mHandler.removeCallbacks(mReleaser);
      if (mHeld) {
        Trace.asyncTraceEnd(Trace.TRACE_TAG_POWER, mTraceName, 0);
        try {
          // 调用PowerMS中releaseWakeLock方法
          mService.releaseWakeLock(mToken, flags);
        } catch (RemoteException e) {
          throw e.rethrowFromSystemServer();
        }
        mHeld = false;
      }
    }
    if (mCount < 0) {
      throw new RuntimeException("WakeLock under-locked " + mTag);
    }
  }
}
```

#### 3.3 PowerMS类中处理wakelock的release

```java
public void releaseWakeLock(IBinder lock, int flags) {
  if (lock == null) {
    throw new IllegalArgumentException("lock must not be null");
  }
  mContext.enforceCallingOrSelfPermission(android.Manifest.permission.WAKE_LOCK, null);
  final long ident = Binder.clearCallingIdentity();
  try {
    releaseWakeLockInternal(lock, flags);
  } finally {
    Binder.restoreCallingIdentity(ident);
  }
}

private void releaseWakeLockInternal(IBinder lock, int flags) {
  synchronized (mLock) {
    int index = findWakeLockIndexLocked(lock);
    if (index < 0) {
      return;
    }
    WakeLock wakeLock = mWakeLocks.get(index);
    if ((flags & PowerManager.RELEASE_FLAG_WAIT_FOR_NO_PROXIMITY) != 0) {
      mRequestWaitForNegativeProximity = true;
    }
    wakeLock.mLock.unlinkToDeath(wakeLock, 0);
    removeWakeLockLocked(wakeLock, index);
  }
}

private void removeWakeLockLocked(WakeLock wakeLock, int index) {
  mWakeLocks.remove(index);
  notifyWakeLockReleasedLocked(wakeLock);
  applyWakeLockFlagsOnReleaseLocked(wakeLock);
  mDirty |= DIRTY_WAKE_LOCKS;
  updatePowerStateLocked();
}
```
