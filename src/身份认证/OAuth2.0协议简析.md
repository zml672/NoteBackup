# OAuth 2.0协议简析 | 字痕随行

OAuth2.0是一个授权协议，CAS对其有单独的模块支持，可以简单了解一下。

网络上的资料很多了，我这里就拣一些重要的说了。

本文基于文件RFC-6749介绍，这个文件的英文原版地址如下：

```html
https://tools.ietf.org/html/rfc6749
```
当然了，肯定有爱心人士翻译成中文：

```html
https://github.com/jeansfish/RFC6749.zh-cn
```
如果读者没接触过这个协议，想要了解的话，我建议英文和中文对照着看，中文便于理解，而英文更准确、更利于之后作为参照物。

好了，资料出处列举完毕，下面就是理解的关键点了。

**授权许可（Authorization Grant）**

为什么先说这个，因为整个协议其实都是围绕着这个来的，它定义了四种方式来获得授权，相当于框定了一个范围。

当然了，也可以扩展许可，相当于自定义，不过一般情况下，谁没事闲的非得自己实现一下。

这四种方式分别是：

1. 授权码（Authorization Code）
2. 隐式授权（Implicit）
3. 资源所有者密码凭据（Resource Owner Password Credentials）
4. 客户端凭据（Client Credentials）

一般来说，我们所见到的都是基于第一种方式，既然是捡重点的说，那本文就只分析第一种，其它的有时间各位可以自行了解。

**授权码许可（Authorization Code Grant）**

这是一个基于重定向的流程，客户端必须能够与资源所有者的用户代理（通常是Web浏览器）进行交互并能够接收来自授权服务器的传入请求（通过重定向）。

举个例子，微信公众号或者企业微信自建应用，通过菜单访问自建应用的时候，要获取当前用户信息，就要按照固定格式拼装一个特别长的携带参数的链接，那个链接触发后会在微信服务器、浏览器、我方服务器之间来回跳转，就是基于这种方式的。

基本的过程如下：

```Plain Text
 +----------+
 | Resource |
 |   Owner  |
 |          |
 +----------+
      ^
      |
     (B)
 +----|-----+          Client Identifier      +---------------+
 |         -+----(A)-- & Redirection URI ---->|               |
 |  User-   |                                 | Authorization |
 |  Agent  -+----(B)-- User authenticates --->|     Server    |
 |          |                                 |               |
 |         -+----(C)-- Authorization Code ---<|               |
 +-|----|---+                                 +---------------+
   |    |                                         ^      v
  (A)  (C)                                        |      |
   |    |                                         |      |
   ^    v                                         |      |
 +---------+                                      |      |
 |         |>---(D)-- Authorization Code ---------'      |
 |  Client |          & Redirection URI                  |
 |         |                                             |
 |         |<---(E)----- Access Token -------------------'
 +---------+       (w/ Optional Refresh Token)
```
**授权码许可 - 第一步 - 授权请求（Authorization Request）**

这里给了一个例子：

```Plain Text
GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
Host: server.example.com
```
这里用GET请求了authorize这个地址，其实在协议文档里面，只定死了请求这个地址时传入的参数，没有定死这个地址，所以我认为地址可以随意指定，但是一般为了方便，默认就用例子里面的了。

固定的传入参数：

* response\_type

必需的。值必须被设置为“code”。

* client\_id

必需的。如2.2节所述的客户端标识。

* redirect\_uri

可选的。如3.1.2节所述。

* scope

可选的。如3.3节所述的访问请求的范围。

* state

推荐的。客户度用于维护请求和回调之间的状态的不透明的值。当重定向用户代理回到客户端时，授权服务器包含此值。该参数应该用于防止如10.12所述的跨站点请求伪造。

以上，带章节的注释说明需要自行查看，在这不会详细说了。

这一步就是发起认证请求，发起的请求要按照指定的方式来传参，这就相当于浏览器里面输入了一个链接，会跳到目标服务器（验证服务器）的指定地址。

**授权码许可 - 第二步 - 授权响应（Authorization Response）**

这里的例子：

```Plain Text
HTTP/1.1 302 Found
Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
```
如果注意到了第一步里面的redirect\_uri这个参数，就知道这是验证服务器发起了重定向，将浏览器页面重定向到redirect\_uri。

同时，按照协议要求，携带了两个参数：

* code

必需的。授权服务器生成的授权码。授权码必须在颁发后很快过期以减小泄露风险。推荐的最长的授权码生命周期是10分钟。客户端不能使用授权码超过一次。如果一个授权码被使用一次以上，授权服务器必须拒绝该请求并应该撤销（如可能）先前发出的基于该授权码的所有令牌。授权码与客户端标识和重定向URI绑定。

* state

必需的，若“state”参数在客户端授权请求中提交。从客户端接收的精确值。

**授权码许可 - 第三步 - 访问令牌请求（Access Token Request）**

这一步，就要用第二步中的code换取access token了：

```Plain Text
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded
grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```
这一步，就不是在浏览器里面跳转做了，而是由客户端获取code之后，由客户端直接请求服务器进行验证。

像我们写的应用，就是由浏览器前端把code发送回应用后端，然后后端用java http client请求验证服务器，来获取返回的token。

这一步携带的参数如下：

* grant\_type

必需的。值必须被设置为“authorization\_code”。

* code

从授权服务器收到的授权码。

* redirect\_uri

必需的，若“redirect\_uri”参数如4.1.1节所述包含在授权请求中，且他们的值必须相同。

* client\_id

必需的，如果客户端没有如3.2.1节所述与授权服务器进行身份认证。

这一步还要获得token，所以会到最后一步。

**授权码许可 - 第四步 - 访问令牌响应（Access Token Response）**

验证成功后，验证服务器会返回一段json：

```Plain Text
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache
{
  "access_token":"2YotnFZFEjr1zCsicMWpAA",
  "token_type":"example",
  "expires_in":3600,
  "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
  "example_parameter":"example_value"
}
```
至此，可以拿着token去访问其它的资源或者数据了。

**最后**

其它的授权许可方式可以自行去看看协议文档，当然协议文档中还包含了一些其它的定义，比如名词解释、错误处理等等。本文只是帮助没接触过该协议或者一头雾水的人员入个门而已。

按照这个文档，实现四种授权许可方式，就代表支持OAuth2.0认证了。

接下来，会介绍一下，在CAS5.3中该如何支持OAuth2.0认证。

如果有问题，欢迎指正讨论。

![image](../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑