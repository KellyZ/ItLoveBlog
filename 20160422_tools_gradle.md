

    android {
        compileSdkVersion ANDROID_BUILD_TARGET_SDK_VERSION as int
        buildToolsVersion ANDROID_BUILD_TOOLS_VERSION
     
        defaultConfig {
        }
     
        buildTypes {
        }
     
        compileOptions {
        }
     
        sourceSets {
        }
     
        lintOptions {
        }
     
        productFlavors {
            flavor1 {
            }
     
            flavor2 {
            }
        }
        signingConfigs {
            release {
                storeFile file(×.keystore)
                storePassword ×××
                keyAlias ××××
                keyPassword ×××
            }
        }
    }
    
defaultConfig如下：

    defaultConfig {
        applicationId com.example.qiujuer.application
        minSdkVersion 15
        targetSdkVersion 21
        versionCode 1
        versionName 1.0
     
        ndk {
            moduleName genius
            cFlags -DANDROID_NDK -D_RELEASE
            ldLibs m, log, jnigraphics
            abiFilters all
        }
    }
    
buildTypes会继承defaultConfig

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.android_media
        }
        debug {
            signingConfig signingConfigs.android_media
        }
    }
    
sourceSets

以下是一些常用的设置，其中最后一个是引用 *.so 文件的时候使用的方法

    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            resources.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
            jniLibs.srcDirs = ['libs']
        }
    }
    
productFlavors：在这里你可以设置你的产品发布的一些东西，比如你现在一共软件需要发布到不同渠道，且不同渠道中的包名不同，那么可以在此进行配置；甚至可以设置不同的 AndroidManifest.xml 文件

    productFlavors {
        flavor1 {
            packageName='com.example.qiujuer.application1'
            manifest.srcFile 'exampleapk/AndroidManifest1.xml'
        }
     
        flavor2 {
            packageName='com.example.qiujuer.application2'
            manifest.srcFile 'exampleapk/AndroidManifest2.xml'
        }
    }
    
## 参考

1. http://www.2cto.com/kf/201501/366464.html
2. http://www.android-studio.org/index.php/docs/guide/135-gradle-2
3. http://tools.android.com/tech-docs/new-build-system/user-guide
4. http://google.github.io/android-gradle-dsl/2.0/