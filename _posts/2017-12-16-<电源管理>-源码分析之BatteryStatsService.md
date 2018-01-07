
### 1、AMS启动BatteryStatsService

```java
        public ActivityManagerService(Context systemContext) {
            ......
            // 创建BatteryStatsService对象，传入/data/system目录及AMS的mHandler
            mBatteryStatsService = new BatteryStatsService(systemDir, mHandler);
            // 调用BatteryStatsImpl的readLocked方法
            mBatteryStatsService.getActiveStatistics().readLocked();
            // 更新BatteryStatsImpl最新信息，写入磁盘
            mBatteryStatsService.scheduleWriteToDisk();
            mOnBattery = DEBUG_POWER ? true
                    : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
            mBatteryStatsService.getActiveStatistics().setCallback(this);
            ......
        }
```

### 2、BatteryStatsService构造函数

```java
        BatteryStatsService(File systemDir, Handler handler) {
            // 创建一个不同于AMS的本地服务线程访问硬盘
            final ServiceThread thread = new ServiceThread("batterystats-sync",
                    Process.THREAD_PRIORITY_DEFAULT, true);
            thread.start();
            mHandler = new BatteryStatsHandler(thread.getLooper());
            // AMS创建BatteryStatsImpl类
            mStats = new BatteryStatsImpl(systemDir, handler, mHandler, this);
        }
```

### 3、AMS调用BSS的publish函数

```java
        private void start() {
            ......
            mBatteryStatsService.publish(mContext);
            ......
        }
```

### 4、BatteryStatsService的publish方法

```java
        public void publish(Context context) {
            mContext = context;
            // BatteryStatsImpl设置mPhoneSignalScanningTimer超时时间（统计搜索信号时间）
            mStats.setRadioScanningTimeout(mContext.getResources().getInteger(
                    com.android.internal.R.integer.config_radioScanningTimeout)
                    * 1000L);
            // 创建衡量软硬件耗电能力的PowerProfile，交给BatteryStatsImpl使用
            mStats.setPowerProfile(new PowerProfile(context));
            // 将BatteryStats服务注册到ServiceManager进程中
            ServiceManager.addService(BatteryStats.SERVICE_NAME, asBinder());
        }
```

### 5、AMS调用BSS的initPowerManagent方法

&emsp;SystemServer调用AMS的initPowerManager方法.

```java
        public void initPowerManagement() {
            ......
            // 调用initPowerManagement方法
            mBatteryStatsService.initPowerManagement();
            ......
        }
```

### 6、BatteryStatsService的initPowerManagement方法

```java
        // 构造函数运行时，power manager还没有初始化。所以稍后初始化低功耗观察者
        public void initPowerManagement() {
            final PowerManagerInternal powerMgr = LocalServices.getService(PowerManagerInternal.class);
            // 向PowerManagerService注册LowPowerMode观察者
            powerMgr.registerLowPowerModeObserver(this);
            // 进入、退出LowPowerMode时，调用BatteryStatsImpl的notePowerSaveMode方法
            mStats.notePowerSaveMode(powerMgr.getLowPowerModeEnabled());
            // 启动WakeupReasonThread
            (new WakeupReasonThread()).start();
        }
```

### 7、BatteryStatsService的内部类WakeupReasonThread

```java
        final class WakeupReasonThread extends Thread {
            ......
            public void run() {
                // 设置前台线程优先级
                Process.setThreadPriority(Process.THREAD_PRIORITY_FOREGROUND);
                // 解码
                mDecoder = StandardCharsets.UTF_8
                        .newDecoder()
                        .onMalformedInput(CodingErrorAction.REPLACE)
                        .onUnmappableCharacter(CodingErrorAction.REPLACE)
                        .replaceWith("?");
                mUtf8Buffer = ByteBuffer.allocateDirect(MAX_REASON_SIZE);
                mUtf16Buffer = CharBuffer.allocate(MAX_REASON_SIZE);
                try {
                    String reason;
                    while ((reason = waitWakeup()) != null) {
                        synchronized (mStats) {
                            // 记录唤醒原因
                            mStats.noteWakeupReasonLocked(reason);
                        }
                    }
                } catch (RuntimeException e) {
                    Slog.e(TAG, "Failure reading wakeup reasons", e);
                }
            }
                ......
                // 写入字节数设置缓存限制
                mUtf8Buffer.limit(bytesWritten);
                // 将缓存由utf-8解码为utf-16，无法映射的予以替代
                mDecoder.decode(mUtf8Buffer, mUtf16Buffer, true);
                mUtf16Buffer.flip();
                // 创建utf-16缓存
                return mUtf16Buffer.toString();
            }
        }
```
