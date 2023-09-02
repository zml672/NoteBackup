# RocketMQ-事务消息 | 字痕随行

选择RocketMQ是因为它支持事务消息，它的事务消息实现过程如下：

1. 先发送一条半消息。
2. 处理业务逻辑。
3. 业务逻辑成功，则确认消息，这时候半消息会正式推送至消费者。业务逻辑失败，则回滚消息，这时候半消息会取消。
4. 因为异常情况，导致无法确认或者回滚时，利用回查接口轮询最终的业务处理结果，再确认或者回滚消息。

上面的过程都是我抄的，只不过边读边理解，然后用自己的话复述了一遍，哈哈。

接下来，进入主题，使用Spring Cloud Stream实现事务消息。

**第一步，生产者创建一个Channel，用来发送消息。**
```java
public interface TestChannel {
    @Output("output-transaction1")
    MessageChannel outputTransaction();
}

```
就是创建一个普通的Channel，特殊的地方在于yml中的配置。

**第二步，生产者新建的Channel配置。**
```Plain Text
spring:
  cloud:
    # Spring Cloud Stream 配置项，对应 BindingServiceProperties 类
    stream:
      # Binding 配置项，也没有什么特殊的
      bindings:
        output-transaction1:
          destination: test-topic
          content-type: text/plain
      # Spring Cloud Stream RocketMQ 配置项
      rocketmq:
        # RocketMQ Binder 配置项，对应 RocketMQBinderConfigurationProperties 类
        binder:
          name-server: 127.0.0.1:9876 # RocketMQ Namesrv 地址
        # RocketMQ 自定义 Binding 配置项，对应 RocketMQBindingProperties Map
        bindings:
          # 特殊的在于这里
          output-transaction1:
            # RocketMQ Producer 配置项，对应 RocketMQProducerProperties 类
            producer:
              group: test-transaction # 生产者分组
              sync: true # 是否同步发送消息，默认为 false 异步。
              transactional: true # 是否发送事务消息，默认为 false。

```
特殊的地方在于将“transactional”置为true。

**第三步，**生产者**业务逻辑处理。**

前面两步的配置，只是可以发送消息，但是真正的业务逻辑处理需要放在一个特殊的类中。
```java
@RocketMQTransactionListener(txProducerGroup = "test-transaction")
public class TransactionListenerImpl implements RocketMQLocalTransactionListener {

    private final static LogCollector LOG_COLLECTOR = LogCollector.getLogCollector(LoggerFactory.getLogger(TransactionListenerImpl.class));

    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message message, Object o) {
        String flag = message.getHeaders().getOrDefault("flag", "-1").toString();
        if ("-1".equals(flag)) {
            //消息处理异常(本地业务事务提交失败)取消发送，直接回滚
            LOG_COLLECTOR.error("消息处理异常(本地业务事务提交失败)取消发送，直接回滚");
            return RocketMQLocalTransactionState.ROLLBACK;
        } else if ("1".equals(flag)) {
            //消息处理产生了其它情况(比如文件写失败，可能有其它的保障机制)，需要回查
            LOG_COLLECTOR.warn("消息处理产生了其它情况(比如文件写失败，可能有其它的保障机制)，需要回查");
            return RocketMQLocalTransactionState.UNKNOWN;
        }
        //消息处理成功(本地业务事务提交完成)
        LOG_COLLECTOR.info("消息处理成功(本地业务事务提交完成)");
        return RocketMQLocalTransactionState.COMMIT;
    }

    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message message) {
        LOG_COLLECTOR.info(message.getPayload().toString() + "回查完成");
        return RocketMQLocalTransactionState.COMMIT;
    }
}

```
executeLocalTransaction：真正的业务逻辑放在这里，上面代码中模拟的也算比较清晰了。

checkLocalTransaction：回查的业务逻辑放在这里，一般会检查一张本地表中的日志状态，确认最终的结果。

**第四步，消费者。**

对于消费者来说，没有任何特别的地方，该怎么接收还怎么接收就可以了，参照之前的《[RocketMQ - 与Spring Cloud Stream结合](http://mp.weixin.qq.com/s?__biz=MzI3NTE2NzczMQ==&mid=2650046376&idx=1&sn=20316ae04ebe83955ddfe6b83d66636f&chksm=f3083f34c47fb62292de468c6b9fac2d61fe798aa96c000f38f2fc3a90ce5c98859d65d52300&scene=21#wechat_redirect)》。

**第五步，生产消息。**
```java
@ResponseBody
@RequestMapping(value = "sendTransaction", method = RequestMethod.GET)
public String sendTransactionMessage(@RequestParam(value = "flag", required = false) String flag) {
    String messageId = IdUtil.simpleUUID();
    Message<String> message = MessageBuilder
            .withPayload("this is a test:" + messageId)
            .setHeader(MessageConst.PROPERTY_TAGS, "testTransaction")
            .setHeader("flag", StrUtil.isNotBlank(flag) ? flag : "-1")
            .build();
    try {
        testChannel.outputTransaction().send(message);
        return messageId + "发送成功";
    } catch (Exception e) {
        LOG_COLLECTOR.error(e.getMessage(), e);
        return messageId + "发送失败，原因：" + e.getMessage();
    }
}

```
以上，RocketMQ和Spring Cloud Stream结合，实现事务消息的示例，如有错误，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑