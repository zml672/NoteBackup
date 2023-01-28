# CAS5.3-支持OAuth2.0 | 字痕随行

CAS现在的版本已经是6.6.x了，它的在线文档也没有5.3.x版本的了，如果还想看官方文档的话，只能去它的开源仓库拉取下来，在项目中有个docs文件夹，这里面可以看到它在这个版本的文档。

现在，言归正传，简单说一下CAS5.3版本如何支持OAuth2.0验证。

**第一步，打开官方文档。**

官方文档说的很清楚，但是也说的不清楚。

在阅读官方文档之前，非常建议了解一下OAuth2.0协议，哪怕是百度找一篇速成一下，也会让你接下来的工作事半功倍。

官方文档的相对地址是：

```Plain Text
\docs\cas-server-documentation\installation\OAuth-OpenId-Authentication.md
```
提炼总结一下，它大概分了四个部分：

1. 引入Maven的坐标文件，支撑该功能实现。
2. 说明OAuth2.0所要求的的请求路径，并且详述每种授权许可所需的参数要求。
3. 如何注册客户端，即Service的Json配置文件。
4. 返回的Profile Json结构，以及如何自定义返回。

**第二步，引入Maven的坐标地址。**

在pom.xml文件中增加以下配置即可：

```xml
<dependency>
  <groupId>org.apereo.cas</groupId>
  <artifactId>cas-server-support-oauth-webflow</artifactId>
  <version>${cas.version}</version>
</dependency>
```
**第三步，增加Service的Json配置文件。**

官方文档中的Register Clients中已经给出了文件结构，并且说明了其中配置项的意义：

```json
{
  "@class" : "org.apereo.cas.support.oauth.services.OAuthRegisteredService",
  "clientId": "clientid",
  "clientSecret": "clientSecret",
  "serviceId" : "^(https|imaps)://<redirect-uri>.*",
  "name" : "OAuthService",
  "id" : 100,
  "supportedGrantTypes": [ "java.util.HashSet", [ "...", "..." ] ],
  "supportedResponseTypes": [ "java.util.HashSet", [ "...", "..." ] ]
}
```
但是以上还不够，所以这里有一些细节额外注意一下：

1. Service配置文件的名称必须是name+"-"+id。
2. serviceId的值是个正则表达式，在验证传入的redirect\_uri是否合法时会用到。所以上面的配置示例才会是“<redirect-uri>.\*”，这<redirect-uri>其实是client的域名。
3. 有一些配置没列在这里，比如：
   1. bypassApprovalPrompt：是否显示资源确认授权那个页面，如果配置成true，验证成功后就会直接跳转到redirect\_uri。不配置或者配置成false，会有一个确认页面等着用户手动点一下。
   2. jsonFormat：不配置或者配置成false，返回的是个字符串。如果配置成true，会返回成Json格式。

**第四步，增加配置。**

在properties增加以下配置：

```java
#OAuth2.0
cas.authn.oauth.refreshToken.timeToKillInSeconds=2592000
cas.authn.oauth.code.timeToKillInSeconds=30
cas.authn.oauth.code.numberOfUses=1
cas.authn.oauth.accessToken.releaseProtocolAttributes=true
cas.authn.oauth.accessToken.timeToKillInSeconds=7200
cas.authn.oauth.accessToken.maxTimeToLiveInSeconds=28800
#决定了profile返回内容的格式，NESTED、FLAT、CUSTOM
cas.authn.oauth.userProfileViewType=NESTED
#让CAS知道自己的地址是什么，以便跳转识别
cas.server.name=http://localhost:8080
cas.server.prefix=http://localhost:8080/cas
```
**第五步，测试一下。**

使用Authorization Code模式，先访问以下地址：

```html
/oauth2.0/authorize?response_type=code&client_id=<ID>&redirect_uri=<CALLBACK>
GET
```
如果在CAS没登录过，会直接先跳到登录界面，验证成功后，会跳转回<CALLBACK>地址，并携带着code参数，如下：

```html
<CALLBACK>?code=CODE
302重定向
```
获得CODE后，使用post请求token的地址，可以获取到access token：

```html
/oauth2.0/accessToken?grant_type=authorization_code&client_id=ID&client_secret=SECRET&code=CODE&redirect_uri=CALLBACK
POST
```
该地址会返回token，携带token访问以下地址可以获取profile：

```html
/oauth2.0/profile?access_token=<TOKEN>
```
该地址会返回人员的JSON格式信息：

```json
Nested配置时：
{
  "id": "casuser",
  "attributes": {
    "email": "casuser@example.org",
    "name": "CAS"
  },
  "something": "else"
}
FLAT配置时：
{
  "id": "casuser",
  "email": "casuser@example.org",
  "name": "CAS",
  "something": "else"
}
```
**最后，自定义。**

这个比较简单，按照官方文档增加一个配置类即可：

```java
package org.apereo.cas.support.oauth;

@Configuration("MyOAuthConfiguration")
@EnableConfigurationProperties(CasConfigurationProperties.class)
public class MyOAuthConfiguration {

    @Bean
    @RefreshScope
    public OAuth20UserProfileViewRenderer oauthUserProfileViewRenderer() {
        ...
    }
}
```
只是需要注意的是，这个配置类是全局的，如果client有多个，需要的profile格式有所不同，那就需要在其内部按照acessToken所属的service进行判断，然后返回各自所需的信息。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑