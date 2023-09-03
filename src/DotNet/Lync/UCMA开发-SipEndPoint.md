# UCMA开发-SipEndPoint | 字痕随行

本文的目的在于讲解如何创建、使用SipEndPoint，为之后构建自动应答机器人做准备。

## Constructors

| |名称|说明|
| ----- | ----- | ----- |
| |SipEndpoint(String, SipAuthenticationProtocols, SipTransportType, String)|创建SipEndpoint的新实例。这个端点是基于服务器的。默认情况下，该平台将为TCP使用端口5060 ，为TLS使用端口5061。要使用一个在这些之外的端口，调用方应尝试注册之前设置端口属性。|
| |SipEndpoint(String, SipAuthenticationProtocols, SipTransportType, String, Int32, Boolean, RealTimeConnectionManager, String)|创建SipEndpoint的新实例。这个端点是基于服务器的。|

## Methods

| |名称|说明|
| ----- | ----- | ----- |
| |BeginRegister(AsyncCallback, Object)|为当前EndPoint启动异步注册操作。|
| |BeginRegister(<br>IEnumerable, AsyncCallback, Object)|为当前EndPoint启动异步注册操作|
| |Register()|同步注册当前的EndPoint，此方法将等待，直到注册完成，不推荐在UI线程内使用|
| |Register(IEnumerable)|同步注册当前的EndPoint，此方法将等待，直到注册完成，不推荐在UI线程内使用|
| |BeginUnregister|开始异步注销当前EndPoint，这个方法总是成功|
| |Unregister|同步注销当前的EndPoint，不推荐在UI线程使用|
| |BeginTerminate|开始终止EndPoint，并且清理活动的会话和资源。端点不再可用（继承自RealTimeEndpoint）|
| |Terminate|终止EndPoint和清理活动的会话和资源，端点不再可用（继承自RealTimeEndpoint）|

## Properties

| |名称|说明|
| ----- | ----- | ----- |
| |CredentialCache|获取凭证缓存|
| |RegistrationState|获取当前EndPoint的注册状态|

## Events

| |名称|说明|
| ----- | ----- | ----- |
| |MessageReceived|收到消息时触发（继承自RealTimeEndpoint）|
| |SessionReceived|收到新的邀请时触发（继承自RealTimeEndpoint）|

## Example

```c#
SipEndpoint sipEndPoint;
try
{
    //strUri为EndPoint地址，必须以“sip:”开始
    sipEndPoint = new SipEndpoint(strUri,
                    SipAuthenticationProtocols.None,
                    SipTransportType.Tls,
                    _strServerName,
                    5061,
                    true,
                    _connectionManager,
                    null);
    sipEndPoint.CredentialCache.Add(
        SipEndpoint.DefaultRtcRealm,
         CredentialCache.DefaultNetworkCredentials);
}
catch (Exception ex)
{
    throw ex;
}
//创建信号头
List headers = new List();
headers.Add(SignalingHeader.MicrosoftSupportedForking);
//如果是未注册状态则注册
if (sipEndpoint.RegistrationState == RegistrationState.Unregistered)
{
    sipEndpoint.Register(headers);
}  
```

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑