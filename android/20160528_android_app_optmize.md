## 调试

1. adb shell dumpsys meminfo <package-name>

    查看 `Activities Views` 是否为零
    多次进入退出后的占用内存`TOTAL`不应变化太大

2. adb shell dumpsys gfxinfo <package-name> -cmd trim 5  触发onTrimMemory回调，然后再按上一条查看内存

3. Traceview:

    DDMS中选择一个进程然后 “Start Method Profiling”

4. systrace:

        `python systrace.py --time=10 -o mynewtrace.html sched gfx view wm`

5. adb shell dumpsys batterystats 查看电量消耗

6. 开启开发者选项中的：
    
    调试GPU过度绘制
    GPU呈现模式分析：Draw、Process、Execute

## 开发环境

1. Android Studio开启：
    
    Analyze->Inspect Code做静态扫描，执行Android Lint

2. 因为依赖要从Jcenter/Maven 仓库上下载，但是网络还是时不时地抽风，这时可以在gradle.properties 配置代理，如：

        // 举例ShadowSocket
        systemProp.http.proxyHost=127.0.0.1
        systemProp.http.proxyPort=1080
        systemProp.https.proxyHost=127.0.0.1
        systemProp.https.proxyPort=1080

3. 设置Gradle离线状态：在Android Studio-> Preference -> Gradle -> Use local gradle distribution，然后选择Gradle 的目录即可，这样就不会跟着项目去下载gradle了。

4. 开启守护进程，并行编译

    设置Gradle Command Option：--parallel --offline
    在gradle.properties中加入：

        org.gradle.daemon=true
        org.gradle.parallel=true
        org.gradle.jvmargs=-Xmx2048m
        -XX:MaxPermSize=512m

5. 当API 不小于21，使用 ART，在 Build 时只做 class to dex，不做 mergeing dex，会省下大量的时间，所以开发时可以使用SDK>=21:

        productFlavors {
            dev {
                minSdkVersion 21
            }
    
            release {
                minSdkVersion 15
            }
        }



## 代码

1. 代码Application里开启StricMode调试模式：

        public void onCreate() {
             if (DEVELOPER_MODE) {
                StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder().detectAll().penaltyLog() .build());
                StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder().detectAll().penaltyLog().build());
             }
             super.onCreate();
        }

    如果主线程有网络或磁盘读写等操作，在logcat中会有"D/StrictMode"tag的日志输出，从而定位到耗时操作的代码。 

2. 在方法上添加@DebugLog来打印方法的耗时：

    添加依赖：classpath 'com.jakewharton.hugo:hugo-plugin:1.2.1'
    应用插件解析下面的注解：apply plugin: 'com.jakewharton.hugo'
    方法上加入注解：

        @DebugLog
        public void test( int a ){
            int b=a*a;
        }

3. 内存泄漏检测：leakcanary
    
    添加依赖：

        debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3.1' // or 1.4-beta1
        releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3.1' // or 1.4-beta1
        testCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3.1' // or 1.4-beta1

    在Application.onCreate中初始化：LeakCanary.install(this);

## 热修复

1. recoo & recoo gradle plugin
2. apply plugin: com.dodola.rocoofix.RocooFixPlugin
3. 在Application中的onCreate中：RocooFix.initPathFromAssets(this, "patch.jar");
4. proguard中：-keep class com.dodola.rocoofix.** {*;}
5. 在build.gradle中：

        rocoo_fix {
            //includePackageNames = ['com/dodola/rocoo.dex/HelloHack.class']
            //preVersionPath = '1'
            enable = true
        }

## 日志、用户行为

1. 日志： bugly

2. 用户行为：

## Clean-Arch架构模型



## 参考

1. http://blog.tingyun.com/web/article/detail/155
2. http://blog.chengyunfeng.com/?p=458
3. https://developer.android.com/studio/profile/optimize-ui.html

    


        