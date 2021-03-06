#### 一、A/B系统更新的特点

&emsp;所谓A/B系统，即有A和B两套系统，一套系统分区，另一套备份分区。这两套系统出厂时一样，此后可能不一样，即一个新版本，另一个旧版本，旧版本升级至新版本，不断更新切换。

&emsp;A/B系统更新又称之为无缝更新(Seamless updates),具有以下几个特点：

&emsp;<1>OTA更新在系统后台运行，该过程中用户仍可以正常使用设备。待更新完成后，需重启一次方可进入新系统。

&emsp;<2>OTA更新失败后(OTA更新无法应用或应用后无法启动)，设备可重启回滚到旧分区继续使用，并重新尝试更新升级。

&emsp;<3>可采用流式升级方式，即无需/data或/cache分区留出足够空间用于存储下载的升级包，也可进行OTA升级。

&emsp;<4>确保在OTA更新期间磁盘上保留一套可以正常启动使用的系统，减少刷机变砖的可能性，减轻售后服务工作量。

#### 二、A/B系统更新的变化

&emsp;Google从Android 7.0开始引入A/B系统更新功能，由于A/B update同Non-A/B update分区设计不兼容，所以7.0之前系统设备无法体验A/B系统更新。

&emsp;与传统Non-A/B系统更新相比，A/B系统更新变化如下：

&emsp;1.分区设计不同；

&emsp;&emsp;<1>Non-A/B仅一套分区，损坏即无法开机使用；

&emsp;&emsp;<2>A/B有slot A和slot B两套分区，损坏一套，另一套依然可能开机使用。

&emsp;2.系统编译方式不同：

&emsp;&emsp;<1>Non-A/B中boot.img存储系统ramdisk，recovery.img存储recovery系统的ramdisk；

&emsp;&emsp;<2>A/B只有boot.img，无recovery.img。

&emsp;3.启动方式不同：

&emsp;&emsp;<1>Non-A/B bootloder通过读取misc分区判断进入Android系统还是Recovery系统；

&emsp;&emsp;<2>A/B bootloader通过特定程序决定选择slot A还是slot B启动。

&emsp;4.升级包内容不同：

&emsp;&emsp;虽然Non A/B和A/B系统的OTA包编译工具和指令一样，但是生成的OTA包内容不同。


#### 三、A/B系统更新过程

&emsp;常见场景及状态变化整理如下：

&emsp;1.普通场景(Normal case)：

&emsp;&emsp;系统从当前slot启动(A或B)，当前系统所处slot状态为bootable,successful, active。

&emsp;2.升级中(Update in progress)：

&emsp;&emsp;slot A检测到升级，在slot B中升级。将slot B标记为unbootable，清除successful标志；slot A仍为bootable, successful和active。

&emsp;3.升级完成，等待重启(Update applied, reboot pending)：

&emsp;&emsp;slot A将B升级后，将B标记为bootable,重启前将B标记为active以便重启后从B启动，需重启验证B后才能将其设置为successful;A变为bootable, successful。

&emsp;4.从新系统升级(System rebooted into new update):

&emsp;&emsp;重启后，bootloader将B标记为active,若A能正常启动运行，则将其标记为successful。重新回到普通场景。

![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2018-06-30-ab-update-introduce/ab_update_slot_status.png)

&emsp;当OTA包可供下载时，升级流程即启动了。由于A/B更新是在后台运行的，用户可能并不知道正在更新，因此更新流程可能会因策略、意外重启、用户操作等而被中断。

&emsp;常见策略控制包括：电量是否足够，用户是否活动，是否支持数据流量更新等等。这些取决于OTA开发是否支持，比如Google OTA即可在server端增加该类限制。

