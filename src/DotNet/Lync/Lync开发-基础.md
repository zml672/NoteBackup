# Lync开发-基础

## 什么是Lync SDK？

此处以Microsoft Lync 2010 SDK为例说明，其SDK包括：

* 在项目中使用的Microsoft.Office.Ocom.dll和Microsoft.Office.uc.dll应用程序集。
* Lync 2010控件(Controls)。
*  Lync 2010的API。
*  Lync 2010的框架程序员指南。
* 示例应用程序。

Microsoft Lync 2010 SDK是一个客户端API，它结合了微软的Lync自动化API的自动化功能与微软的统一通信客户端API的功能。访问一个使用Lync 2010 SDK开发的应用程序中的Lync API功能，必须在本地主机上启动一个Microsoft Lync 2010 Client。

## Lync SDK的获取与安装

你可以在以下地址下载适合自己所需要的安装包：

[Microsoft Lync 2010 SDK](http://www.microsoft.com/en-us/download/details.aspx?id=18898 "Microsoft Lync 2010 SDK")

[Microsoft Lync 2013 SDK](http://www.microsoft.com/en-us/download/details.aspx?id=36824 "Microsoft Lync 2013 SDK")

下载完毕后，在开发机上运行即可。

## 什么是Microsoft Lync Controls？

Lync SDK中提供了一些控件，每个Lync Control提供了一个特定的功能，如搜索、在线状态等，每个控件的外观复制了Lync Client的UI。

需要注意的是：如果Lync UI Suppression(Lync UI 抑制模式)被打开，则不能使用Lync Controls。

## 什么是Lync UI Suppression？

Lync UI Suppression即Lync UI抑制模式。当开启Lync UI Suppression时，可以完全隐藏Microsoft Lync 2010客户端界面(双击安装路径下的communicator.exe时毫无反应)。UI抑制是非常有用的，它可以让我们开发定制另类的UI。但是需要注意的是，Lync UI Suppression开启时，Automation和Lync Controls是不可用的。

## 如何开启Lync UI Suppression？

如果电脑是64位操作系统，可以修改注册表中的键“HKLM\\SOFTWARE\\Wow6432Node\\Microsoft\\Communicator\\UISuppressionMode”的值为“1”，即可开启。

如果电脑是32位操作系统，可以修改注册表中的键“HKLM\\SOFTWARE\\Microsoft\\Communicator\\UISuppressionMode”的值为“1”，即可开启。

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑