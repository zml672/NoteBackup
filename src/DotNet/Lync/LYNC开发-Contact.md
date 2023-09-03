# LYNC开发-Contact | 字痕随行

本文将描述如何使用Microsoft Lync SDK获取联系人信息。

## 获取联系人信息
可以通过Contact对象所提供的方法获取联系人信息，Contact对象隶属于命名空间Microsoft.Lync.Model，在获取联系人信息时，会使用到以下属性和方法：

| |名称|说明|
| ----- | ----- | ----- |
|方法|GetContactInformation(ContactInformationType)|从Contact对象中获取单一的联系人信息|
|属性|ContactManager|获取此联系人的父联系人和组管理|
|属性|CustomGroups|获取此联系人的联系人组列表|
|属性|Uri|获取联系人的Uri|

其中枚举ContactInformationType主要内容如下：

| |名称|说明|
| ----- | ----- | ----- |
| |Availability|联系人可用性(在线状态)，联系人信息项的值类型是AvailabilityType枚举。|
| |Activity|联系人的当前活动（例如，在手机上，在会议上，或可用）。联系人信息项的值类型为String。|
| |DisplayName|联系人的显示名称。联系人信息项的值类型为String。|
| |PersonalNote|个人注释。联系人信息项的值类型为String。|
| |Photo|联系人的照片。联系人信息项的值类型是Stream对象。|

* 获得联系人信息的示例代码如下：  
```c#
//获取contact 
contact = LyncClient.GetClient().ContactManager.GetContactByUri(strSIP);
//获取联系人的显示名称
contact.GetContactInformation(ContactInformationType.DisplayName).ToString()
//获取联系人的在线状态
(ContactAvailability)contact.GetContactInformation(
        ContactInformationType.Availability);
//获取联系人的Uri
contact.Uri;
```
* 获得联系人的联系人组列表示例代码如下：  
```c#
foreach (Group tempGroup in LyncClient.GetClient().ContactManager.Groups)
{

}
```

MSDN参考资料：[Get started with Lync contact lists](http://msdn.microsoft.com/en-us/library/jj937302(v=office.15).aspx "Get started with Lync contact lists")

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑
