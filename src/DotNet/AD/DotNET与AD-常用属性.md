# DotNET与AD-常用属性 | 字痕随行

AD中节点为“User”类型的常用属性如下：

|名称|说明|备注|
| ----- | ----- | ----- |
|sAMAccountName|用户登录名|Windows2000前|
|userPrincipalName|用户登录名| |
|userAccountControl|帐户行为控制标志| |
|cn|公共名称|通常以此属性标识该用户，name的值相同|
|name|名称|通常以此属性标识该用户，同公共名称CN|
|sn|姓| |
|givenName|名| |
|displayName|显示名称| |
|mail|电子邮件| |

重要的公共属性如下：

|名称|说明|备注|
| ----- | ----- | ----- |
|objectGUID|ID|主键标识|
|objectClass|类型|person,user,organizationalUnit|

"userAccountControl"的对应值如下：

|属性标志|十六进制值|十进制值|说明|
| ----- | ----- | ----- | ----- |
|SCRIPT|0x0001|1|将运行登录脚本|
|ACCOUNTDISABLE|0x0002|2|禁用用户帐户|
|HOMEDIR\_REQUIRED|0x0008|8|需要主文件夹|
|LOCKOUT|0x0010|16| |
|PASSWD\_NOTREQD|0x0020|32|不需要密码|
|PASSWD\_CANT\_CHANGE|0x0040|64|用户不能更改密码。可以读取此标志，但不能直接设置它|
|ENCRYPTED\_TEXT\_PWD\_ALLOWED|0x0080|128|用户可以发送加密的密码|
|TEMP\_DUPLICATE\_ACCOUNT|0x0100|256|此帐户属于其主帐户位于另一个域中的用户。此帐户为用户提供访问该域的权限，但不提供访问信任该域的任何域的权限。有时将这种帐户称为“本地用户帐户”。|
|NORMAL\_ACCOUNT|0x0200|512|这是表示典型用户的默认帐户类型|
|INTERDOMAIN\_TRUST\_ACCOUNT|0x0800|2048|对于信任其他域的系统域，此属性允许信任该系统域的帐户|
|WORKSTATION\_TRUST\_ACCOUNT|0x1000|4096|这是运行 Microsoft Windows NT 4.0 Workstation、Microsoft Windows NT 4.0 Server、Microsoft Windows 2000 Professional 或 Windows 2000 Server 并且属于该域的计算机的计算机帐户。|
|SERVER\_TRUST\_ACCOUNT|0x2000|8192|这是属于该域的域控制器的计算机帐户|
|DONT\_EXPIRE\_PASSWORD|0x10000|65536|表示在该帐户上永远不会过期的密码|
|MNS\_LOGON\_ACCOUNT|0x20000|131072|这是 MNS 登录帐户|
|SMARTCARD\_REQUIRED|0x40000|262144|设置此标志后，将强制用户使用智能卡登录|
|TRUSTED\_FOR\_DELEGATION|0x80000|524288|设置此标志后，将信任运行服务的服务帐户（用户或计算机帐户）进行 Kerberos 委派。任何此类服务都可模拟请求该服务的客户端。若要允许服务进行 Kerberos 委派，必须在服务帐户的userAccountControl 属性上设置此标志|
|NOT\_DELEGATED|0x100000|1048576|设置此标志后，即使将服务帐户设置为信任其进行 Kerberos 委派，也不会将用户的安全上下文委派给该服务|
|USE\_DES\_KEY\_ONLY|0x200000|2097152|(Windows 2000/Windows Server 2003) 将此用户限制为仅使用数据加密标准 (DES) 加密类型的密钥|
|DONT\_REQ\_PREAUTH|0x400000|4194304|(Windows 2000/Windows Server 2003) 此帐户在登录时不需要进行 Kerberos 预先验证|
|PASSWORD\_EXPIRED|0x800000|8388608|(Windows 2000/Windows Server 2003) 用户的密码已过期|
|TRUSTED\_TO\_AUTH\_FOR\_DELEGATION|0x1000000|16777216|(Windows 2000/Windows Server 2003) 允许该帐户进行委派。这是一个与安全相关的设置。应严格控制启用此选项的帐户。此设置允许该帐户运行的服务冒充客户端的身份，并作为该用户接受网络上其他远程服务器的身份验证|

"userAccountControl"的属性标志是累计的，针对66050可以如下面一样解析：
66050=65536+512+2，分别表示“65536” - 密码永不过期，“512” - 用户状态正常，“2” - 用户被禁用。

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑