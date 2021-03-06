### 一、初识Doze模式

&emsp;&emsp;从Android 6.0(API级别23)开始，Android引入了两个省电功能，可通过管理应用在设备未连接至电源时的行为方式为用户延长电池寿命。低电耗模式(Doze)通过在设备长时间处于闲置状态时推迟应用的后台CPU和网络Activity来减少电池消耗。应用待机模式(App Standby)可推迟用户近期未与之交互的应用的后台网络Activity。


### 二、理解Doze模式

#### 2.1 进入Doze模式时机

&emsp;1、如果用户设备未插入电源;

&emsp;2、处于静止状态一段时间且屏幕关闭。

#### 2.2 进入Doze模式应用变化

&emsp;1、系统会尝试通过限制应用对网络和CPU密集型服务的访问来节省电量。

&emsp;2、还可以阻止应用访问网络并推迟其作业、同步和标准闹铃。

&emsp;3、系统会定期退出Doze模式一会儿，好让应用完成其已推迟的Activity。在此维护时段内，系统会运行所有待定同步、作业和闹铃并允许应用访问网络。

&emsp;4、在每个维护时段结束后，系统会再次进入Doze模式，暂停网络访问并推迟作业、同步和闹铃。随着时间的推移，系统安排维护时段的次数越来越少。

![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2017-11-20-doze-mode-introduce/doze_m.bmp)

#### 2.3 退出Doze模式时机

&emsp;1、移动设备；

&emsp;2、打开屏幕；

&emsp;3、连接到充电器唤醒设备。

#### 2.4 Doze模式类型
&emsp;&emsp;在Android M上只有Deep Doze模式，在Android N上既有Deep Doze模式也有Light Doze模式，其区别与联系如下：

&emsp;<1>Deep Doze模式需要静止一会儿，但是Light Doze模式无需静止；

&emsp;<2>Deep Doze和Light Doze模式能同时在设备上运行；

&emsp;<3>设备首先进入Light Doze模式，然后才能进入Deep Doze模式；

&emsp;<4>如果设备进入Deep Doze模式，则忽略Light Doze模式。

![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2017-11-20-doze-mode-introduce/doze_n.bmp)

#### 2.5 Doze模式限制(Deep Mode)

&emsp;1、暂停访问网络。

&emsp;2、系统将忽略wake locks。

&emsp;3、标准AlarmManager闹铃(包括setExact()和setWindow())推迟到下一维护时段。

&emsp;&emsp;<1>如果需要设置在Doze模式下触发的闹铃，需使用setAndAllowWhileIdle()或setExactAndAllowWhileIdle()。

&emsp;&emsp;<2>一般情况下，使用setAlarmClock()设置的闹铃将继续触发，但系统会在这些闹铃触发之前不久退出Doze模式。

&emsp;4、系统不执行Wi-Fi扫描。

&emsp;5、系统不允许运行同步适配器。

&emsp;6、系统不允许运行JobScheduler。

&emsp;注：1、5、6适用于Light Doze模式。

#### 2.6 Doze模式检查清单

&emsp;1、尽可能使用 FCM 进行下游消息传递。

&emsp;2、如果用户必须立即查看通知，需使用FCM高优先级消息。

&emsp;3、在初始消息负载中提供足够的信息，这样随后就无需访问网络。

&emsp;4、使用setAndAllowWhileIdle()和setExactAndAllowWhileIdle()设置关键闹铃。

&emsp;5、在Doze模式下测试应用。

#### 2.7 Doze代码分布

&emsp;&emsp;为实现Doze模式，Android M中新增了一个DeviceIdle服务，其实现代码位置为：

```
    frameworks/base/services/core/java/com/android/server/DeviceIdleController.java。
```

&emsp;&emsp;SystemServer在开机时会启动这个服务：

```
    mSystemServiceManager.startService(DeviceIdleController.class)
```

&emsp;&emsp;除了上述的Doze模式限制控制逻辑外，还在NetworkPolicyManagerService、JobSchedulerService、SyncManager、PowerManagerService和AlarmManagerService中加入了对Doze状态的监听和查询接口来进行响应的操作。在DeviceIdleController中实现对设备状态的控制和改变，并且通知其他相关注册了AppIdleStateChangeListener接口的服务进行处理，而反过来这些服务也可以向DeviceIdleController查询device的状态，是一种交互的关系。其控制逻辑和策略实现的代码关系如下图所示：

![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2017-11-20-doze-mode-introduce/doze_device_idle_controller.bmp)

#### 2.8 Doze状态机

&emsp;1、 Android M

&emsp;&emsp;<1>当设备亮屏或者处于正常使用状态时即为ACTIVE状态；

&emsp;&emsp;<2>ACTIVE状态下不插充电器或USB且灭屏，则设备就会切换到INACTIVE状态；

&emsp;&emsp;<3>INACTIVE状态经过30分钟，期间检测没有打断状态的行为，Doze就切换到IDLE_PENDING的状态；

&emsp;&emsp;<4>然后再经过30分钟以及一系列的判断，状态切换到SENSING；

&emsp;&emsp;<5>在SENSING状态下会去检测是否有地理位置变化，没有的话就切到LOCATION状态；

&emsp;&emsp;<6>LOCATION状态下再经过30s的检测时间之后就进入了Doze的核心状态IDLE；

&emsp;&emsp;<7>在IDLE模式下每隔一段时间就会进入一次IDLE_MAINTANCE，此间用来处理之前被挂起的一些任务；

&emsp;&emsp;<8>IDLE_MAINTANCE状态持续5分钟之后会重新回到IDLE状态；

&emsp;&emsp;<9>在除ACTIVE以外的所有状态中，检测到打断的行为如亮屏、插入充电器，位置的改变等状态就会回到ACTIVE，重新开始下一个轮回。

![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2017-11-20-doze-mode-introduce/doze_state_m.bmp)

&emsp;2、Android N

&emsp;&emsp;详见Deep/Light Mode介绍


### 三、理解App standby模式

&emsp;&emsp;App Standby模式允许系统判定应用在用户未主动使用它时处于空闲状态。当用户有一段时间未触摸应用时，系统便会作出此判定，以下条件均不适用：

&emsp;<1>用户显式启动应用。

&emsp;<2>应用当前有一个进程位于前台(表现为Activity或前台服务形式，或被另一Activity 或前台服务占用)。

&emsp;<3>应用生成用户可在锁屏或通知托盘中看到的通知。

&emsp;&emsp;当用户将设备插入电源时，系统将从待机状态释放应用，从而让它们可以自由访问网络并执行任何待定作业和同步。如果设备长时间处于空闲状态，系统将按每天大约一次的频率允许空闲应用访问网络


### 四、测试Doze模式和App Standby模式

#### 4.1 Doze模式

&emsp;1、使用(Android 6.0(API级别23)或更高版本的系统映像配置硬件设备或虚拟设备。

&emsp;2、将设备连接到开发计算机并安装应用

&emsp;3、运行应用并使其保持活动状态

&emsp;4、关闭设备屏幕。(应用保持活动状态)

&emsp;5、通过运行以下命令强制系统在Doze模式之间循环切换：

```shell
    $ adb shell dumpsys battery unplug
    $ adb shell dumpsys deviceidle step light/deep
```

&emsp;&emsp;可能需要多次运行第二个命令。不断地重复，直到设备变为空闲状态。

&emsp;6、通过运行下面指令enable Doze、dump Doze：

```shell
    $ adb shell dumpsys deviceidle enable light/deep/all
    $ adb shell dumpsys deviceidle
```

&emsp;&emsp;MTK平台可将persist.config.AutoPowerModes设置为1来enable Doze

&emsp;7、在重新激活设备后观察应用的行为。确保应用在设备退出Doze模式时正常恢复。

#### 4.2 App Standby模式

&emsp;1、使用Android 6.0(API级别23)或更高版本的系统映像配置硬件设备或虚拟设备。

&emsp;2、运行应用并使其保持活动状态。

&emsp;3、通过运行以下命令强制应用进入App Standby模式：

```shell
    $ adb shell dumpsys battery unplug
    $ adb shell am set-inactive <packageName> true
```

&emsp;4、使用以下命令模拟唤醒应用：

```shell
    $ adb shell am set-inactive <packageName> false
    $ adb shell am get-inactive <packageName>
```

&emsp;5、观察唤醒后的应用行为。确保应用从待机模式中正常恢复。特别地，应检查应用的通知和后台作业是否按预期继续运行。


### 五、其它

#### 5.1 FCM简介

&emsp;&emsp;Firebase Cloud Messaging(FCM)是一项云端至设备的服务，允许支持在后端服务与Android设备上的应用之间实时进行下游消息传递。FCM提供了单一持久的云连接；所有需要实时传递消息的应用均可共享此连接。此共享连接使多个应用无需消耗电池即可维持自身单独的持久连接，避免快速耗尽电池，从而显著优化电池消耗。 因此，如果应用需要与后端服务进行消息传递集成，我们强烈建议尽量使用FCM，而非维持自身持久的网络连接。

&emsp;&emsp;FCM经过优化，可通过高优先级FCM消息用于Doze模式和App Standby模式。FCM高优先级消息允许可靠地唤醒应用访问网络，即使用户设备处于Doze模式或应用处于AppStandby模式也不例外。在Doze模式或App Standby模式下，系统将传递消息并允许应用临时访问网络服务和部分唤醒锁，然后将设备或应用恢复到空闲状态。

&emsp;&emsp;高优先级FCM消息不会影响Doze模式，也不会影响任何其他应用的状态。这意味着应用可以使用这些消息进行有效的通信，同时尽可能减少对整个系统和设备的电池影响。

&emsp;&emsp;作为一项常规最佳做法，如果应用需要下游消息传递，则应使用FCM。如果服务器和客户端已经使用FCM，请确保服务对关键消息使用高优先级消息，因为即使设备处于Doze模式，这也会可靠地唤醒应用。

#### 5.2 对其它用例支持

![](https://xiezeyangcn.github.io/alexblog.github.io/assets/images/2017-11-20-doze-mode-introduce/doze_support_case.bmp)

&emsp;&emsp;通过妥善管理网络连接、闹铃、作业和同步并使用FCM高优先级消息，几乎所有应用都应该能够支持Doze模式。对于一小部分用例，这可能还不够。对于此类用例，系统为部分免除Doze模式和AppStandby模式优化的应用提供了一份可配置的白名单。

&emsp;&emsp;在Doze模式和App Standby模式期间，加入白名单的应用可以使用网络并保留部分wake locks。不过，正如其他应用一样，其他限制仍然适用于加入白名单的应用。 例如，加入白名单的应用的作业和同步将推迟(在API级别23及更低级别中)，并且其常规AlarmManager 闹铃不会触发。通过调用isIgnoringBatteryOptimizations()，应用可以检查自身当前是否位于豁免白名单中。

&emsp;&emsp;用户可以在Settings>Battery>BatteryOptimization中手动配置该白名单。或者，系统会为应用提供请求用户将应用加入白名单的方式：

&emsp;<1>应用可以触发ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS Intent，让用户直接进入 Battery Optimization，他们可以在其中添加应用。

&emsp;<2>具有REQUEST_IGNORE_BATTERY_OPTIMIZATIONS权限的应用可以触发系统对话框，让用户无需转到“设置”即可直接将应用添加到白名单。应用将通过触发ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS Intent来触发该对话框。

&emsp;<3>用户可以根据需要手动从白名单中移除应用。

&emsp;&emsp;在请求用户将应用添加到白名单之前，请确保应用符合加入白名单的可接受用例。

&emsp;注：除非应用的核心功能受到不利影响，否则Google Play政策禁止应用请求直接豁免Android 6.0+中的电源管理功能(Doze模式和App Standby模式)。
