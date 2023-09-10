# 开启Exchange账户 | 字痕随行

原来做过一个项目，用网页来添加AD账号并开启Exchange2007邮箱账户。添加AD账户还比较轻松容易，但是开启Exchange账户的时候遇到了一些麻烦，最主要的是权限还有PowerShell的参数问题。

如何添加AD账户，网上一搜一大堆，基本就是那样，也不需要特殊的权限，只是每次声明DirectoryEntry时，需要带上具有与管理权限的用户。DirectoryEntry entry = new DirectoryEntry(LDAP，域管理员账号,域管理员密码,AuthenticationTypes.Secure);这里的域管理员账号最好是[admin@xxx.com.cn](mailto:admin@xxx.com.cn)这种格式的，如果不是这种格式的话，会报一个错误，不过这错误拿到网上一搜一大堆解决方案。

接下来就是开通Exchange2007账户了。因为Exchange2007开始微软开始使用PowerShell命令行来进行Exchange邮箱管理，我们必须运行脚本来进行账户开通和删除。

开始需要进行一些准备工作。需要准备一个账户，这个账户A要有两个权限，一个是Account Operators，另外一个是Exchange Recipient Administrators。第一个权限是为了管理域用户，第二个权限是为了管理Exchange2007。

然后还需要一个DLL文件，这个文件是用来运行PowerShell的，这个文件可以在C:\\Program Files\\Reference Assemblies\\Microsoft\\WindowsPowerShell\\v1.0\\System.Management.Automation.dll找到。前提是你安装了PowerShell。反正我电脑上有，如果你没有搜索吧。这样准备工作就完成了。

新建立一个web服务，这是因为运行Shell脚本的程序必须部署在exchange2007服务器上，否则的话会报错误。当然了你把这整个系统全都扔到exchange服务器上也没问题。

我开通账户用的是Enable-Mailbox这个命令，因为我已经在AD里面建好账户了。如果你需要这些命令的具体说明可以去MSDN上查看。

下面用C#运行这个命令的函数主体，你需要在工程里面引用上面所提到的那个DLL文件，下面的类基本上都是在这个DLL文件里面的。
```csharp
RunspaceConfiguration runspaceConf = RunspaceConfiguration.Create();
PSSnapInException PSException = null;
PSSnapInInfo info = runspaceConf.AddPSSnapIn("Microsoft.Exchange.Management.PowerShell.Admin", out PSException);
Runspace runspace = RunspaceFactory.CreateRunspace(runspaceConf);
5. runspace.Open();
Pipeline pipeline = runspace.CreatePipeline();
Command command = new Command("Enable-Mailbox");
command.Parameters.Add("Identity", identity);//人员组织机构位置
command.Parameters.Add("Database", ConfigurationManager.AppSettings["MailDataBase"].Trim());  //Exchange数据库位置
pipeline.Commands.Add(command);
Collection result = pipeline.Invoke();
```

这个函数里面有一个需要注意的地方，也是我调试了半天才得出的结论，identity必须符合这样的“pyc.local/XXX/XX/人员的sn”格式，否则运行的时候它也不报错，就是建立不成功。

下面是删除的函数：
```csharp
RunspaceConfiguration runspaceConf = RunspaceConfiguration.Create();
PSSnapInException PSException = null;
PSSnapInInfo info = runspaceConf.AddPSSnapIn("Microsoft.Exchange.Management.PowerShell.Admin", out PSException);
Runspace runspace = RunspaceFactory.CreateRunspace(runspaceConf);
runspace.Open();
Pipeline pipeline = runspace.CreatePipeline();
Command command = new Command("Remove-Mailbox");
command.Parameters.Add("Identity", identity);
command.Parameters.Add("Confirm", false);//添加此参数以防止出现Cannot invoke this function because the current host does not implement it
pipeline.Commands.Add(command);
Collection result = pipeline.Invoke();
```

这个函数的identity可以是别名，可以是人员唯一的登录名，这个可以去MSDN上查看。一定要添加confirm这个参数，并且值是false，否则就是报上面注释的错误。

这样删除和开通的函数就有了，现在需要把它部署到exchang服务器上去，这时候我们就需要用到上面建立的账号A了。

在IIS里面新建一个网站，新建一个程序池，这个新建网站在新建立的程序池上运行。程序池的属性->标示选择配置，输入A账户名和密码。这样我们运行这个web服务使用的就是A账户，而A账户拥有操作AD和Exchange的权限，就能顺利的运行操作命令了。

基本上到此程序就能够跑通了，当时，我在这短短的几行代码上花费了3天，因为基本上这些问题都没有中文的资料，还是老外牛奔啊，不过讨论的也比较少，在我连蒙带猜带用我二把刀英文外带使劲google的情况下终于让我跑通了这个程序，真不容易，再次记录，希望以后少走弯路。

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑