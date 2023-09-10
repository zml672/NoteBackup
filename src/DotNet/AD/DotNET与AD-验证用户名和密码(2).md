# DotNET与AD-验证用户名和密码2 | 字痕随行

假设现在有一个需求：将AD中的人员名称定时同步至某个站点的数据库内，该站点非匿名访问，登录时需要进行域验证。

针对这个需求，需要至少一张数据库表（在此命名为TB\_UserInfo），表结构设计如下：

|字段名称|类型|备注|
| ----- | ----- | ----- |
|UserID|varchar|主键，用户ID，与AD中user的ID属性相同|
|UserName|varchar|用户名称，与AD中user的sAMAccountName属性相同|
|DisplayName|Varchar|用户显示名称，与AD中user的DisplayName属性相同|

同步程序设计如下：

1. 使用Windows Service或者桌面应用程序开发，安装至服务器后设置为开机自启动。
2. 假设数据量在千条之内，则将AD中指定节点下的人员数据和数据库中表TB\_UserInfo中的数据读取至内存集合（ADUserCollection和DBUserCollection）中，在数据库中更新属于ADUserCollectio且属于DBUserCollection的数据；在数据库中删除属于DBUserCollection但不属于ADUserCollection的数据；在数据库中新增属于ADUserCollection但不属于DBUserCollection的数据。

示例代码如下：
```csharp
/// 
/// 人员信息
/// 
public class UserInfo
{
        /// 
        /// 人员ID
        /// 
        public string UserID  {  get ; set; }
        /// 
        /// 用户名
        /// 
        public string UserName  {  get; set; }
        /// 
        /// 显示名称
        /// 
        public string DisplayName  {  get; set; }
}
/// 
/// 为UserInfo自定义的比较类
/// 
public class UserInfoComparer : IEqualityComparer
{
        public bool Equals(UserInfo x, UserInfo y)
        {
                return x.UserID == y.UserID;
        }
        public int GetHashCode(UserInfo userInfo)
        {
                return userInfo.UserID.GetHashCode();
        }
}
/// 
/// 将活动目录中的人员信息同步至数据库
/// 
public void SynchronizationADUserToDB()
{
        List lstADUser = GetUserFromAD();
        List lstDBUser = GetUserFromDB();
        //取得存在于DB中但不存在于AD中的人员集合
        List lstImportUser =
                lstDBUser.Except(lstADUser, new UserInfoComparer()).ToList();
        foreach (UserInfo userInfo in lstImportUser)
        {
                DeleteUserToDB (userInfo);
        }
        //取得存在于AD中且存在于DB中的人员集合
        lstImportUser 
	        = lstADUser.Intersect(lstDBUser, new UserInfoComparer()).ToList();
        foreach (UserInfo userInfo in lstImportUser)
        {
                UpdateUserToDB (userInfo);
                lstADUser.Remove(userInfo);
        }
        //取得存在于AD中但不存在于DB中的人员集合
        lstImportUser = lstADUser;
        foreach (UserInfo userInfo in lstImportUser)
        {
                InsertUserToDB(userInfo);
        }
}
```

站点的程序设计比较简单，只需要在登录时先验证人员是否存在于数据库中，然后再验证是否可以使用输入的凭证信息登录AD，示例代码如下：
```csharp
//通过输入的账户信息验证是否存在于数据库中
//验证是否可以使用输入的账户信息登录AD
DirectoryEntry entry = null;
try
{        
	entry = new DirectoryEntry(
		strLDAP, strUserName, strPWD, AuthenticationTypes.Secure);
    object objID = entry.NativeGuid;
	return true;
}
catch
{
        return false;
}
//如果既存在于数据库同时也可以登录AD，则该用户为合法用户
```

至此，所假设的需求已经实现。如果AD内user数量巨大，最好不要使用此示例中所提供的导入方法，嵌套遍历会使效率极其低下。

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑