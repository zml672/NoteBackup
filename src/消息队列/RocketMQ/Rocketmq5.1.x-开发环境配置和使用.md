这篇文章源自我想试验一下：一个Flowable应用作为流程模型发布端，其它应用接收到发布请求后，在本地进行部署。

这中间消息队列就是必不可少的组件，正好Rocketmq的最新版本也已经到5了，索性一起试验一下。

## 部署

很早之前有一篇文章介绍过如何在本地部署Rocketmq，当时的版本还是4.8.0。部署5.1.x时大部分的坑和关键点还是一样的，照着配置一下就可以了。

最大的不同在于5.1.x增加了Proxy，所以除了要像之前启动Namesrv、Broker之外，还要启动Proxy。

如果没有启动Proxy，又按照下文的方式生成消息，就会报出gRpc异常。

这里给出常用的启动命令以供参考：

```shell
# 启动namesrv，默认情况下，会占用9876端口
.\mqnamesrv.cmd

# 启动broker
.\mqbroker.cmd -n 127.0.0.1:9876 autoCreateTopicEnable=true

# 启动proxy，proxy默认占用8081端口
 .\mqproxy -n 127.0.0.1:9876 -pc ..\conf\rmq-proxy.json

```
## Java客户端

此处以SpringBoot集成为例，starter的仓库地址如下：
```
https://github.com/apache/rocketmq-spring
```

这个仓库最后一次release版本是2.2.3，并不支持Rocketmq5。如果只是想为了测试一把比较方便的话，可以使用snapshot版本。

使用的方式是：
1. 把master分支pull到本地。
2. 使用Maven install命令，把v5部分安装到本地仓库。
3. 可以使用Maven引入了。

最后引入的方式如下：
```xml
<dependency>  
    <groupId>org.apache.rocketmq</groupId>  
    <artifactId>rocketmq-v5-client-spring-boot-starter</artifactId>
    <version>2.2.4-SNAPSHOT</version>
</dependency>
```

## 示例代码

### 生产者

application.properties：
```
# rocketmq  
rocketmq.producer.endpoints=localhost:8081  
rocketmq.producer.topic=normalTopic
```
发送消息：
```java
@RestController  
@RequestMapping("test")  
public class TestController {  
  
    @Resource  
    private RocketMQClientTemplate rocketMQClientTemplate;  
  
    @GetMapping("pushMessage")  
    public void pushMessage() {  
        SendReceipt sendReceipt = rocketMQClientTemplate.syncSendNormalMessage(  
                "normalTopic",  
                new HashMap<String, String>(){{  
                    put("name", "test");  
                }});  
        System.out.printf("normalSend to topic %s sendReceipt=%s %n", "normalTopic", sendReceipt);  
    }  
}

```
### 消费者

application.properties：
```
# rocketmq  
rocketmq.simple-consumer.endpoints=localhost:8081  
rocketmq.simple-consumer.consumer-group=normalGroup  
rocketmq.simple-consumer.topic=normalTopic  
rocketmq.simple-consumer.tag=*  
rocketmq.simple-consumer.filter-expression-type=tag

```
消费消息：
```java
@Service  
@RocketMQMessageListener(  
        endpoints = "${rocketmq.simple-consumer.endpoints:}",  
        topic = "${rocketmq.simple-consumer.topic:}",  
        consumerGroup = "${rocketmq.simple-consumer.consumer-group:}",  
        tag = "${rocketmq.simple-consumer.tag:}")  
public class DefaultMQListener implements RocketMQListener {  
  
    @Override  
    public ConsumeResult consume(MessageView messageView) {  
        System.out.println("handle my message:" + messageView);  
        String msgBody = Charset.defaultCharset().decode(messageView.getBody()).toString();  
        System.out.println("message body:" + msgBody);  
        return ConsumeResult.SUCCESS;  
    }  
}
```

