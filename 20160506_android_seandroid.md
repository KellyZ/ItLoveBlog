## 说明

![](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/seandroid_framework.png)

以SELinux文件系统接口为边界，SEAndroid安全机制包含有内核空间和用户空间两部分支持。在内核空间，主要涉及到一个称为SELinux LSM的模块。而在用户空间中，涉及到Security Context、Security Server和SEAndroid Policy等模块。

Security Server负责安全访问控制逻辑，即由它来决定一个主体访问一个客体是否是合法的。这里说的主体一般就是指进程，而客体就是主体要访问的资源，例如文件。


## 安全上下文

SEAndroid是一种基于安全策略的MAC安全机制。这种安全策略又是建立在对象的安全上下文的基础上的。

这里所说的**对象**分为两种类型，一种称主体（Subject），一种称为客体（Object）。主体通常就是指进程，而客观就是指进程所要访问的资源，例如文件、系统属性等。

**安全上下文**实际上就是一个附加在对象上的标签（Tag）。这个标签实际上就是一个字符串，它由四部分内容组成，分别是SELinux用户、SELinux角色、类型、安全级别，每一个部分都通过一个冒号来分隔，格式为“user:role:type:sensitivity”。

可以通过ls -Z和ps -Z就可以查看文件和进程对象的安全上下文字符串了。

在安全上下文中，只有类型（Type）才是最重要的，SELinux用户、SELinux角色和安全级别都几乎可以忽略不计的。正因为如此，SEAndroid安全机制又称为是基于TE（Type Enforcement）策略的安全机制。

对于进程来说，SELinux用户和SELinux角色只是用来限制进程可以标注的类型。而对于文件来说，SELinux用户和SELinux角色就可以完全忽略不计。为了完整地描述一个文件的安全上下文，通常将它的SELinux角色固定为object_r，而将它的SELinux用户设置为创建它的进程的SELinux用户。

### 用户与角色

在SEAndroid中，只定义了一个SELinux用户u，因此我们通过ps -Z和ls -Z命令看到的所有的进程和文件的安全上下文中的SELinux用户都为u。同时，SEAndroid也只定义了一个SELinux角色r，因此，我们通过ps -Z命令看到的所有进程的安全上下文中的SELinux角色都为r。

通过external/sepolicy/users：
   
    user u roles { r } level s0 range s0 - mls_systemhigh;
    
和external/sepolicy/roles：

    role r;
    role r types domain;

### 类型

上述两个文件定义了u、r和domain可以组合在一起形成一个合法的安全上下文，那其他形式的安全上下文组合是如何定义的呢？在另外一个文件external/sepolicy/init.te中，通过type语句声明其他类型：

    type init, domain;

在SEAndroid中，我们通常将用来标注文件的安全上下文中的类型称为file_type，而用来标注进程的安全上下文的类型称为domain。
并且每一个用来描述文件安全上下文的类型都将file_type设置为其属性，每一个用来进程安全上下文的类型都将domain设置为其属性。

文件external/sepolicy/file.te，可以看到App数据文件的类型声明：

    type app_data_file, file_type, data_file_type;
    
上述语句表明类型app_data_file具有属性file_type，即它是用来描述文件的安全上下文的。

### 安全级别

在实际使用中，安全级别是由敏感性（Sensitivity）和类别（Category）两部分内容组成的，格式为“sensitivity[:category_set]”，其中，category_set是可选的。例如，假设我们定义有s0、s1两个Sensitivity，以c0、c1、c2三个Category，那么“s0:c0,c1”表示的就是Sensitivity为s0、Category为c0和c1的一个安全级别。

### 上下文描述

主要来看App进程、App数据文件、系统文件和系统属性的安全上下文。这四种类型对象的安全上下文通过四个文件来描述：mac_permissions.xml、seapp_contexts、file_contexts和property_contexts，它们均位于external/sepolicy目录中。

### App进程

external/sepolicy/mac_permissions.xml :

    <?xml version="1.0" encoding="utf-8"?>
    <policy>
    
        <!-- Platform dev key in AOSP -->
        <signer signature="@PLATFORM" >
          <seinfo value="platform" />
        </signer>
    
        <!-- All other keys -->
        <default>
          <seinfo value="default" />
        </default>
    
    </policy>
    
这个seinfo描述的其实并不是安全上下文中的Type，它是用来在另外一个文件external/sepolicy/seapp_contexts中查找对应的Type的。

    isSystemServer=true domain=system_server
    user=system domain=system_app type=system_app_data_file
    user=bluetooth domain=bluetooth type=bluetooth_data_file
    user=nfc domain=nfc type=nfc_data_file
    user=radio domain=radio type=radio_data_file
    user=shared_relro domain=shared_relro
    user=shell domain=shell type=shell_data_file
    user=_isolated domain=isolated_app
    user=_app seinfo=platform domain=platform_app type=app_data_file
    user=_app domain=untrusted_app type=app_data_file
    
比如平台签名的APP匹配的就是：user=_app seinfo=platform domain=platform_app type=app_data_file，因此平台签名的App所运行在的进程domain为“platform_app”，并且它的数据文件的file_type为“platform_app_data_file”。

第三方签名的App的seinfo为“default”。用户空间的Security Server在为它查找对应的Type时，使用的user输入也为"_app"。但在seapp_contexts文件中，没有一行对应的user和seinfo分别为“_app”和“default”。但是有一行是最匹配的，即user=_app domain=untrusted_app type=app_data_file。

### 系统文件

系统文件的安全上下文定义在：external/sepolicy/file_contexts

    ###########################################
    # Root
    /                       u:object_r:rootfs:s0
    
    # Data files
    /adb_keys               u:object_r:adb_keys_file:s0
    /default\.prop          u:object_r:rootfs:s0
    /fstab\..*              u:object_r:rootfs:s0
    /init\..*               u:object_r:rootfs:s0
    /res(/.*)?              u:object_r:rootfs:s0
    /ueventd\..*            u:object_r:rootfs:s0
    
    # Executables
    /charger                u:object_r:rootfs:s0
    /init                   u:object_r:rootfs:s0
    /sbin(/.*)?             u:object_r:rootfs:s0
    
    # Empty directories
    /lost\+found            u:object_r:rootfs:s0
    /proc                   u:object_r:rootfs:s0
    
    # SELinux policy files
    /file_contexts          u:object_r:rootfs:s0
    /property_contexts      u:object_r:rootfs:s0
    /seapp_contexts         u:object_r:rootfs:s0
    /sepolicy               u:object_r:rootfs:s0
    ......
    #############################
    # System files
    #
    /system(/.*)?           u:object_r:system_file:s0
    /system/bin/sh          --      u:object_r:shell_exec:s0
    /system/bin/run-as      --      u:object_r:runas_exec:s0
    /system/bin/bootanimation u:object_r:bootanim_exec:s0
    /system/bin/app_process32       u:object_r:zygote_exec:s0
    /system/bin/app_process64       u:object_r:zygote_exec:s0
    /system/bin/servicemanager      u:object_r:servicemanager_exec:s0
    /system/bin/surfaceflinger      u:object_r:surfaceflinger_exec:s0
    /system/bin/drmserver   u:object_r:drmserver_exec:s0
    /system/bin/dumpstate   u:object_r:dumpstate_exec:s0
    /system/bin/vold        u:object_r:vold_exec:s0
    /system/bin/netd        u:object_r:netd_exec:s0
    /system/bin/rild        u:object_r:rild_exec:s0
    /system/bin/mediaserver u:object_r:mediaserver_exec:s0
    /system/bin/mdnsd       u:object_r:mdnsd_exec:s0
    /system/bin/installd    u:object_r:installd_exec:s0
    /system/bin/keystore    u:object_r:keystore_exec:s0
    /system/bin/debuggerd   u:object_r:debuggerd_exec:s0
    /system/bin/debuggerd64 u:object_r:debuggerd_exec:s0
    /system/bin/wpa_supplicant      u:object_r:wpa_exec:s0
    /system/bin/sdcard      u:object_r:sdcardd_exec:s0
    /system/bin/dhcpcd      u:object_r:dhcp_exec:s0
    /system/bin/mtpd        u:object_r:mtp_exec:s0
    /system/bin/pppd        u:object_r:ppp_exec:s0
    /system/bin/tf_daemon   u:object_r:tee_exec:s0
    /system/bin/racoon      u:object_r:racoon_exec:s0
    /system/xbin/su         u:object_r:su_exec:s0
    /system/vendor/bin/gpsd u:object_r:gpsd_exec:s0
    /system/bin/dnsmasq     u:object_r:dnsmasq_exec:s0
    /system/bin/hostapd     u:object_r:hostapd_exec:s0
    /system/bin/clatd       u:object_r:clatd_exec:s0
    /system/bin/lmkd        u:object_r:lmkd_exec:s0
    /system/bin/inputflinger u:object_r:inputflinger_exec:s0
    /system/bin/logd        u:object_r:logd_exec:s0
    /system/bin/uncrypt     u:object_r:uncrypt_exec:s0
    /system/bin/logwrapper  u:object_r:system_file:s0
    /system/bin/vdc         u:object_r:vdc_exec:s0
    /system/bin/install-recovery.sh u:object_r:install_recovery_exec:s0
    /system/bin/dex2oat     u:object_r:dex2oat_exec:s0
    ......
    
### 系统属性

系统属性上下文定义在external/sepolicy/property_contexts：

    ##########################
    # property service keys
    #
    #
    net.rmnet               u:object_r:net_radio_prop:s0
    net.gprs                u:object_r:net_radio_prop:s0
    net.ppp                 u:object_r:net_radio_prop:s0
    net.qmi                 u:object_r:net_radio_prop:s0
    net.lte                 u:object_r:net_radio_prop:s0
    net.cdma                u:object_r:net_radio_prop:s0
    net.dns                 u:object_r:net_radio_prop:s0
    sys.usb.config          u:object_r:system_radio_prop:s0
    ril.                    u:object_r:radio_prop:s0
    gsm.                    u:object_r:radio_prop:s0
    persist.radio           u:object_r:radio_prop:s0
    ......
    
## 安全策略

SEAndroid安全机制中的安全策略是在安全上下文的基础上进行描述的，也就是说，它通过主体和客体的安全上下文的匹配关系，定义主体是否有权限访问客体。

SEAndroid安全机制主要是使用对象安全上下文中的类型来定义安全策略，这种安全策略就称Type Enforcement，简称TE。在external/sepolicy目录中，所有以.te为后缀的文件经过编译之后，就会生成一个sepolicy文件。这个sepolicy文件会打包在ROM中，并且保存在设备上的根目录下，即它在设备上的路径为/sepolicy。

    > ls external/sepolicy/*.te
    external/sepolicy/adbd.te                 external/sepolicy/inputflinger.te      external/sepolicy/runas.te
    external/sepolicy/app.te                  external/sepolicy/installd.te          external/sepolicy/sdcardd.te
    external/sepolicy/binderservicedomain.te  external/sepolicy/install_recovery.te  external/sepolicy/servicemanager.te
    external/sepolicy/bluetooth.te            external/sepolicy/isolated_app.te      external/sepolicy/service.te
    external/sepolicy/bootanim.te             external/sepolicy/kernel.te            external/sepolicy/shared_relro.te
    external/sepolicy/clatd.te                external/sepolicy/keystore.te          external/sepolicy/shell.te
    external/sepolicy/debuggerd.te            external/sepolicy/lmkd.te              external/sepolicy/surfaceflinger.te
    external/sepolicy/device.te               external/sepolicy/logd.te              external/sepolicy/su.te
    external/sepolicy/dex2oat.te              external/sepolicy/mdnsd.te             external/sepolicy/system_app.te
    external/sepolicy/dhcp.te                 external/sepolicy/mediaserver.te       external/sepolicy/system_server.te
    external/sepolicy/dnsmasq.te              external/sepolicy/mtp.te               external/sepolicy/tee.te
    external/sepolicy/domain.te               external/sepolicy/netd.te              external/sepolicy/ueventd.te
    external/sepolicy/drmserver.te            external/sepolicy/net.te               external/sepolicy/unconfined.te
    external/sepolicy/dumpstate.te            external/sepolicy/nfc.te               external/sepolicy/uncrypt.te
    external/sepolicy/file.te                 external/sepolicy/platform_app.te      external/sepolicy/untrusted_app.te
    external/sepolicy/gpsd.te                 external/sepolicy/ppp.te               external/sepolicy/vdc.te
    external/sepolicy/hci_attach.te           external/sepolicy/property.te          external/sepolicy/vold.te
    external/sepolicy/healthd.te              external/sepolicy/racoon.te            external/sepolicy/watchdogd.te
    external/sepolicy/hostapd.te              external/sepolicy/radio.te             external/sepolicy/wpa.te
    external/sepolicy/init_shell.te           external/sepolicy/recovery.te          external/sepolicy/zygote.te
    external/sepolicy/init.te                 external/sepolicy/rild.te

比如platform_app.te文件就定义了平台签名APP的安全策略：

    ###
    ### Apps signed with the platform key.
    ###
    
    type platform_app, domain;
    app_domain(platform_app)
    # Access the network.
    net_domain(platform_app)
    # Access bluetooth.
    bluetooth_domain(platform_app)
    # Read from /data/local/tmp or /data/data/com.android.shell.
    allow platform_app shell_data_file:dir search;
    allow platform_app shell_data_file:file { open getattr read };
    # Populate /data/app/vmdl*.tmp, /data/app-private/vmdl*.tmp files
    # created by system server.
    allow platform_app { apk_tmp_file apk_private_tmp_file }:dir rw_dir_perms;
    allow platform_app { apk_tmp_file apk_private_tmp_file }:file rw_file_perms;
    allow platform_app apk_private_data_file:dir search;
    # ASEC
    allow platform_app asec_apk_file:dir create_dir_perms;
    allow platform_app asec_apk_file:file create_file_perms;
    
    # Access to /data/media.
    allow platform_app media_rw_data_file:dir create_dir_perms;
    allow platform_app media_rw_data_file:file create_file_perms;
    
    # Write to /cache.
    allow platform_app cache_file:dir create_dir_perms;
    allow platform_app cache_file:file create_file_perms;

1. platform_app接下来会通过app_domain、net_domain、bluetooth_domain宏分别加入到其它的domain中去，以便可以获得相应的权限，宏定义在external/sepolicy/te_macros文件中。
2. SEAndroid使用的是最小权限原则，也就是说，只有通过allow语句声明的权限才是允许的，而其它没有通过allow语句声明的权限都是禁止，这样就可以最大限度地保护系统中的资源

## SEAndroid启动加载

SEAndroid安全机制的安全策略经过编译后会得到一个sepolicy文件，并且最终保存在设备上的根据目录下。注意，如果我们什么也不做，那么保存在这个sepolicy文件中的安全策略是不会自动加载到内核空间的SELinux LSM模块去的。它需要我们在系统启动的过程中进行加载。

系统中第一个启动的进程是init进程。我们知道，Init进程在启动的过程中，执行了很多的系统初始化工作，其中就包括加载SEAndroid安全策略的工作，如下所示system/core/init/init.c：

    int main(int argc, char **argv)
    {
        ......
        union selinux_callback cb;
        cb.func_log = log_callback;
        selinux_set_callback(SELINUX_CB_LOG, cb);
    
        cb.func_audit = audit_callback;
        selinux_set_callback(SELINUX_CB_AUDIT, cb);
    
        selinux_initialize();
        ......
    }
    
    static void selinux_initialize(void)
    {
        if (selinux_is_disabled()) {
            return;
        }
    
        INFO("loading selinux policy\n");
        if (selinux_android_load_policy() < 0) {
            ERROR("SELinux: Failed to load policy; rebooting into recovery mode\n");
            android_reboot(ANDROID_RB_RESTART2, 0, "recovery");
            while (1) { pause(); }  // never reached
        }
    
        selinux_init_all_handles();
    
        bool is_enforcing = selinux_is_enforcing();
    
        INFO("SELinux: security_setenforce(%d)\n", is_enforcing);
        security_setenforce(is_enforcing);
    }
    
    void selinux_init_all_handles(void)
    {
        sehandle = selinux_android_file_context_handle();
        selinux_android_set_sehandle(sehandle);
        sehandle_prop = selinux_android_prop_context_handle();
    }
    
通过调用libselinux的函数来打开前面分析file_contexts和property_contexts文件，以便可以用来查询系统文件和系统属性的安全上下文。

函数selinux_android_load_policy则是用来加载安全策略到内核空间的SELinux LSM模块中去，函数定义在文件external/libselinux/src/android.c中：

    static const char *const sepolicy_file[] = {
        "/sepolicy",
        "/data/security/current/sepolicy",
        NULL };

    int selinux_android_load_policy(void)
    {
            const char *mnt = SELINUXMNT;
            int rc;
            rc = mount(SELINUXFS, mnt, SELINUXFS, 0, NULL);
            if (rc < 0) {
                    if (errno == ENODEV) {
                            /* SELinux not enabled in kernel */
                            return -1;
                    }
                    if (errno == ENOENT) {
                            /* Fall back to legacy mountpoint. */
                            mnt = OLDSELINUXMNT;
                            rc = mkdir(mnt, 0755);
                            if (rc == -1 && errno != EEXIST) {
                                    selinux_log(SELINUX_ERROR,"SELinux:  Could not mkdir:  %s\n",
                                            strerror(errno));
                                    return -1;
                            }
                            rc = mount(SELINUXFS, mnt, SELINUXFS, 0, NULL);
                    }
            }
            if (rc < 0) {
                    selinux_log(SELINUX_ERROR,"SELinux:  Could not mount selinuxfs:  %s\n",
                                    strerror(errno));
                    return -1;
            }
            set_selinuxmnt(mnt);
    
        return selinux_android_load_policy_helper(false);
    }

    static int selinux_android_load_policy_helper(bool reload)
    {
            int fd = -1, rc;
            struct stat sb;
            void *map = NULL;
    
            /*
             * If reloading policy and there is no /data policy or
             * that /data policy has the wrong version or the /data
             * policy is disabled via safe mode, then just return.
             * There is no point in reloading policy from / a second time.
             */
            if (reload && !selinux_android_use_data_policy())
                    return 0;
    
            fd = open(sepolicy_file[policy_index], O_RDONLY | O_NOFOLLOW);
            if (fd < 0) {
                    selinux_log(SELINUX_ERROR, "SELinux:  Could not open sepolicy:  %s\n",
                                    strerror(errno));
                    return -1;
            }
            if (fstat(fd, &sb) < 0) {
                    selinux_log(SELINUX_ERROR, "SELinux:  Could not stat %s:  %s\n",
                                    sepolicy_file[policy_index], strerror(errno));
                    close(fd);
                    return -1;
            }
            map = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
            if (map == MAP_FAILED) {
                    selinux_log(SELINUX_ERROR, "SELinux:  Could not map %s:  %s\n",
                            sepolicy_file[policy_index], strerror(errno));
                    close(fd);
                    return -1;
            }
    
            rc = security_load_policy(map, sb.st_size);
            if (rc < 0) {
                    selinux_log(SELINUX_ERROR, "SELinux:  Could not load policy:  %s\n",
                            strerror(errno));
                    munmap(map, sb.st_size);
                    close(fd);
                    return -1;
            }
    
            munmap(map, sb.st_size);
            close(fd);
            selinux_log(SELINUX_INFO, "SELinux: Loaded policy from %s\n", sepolicy_file[policy_index]);
    
            return 0;
    }
    
1. 以/sys/fs/selinux为安装点，安装一个类型为selinuxfs的文件系统，也就是SELinux文件系统，用来与内核空间的SELinux LSM模块通信。
2. 如果不能在/sys/fs/selinux这个安装点安装SELinux文件系统，那么再以/selinux为安装点，安装SELinux文件系统。
3. 成功安装SELinux文件系统之后，接下来就调用另外一个函数selinux_android_load_policy_helper来将SEAndroid安全策略加载到内核空间的SELinux LSM模块中去。
4. 最后调用security_load_policy将已经通过mmap映射到内存的sepolicy文件加载到内核空间SELinux LSM模块中去。

security_load_policy定义在external/libselinux/src/load_policy.c：

    int security_load_policy(void *data, size_t len)
    {
            char path[PATH_MAX];
            int fd, ret;
    
            if (!selinux_mnt) {
                    errno = ENOENT;
                    return -1;
            }
    
            snprintf(path, sizeof path, "%s/load", selinux_mnt);
            fd = open(path, O_RDWR);
            if (fd < 0)
                    return -1;
    
            ret = write(fd, data, len);
            close(fd);
            if (ret < 0)
                    return -1;
            return 0;
    }
    
selinux_mnt是一个全局变量，它描述的是SELinux文件系统的安装点。在我们这个情景中，它的值就等于/sys/fs/selinux。

函数security_load_policy的实现很简单，它首先打开/sys/fs/selinux/load文件，然后将参数data所描述的安全策略写入到这个文件中去。由于/sys/fs/selinux是由内核空间的SELinux LSM模块导出来的文件系统接口，因此当我们将安全策略写入到位于该文件系统中的load文件时，就相当于是将安全策略从用户空间加载到SELinux LSM模块中去了。以后SELinux LSM模块中的Security Server就可以通过它来进行安全检查

## Security Server

用户空间的Security Server主要是用来保护用户空间资源的，以及用来操作内核空间对象的安全上下文的，它由应用程序安装服务PackageManagerService、应用程序安装守护进程installd、应用程序进程孵化器Zygote进程以及init进程组成。其中，PackageManagerService和installd负责创建App数据目录的安全上下文，Zygote进程负责创建App进程的安全上下文，而init进程负责控制系统属性的安全访问。

应用程序安装服务PackageManagerService在启动的时候，会在/etc/security目录中找到我们前面分析的mac_permissions.xml文件，然后对它进行解析，得到App签名或者包名与seinfo的对应关系。当PackageManagerService安装App的时候，它就会根据其签名或者包名查找到对应的seinfo，并且将这个seinfo传递给另外一个守护进程installed。

守护进程installd负责创建App数据目录。在创建App数据目录的时候，需要给它设置安全上下文，使得SEAndroid安全机制可以对它进行安全访问控制。Installd根据PackageManagerService传递过来的seinfo，并且调用libselinux库提供的selabel_lookup函数到前面我们分析的seapp_contexts文件中查找到对应的Type。有了这个Type之后，installd就可以给正在安装的App的数据目录设置安全上下文了，这是通过调用libselinux库提供的lsetfilecon函数来实现的。

在Android系统中，Zygote进程负责创建应用程序进程。应用程序进程是SEAndroid安全机制中的主体，因此它们也需要设置安全上下文，这是由Zygote进程来设置的。组件管理服务ActivityManagerService在请求Zygote进程创建应用程序进程之前，会到PackageManagerService中去查询对应的seinfo，并且将这个seinfo传递到Zygote进程。于是，Zygote进程在fork一个应用程序进程之后，就会使用ActivityManagerService传递过来的seinfo，并且调用libselinux库提供的selabel_lookup函数到前面我们分析的seapp_contexts文件中查找到对应的Domain。有了这个Domain之后，Zygote进程就可以给刚才创建的应用程序进程设置安全上下文了，这是通过调用libselinux库提供的lsetcon函数来实现的。

前面提到，在Android系统中，属性也是一项需要保护的资源。Init进程在启动的时候，会创建一块内存区域来维护系统中的属性，接着还会创建一个Property服务。这个Property服务通过socket提供接口给其它进程访问Android系统中的属性。其它进程通过socket来和Property服务通信时，Property服务可以获得它的安全上下文。有了这个安全上下文之后，Property服务就可以通过libselinux库提供的selabel_lookup函数到前面我们分析的property_contexts去查找要访问的属性的安全上下文了。有了这两个安全上下文之后，Property服务就可以决定是否允许一个进程访问它所指定的属性了。


## 参考

1. http://blog.csdn.net/luoshengyang/article/details/37613135 （SEAndroid机制框架）
2. http://blog.csdn.net/luoshengyang/article/details/37749383 （对文件的保护）
3. http://blog.csdn.net/luoshengyang/article/details/38054645 （对进程的保护）
4. http://blog.csdn.net/luoshengyang/article/details/38102011 （对属性的保护）
5. http://blog.csdn.net/luoshengyang/article/details/38326729 （对Binder IPC的保护）