
### 1、BatteryStatsService调用BatteryStatsImpl

```java
        BatteryStatsService(File systemDir, Handler handler) {
            ....
            // AMS创建BatteryStatsImpl类
            mStats = new BatteryStatsImpl(systemDir, handler, mHandler, this);
        }
```

### 2、BatteryStatsImpl构造函数

```java
        public BatteryStatsImpl(File systemDir, Handler handler, ExternalStatsSync externalSync,
                                PlatformIdleStateCallback cb) {
            this(new SystemClocks(), systemDir, handler, externalSync, cb);
        }

        public BatteryStatsImpl(Clocks clocks, File systemDir, Handler handler,
                ExternalStatsSync externalSync, PlatformIdleStateCallback cb) {
            init(clocks);
            // 创建batterystats.bin文件
            if (systemDir != null) {
                mFile = new JournaledFile(new File(systemDir, "batterystats.bin"),
                        new File(systemDir, "batterystats.bin.tmp"));
            } else {
                mFile = null;
            }
            ......
        }
```

```java
        private void init(Clocks clocks) {
            mClocks = clocks;
            // NetworkStats是实现Parcelable接口的对象，用于统计网络相关数据
            mMobileNetworkStats = new NetworkStats[] {
                    new NetworkStats(mClocks.elapsedRealtime(), 50),
                    new NetworkStats(mClocks.elapsedRealtime(), 50),
                    new NetworkStats(mClocks.elapsedRealtime(), 50)
            };
            mWifiNetworkStats = new NetworkStats[] {
                    new NetworkStats(mClocks.elapsedRealtime(), 50),
                    new NetworkStats(mClocks.elapsedRealtime(), 50),
                    new NetworkStats(mClocks.elapsedRealtime(), 50)
            };
        }
```