在frameworks/native/cmds有：

    atrace  bugreport  dumpstate  dumpsys  flatland  installd  ip-up-vpn  rawbu  service  servicemanager
    
在frameworks/base/cmds有：

    am           appwidget  bootanimation  content  ime          media        screencap  uiautomator
    appops       backup     bu             dpm      input        pm           settings   wm
    app_process  bmgr       bugbox         idmap    interrupter  requestsync  svc
    
## dumpsys

dumpsys特指dump注册在ServiceManager里的`Service`, 每个`Service`都可以通过实现`dump`接口来说明自己的信息

1. dumpsys -l (显示当前运行的service，**标记的是常用）
        
        window          **
        SurfaceFlinger   **
        account    **
        activity   **
        alarm      **
        cpuinfo     **
        jobscheduler       **
        notification        **
        package     **
        permission      **
        themes      **
        meminfo     **
        
        AtCmdFwd
        accessibility
        DockObserver
        android.security.keystore
        appops
        appwidget
        audio
        backup
        battery
        batteryproperties
        batterystats
        bluetooth_manager
        carrier_config
        clipboard
        cneservice
        com.qualcomm.location.izat.IzatService
        com.qualcomm.qti.auth.fidocryptodaemon
        commontime_management
        connectivity
        consumer_ir
        content
        country_detector
        dbinfo
        device_policy
        device_status
        deviceidle
        devicestoragemonitor
        diskstats
        display
        display.qservice
        dpmservice
        dreams
        drm.drmManager
        dropbox
        ethernet
        extphone
        fingerprint
        fingerprints_service
        gfxinfo
        graphicsstats
        imms
        ims
        input
        input_method
        iphonesubinfo
        isms
        isub
        launcherapps
        location
        lock_settings
        media.audio_flinger
        media.audio_policy
        media.camera
        media.camera.proxy
        media.player
        media.radio
        media.resource_manager
        media.sound_trigger_hw
        media_projection
        media_router
        media_session
        midi
        mount
        netpolicy
        netstats
        network_management
        network_score
        nfc
        phone
        power
        print
        processinfo
        procstats
        qti.ims.connectionmanagerservice
        qtitetherservice
        restrictions
        rttmanager
        samplingprofiler
        scheduling_policy
        search
        sensorservice
        serial
        servicediscovery
        simphonebook
        sip
        statusbar
        telecom
        telephony.registry
        textservices
        trust
        uimode
        updatelock
        usagestats
        usb
        user
        vendor.qcom.PeripheralManager
        vibrator
        voiceinteraction
        wallpaper
        wbc_service
        webviewupdate
        wfdservice
        wifi
        wifip2p
        wifiscanner
        
2. dumpsys activity

3. dumpsys package

4. dumpsys alarm/jobscheduler

## am

    usage: am [subcommand] [options]
    usage: am start [-D] [-W] [-P <FILE>] [--start-profiler <FILE>]
                   [--sampling INTERVAL] [-R COUNT] [-S] [--opengl-trace]
                   [--user <USER_ID> | current] <INTENT>
           am startservice [--user <USER_ID> | current] <INTENT>
           am stopservice [--user <USER_ID> | current] <INTENT>
           am force-stop [--user <USER_ID> | all | current] <PACKAGE>
           am kill [--user <USER_ID> | all | current] <PACKAGE>
           am kill-all
           am broadcast [--user <USER_ID> | all | current] <INTENT>
           am instrument [-r] [-e <NAME> <VALUE>] [-p <FILE>] [-w]
                   [--user <USER_ID> | current]
                   [--no-window-animation] [--abi <ABI>] <COMPONENT>
           am profile start [--user <USER_ID> current] [--sampling INTERVAL] <PROCESS> <FILE>
           am profile stop [--user <USER_ID> current] [<PROCESS>]
           am dumpheap [--user <USER_ID> current] [-n] <PROCESS> <FILE>
           am set-debug-app [-w] [--persistent] <PACKAGE>
           am clear-debug-app
           am set-watch-heap <PROCESS> <MEM-LIMIT>
           am clear-watch-heap
           am monitor [--gdb <port>]
           am hang [--allow-restart]
           am restart
           am idle-maintenance
           am screen-compat [on|off] <PACKAGE>
           am package-importance <PACKAGE>
           am to-uri [INTENT]
           am to-intent-uri [INTENT]
           am to-app-uri [INTENT]
           am switch-user <USER_ID>
           am start-user <USER_ID>
           am stop-user [-w] <USER_ID>
           am stack start <DISPLAY_ID> <INTENT>
           am stack movetask <TASK_ID> <STACK_ID> [true|false]
           am stack resize <STACK_ID> <LEFT,TOP,RIGHT,BOTTOM>
           am stack split <STACK_ID> <v|h> [INTENT]
           am stack list
           am stack info <STACK_ID>
           am task lock <TASK_ID>
           am task lock stop
           am task resizeable <TASK_ID> [true|false]
           am task resize <TASK_ID> <LEFT,TOP,RIGHT,BOTTOM>
           am get-config
           am set-inactive [--user <USER_ID>] <PACKAGE> true|false
           am get-inactive [--user <USER_ID>] <PACKAGE>
           am send-trim-memory [--user <USER_ID>] <PROCESS>
                   [HIDDEN|RUNNING_MODERATE|BACKGROUND|RUNNING_LOW|MODERATE|RUNNING_CRITICAL|COMPLETE]

## pm

    usage: pm list packages [-f] [-d] [-e] [-s] [-3] [-i] [-u] [--user USER_ID] [FILTER]
       pm list permission-groups
       pm list permissions [-g] [-f] [-d] [-u] [GROUP]
       pm list instrumentation [-f] [TARGET-PACKAGE]
       pm list features
       pm list libraries
       pm list users
       pm path PACKAGE
       pm dump PACKAGE
       pm install [-lrtsfd] [-i PACKAGE] [--user USER_ID] [PATH]
       pm install-create [-lrtsfdp] [-i PACKAGE] [-S BYTES]
               [--install-location 0/1/2]
               [--force-uuid internal|UUID]
       pm install-write [-S BYTES] SESSION_ID SPLIT_NAME [PATH]
       pm install-commit SESSION_ID
       pm install-abandon SESSION_ID
       pm uninstall [-k] [--user USER_ID] PACKAGE
       pm set-installer PACKAGE INSTALLER
       pm move-package PACKAGE [internal|UUID]
       pm move-primary-storage [internal|UUID]
       pm clear [--user USER_ID] PACKAGE
       pm enable [--user USER_ID] PACKAGE_OR_COMPONENT
       pm disable [--user USER_ID] PACKAGE_OR_COMPONENT
       pm disable-user [--user USER_ID] PACKAGE_OR_COMPONENT
       pm disable-until-used [--user USER_ID] PACKAGE_OR_COMPONENT
       pm hide [--user USER_ID] PACKAGE_OR_COMPONENT
       pm unhide [--user USER_ID] PACKAGE_OR_COMPONENT
       pm grant [--user USER_ID] PACKAGE PERMISSION
       pm revoke [--user USER_ID] PACKAGE PERMISSION
       pm reset-permissions
       pm set-app-link [--user USER_ID] PACKAGE {always|ask|never|undefined}
       pm get-app-link [--user USER_ID] PACKAGE
       pm set-install-location [0/auto] [1/internal] [2/external]
       pm get-install-location
       pm set-permission-enforced PERMISSION [true|false]
       pm trim-caches DESIRED_FREE_SPACE [internal|UUID]
       pm create-user [--profileOf USER_ID] [--managed] USER_NAME
       pm remove-user USER_ID
       pm get-max-users

## wm

    usage: wm [subcommand] [options]
       wm size [reset|WxH|WdpxHdp]
       wm density [reset|DENSITY]
       wm overscan [reset|LEFT,TOP,RIGHT,BOTTOM]
       wm scaling [off|auto]
       wm screen-capture [userId] [true|false]

## idmap

    > idmap --help
    NAME
      idmap - create or display idmap files
    
    SYNOPSIS
          idmap --help
          idmap --fd target overlay fd
          idmap --path target overlay idmap
          idmap --scan dir-to-scan target-to-look-for target dir-to-hold-idmaps
          idmap --inspect idmap
    
    DESCRIPTION
          Idmap files play an integral part in the runtime resource overlay framework. An idmap
          file contains a mapping of resource identifiers between overlay package and its target
          package; this mapping is used during resource lookup. Idmap files also act as control
          files by their existence: if not present, the corresponding overlay package is ignored
          when the resource context is created.
    
          Idmap files are stored in /data/resource-cache. For each pair (target package, overlay
          package), there exists exactly one idmap file, or none if the overlay should not be used.
    
    NOMENCLATURE
          target: the original, non-overlay, package. Each target package may be associated with
                  any number of overlay packages.
    
          overlay: an overlay package. Each overlay package is associated with exactly one target
                   package, specified in the overlay's manifest using the <overlay target="..."/>
                   tag.
    
    OPTIONS
          --help: display this help
    
          --fd: create idmap for target package 'target' (path to apk) and overlay package 'overlay'
                (path to apk); write results to file descriptor 'fd' (integer). This invocation
                version is intended to be used by a parent process with higher privileges to call
                idmap in a controlled way: the parent will open a suitable file descriptor, fork,
                drop its privileges and exec. This tool will continue execution without the extra
                privileges, but still have write access to a file it could not have opened on its
                own.
    
          --path: create idmap for target package 'target' (path to apk) and overlay package
                  'overlay' (path to apk); write results to 'idmap' (path).
    
          --scan: non-recursively search directory 'dir-to-scan' (path) for overlay packages with
                  target package 'target-to-look-for' (package name) present at 'target' (path to
                  apk). For each overlay package found, create an idmap file in 'dir-to-hold-idmaps'
                  (path).
    
          --inspect: decode the binary format of 'idmap' (path) and display the contents in a
                     debug-friendly format.
    
    EXAMPLES
          Create an idmap file:
    
          $ adb shell idmap --path /system/app/target.apk \
                                   /vendor/overlay/overlay.apk \
                                   /data/resource-cache/vendor@overlay@overlay.apk@idmap
    
          Display an idmap file:
    
          $ adb shell idmap --inspect /data/resource-cache/vendor@overlay@overlay.apk@idmap
          SECTION      ENTRY         VALUE      COMMENT
          IDMAP HEADER magic         0x706d6469
                       base crc      0xb65a383f
                       overlay crc   0x7b9675e8
                       base mtime    0x1eb47d51
                       overlay mtime 0x185f87a2
                       base path     .......... /path/to/target.apk
                       overlay path  .......... /path/to/overlay.apk
          DATA HEADER  target pkg    0x0000007f
                       types count   0x00000003
          DATA BLOCK   target type   0x00000002
                       overlay type  0x00000002
                       entry count   0x00000001
                       entry offset  0x00000000
                       entry         0x00000000 drawable/drawable
          DATA BLOCK   target type   0x00000003
                       overlay type  0x00000003
                       entry count   0x00000001
                       entry offset  0x00000000
                       entry         0x00000000 xml/integer
          DATA BLOCK   target type   0x00000004
                       overlay type  0x00000004
                       entry count   0x00000001
                       entry offset  0x00000000
                       entry         0x00000000 raw/lorem_ipsum
    
          In this example, the overlay package provides three alternative resource values:
          drawable/drawable, xml/integer, and raw/lorem_ipsum
    
    NOTES
          This tool and its expected invocation from installd is modelled on dexopt.

## logcat （system/core/logcat）

    Usage: logcat [options] [filterspecs]
    options include:
      -s              Set default filter to silent.
                      Like specifying filterspec '*:S'
      -f <filename>   Log to file. Default is stdout
      -r <kbytes>     Rotate log every kbytes. Requires -f
      -n <count>      Sets max number of rotated logs to <count>, default 4
      -v <format>     Sets the log print format, where <format> is:
    
                          brief color long printable process raw tag thread
                          threadtime time usec
    
      -D              print dividers between each log buffer
      -c              clear (flush) the entire log and exit
      -d              dump the log and then exit (don't block)
      -t <count>      print only the most recent <count> lines (implies -d)
      -t '<time>'     print most recent lines since specified time (implies -d)
      -T <count>      print only the most recent <count> lines (does not imply -d
      -T '<time>'     print most recent lines since specified time (not imply -d)
                      count is pure numerical, time is 'MM-DD hh:mm:ss.mmm'
      -g              get the size of the log's ring buffer and exit
      -L              dump logs from prior to last reboot
      -b <buffer>     Request alternate ring buffer, 'main', 'system', 'radio',
                      'events', 'crash' or 'all'. Multiple -b parameters are
                      allowed and results are interleaved. The default is
                      -b main -b system -b crash.
      -B              output the log in binary.
      -S              output statistics.
      -G <size>       set size of log ring buffer, may suffix with K or M.
      -p              print prune white and ~black list. Service is specified as
                      UID, UID/PID or /PID. Weighed for quicker pruning if prefix
                      with ~, otherwise weighed for longevity if unadorned. All
                      other pruning activity is oldest first. Special case ~!
                      represents an automatic quicker pruning for the noisiest
                      UID as determined by the current statistics.
      -P '<list> ...' set prune white and ~black list, using same format as
                      printed above. Must be quoted.
    
    filterspecs are a series of
      <tag>[:priority]

常用用法：

1. logcat -c ： 清空log
2. logcat -s *:E
3. logcat -v 
4. logcat -t/-T <count>/'<time>'
5. logcat -b main/system/radio/events/crash/all

## 参考

1. http://blog.csdn.net/wlqingwei/article/details/43970299 （dumpsys原理）
2. http://gityuan.com/2016/02/27/am-command/
3. http://gityuan.com/2016/02/28/pm-command/