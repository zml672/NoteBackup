# UCMA开发-互动消息机器人的实现 | 字痕随行

引用原文自：[http://bbs.winos.cn/thread-73856-1-1.html](http://bbs.winos.cn/thread-73856-1-1.html "UCMA - 互动消息机器人的实现")

这次将的是开发OCS 互动消息机器人，通过TCP连接方式。互动机器人的重点是接受会话，所以终端需要去OCS服务器上注册过。使用的终端类为SipEndpoint。

准备参数：

* robot的sip地址 必须是在AD中存在的OC用户。
* OCS的FQDN

## 实现步骤

一些相同和上面相同的就不介绍了。

1. 创建项目，添加引用。
2. 创建RealTimeServerTcpConnectionManager。
3. 创建SipEndpoint。
```c#
SipEndpoint sipEndpoint = new SipEndpoint(_ownerUri
	, SipAuthenticationProtocols.Ntlm
	, SipTransportType.Tcp
	, _ocsFQDN
	, 5060
	, true
	, rtsTcpConnectionMgr
	, null
);
```
4. 注册SipEndpoint。
```c#
sipEndpoint.Register();
```
5. SipEndpoint 添加Session接收事件。
```c#
sipEndpoint.SessionReceived += sipEndpoint\_SessionReceived;
```
6. SessionReceived 事件的处理。
```c#
private void sipEndpoint\_SessionReceived(object sender, SessionReceivedEventArgs e)
{
   Console.WriteLine("a session received...");
   SignalingSession session = e.Session;
   session .OfferAnswerNegotiation = \_sipOfferAnswer;
   Console.WriteLine("Participate session");
   session .BeginParticipate(CompleteParticipate, session);
}
```
在这一步中，主要是接收其他终端发送过来的Invite请求，然后我们在接收到Invite后需要回复一条加入Session的请求，我使用异步方式加入。

7. Session participate的callback处理。
```c#
private void CompleteParticipate(IAsyncResult ar)
{
    SignalingSession session = ar.AsyncState as SignalingSession;
    try
    {
        session.EndParticipate(ar);
        session.MessageReceived += session\_MessageReceived;
    }
    catch (System.Exception e)
    {
        Console.WriteLine(e.ToString());
    }
}
```
在callback过程中，我们为session添加了MessageReceived事件。

8、MessageReceived 事件的处理。
```c#
private void session\_MessageReceived(object sender, MessageReceivedEventArgs e)
{
    SignalingSession session = sender as SignalingSession;
    if (e.MessageType== MessageType.Message)
    {
        Console.WriteLine(e.TextBody);
        session.SendMessage(MessageType.Message
            , new System.Net.Mime.ContentType("text/plain")
            , Encoding.UTF8.GetBytes("message received.")
        );
    }
}
```
在这里我们把收到的消息显示出来了，并回复了一条"message received"。

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑