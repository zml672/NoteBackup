# CAS5.3-Springboot集成客户端验证 | 字痕随行

距离今年结束还有半个月，距离春节还有一个半月，年底还有项目要上线，不知道还能更新几篇。

前面几篇说了说如何构建定制化的CAS Server，这篇开始说一说WEB项目如何使用CAS进行身份验证。

SpringMVC项目不打算单独开篇了，毕竟CAS就不是个新玩意，再搞个老古董，都2022年了，一点意义都没有。

SpringBoot集成的话，就几个关键步骤：

1. 引入Jar包。
2. yml中增加配置。
3. 增加Filter过滤器。
4. 增加Configuration配置。

**首先**，在pom文件中引入配置：

**然后**，在yml中增加配置项：

```Plain Text
cas:
  server-url-prefix: http://localhost:8443/cas
  server-login-url: http://localhost:8443/cas/login
  client-host-url: http://localhost:8080/
  validation-type: CAS

```
**再然后**，增加Filter过滤器：

```Plain Text
public class CasContextFilter implements Filter {

    public void init(FilterConfig filterConfig) throws ServletException {

    }

    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) servletRequest;
        HttpSession session = req.getSession(false);
        Assertion assertion = session != null ? (Assertion) session.getAttribute("_const_cas_assertion_") : null;
        if (null != session) {
            session.setAttribute("USER_ID_", assertion.getPrincipal().getName());
        }
        filterChain.doFilter(servletRequest, servletResponse);
    }

    public void destroy() {

    }
}

```
配置Filter，使其生效：

```Plain Text
@Bean
public FilterRegistrationBean casContextFilter() {
    FilterRegistrationBean<CasContextFilter> filterRegistrationBean = new FilterRegistrationBean<>();
    filterRegistrationBean.setFilter(new CasContextFilter());
    filterRegistrationBean.setEnabled(true);
    //拦截的路径，一般通过这个路径来做跳转处理
    filterRegistrationBean.addUrlPatterns("/sso/*");
    filterRegistrationBean.setOrder(2);
    return filterRegistrationBean;
}

```
这个过滤器其实是为了认证完毕后，将CAS提供的认证信息写入到Session中。

最关键的其实就是这句：

```Plain Text
Assertion assertion = session != null ? (Assertion) session.getAttribute("_const_cas_assertion_") : null;

```
**最后**，使用CAS的认证过滤器拦截单点登录的地址：

```Plain Text
@Value("${cas.server-url-prefix}")
private String serverUrlPrefix;

@Value("${cas.client-host-url}")
private String clientHostUrl;

@Override
public void configureAuthenticationFilter(FilterRegistrationBean authenticationFilter) {
    authenticationFilter.setFilter(new AuthenticationFilter());
    authenticationFilter.addUrlPatterns("/sso/*");
    authenticationFilter.addInitParameter("casServerLoginUrl", serverUrlPrefix);
    authenticationFilter.addInitParameter("serverName", clientHostUrl);
    authenticationFilter.setOrder(1);
}

```
一定要注意两个Filter的顺序，它们的功用是完全不同的。

做一个/sso/test来测试一下：

```Plain Text
@RestController
@RequestMapping("/sso")
public class SSOController {

    @GetMapping("test")
    public String getTest(HttpServletRequest request) {
        return "this is a test," + request.getSession().getAttribute("USER_ID_");
    }
}

```
当然需要启动之前自定义的CAS Server，输入：

```Plain Text
http://localhost/sso/test

```
就会自动跳转到：

```Plain Text
http://localhost:8443/cas/login

```
登录成功后，会自动跳回/sso/test，并且打印出：

```Plain Text
this is a test,用户名

```
![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑