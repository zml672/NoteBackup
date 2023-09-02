# CAS5.3-Nginx转发OAuth2.0请求导致验证失败的问题分析 | 字痕随行

先说说环境吧：

* CAS在Nginx后面，所有的请求经由Nginx转发。
* CAS开启了OAuth2.0模块

其它的都是正常配置了，但是当OAuth2.0请求验证时，总会跳转到一个内网地址，就很纳闷为什么。

最后，只能通过分析源码来寻找原因了。

通过浏览器的开发助手，我们知道卡住的位置在callbackAuthorize这个地址，它的302重定向会到一个内网地址，响应头如下：
```Plain Text
cache-control: no-cache, no-store, max-age=0, must-revalidate
content-language: zh-CN
content-length: 0
date: Thu, 09 Mar 2023 05:41:13 GMT
expires: 0
location: http://192.168.1.1:39999/oauth2.0/authorize?client_id=cliendId&redirect_uri=callback地址&response_type=code&state=uuid
pragma: no-cache
server: nginx/1.x.x
x-content-type-options: nosniff
x-frame-options: DENY
x-xss-protection: 1; mode=block
```
按道理都有redirect地址，怎么都不会跳转到一个内网的地址，尤其只是域名更换了，后面的地址链接还都是正确的。

直接看callbackAuthorize这个地址的处理逻辑就好了，为什么会response这么一个location出来？下面是该controller的代码：
```java
@GetMapping(path = OAuth20Constants.BASE_OAUTH20_URL + '/' + OAuth20Constants.CALLBACK_AUTHORIZE_URL)
public ModelAndView handleRequest(final HttpServletRequest request, final HttpServletResponse response) {
    final J2EContext context = new J2EContext(request, response, this.oauthConfig.getSessionStore());
    final DefaultCallbackLogic callback = new DefaultCallbackLogic();
    //直接看这个方法，这方法决定了response的header
    callback.perform(context, oauthConfig, J2ENopHttpActionAdapter.INSTANCE,
        null, Boolean.TRUE, Boolean.FALSE,
        Boolean.FALSE, Authenticators.CAS_OAUTH_CLIENT);
    final String url = StringUtils.remove(response.getHeader("Location"), "redirect:");
    final ProfileManager manager = Pac4jUtils.getPac4jProfileManager(request, response);
    return oAuth20CallbackAuthorizeViewResolver.resolve(context, manager, url);
}
```
然后就是Pac4j的内部代码了：
```java
protected HttpAction redirectToOriginallyRequestedUrl(final C context, final String defaultUrl) {
    final String requestedUrl = (String) context.getSessionStore().get(context, Pac4jConstants.REQUESTED_URL);
    String redirectUrl = defaultUrl;
    if (isNotBlank(requestedUrl)) {
        context.getSessionStore().set(context, Pac4jConstants.REQUESTED_URL, null);
        redirectUrl = requestedUrl;
    }
    logger.debug("redirectUrl: {}", redirectUrl);
    return HttpAction.redirect(context, redirectUrl);
}
```
注意上面代码中的requestedUrl，是从sessionStore里面取出的。那下一步就很明确了，这url是怎么存进去的。

找这个url的思路如下：

* 身份验证肯定有个验证的方法，所以这段验证逻辑在哪，会导致它要转到login路径。
* Pac4j是个登录验证的封装工具。

所以，就是想一想，就是要找登录验证的地方，就找到了下面这段代码：
```java
@ConditionalOnMissingBean(name = "requiresAuthenticationAuthorizeInterceptor")
@Bean
public SecurityInterceptor requiresAuthenticationAuthorizeInterceptor() {
    return new SecurityInterceptor(oauthSecConfig.getIfAvailable(), Authenticators.CAS_OAUTH_CLIENT);
}
```
继续找到preHandle，就会找到RequestUrl是怎么保存进来的：
```java
protected void saveRequestedUrl(final C context, final List<Client> currentClients) {
    if (ajaxRequestResolver == null || !ajaxRequestResolver.isAjax(context)) {
        //就这里了，怎么获取RequestURL的
        final String requestedUrl = context.getFullRequestURL();
        logger.debug("requestedUrl: {}", requestedUrl);
        context.getSessionStore().set(context, Pac4jConstants.REQUESTED_URL, requestedUrl);
    }
}
```
然后，就看到了request.getRequestURL()。

当时我就给跪了，要是Nginx转发的，那不就是个内网地址，真实的地址都放到header了。

原始代码是真的不想改了，只能想想别的变通方法了，不行就把Nginx去了吧。

以上，如果有错误，欢迎探讨和指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑