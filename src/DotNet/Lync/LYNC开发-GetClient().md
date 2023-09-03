# LYNC开发-GetClient() | 字痕随行

# 获取客户端模型
使用Lync SDK开发时，最重要的是要先获取客户端模型，即Microsoft.Lync.Model. LyncClient，我们可以使用以下方法获取客户端模型：
```c#
_lyncClient = LyncClient.GetClient();
```

如果没有处于UI抑制模式下，使用上面的方法已经可以获取到客户端模型了，并且可以使用客户端模型做一些操作，比如登录。但是在此之前，必须将Microsoft Lync Client启动起来，否则就会出现错误；如果处于UI抑制模式下，使用以上的方式可以获得客户端模型，但是不能使用，因为此时进程communicator.exe并没有运行。

# 初始化客户端模型
如果Micorosoft Lync Client处于UI抑制模式下，我们运行程序时是没有反应的，使用GetClient()方法，我们只会获得客户端模型，而不会启动communicator.exe。所以在我们获取到客户端模型后，还需要调用相应的方法初始化客户端模型，以便启动communicator.exe。

我们可以使用以下方法初始化客户端模型：
```c#
if (_lyncClient.InSuppressedMode)
{
    if (_lyncClient.State == ClientState.Uninitialized)
    {
        _lyncClient.BeginInitialize(LyncClientInitializeCallback, _lyncClient);;
    }
}
private void LyncClientInitializeCallback(IAsyncResult ar)
{
    if (ar.IsCompleted)
    {
        ((LyncClient)ar.AsyncState).EndInitialize(ar);
    }
}
```

上述代码中的LyncClient.BeginInitializ方法是用来在UI抑制模式下初始化LyncClient的，在非UI抑制模式下，并不需要调用此方法。

MSDN参考链接：[Understanding UI Suppression in Lync SDK](http://msdn.microsoft.com/zh-cn/library/hh345230.aspx "Understanding UI Suppression in Lync SDK")

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑