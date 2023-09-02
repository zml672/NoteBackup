# Dubbo–Http协议实现Restful接口 | 字痕随行

每天开车望红灯，时间都被浪费在路上，无奈。

本章记录一下如何使用Http协议实现Restful接口，如果想了解Rest协议的实现，可以看这里：[Dubbo - Rest协议实现Restful接口](http://www.blackzs.com/archives/1666)。

本示例基于：
```text
Spring Boot 2.2.5.RELEASE

Spring Cloud vHoxton.SR3

Spring Cloud Alibaba 2.2.0.RELEASE
```

provider的pom文件引入Jar包如下：
```xml
<dependencies>

    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jetty</artifactId>
    </dependency>

    <!--服务注册与发现-->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-dubbo</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!--对外接口-->
    <dependency>
        <groupId>com.xxxx</groupId>
        <artifactId>xxxx-examples-service-api</artifactId>
    </dependency>

</dependencies>

```
application.yml配置如下：
```yml
server:
  port: 8081
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/xml,text/plain
dubbo:
  scan:
    base-packages: com.xxxx.examples.service.http.provider
  protocols:
    dubbo:
      name: dubbo
      port: -1
  registry:
    address: spring-cloud://localhost

spring:
  http:
    encoding:
      charset: UTF-8
  application:
    name: xxxx-examples-service-http-provider
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
  main:
    allow-bean-definition-overriding: true

```
测试用的对外接口实现类如下：
```java
@RestController
@RequestMapping(("/example/service/test"))
@Service(protocol = {"dubbo"})
public class TestApiServiceImpl implements TestApiService {

    @Override
    @GetMapping("echo")
    public String echoString() {
        return "hello, welcome";
    }

    @Override
    @GetMapping("echo/{id}")
    public String echoString(@PathVariable String id) {
        return "you input: " + id;
    }
}

```
启动类如下：
```java
@SpringBootApplication
@EnableDiscoveryClient
public class ExampleServiceHttpProviderApp
{
    public static void main( String[] args )
    {
        SpringApplication.run(ExampleServiceHttpProviderApp.class, args);
    }
}

```
consumer的pom文件如下：
```xml
<dependencies>

    <!-- spring boot web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jetty</artifactId>
    </dependency>

    <!--服务注册与发现-->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-dubbo</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!--对外接口-->
    <dependency>
        <groupId>com.xxxx</groupId>
        <artifactId>xxxx-examples-service-api</artifactId>
    </dependency>

</dependencies>

```
接口的调用方式和上一篇一样：
```java
//注意LoadBalanced
@Configuration
public class RestConfig {
    @Bean
    @LoadBalanced
    @DubboTransported
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
@RestController
@RequestMapping("test")
public class TestController {

    @Reference
    private TestApiService testApiService;

    @Resource
    private RestTemplate restTemplate;

    @GetMapping("/rpc/echo")
    public String echoRpc() {
        String echoString = testApiService.echoString();
        return echoString;
    }

    @GetMapping("/rest/echo")
    public String echoRest() {
        try {
            //注意使用服务名称调用
            String echoString = restTemplate.getForObject("http://xxx-examples-service-http-provider/example/service/test/echo", String.class, "1");
            return echoString;
        } catch (Exception e) {
            return null;
        }
    }
}

```
启动类也是一样的：
```java
@SpringBootApplication
@EnableDiscoveryClient
public class ExampleServiceConsumerApp
{
    public static void main( String[] args )
    {
        SpringApplication.run(ExampleServiceConsumerApp.class, args);
    }
}

```

以上，如果有错误，欢迎探讨和指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑