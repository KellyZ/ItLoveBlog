## 生成R.java

    aapt p[ackage] [-d][-f][-m][-u][-v][-x][-z][-M AndroidManifest.xml] \
           [-0 extension [-0 extension ...]] [-g tolerance] [-j jarfile] \
           [--debug-mode] [--min-sdk-version VAL] [--target-sdk-version VAL] \
           [--app-version VAL] [--app-version-name TEXT] [--custom-package VAL] \
           [--rename-manifest-package PACKAGE] \
           [--rename-instrumentation-target-package PACKAGE] \
           [--utf16] [--auto-add-overlay] \
           [--max-res-version VAL] \
           [-I base-package [-I base-package ...]] \
           [-A asset-source-dir]  [-G class-list-file] [-P public-definitions-file] \
           [-S resource-sources [-S resource-sources ...]] \
           [-F apk-file] [-J R-file-dir] \
           [--product product1,product2,...] \
           [-c CONFIGS] [--preferred-density DENSITY] \
           [--split CONFIGS [--split CONFIGS]] \
           [--feature-of package [--feature-after package]] \
           [raw-files-dir [raw-files-dir] ...] \
           [--output-text-symbols DIR]
    
Package the android resources.  It will read assets and resources that are supplied with the -M -A -S or raw-files-dir arguments.  The -J -P -F and -R
options control which files are output.

示例：

    aapt package -f -m -J ./gen -S res -M AndroidManifest.xml -I D:\android_sdk\platforms\android-22\android.jar
    
    -f  force overwrite of existing files
    -m  make package directories under location specified by -J
    -J  specify where to output R.java resource constant definitions
    -S  directory in which to find resources.  Multiple directories will be scanned and the first match found (left to right) will take precedence.
    -M  specify full path to AndroidManifest.xml to include in zip
    -I  add an existing package to base include set


##  编译java文件和jar包

    javac -encoding GBK -target 1.5 -bootclasspath D:\android_sdk_for_studio\platforms\android-22\android.jar -d bin src\<package_name>\*.java gen\net\mobctrl\normal\apk\R.java -classpath libs\*.jar
    
## dx工具打包成classes.dex

    dx --dex --output=\<path-to-output>\classes.dex bin

    --dex: Convert a set of classfiles into a dex file
    
## 编译成资源文件

    aapt package -f -M AndroidManifest.xml -S res -I D:\android_sdk\platforms\android-22\android.jar -F bin\resources.ap_ --non-constant-id
    
    -F specify the apk file to output
    
    --non-constant-id
    Make the resources ID non constant. This is required to make an R java class that does not contain the final value but is used to make reusable compiled libraries that need to access resources.
    
## 使用sdklib.jar生成apk

    java -cp D:\android_sdk\tools\lib\sdklib.jar com.android.sdklib.build.ApkBuilderMain bin\MyCommond.apk -v -u -z bin\resources.ap_ -f bin\classes.dex -rf src
    
    resources.ap_和classes.dex都是前面生成的
    
## 使用jarsigner签名

    java –jar signapk.jar [-w] publickey.x509[.pem] privatekey.pk8 input.jar output.jar 

    -w 是指对ROM签名时需使用的参数
    input.jar 要签名的apk或者rom 
    output.jar 签名后生成的apk或者rom
    
    # 用keystore:
    jarsigner -verbose -keystore test.keystore -storepass 123456 -keypass 123456 -signedjar projectdemo-signed.apk test.apk test 

## 参考

1. http://blog.csdn.net/nupt123456789/article/details/50494278

