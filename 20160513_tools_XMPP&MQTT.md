## 消息推送简介

消息推送分两个层面讲：

1. 首先是连接，在客户端与服务端建立轻量级且可靠的连接：长连接+心跳（心跳时间间隔最为重要）
2. 然后是协议，在连接之上的数据协议，数据协议自然是要解决业务问题，对业务的分析与扩展很重要。

心跳包问题补充：

你说的不发心跳包也可以维持长链接，这是对的，因为tcp是有保活定时器的，默认是用保活定时器来维持长连接，但是为什么要发心跳包？因为保活定时器的周期是两小时。这属于一个补充点。

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

## 移动网络的特点

当一台智能手机连上移动网络时，其实并没有真正连接上Internet，运营商分配给手机的IP其实是运营商的内网IP，手机终端要连接上Internet还必须通过运营商的网关进行IP地址的转换，这个网关简称为NAT(NetWork Address Translation)，简单来说就是手机终端连接Internet 其实就是移动内网IP，端口，外网IP之间相互映射。相当于在手机终端在移动无线网络这堵墙上打个洞与外面的Internet相连。

由于大部分的移动无线网络运营商为了减少网关NAT映射表的负荷，如果一个链路有一段时间没有通信时就会删除其对应表，造成链路中断，正是这种刻意缩短空闲连接的释放超时，原本是想节省信道资源的作用，没想到让互联网的应用不得以远高于正常频率发送心跳来维护推送的长连接。这也是为什么会有之前的信令风暴，微信摇收费的传言，因为这类的应用发送心跳的频率是很短的，既造成了信道资源的浪费，也造成了手机电量的快速消耗。


## 参考

1. https://segmentfault.com/a/1190000000656509
2. http://www.coderyi.com/archives/434

3. http://blog.csdn.net/clh604/article/details/20167263
4. http://www.cnblogs.com/hanyonglu/archive/2012/03/04/2378971.html
5. http://www.jianshu.com/p/584707554ed7