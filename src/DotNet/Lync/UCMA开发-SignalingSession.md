# UCMA开发-SignalingSession | 字痕随行

本文的目的在于讲解如何创建、使用SignalingSession，为之后构建自动应答机器人做准备。

## Constructors

| |名称|说明|
| ----- | ----- | ----- |
| |SignalingSession(RealTimeEndpoint, RealTimeAddress)|使用端点(EndPoint)和目标初始化一个信令会话(SignalingSession)|

## Methods

| |名称|说明|
| ----- | ----- | ----- |
| |BeginAccept(AsyncCallback, Object)|接受会话|
| |BeginEstablish(AsyncCallback, Object)|建立会话|
| |BeginParticipate(AsyncCallback, Object)|加入一个会话，这个方法需要调用(用户传入和传出消息)已建立的会话|
| |BeginSendMessage(MessageType, ContentType, Byte \[\],AsyncCallback, Object)|发送一条消息，会话应该在连接状态|
| |BeginSendMessage(MessageType, ContentType, Byte \[\],IEnumerable, AsyncCallback, Object)| |
| |BeginTerminate(AsyncCallback, Object)|异步终止会话，本次会话将不再可用|

## Properties

| |名称|说明|
| ----- | ----- | ----- |
| |OfferAnswerNegotiation|获取或设置由调用者实现并提供的应答协商接口|
| |State|获取或设置会话的状态|

## Events

| |名称|说明|
| ----- | ----- | ----- |
| |MessageReceived|收到消息时触发|

## Example

```c#
//直接建立SignalingSession，并发送消息
RealTimeAddress target = new RealTimeAddress(strRemoteUri);
SignalingSession session = new SignalingSession(sipEndPoint, target);
session.OfferAnswerNegotiation = this;
try
{
    session.EndEstablish(session.BeginEstablish(null, null));
    break;
}
catch
{
    Thread.Sleep(10);
}
ContentType contentType = new ContentType("text/plain; charset=UTF-8");
byte[] msgBody = Encoding.UTF8.GetBytes(strMessage);
try
{
    session.SendMessage(
        MessageType.Message,
        contentType,
        msgBody);
}
catch (Exception ex)
{
    throw ex;
}
session.EndTerminate(session.BeginTerminate(null, null));
//通过SipEndPoint的SessionReceived事件来建立
void SipEndpoint_SessionReceived(object sender, SessionReceivedEventArgs e)
{
    e.Session.OfferAnswerNegotiation = this;
    //开始参与该会话Session
    try
    {
        e.Session.BeginParticipate(
            new AsyncCallback(ParticipateCallback), e.Session);
    }
    catch
    {
    }
}
//参与处理回发事件
void ParticipateCallback(IAsyncResult ar)
{
    SignalingSession session = ar.AsyncState as SignalingSession;
    SipMessageData response = null;
    try
    {
        response = session.EndParticipate(ar);
        session.SendMessage(
            MessageType.Message,
            contentType,
            msgBody);
    }
    catch
    {
    }
}
```

关于“Session.OfferAnswerNegotiation = this;”的解释：

this其实代表了继承IOfferAnswer接口的类，在示例中，因为代码段所属的类继承并实现了IOfferAnswer接口的，所以可以直接将自己赋值给OfferAnswerNegotiation属性，IOfferAnswer的实现见下面的示例代码：

```c#
#region IOfferAnswer 接口实现
//Occurs when we receive and INVITE with no offer
public ContentDescription GetAnswer(object sender, ContentDescription offer)
{
    return GetContentDescription((SignalingSession)sender);
}
//Occurs when we receive an invite with an offer
public ContentDescription GetOffer(object sender)
{
    return GetContentDescription((SignalingSession)sender);
}
//Occurs in Reinvite cases
public void HandleOfferInInviteResponse(object sender, OfferInInviteResponseEventArgs e)
{
    return;
}
//Occurs in Reinvite cases
public void HandleOfferInReInvite(object sender, OfferInReInviteEventArgs e)
{
    return;
}
//Occurs when we initiate the invite
public void SetAnswer(object sender, ContentDescription answer)
{
    SignalingSession session = sender as SignalingSession;
    byte[] Answer = answer.GetBody();
    if (Answer != null)
    {
        Sdp<SdpGlobalDescription, SdpMediaDescription> sessionDescription =
            new Sdp<SdpGlobalDescription, SdpMediaDescription>();
        if (!sessionDescription.TryParse(Answer))
        {
            session.BeginTerminate(null, session);
            return;
        }
        else
        {
            IList ActiveMediaTypes =
                sessionDescription.MediaDescriptions;
            if ((ActiveMediaTypes.Count == 1) &&
                (ActiveMediaTypes[0].MediaName.Equals("message",
                    StringComparison.Ordinal)) &&
                (ActiveMediaTypes[0].Port > 0) &&
                (ActiveMediaTypes[0].TransportProtocol.Equals("sip",
                    StringComparison.OrdinalIgnoreCase)))
            {
            }
            else
            {
                session.BeginTerminate(null, session);
            }
        }
    }
}
//Retrieves the content description for offers and answers
private ContentDescription GetContentDescription(SignalingSession session)
{
    IPAddress ipAddress;
    // This method is called back every time an outbound INVITE is sent.
    if (session.Connection != null)
    {
        ipAddress = session.Connection.LocalEndpoint.Address;
    }
    else
    {
        ipAddress = IPAddress.Any;
    }
    Sdp<SdpGlobalDescription, SdpMediaDescription> sessionDescription =
        new Sdp<SdpGlobalDescription, SdpMediaDescription>();
    //Set the origin line of the SDP
    //s, t, and v lines are automatically constructed
    sessionDescription.GlobalDescription.Origin.Version = 0;
    sessionDescription.GlobalDescription.Origin.SessionId = "0";
	sessionDescription.GlobalDescription.Origin.UserName = "-";
	sessionDescription.GlobalDescription.
	  Origin.Connection.Set(ipAddress.ToString());
	//Set the connection line
	sessionDescription.GlobalDescription.Connection.TrySet(ipAddress.ToString());
	SdpMediaDescription mditem = new SdpMediaDescription("message");
	mditem.Port = 5061;
    mditem.TransportProtocol = "sip";
	mditem.Formats = "null";
	SdpAttribute aitem = new SdpAttribute("accept-types", "text/plain");
	mditem.Attributes.Add(aitem);
	//Append the Media description to the Global description
	sessionDescription.MediaDescriptions.Add(mditem);
	ContentType ct = new ContentType("application/sdp");
	return new ContentDescription(ct, sessionDescription.GetBytes());
}
#endregion
```

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑