
https://console.firebase.google.com/

1. 新建项目 （容器集）
2. 功能：Notifications（消息推送）、Remote Confit、Analytics、Hosting、Database、Auth、Storage、Dynamic Links、Test Lab、Crash Reporting、AdMob；


## Notifications

是否和GCM一样？国内网络不通？不符合国内运营商特性？

1. 创建Android应用；
2. 会下载一个google-service.json;
3. 将google-service.json放到项目文件中；

![](./images/firebase_step2.png)

4. 配置google-services依赖；

![](./images/firebase_step3.png)

5. 如果网络问题会报错：Error:Failed to resolve:com.google.firebase:firebase-core:9.0.0，解决办法见[这里](http://stackoverflow.com/questions/37310188/failed-to-resolve-com-google-firebasefirebase-core9-0-0/37310513#37310513),升级Google Play Services (rev 30) and Google Repository (rev 26)