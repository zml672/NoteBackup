# UCMA开发-总览介绍 | 字痕随行

基本上所有的介绍都基于命名空间Microsoft.Rtc.Signaling，如果有其它的相关介绍，会追加更新。

## Microsoft.Rtc.Signaling

此命名空间内的类提供了连接到主机、调度会话、控制通道以便使一个EndPoint可以邀请另外一个EndPoint，它封装了低层次的会话发起协议(SIP)功能。下图显示了该命名空间的主要组成：

![](../../images/UCMA%E5%BC%80%E5%8F%91-%E6%80%BB%E8%A7%88%E4%BB%8B%E7%BB%8D/image-20230903163643331.png)

### Connection Manager

连接管理器的主要功能是管理传入和传出连接。一个类继承自抽象RealTimeConnectionManager类的实例，可以用来管理出站连接。一个RealTimeServerConnectionManager实现(RealTimeServerTcpConnectionManager或RealTimeServerTlsConnectionManager)用于管理传入的连接。

![](../../images/UCMA%E5%BC%80%E5%8F%91-%E6%80%BB%E8%A7%88%E4%BB%8B%E7%BB%8D/image-20230903163726492.png)

其中，RealTimeTcpServerConnectionManager是TCP连接，RealTimeTlsServerConnectionManager是TLS安全连接。

### EndPoint

EndPoint是在SIP网络中的路由实体。一个实时通信的应用程序创建一个实时的端点，使用户能够利用多种类型的多个设备实时地与其他用户进行通信，每个设备对应于该用户的唯一的EndPoint。例如，一个用户透过一个EndPoint从一个桌面客户端发送即时消息,同一用户在移动电话上应答呼叫是透过另一个EndPoint,这种情况被称为多点状态（MPOP）。

SipEndPoint类派生自RealTimeEndPoint，SipEndPoint必须在SIP服务器上注册，才能使用这种类型的其他端点进行通信。应用程序可以使用一个SipEndpoint实例来使用户能够发布或订阅数据的订阅会话，或使用SignalingSession来发送或接收邀请。

SipPeerToPeerEndPoint类派生自RealTimeEndPoint，可以不需要在服务器存在的情况下创建，但是存在一定的局限性，由于不和真实的Server创建连接，那么就无法监听来自服务器的接入和接出的连接，只能限于本身和其他机器之间的通信。

### SignalingSession

SignalingSession提供一个EndpPoint可以邀请另一个EndPoint参加一些活动或建立媒体沟通交流的控制通道。两个EndPoint可以使用已建立的SignalingSession彼此之间交换控制短消息以及文本信息，支持的消息类型为列举的消息类型枚举。

可参考的资料：

[http://www.cnblogs.com/vipyoumay/archive/2012/01/12/2320801.html](http://www.cnblogs.com/vipyoumay/archive/2012/01/12/2320801.html "Lync及UCMA介绍")

[http://msdn.microsoft.com/en-us/library/office/dn465974(v=office.15).aspx](http://msdn.microsoft.com/en-us/library/office/dn465974(v=office.15).aspx "UCMA 4.0 details")

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑