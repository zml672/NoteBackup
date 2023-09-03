# UCMA开发-ConnectionManager

本文的目的在于讲解如何创建RealTimeServerTlsConnectionManager，为之后构建自动应答机器人做准备。

## Constructors

| |名称|说明|
| ----- | ----- | ----- |
| |RealTimeServerTlsConnectionManager(String,Byte\[\])|使用默认的本地主机名称和给定的证书信息实例化|
| |RealTimeServerTlsConnectionManager(String, String, Byte\[\])|使用给定的本地主机名称和给定的证书信息实例化|

## Methods

| |名称|说明|
| ----- | ----- | ----- |
| |StartListening|开始监听指定的地址和端口（继承自RealTimeServerConnectionManager）|
| |StopListening|停止监听新的连接（继承自RealTimeServerConnectionManager）|

## Properties

| |名称|说明|
| ----- | ----- | ----- |
| |IsListening|获取监听是否启用（继承自RealTimeServerConnectionManager。）|
| |ListeningPort|获取监听端口（继承自RealTimeServerConnectionManager。）|
| |NeedMutualTls|获取或设置一个MutualTls连接对于传出的Tls连接是否是必须的|

## Example
```c#
RealTimeConnectionManager _connectionManager;
RealTimeServerTlsConnectionManager _serverTlsConnectionManager;
try
{
    _serverTlsConnectionManager = new RealTimeServerTlsConnectionManager(
        _strCertificateIssuerName, _strCertificateSerialNumber);
}
catch (TlsFailureException ex)
{
    throw ex;
}
_serverTlsConnectionManager.NeedMutualTls = true;
_connectionManager = (RealTimeConnectionManager)_serverTlsConnectionManager;
IPAddress localIpAddress = Dns.GetHostAddresses(Dns.GetHostName())[1];
if (!_serverTlsConnectionManager.IsListening)
{
    _serverTlsConnectionManager.StartListening(new IPEndPoint(localIpAddress, 0));
}
```

如何获取证书，可以参考如下代码：
```c#
//using System.Security.Cryptography.X509Certificates;
X509Store store = new X509Store(StoreName.My, StoreLocation.LocalMachine);
try
{
   store.Open(OpenFlags.ReadOnly | OpenFlags.OpenExistingOnly);
}
catch (System.Security.SecurityException)
{
   MessageBox.Show("你没有权限枚举本地计算机存储的证书");
   return;
}
if (store.Certificates.Count < 1)
{
    MessageBox.Show("请确保证书存在");
    return;
}
//遍历寻找适合的证书
foreach (X509Certificate2 certificate in store.Certificates)
{
    StringBuilder sb = new StringBuilder();
    XmlWriter writer = XmlWriter.Create(sb);
    writer.WriteStartElement("root");
    writer.WriteBinHex(
        certificate.GetSerialNumber(), 0, certificate.GetSerialNumber().Length);
    writer.WriteEndElement();
    writer.Close();
    this.txtReport.AppendText(
        "IssuerName:" + certificate.Issuer
        + "\\r\\nSerialNumber:" + sb.ToString() + "\\r\\n\\r\\n");
}
```

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑