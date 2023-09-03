# LYNC开发-登录 | 字痕随行

本文将描述如何使用Microsoft Lync SDK控制Microsoft Lync Client完成登录。
## 图示

![](../../images/LYNC%E5%BC%80%E5%8F%91-%E7%99%BB%E5%BD%95/image-20230903163902490.png)

## 说明

* 初始化客户端模型，使用[《Lync开发-GetClient()》](http://www.blackzs.com/archives/176 "LYNC开发-GetClient()")中所介绍的方法获取LyncClient。
* 注册客户端模型事件“StateChanged”和“CredentialRequested”。  
```c#
lyncClient.StateChanged += LyncClient_StateChanged;
lyncClient.CredentialRequested += LyncClient_CredentialRequested;
```
* 调用BeginSignIn方法，开始登录。  
```c#
lyncClient.BeginSignIn(_strSIP, null, null, LyncSignInCallback, _lyncClient);
```
* 触发CredentialRequested事件。  
```c#
void LyncClient_CredentialRequested(object sender, CredentialRequestedEventArgs e)
{
    if (e.Type == CredentialRequestedType.SignIn)
    {
        e.Submit(strUserName, strPassWord, blIsRememberPWD);
    }
}
```
* 调用EndSignIn方法，结束登录。  
```c#
void LyncSignInCallback(IAsyncResult ar)
{
    if (ar.IsCompleted)
    {
        try
        {
             ((LyncClient)ar.AsyncState).EndSignIn(ar);
        }
        catch
        {
            throw;
        }
     }
}
```
* 触发StateChanged事件。  
```c#
void LyncClient_StateChanged(object sender, ClientStateChangedEventArgs e)
{
    if (e.NewState == ClientState.SignedIn)
    {
        //登录成功
     }
}
```
## 注意事项

调用BeginSignIn方法时，如果第二个和第三个参数输入为null，则会触发CredentialRequested事件。如果输入域账户名称和密码，在正确的情况下会成功登录，并不会触发CredentialRequested事件。

触发CredentialRequested事件，调用Submit方法提交用户信息时，如果用户凭证正确，则登录成功；如果用户凭证不正确，则会再次触发CredentialRequested事件。如果\_blIsRememberPWD 等于true，会生成相应的用户证书，下一次调用BeginSignIn方法时，只需要SIP地址（第一个参数），就可以成功登录。

MSDN参考：[How to: Sign In to Lync with UI Suppressed](http://msdn.microsoft.com/zh-cn/library/hh378603 "How to: Sign In to Lync with UI Suppressed")

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑