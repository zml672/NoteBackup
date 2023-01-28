# CAS5.3-Restful方式认证 | 字痕随行

这篇应该是CAS系列的最后一篇了，就这么晃荡晃荡，就快过年了。

CAS高版本都支持使用Restful方式进行认证，主要是通过其开放的3个接口实现的。

**第一个，获取TGT。**

TGT，可以理解为一个存活时间稍长的身份凭证，这个凭证有两个有效期配置：

```Plain Text
# Set to a negative value to never expire tickets
# cas.ticket.tgt.maxTimeToLiveInSeconds=28800
# cas.ticket.tgt.timeToKillInSeconds=7200

```
很多文章都说，如果7200秒没有移动过鼠标，这个凭证就会过期。

但是，一个B/S系统，这Server端是怎么监控Client端的？

所以，我觉得，可能是如果7200秒内没有任何请求就会过期。

28800是最大存续时间，意思就是不管如何，这个时间之后，TGT就会失效。

获取TGT的代码如下：

```Plain Text
public String getTgt(String loginName, String loginPwd) {
    String tgtUrl = "https://ip:port/cas/v1/tickets";
    //使用hutool实现
    HttpResponse httpResponse = HttpRequest
            .post(tgtUrl)
            .contentType("application/x-www-form-urlencoded;charset=UTF-8")
            .header(Header.ACCEPT, "application/json;charset=UTF-8")
            .form(new HashMap<String, Object>() {
                private static final long serialVersionUID = -138163390499485640L;

                {
                    put("username", loginName);
                    put("password", loginPwd);
                }
            })
            .execute();
    return httpResponse.body();
}

```
**第二个，获取GT。**

GT，使用TGT换取的一次性票据，它和Service是绑定的。

所谓的Service其实就是需要验证的完整Url，比如上一篇的/sso/test。

默认配置下，GT获取以后，需要在10秒内进行认证，并且只能使用一次。

获取GT的代码如下：

```Plain Text
public String getGt(String tgt, String service) {
    String stUrl = "https://ip:port/cas/v1/tickets/" + tgt;
    HttpResponse httpResponse = HttpRequest
            .post(stUrl)
            .contentType("application/x-www-form-urlencoded;charset=UTF-8")
            .header(Header.ACCEPT, "application/json;charset=UTF-8")
            .form(new HashMap<String, Object>() {
                private static final long serialVersionUID = -6472219360813373597L;

                {
                    put("service", service);
                }
            })
            .execute();
    return httpResponse.body();
}

```
**第三个，验证GT。**

获取GT之后，就可以在目标系统上进行验证了，通过验证之后，就可以获得用户的身份信息了。

代码如下：

```Plain Text
public String getValidate(String ticket, String service) {
    String validateUrl = "https://ip:port/cas/p3/serviceValidate";
    HttpResponse httpResponse = HttpRequest
            .get(validateUrl)
            .form(new HashMap<String, Object>() {
                private static final long serialVersionUID = -7333706958754801580L;

                {
                    put("service", service);
                    put("ticket", ticket);
                }
            })
            .header(Header.ACCEPT, "application/json;charset=UTF-8")
            .execute();
    return httpResponse.body();
}

```
该接口返回的[cas:user](cas:user)中的内容，就是登录账号。

如果目标系统已经按照上一篇完成了CAS的配置，其实可以直接在拦截的Url后面加上ticket参数，比如：

```Plain Text
http://localhost/sso/test?ticket=获得的gt

```
如果Url和GT匹配，会自动通过验证的，有兴趣的同学可以自行尝试。

好了，以上，如果有错误，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑