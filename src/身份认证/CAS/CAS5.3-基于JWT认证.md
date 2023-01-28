# CAS5.3-基于JWT认证 | 字痕随行

周末吃了个非常好吃的自助--温野菜，强烈推荐。

基于上一章，本章将介绍CAS Server如何基于JwtToken，进行身份验证。

首先需要了解一下CAS Server如何开启JWT Token认证支持，可以参考官方指南：

```Plain Text
https://apereo.github.io/cas/5.3.x/installation/JWT-Authentication.html

```
其实，比较简单，大概的步骤如下：

1. 引入Jar包，开启支持。

2. 设置CAS Properties。

3. 在服务注册表中配置服务。

4. 在之前自定义的Handler中实现自定义认证逻辑。

基于之前的代码，在pom.xml文件中增加：

```Plain Text
<dependency>
    <groupId>org.apereo.cas</groupId>
    <artifactId>cas-server-support-token-core</artifactId>
    <version>${cas.version}</version>
</dependency>
<dependency>
    <groupId>org.apereo.cas</groupId>
    <artifactId>cas-server-support-token-webflow</artifactId>
    <version>${cas.version}</version>
</dependency>

```
在application.properties文件中增加：

```Plain Text
# JWT认证
cas.authn.token.name=token
cas.authn.token.crypto.encryptionEnabled=true
cas.authn.token.crypto.signingEnabled=true

```
新注册服务，在resources/service中新增文件Jwt-0000001.json：

```Plain Text
{
  "@class" : "org.apereo.cas.services.RegexRegisteredService",
  "serviceId" : "^(https|http|imaps)://.*",
  "name" : "JwtService",
  "id" : 1,
  "properties" : {
    "@class" : "java.util.HashMap",
    "jwtSigningSecret" : {
      "@class" : "org.apereo.cas.services.DefaultRegisteredServiceProperty",
      "values" : [ "java.util.HashSet", [ "12345678901234567890123456789012345678901" ] ]
    },
    "jwtSigningSecretAlg" : {
      "@class" : "org.apereo.cas.services.DefaultRegisteredServiceProperty",
      "values" : [ "java.util.HashSet", [ "HS256" ] ]
    }
  }
}

```
改造一下之前的UsernamePasswordCaptchaAuthenticationHandler，代码如下：

```Plain Text
public class UsernamePasswordCaptchaAuthenticationHandler extends QueryDatabaseAuthenticationHandler {

    private final String sql;

    public UsernamePasswordCaptchaAuthenticationHandler(String name, ServicesManager servicesManager, PrincipalFactory principalFactory, Integer order, DataSource dataSource, String sql, String fieldPassword, String fieldExpired, String fieldDisabled, Map<String, Object> attributes) {
        super(name, servicesManager, principalFactory, order, dataSource, sql, fieldPassword, fieldExpired, fieldDisabled, attributes);
        this.sql = sql;
    }

    @Override
    public boolean supports(Credential credential) {
        //判断传递过来的Credential 是否是自己能处理的类型
        return credential instanceof UsernamePasswordCredential || credential instanceof TokenCredential;
    }

    @Override
    protected AuthenticationHandlerExecutionResult doAuthentication(Credential credential) throws GeneralSecurityException, PreventedException {
        if (credential instanceof UsernamePasswordCaptchaCredential) {
            return doLoginRequestAuthentication(credential);
        } else if (credential instanceof TokenCredential) {
            return doTokenAuthentication(credential);
        }

        throw new FailedLoginException("没有匹配的Handler");
    }

    private AuthenticationHandlerExecutionResult doLoginRequestAuthentication(Credential credential) throws GeneralSecurityException, PreventedException {
        UsernamePasswordCaptchaCredential thisCredential = (UsernamePasswordCaptchaCredential) credential;
        //从Session中读取验证码
        String vcode;
        try {
            HttpSession httpSession = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest().getSession();
            Object object = httpSession.getAttribute(ContextConstant.VALIDAT_CODE_KEY);
            httpSession.removeAttribute(ContextConstant.VALIDAT_CODE_KEY);
            vcode = null == object ? null : object.toString();
        } catch (Exception e) {
            throw new FailedLoginException("无法获取验证码");
        }
        //比对验证码
        if (StrUtil.isNotBlank(thisCredential.getCaptcha())
                && thisCredential.getCaptcha().equals(vcode)) {
            super.doAuthentication(credential);
            return createHandlerResult(
                    thisCredential,
                    principalFactory.createPrincipal(thisCredential.getUsername(), new HashMap<>()),
                    new ArrayList<>()
            );
        }

        throw new FailedLoginException("验证码错误");
    }

    private AuthenticationHandlerExecutionResult doTokenAuthentication(Credential credential) throws FailedLoginException {
        TokenCredential tokenCredential = (TokenCredential) credential;
        Map<String, Object> dbFields = this.getJdbcTemplate().queryForMap(this.sql, tokenCredential.getId());
        if (MapUtil.isNotEmpty(dbFields)) {
            return createHandlerResult(
                    tokenCredential,
                    principalFactory.createPrincipal(tokenCredential.getId(), new HashMap<>()),
                    new ArrayList<>()
            );
        }
        throw new FailedLoginException("用户验证错误");
    }
}

```
通过Postman请求如下地址：

```Plain Text
POST /login?service=http://localhost:8081/&token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIyMDE4MTQzMiIsImlzcyI6ImNlcyJ9.-43ymRRDLKHtm5TJehVXK4kMCMgtn6IVBUSiS7HNAsA
 HTTP/1.1
Host: localhost:8080

```
会获得请求中service的ST凭据。

至于JWT的相关知识，还需要自行了解，或者后面单独开一章介绍一下。

以上，如有错误，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑