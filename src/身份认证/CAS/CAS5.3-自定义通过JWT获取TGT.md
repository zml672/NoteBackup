# CAS5.3-自定义通过JWT获取TGT | 字痕随行

这天气突然就热了起来，终于要进入我最喜欢的季节了。

上一篇[CAS5.3-Restful方式认证](http://mp.weixin.qq.com/s?__biz=MzI3NTE2NzczMQ==&mid=2650046496&idx=1&sn=01ffaebd4c231308dc4cf05cd9805f4c&chksm=f308c0bcc47f49aa491b97c5a7098aa970c8090f9929b975bce3d85f4b499bf7e122ad2e65b4&scene=21#wechat_redirect)，必须使用用户名和密码才能获取TGT。

翻了翻它的Rest Support，发现使用cas-server-support-rest-tokens，也仅仅是用JWT替代TGT而已。

也就是说，还是必须使用用户名和密码，只不过获得的TGT变成了JWT，然后拿着JWT去获取ST。

本篇尝试通过自定义的方式，实现以下需求：

1. 通过JWT获得TGT，不再需要用户名和密码。
2. JWT的秘钥位于[CAS5.3-基于JWT认证](http://mp.weixin.qq.com/s?__biz=MzI3NTE2NzczMQ==&mid=2650046472&idx=1&sn=19e1ce3cd096306681c62181ea714b9b&chksm=f308c094c47f4982727638dd1eec567b4d23e40a97eddb2c243c6fb9d35120c893ab5b67322d&scene=21#wechat_redirect)中的Serivce配置文件中。
3. 获得的TGT在CAS Server中有效，可以通过其获取的ST实现其它系统的单点登录。

这里需要参考CAS Server的几段源码，需要下载Cas Server的源工程。

**第一段代码**

参考它如何通过用户名和密码获得TGT的，需要从

```java
org.apereo.cas.support.rest.resources.TicketGrantingTicketResource

```
这个类入手。

一路跟踪下去，会发现这样的代码：

```java
protected TicketGrantingTicket createTicketGrantingTicketForRequest(final MultiValueMap<String, String> requestBody,
                                                                    final HttpServletRequest request,
                                                                    final HttpServletResponse response) throws Exception {
    //主要得看这地方，怎么得到的authenticationResult，如何仿造
    val authenticationResult = authenticationService.authenticate(requestBody, request, response);
    val result = authenticationResult.orElseThrow(FailedLoginException::new);
    return centralAuthenticationService.createTicketGrantingTicket(result);
}

```
找一下AuthenticationResult的实现类，发现DefaultAuthenticationResult，进而发现依赖于Authentication。

最终组装的这段代码如下：

```java
Principal principal = PrincipalFactoryUtils.newPrincipalFactory().createPrincipal(authenticate.getPrincipal().getId(), new HashMap<>(0));
CredentialMetaData meta = new BasicCredentialMetaData();
Authentication authentication = new DefaultAuthenticationBuilder(principal)
        .addCredential(meta)
        .addSuccess("tokenTgtHandler", new DefaultAuthenticationHandlerExecutionResult(usernamePasswordCaptchaAuthenticationHandler, meta))
        .setAttributes(new HashMap<>(0))
        .build();
AuthenticationResult authenticationResult = new DefaultAuthenticationResult(authentication, null);
TicketGrantingTicket ticketGrantingTicket = centralAuthenticationService.createTicketGrantingTicket(authenticationResult);

```
这里就引出**第二段参考代码，**怎么获得当前的人员标识。

当前的人员肯定在JWT里面，但是怎么解析JWT呢？

直接可以参考login是怎么通过JWT认证的，可以断点跟踪一下，最终的代码如下：

```java
Credential credential = new TokenCredential(token, serviceInstance);
AuthenticationHandlerExecutionResult authenticate;
authenticate = authenticationHandler.authenticate(credential)

```
**最后**

新建一个Controller，然后创建一个方法，把上面的代码复制过去，调整一下，就可以尝试了。

```java
http://localhost/getTgt?token=JWT&service=http://localhost

```
通过返回的ticketGrantingTicket.getId()，就可以获取ST了。

以上，如有错误，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑