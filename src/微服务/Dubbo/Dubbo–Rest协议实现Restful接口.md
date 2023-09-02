# Dubbo–Rest协议实现Restful接口 | 字痕随行

最近做了一些Dubbo的示例，在此记录一下。

本章记录一下使用Rest协议实现Http Restful接口。

本示例基于：
```text
Spring Boot 2.2.5.RELEASE

Spring Cloud vHoxton.SR3

Spring Cloud Alibaba 2.2.0.RELEASE

Netty 4.1.42.final

Resteasy 3.11.0.Final
```

**provider**的pom文件引入Jar包如下：
```xml
<dependencies>

    <!--单元测试-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!--服务注册与发现-->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-dubbo</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-nacos-discovery</artifactId>
        <exclusions>
            <!-- 过滤掉jsr311,防止与rs-api冲突,导致缺少method -->
            <exclusion>
                <groupId>javax.ws.rs</groupId>
                <artifactId>jsr311-api</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

    <!-- Dubbo REST support dependencies -->
    <dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-all</artifactId>
    </dependency>
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
    </dependency>
    <dependency>
        <groupId>org.jboss.resteasy</groupId>
        <artifactId>resteasy-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.jboss.resteasy</groupId>
        <artifactId>resteasy-netty4</artifactId>
    </dependency>
    <dependency>
        <groupId>javax.validation</groupId>
        <artifactId>validation-api</artifactId>
    </dependency>
    <dependency>
        <groupId>org.hibernate.validator</groupId>
        <artifactId>hibernate-validator</artifactId>
    </dependency>

    <!--对外接口-->
    <dependency>
        <groupId>com.xxx</groupId>
        <artifactId>xxx-examples-service-api</artifactId>
    </dependency>
</dependencies>

```
application.yml配置如下：
```yml
dubbo:
  scan:
    base-packages: com.xxx.examples.service.rest.provider
  protocols:
    dubbo:
      name: dubbo
      port: -1
    rest:
      name: rest
      port: 8888
      server: netty
  registry:
    address: spring-cloud://localhost

spring:
  application:
    name: xxx-examples-service-rest-provider
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
  main:
    allow-bean-definition-overriding: true

```
测试用的对外接口实现类如下：
```java
/**
 * 服务示例对外测试接口实现
 */
@Path("/example/service/test")
@Service(protocol = {"dubbo", "rest"})
public class TestApiServiceImpl implements TestApiService {

    @GET
    @Path("echo")
    @Override
    public String echoString() {
        return "hello, welcome";
    }

    @GET
    @Path("echo/{id}")
    @Override
    public String echoString(@PathParam("id") String id) {
        return "you input: " + id;
    }
}

```
启动类如下：
```java
/**
 * 服务示例对外测试接口实现
 */
@SpringBootApplication
@EnableDiscoveryClient
public class ExampleServiceRestProviderApp
{
    public static void main( String[] args )
    {
        new SpringApplicationBuilder(ExampleServiceRestProviderApp.class)
                .web(WebApplicationType.NONE)
                .run(args);
    }
}

```
**consumer**的pom文件如下：
```xml
<dependencies>
    <!--单元测试-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

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
        <exclusions>
            <!-- 过滤掉jsr311,防止与rs-api冲突,导致缺少method -->
            <exclusion>
                <groupId>javax.ws.rs</groupId>
                <artifactId>jsr311-api</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

    <!-- Dubbo REST support dependencies -->
    <dependency>
        <groupId>org.jboss.resteasy</groupId>
        <artifactId>resteasy-client</artifactId>
    </dependency>
    <dependency>
        <groupId>javax.validation</groupId>
        <artifactId>validation-api</artifactId>
    </dependency>
    <dependency>
        <groupId>org.hibernate.validator</groupId>
        <artifactId>hibernate-validator</artifactId>
    </dependency>

    <!--对外接口-->
    <dependency>
        <groupId>com.xxx</groupId>
        <artifactId>xxx-examples-service-api</artifactId>
    </dependency>
</dependencies>

```
application.yml配置如下：
```yml
server:
  port: 8082
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/xml,text/plain
dubbo:
  registry:
    address: spring-cloud://localhost
  cloud:
    # The subscribed services in consumer side
    subscribed-services: xxx-examples-service-rest-provider

spring:
  http:
    encoding:
      charset: UTF-8
  application:
    name: xxx-examples-service-consumer
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
  main:
    allow-bean-definition-overriding: true

```
接口调用如下：
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

```
```java
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
            String echoString = restTemplate.getForObject("http://xxx-examples-service-rest-provider/example/service/test/echo", String.class, "1");
            return echoString;
        } catch (Exception e) {
            return null;
        }
    }
}

```
启动类如下：
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