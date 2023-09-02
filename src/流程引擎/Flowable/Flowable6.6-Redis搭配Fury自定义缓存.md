# Flowable6.6-Redis搭配Fury自定义缓存 | 字痕随行

之前通过一些简单的代码试验过序列/反序列化ProcessDefinitionCacheEntry，这次就使用Redis完整的试验一下。

选定的序列/反序列化工具为Fury，Redis版本是x64-5.0.14.1。

首先，在工程内引入Fury。

```xml
<dependency>
    <groupId>org.furyio</groupId>
    <artifactId>fury-core</artifactId>
    <version>0.1.0-alpha.1</version>
</dependency>
```
然后，创建一个RedisSerializer的实现类。

```java
public class FurySerializer<T> implements RedisSerializer<T> {
    //使用线程安全模式
    ThreadSafeFury fury = Fury.builder()
            .withLanguage(Language.JAVA)
            // enable referecne tracking for shared/circular reference.
            // Disable it will have better performance if no duplciate reference.
            .withRefTracking(true)
            // Allow to deserialize objects unknown types,
            // more flexible but less secure.
            .withSecureMode(false)
            .buildThreadSafeFury();

    @Override
    public byte[] serialize(T t) throws SerializationException {
        return fury.serialize(t);
    }

    @Override
    public T deserialize(byte[] bytes) throws SerializationException {
        if (null == bytes) {
            return null;
        }
        return (T) fury.deserialize(bytes);
    }
}
```
再创建Redis的配置类，这里只展示主要的代码。

```java
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {

    /**
     * retemplate相关配置
     */
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {

        RedisTemplate<String, Object> template = new RedisTemplate<>();
        // 配置连接工厂
        template.setConnectionFactory(factory);

        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new FurySerializer<>());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new FurySerializer<>());

        return template;
    }
    
    //这里省略了一些数据操作配置
}
```
自定义一个使用Redis的缓存类。

```java
public class ProcessDefinitionRedisCache implements DeploymentCache<ProcessDefinitionCacheEntry> {

    @Override
    public ProcessDefinitionCacheEntry get(String id) {
        //BeanFactory是自定义的，可以帮助取出Spring容器内的对象
        RedisTemplate<String, Object> redisTemplate = BeanFactory.getBean("redisTemplate", RedisTemplate.class);
        if (null == redisTemplate) {
            return null;
        }
        Object obj = redisTemplate.opsForValue().get(id);
        if (null == obj) {
            return null;
        }
        ProcessDefinitionCacheEntry cacheEntry = (ProcessDefinitionCacheEntry) redisTemplate.opsForValue().get(id);

        return cacheEntry;
    }

    @Override
    public void add(String id, ProcessDefinitionCacheEntry processDefinitionCacheEntry) {
        RedisTemplate<String, Object> redisTemplate = BeanFactory.getBean("redisTemplate", RedisTemplate.class);
        if (null != redisTemplate) {
            redisTemplate.opsForValue().set(id, processDefinitionCacheEntry);
        }
    }
```
Flowable的配置类内，需要使用自定义的缓存。

```java
@Primary
@Bean(name = "processEngineConfiguration")
public SpringProcessEngineConfiguration getSpringProcessEngineConfiguration(@Qualifier("dataSource") DataSource dataSource, @Qualifier("transactionManager") DataSourceTransactionManager transactionManager) {
    SpringProcessEngineConfiguration configuration = new SpringProcessEngineConfiguration();
    //此处省略一大堆配置
    //使用自定义的缓存
    configuration.setProcessDefinitionCache(new ProcessDefinitionRedisCache());
    return configuration;
}
```
最后，可以直接启动一个流程跑一下。我跑了一个比较简单的流程，没有异常产生。

以上，如果有错误，欢迎探讨和指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑