# UCMA开发-自动回复机器人 | 字痕随行

本文将基于之前介绍的开发基础，来讲述如何使用UCMA创建一个可以自动回复的机器人程序。

## 第一步

创建ConnectionManager，这是通讯的基础。参考《UCMA开发之ConnectionManager》一文中所介绍的方法完成创建。

## 第二步

生成SipEndPoint。此SipEndPoint就代表所要创建的机器人，SipEndPoint创建时依赖唯一的SIP地址，所以创建此机器人后，凡是发送给此SIP地址的信息，都会获得自动回复。可以参考《UCMA开发-SipEndPoint》一文中所介绍的方法完成创建，但是实际的创建要稍显复杂一些。
```c#
SipEndpoint sipEndPoint;
try
{
    //《UCMA开发之SipEndPoint》中的new SipEndPoint()代码段
    ……
    //注册Session接收事件
    //如果发起了新的会话就会被触发，此时需要参与至新会话中
    sipEndPoint.SessionReceived += SipEndpoint\_SessionReceived;
}
catch (Exception ex)
{
    throw ex;
}
//如果是注册状态则注销，为下一次注册准备
if (sipEndPoint.RegistrationState == RegistrationState.Registered)
{
    sipEndPoint.Unregister();
}
//创建信号头
List headers = new List();
headers.Add(SignalingHeader.MicrosoftSupportedForking);
//如果是未注册状态则注册
if (sipEndPoint.RegistrationState == RegistrationState.Unregistered)
{
    sipEndPoint.Register(headers);
}
//如果是注册状态则注销，为下一次注册准备
if (sipEndPoint.RegistrationState == RegistrationState.Registered)
{
    sipEndPoint.Unregister();
}
//如果未注册则注册
//两次注册以保证服务器正确发布注册终端及其端口号
if (sipEndPoint.RegistrationState == RegistrationState.Unregistered)
{
    sipEndPoint.Register(headers);
}
```
## 第三步

处理SessionReceived事件。当其他联系人向第二步生成的SipEndPoint发送消息时，会首先创建新的SignalingSession，并且触发SessionReceived事件将其抛出，当事件被触发时，需要控制当前的SipEndPoint参与至此会话。
```c#
void SipEndpoint\_SessionReceived(object sender, SessionReceivedEventArgs e)
{
    //参见《UCMA开发之SignalingSession》
    e.Session.OfferAnswerNegotiation = this;
    //开始参与该会话Session
    e.Session.BeginParticipate(new AsyncCallback(ParticipateCallback), e.Session);
}
void ParticipateCallback(IAsyncResult ar)
{
    SignalingSession session = ar.AsyncState as SignalingSession;
    SipMessageData response = null;
    try
    {
        response = session.EndParticipate(ar);
        //参与至新会话后，就可以注册消息接收事件
        //当新消息到达时，会触发此事件
        session.MessageReceived += SipEndpoint\_MessageReceived;
    }
    catch(Exception ex)
    {
        throw ex;
    }
}
```
## 第四步

处理SignalSession的MessageReceived事件。在这一步中，其实就是机器人的最终实现，可以根据联系人发送的消息内容进行回复。
```c#
void SipEndpoint\_MessageReceived(object sender, MessageReceivedEventArgs e)
{
    SignalingSession session = sender as SignalingSession;
     //如果信息类型是消息，则触发接收事件并自动进行回复
     if (e.MessageType == MessageType.Message)
     {
         //自动回复
          session.SendMessage(MessageType.Message
                    , new System.Net.Mime.ContentType("text/plain")
                    , Encoding.UTF8.GetBytes("信息已被机器人自动接收！"));
    }
}
```

在第四步中，其实可以按照业务逻辑完成不同的操作或者回复不同的消息，比如：当天天气信息，联系人电话等等。这个示例就是介绍如何使用UCMA制作一个无人值守的机器人，在创建时主要要注意以下几点：

* 使用最新的凭证来创建RealTimeConnectionManager，否则很容易在接下来的的步骤中发生“UnKnown Error”。
* 将SipEndPoint注册两次，以保证其能够正确发布。
* 必须调用BeginParticipate方法参与至新的会话中，否则对方有可能会收不到自动回复的消息。  

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑