## Android-CleanArchitecture

架构实现参考实例：https://github.com/android10/Android-CleanArchitecture

参考说明：http://www.csdn.net/article/2015-08-20/2825506

![](./images/clean_arch_depen.png)

> Architectural reactive approach

![](./images/clean_arch_reactive.png)

Subscriber & Observable通过响应式模式框架：RxJava实现；

通过Dagger2依赖注入可以将对象的实例存在于在一个隔离和解耦地方，这些对象在外部注入和配置；

> 各分层测试方法：

1. 表现层：用Espresso 2和Android Instrumentation测试框架测试UI。
2. 领域层：JUnit + Mockito —— 它是Java的标准模块。
3. 数据层：将测试组合换成了Robolectric 3 + JUnit + Mockito。这一层的测试曾经存在于单独的Android模块。由于当时（当前示例程序的第一个版本）没有内置单元测试的支持，也没有建立像robolectric那样的框架，该框架比较复杂，需要一群黑客的帮忙才能让其正常工作。

OkHttp/Retrofit： http://blog.myitlove.com/androidxi-lie-011-httplibsxuan-ze/

Rxjava/RxAndroid： 

Dagger2： 

## repository pattern（资源库）模式

仓库模式: https://msdn.microsoft.com/en-us/library/ff649690.aspx

![](./images/IC340233.png)

协调领域和数据映射层，利用类似于集合的接口来访问领域对象；

Repository是一个独立的层，介于领域层与数据映射层（数据访问层）之间。它的存在让领域层感觉不到数据访问层的存在，它提供一个类似集合的接口提供给领域层进行领域对象的访问。Repository是仓库管理员，领域层需要什么东西只需告诉仓库管理员，由仓库管理员把东西拿给它，并不需要知道东西实际放在哪。

参考文档：http://www.cnblogs.com/dudu/archive/2011/05/25/repository_pattern.html


## Expresso

官网：https://google.github.io/android-testing-support-library/docs/espresso/

## Mockito

https://github.com/mockito/mockito


