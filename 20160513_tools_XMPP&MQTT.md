## 消息推送简介

消息推送分两个层面讲：

1. 首先是连接，在客户端与服务端建立轻量级且可靠的连接：长连接+心跳（心跳时间间隔最为重要）
     * 慢心跳原理：NAT，运营商信令问题，5分钟&28分钟；
     * Mobile唤醒：Alarm 和 PowerManager Wakeup
     * 网络休眠
2. 然后是协议，在连接之上的数据协议，数据协议自然是要解决业务问题，对业务的分析与扩展很重要。
     * XMPP
     * MQTT
     * 自定义：注册认证、定向推送、消息保留、消息确认

心跳包问题补充：

你说的不发心跳包也可以维持长链接，这是对的，因为tcp是有保活定时器的，默认是用保活定时器来维持长连接，但是为什么要发心跳包？因为保活定时器的周期是两小时。这属于一个补充点。

## 移动网络的特点

当一台智能手机连上移动网络时，其实并没有真正连接上Internet，运营商分配给手机的IP其实是运营商的内网IP，手机终端要连接上Internet还必须通过运营商的网关进行IP地址的转换，这个网关简称为NAT(NetWork Address Translation)，简单来说就是手机终端连接Internet 其实就是移动内网IP，端口，外网IP之间相互映射。相当于在手机终端在移动无线网络这堵墙上打个洞与外面的Internet相连。

由于大部分的移动无线网络运营商为了减少网关NAT映射表的负荷，如果一个链路有一段时间没有通信时就会删除其对应表，造成链路中断，正是这种刻意缩短空闲连接的释放超时，原本是想节省信道资源的作用，没想到让互联网的应用不得以远高于正常频率发送心跳来维护推送的长连接。这也是为什么会有之前的信令风暴，微信摇收费的传言，因为这类的应用发送心跳的频率是很短的，既造成了信道资源的浪费，也造成了手机电量的快速消耗。

## AlarmManager

Alarm manager功能相对比较简单，相关代码位于

一.frameworks/base/core/jni/server/com_android_server_AlarmManagerService.cpp

这部分代码直接管理硬件时钟，设备名为/dev/alarm。包括打开设备，关闭设备，设置时区，设置触发时间（timeout），以及等待时钟触发。

二. frameworks/base/services/java/com/android/server/AlarmManagerService.java

这部分封装目录一中的代码，向上提供java接口，同时与客户端（如calendar）交互，接收来自客户端的时钟设置请求，并在时钟触发时通知客户端。Alarm是在预定的时间上触发Intent的一种独立的方法。

> adb shell dumpsys alarm 解析

参考：http://stackoverflow.com/questions/28742884/how-to-read-adb-shell-dumpsys-alarm-output

>> Q1: Batches

   从API 19开始,当应用设置ALARM的时候，系统不会将这些ALARM在设置的准确时间内触发，而将用一种批量触发（batches mode）的策略，这样可以最小化地使系统从休眠状态醒来，最低程度地减少电池的消耗，即将一批触发时间接近的Alarm，压缩到某一个时间段内一起触发，而不是一个个触发，这样系统会很难休眠。这种机制称为batch，也有厂商称为唤醒对齐：`Pending alarm batches: 31`
    
        Current Alarm Manager state:
        
        //相对开机时间计算即SystemClock.elapsedRealtime()： +3d23h23m54s877ms=877+54000+1380000+82800000+259200000=343434877ms
        nowRTC=1463364959595=2016-05-16 10:15:59 nowELAPSED=+3d23h23m54s877ms
        
        Next non-wakeup alarm: +404ms = 2016-05-16 10:15:59
        
        Next wakeup: +3m15s805ms = 2016-05-16 10:19:15
        
        Num time change events: 1
        
        
        
        Pending alarm batches: 31
        
        //触发时间343435281 = 343434877ms + 404ms
        Batch{25cf8667 num=1 start=343435281 end=343435281 STANDALONE}:
            ELAPSED #0: Alarm{2ef9cc14 type 3 when 343435281 android}
        
            tag=*alarm*:android.intent.action.TIME_TICK
        
            type=3 whenElapsed=+404ms when=+404ms
        
            window=0 repeatInterval=0 count=0
        
            operation=PendingIntent{3c09604b: PendingIntentRecord{30570a28 android broadcastIntent}}
        Batch{8022cbd num=1 start=343630682 end=343630682 STANDALONE}:
            ......
        
   先看下两个示例：
   
    示例1： 相对时间，一分钟后触发
    alarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime()
                 + 60*1000, pendingIntent);
    示例2： 绝对时间，一分钟后触发 （绝对时间可能有个问题就是用户或网络可能改时间）
    alarmManager.set(AlarmManager.RTC_WAKEUP, System.currentTimeMillis()
                 + 60*10000, pendingIntent);

   dump出来的`Batch{25cf8667 num=1 start=343435281 end=343435281 STANDALONE}`就是相对时间；
   
>> Q2: Alarms

   每一个Alarm会如下：
   
    ELAPSED #0: Alarm{2ef9cc14 type 3 when 343435281 android}

    tag=*alarm*:android.intent.action.TIME_TICK

    type=3 whenElapsed=+404ms when=+404ms

    window=0 repeatInterval=0 count=0

    operation=PendingIntent{3c09604b: PendingIntentRecord{30570a28 android broadcastIntent}}
    
   * 第一行的android表示的是哪个package设置的Alarm；
   * typ=3表示的是RTC_WAKEUP, RTC, ELAPSED_WAKEUP, or ELAPSED中的值（0~3）；
   * whenElapsed=+404ms表示的是多长时间后触发；
   * window=0代表的是AlarmManager.WINDOW_EXACT=0或AlarmManager.WINDOW_HEURISTIC=-1，和设置Alarm时调用的setExact()、setAlarmClock()、setInexactRepeating()有关；
   * repeatInterval=0 count=0代表循环触发的时间间隔及次数；
   * operation=PendingIntent表示的是通过getService, getBroadcast, getActivity, or getActivities设置的Alarm对应的触发动作；可以通过`adb shell dumpsys activity intents | grep 30570a28` 来查看该Intent的详细信息；

>> Broadcast Ref Count

        Past-due non-wakeup alarms: (none)

        Number of delayed alarms: 3971, total delay time: +1d7h0m17s422ms
        
        Max delay time: +10m46s931ms, max non-interactive time: +6h27m24s849ms
        
        Broadcast ref count: 0
    
        Top Alarms:
        
   Broadcast Ref Count: 0 simply means that at the time that dumpsys alarm was run, it was not in the middle of sending any broadcasts
   
>> Top Alarms

        Top Alarms:

        +48d2h55m36s622ms running, 0 wakeups, 1425 alarms: 1000:android
    
          *alarm*:android.intent.action.TIME_TICK
    
        +36d10h23m25s671ms running, 0 wakeups, 13 alarms: u0a75:com.google.android.gms
    
          *alarm*:com.google.android.intent.action.SEND_IDLE
    
        +32d15h27m9s615ms running, 0 wakeups, 15 alarms: u0a69:com.tencent.mm
    
          *alarm*:ALARM_ACTION(14984)
    
        +31d20h49m3s999ms running, 0 wakeups, 22 alarms: u0a101:com.tencent.mobileqq
    
          *alarm*:com.tencent.mobileqq.msf.WatchdogForInfoLogin
    
        +31d4h19m10s1ms running, 0 wakeups, 46 alarms: u0a69:com.tencent.mm
    
          *alarm*:ALARM_ACTION(12022)
    
        +30d0h13m40s365ms running, 0 wakeups, 3 alarms: u0a69:com.tencent.mm
    
          *alarm*:ALARM_ACTION(28115)
    
        +26d20h18m17s181ms running, 0 wakeups, 5 alarms: u0a112:com.google.android.googlequicksearchbox
    
          *alarm*:com.google.android.googlequicksearchbox/com.google.android.apps.gsa.tasks.VelvetBackgroundTasksIntentService
    
        +26d8h17m25s785ms running, 0 wakeups, 15 alarms: u0a111:com.android.vending
    
          *alarm*:com.android.vending/com.google.android.finsky.receivers.FlushLogsReceiver
    
        +25d18h0m51s261ms running, 0 wakeups, 17 alarms: u0a143:com.facebook.katana
    
          *alarm*:com.facebook.common.executors.WakingExecutorService.ACTION_ALARM.com.facebook.katana.Mqtt_Wakeup
    
        +24d0h27m35s66ms running, 0 wakeups, 59 alarms: u0a111:com.android.vending
    
          *alarm*:com.android.vending/com.google.android.finsky.services.ContentSyncService

   表示的是：根据应用的唤醒系统的时间排行，取最长时间的前十名，然后按照降序列出，有助于找出第三方app因为编程不规范，而导致极度耗电
   
>> Alarm Stats

        u0a69:com.tencent.mm +2d10h23m58s182ms running, 0 wakeups:

            +32d15h27m9s615ms 0 wakes 15 alarms: *alarm*:ALARM_ACTION(14984)
            
            +31d4h19m10s1ms 0 wakes 46 alarms: *alarm*:ALARM_ACTION(12022)
            
            +30d0h13m40s365ms 0 wakes 3 alarms: *alarm*:ALARM_ACTION(28115)
            
            +21d17h35m46s816ms 0 wakes 3 alarms: *alarm*:ALARM_ACTION(14890)
            
            +7d3h48m31s645ms 0 wakes 6 alarms: *alarm*:ALARM_ACTION(11694)
            
            +4d0h13m11s819ms 0 wakes 1 alarms: *alarm*:ALARM_ACTION(30691)
            
            +3d12h32m57s110ms 0 wakes 0 alarms: *alarm*:ALARM_ACTION(27931)
            
            +2m29s42ms 0 wakes 798 alarms: *alarm*:com.tencent.mm.TrafficStatsReceiver
            
            +1m1s914ms 0 wakes 288 alarms: *alarm*:com.tencent.mm/.booter.MMReceivers$AlarmReceiver
        
        u0a75:com.google.android.gms +3d1h55m4s474ms running, 0 wakeups:
        
            +36d10h23m25s671ms 0 wakes 13 alarms: *alarm*:com.google.android.intent.action.SEND_IDLE
            
            +18d17h51m38s197ms 0 wakes 19 alarms: *alarm*:com.google.android.gms.icing.proxy.action.SMS_CHANGED
            
            +16d8h2m33s233ms 0 wakes 0 alarms: *alarm*:com.google.android.gms.icing.INDEX_ONETIME_MAINTENANCE
            
            +15d22h46m50s984ms 0 wakes 5 alarms: *alarm*:com.google.android.gms.gcm.ACTION_CHECK_QUEUE
            
            ......
            
   可以查看上次重启后各Alarm的触发情况
   


## Powmanager WakeLock

    Flag Value              CPU  Screen  Keyboard
    PARTIAL_WAKE_LOCK	    On*	 Off	 Off
    SCREEN_DIM_WAKE_LOCK	On	 Dim	 Off
    SCREEN_BRIGHT_WAKE_LOCK	On	 Bright	 Off
    FULL_WAKE_LOCK	        On	 Bright	 Bright

查看WakeLock，分为linux层和应用层：

** linux层 **：

    > cat /sys/power/wake_lock
        PowerManagerService.Display
        PowerManagerService.WakeLocks

PowerManagerService.Display : 这是屏开着是PowerManagerService对linux层设的wakelock

PowerManagerService.WakeLocks: 这是应用层设的wakelock, 所以应用程序的设的wakelock在linux层表现成这个wakelock

PowerManagerService会维护所有应用程序的一个wakelock表，当不为空时，向linux层设置PowerManagerService.WakeLocks, 为空时取消这个wakelock

** 应用层 ** ：

    > dumpsys power
        Wake Locks: size=2

        PARTIAL_WAKE_LOCK              '*sync*/gmail-ls/com.google/work2012kk@gmail.com' (uid=1000, pid=1122, ws=WorkSource{10122})
        
        PARTIAL_WAKE_LOCK              '*sync*/com.android.contacts/com.google/work2012kk@gmail.com' (uid=1000, pid=1122, ws=WorkSource{10075})

Android设备各组件耗电量值见文件frameworks/base/core/res/res/xml/power_profile.xml：

    <device name="Android">
        <!-- Most values are the incremental current used by a feature,
           in mA (measured at nominal voltage).
           The default values are deliberately incorrect dummy values.
           OEM's must measure and provide actual values before
           shipping a device.
           Example real-world values are given in comments, but they
           are totally dependent on the platform and can vary
           significantly, so should be measured on the shipping platform
           with a power meter. -->
        <item name="none">0</item>
        <item name="screen.on">0.1</item>  <!-- ~200mA -->
        <item name="screen.full">0.1</item>  <!-- ~300mA -->
        <item name="bluetooth.active">0.1</item> <!-- Bluetooth data transfer, ~10mA -->
        <item name="bluetooth.on">0.1</item>  <!-- Bluetooth on & connectable, but not connected, ~0.1mA -->
        <item name="wifi.on">0.1</item>  <!-- ~3mA -->
        <item name="wifi.active">0.1</item>  <!-- WIFI data transfer, ~200mA -->
        <item name="wifi.scan">0.1</item>  <!-- WIFI network scanning, ~100mA -->
        <item name="dsp.audio">0.1</item> <!-- ~10mA -->
        <item name="dsp.video">0.1</item> <!-- ~50mA -->
        <item name="radio.active">0.1</item> <!-- ~200mA -->
        <item name="radio.scanning">0.1</item> <!-- cellular radio scanning for signal, ~10mA -->
        <item name="gps.on">0.1</item> <!-- ~50mA -->
        <!-- Current consumed by the radio at different signal strengths, when paging -->
        <array name="radio.on"> <!-- Strength 0 to BINS-1 -->
          <value>0.2</value> <!-- ~2mA -->
          <value>0.1</value> <!-- ~1mA -->
        </array>
        <!-- Different CPU speeds as reported in
           /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state -->
        <array name="cpu.speeds">
          <value>400000</value> <!-- 400 MHz CPU speed -->
        </array>
        <!-- Current when CPU is idle -->
        <item name="cpu.idle">0.1</item>
        <!-- Current at each CPU speed, as per 'cpu.speeds' -->
        <array name="cpu.active">
          <value>0.1</value>  <!-- ~100mA -->
        </array>
        <!-- This is the battery capacity in mAh (measured at nominal voltage) -->
        <item name="battery.capacity">1000</item>
        
        <array name="wifi.batchedscan"> <!-- mA -->
          <value>.0002</value> <!-- 1-8/hr -->
          <value>.002</value>  <!-- 9-64/hr -->
          <value>.02</value>   <!-- 65-512/hr -->
          <value>.2</value>    <!-- 513-4,096/hr -->
          <value>2</value>    <!-- 4097-/hr -->
        </array>
        </device>

相关值的官网说明：https://source.android.com/devices/tech/power/values.html

（举个例子让大家好理解）比如一个手机的电池容量是4000mAh，举个例子刷微博：CPU开启100mA，wifi开启连接200mA，屏幕亮起200mA，按保守算500mA，可以连续使用4000mAh/500mA=8h


## XMPP

在IETF 中，把IM协议划分为四种协议，即即时信息和出席协议(Instant Messaging and Presence Protocol, IMPP)、出席和即时信息协议(Presence and Instant Messaging Protocol, PRIM)、针对即时信息和出席扩展的会话发起协议(Session Initiation Protocol for Instant Messaging and Presence Leveraging Extensions, SIMPLE)，以及可扩展的消息出席协议(XMPP)

> 基本的XMPP 客户端必须实现以下标准协议（XEP-0211）：

1. RFC3920 核心协议Core
2. RFC3921 即时消息和出席协议Instant Messaging and Presence
3. XEP-0030 服务发现Service Discovery
4. XEP-0115 实体能力Entity Capabilities

> 基本的XMPP 服务器必须实现以下标准协议

1. RFC3920 核心协议Core
2. RFC3921 即时消息和出席协议Instant Messaging and Presence
3. XEP-0030 服务发现Service Discovery


> XMPP消息格式：

XMPP通信原语有3种：message、presence和iq


1. XMPP协议数据冗余达50%以上；

## MQTT



## 参考

1. https://segmentfault.com/a/1190000000656509
2. http://www.coderyi.com/archives/434

3. http://blog.csdn.net/clh604/article/details/20167263
4. http://www.cnblogs.com/hanyonglu/archive/2012/03/04/2378971.html
5. http://www.jianshu.com/p/584707554ed7

6. http://stackoverflow.com/questions/5938213/android-alarmmanager-rtc-wakeup-vs-elapsed-realtime-wakeup/15400014#15400014
7. http://stackoverflow.com/questions/28742884/how-to-read-adb-shell-dumpsys-alarm-output