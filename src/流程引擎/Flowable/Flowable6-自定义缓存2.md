# Flowable6 - 自定义缓存(2) | 字痕随行
原创 字痕随行 字痕随行

收录于话题

#流程引擎

53个

接上一篇的自定义缓存，这次具体说说如何自定义缓存，以及使用了Redis后，我是如何能够让它正常运行。

**首先**，自定义缓存需要实现一个接口，代码如下：

```Java
public class ProcessDefinitionRedisCache implements DeploymentCache<ProcessDefinitionCacheEntry> {

    @Override
    public ProcessDefinitionCacheEntry get(String id) {
    }

    @Override
    public boolean contains(String id) {
    }

    @Override
    public void add(String id, ProcessDefinitionCacheEntry processDefinitionCacheEntry) {
    }

    @Override
    public void remove(String id) {
    }

    @Override
    public void clear() {
    }

    @Override
    public Collection<ProcessDefinitionCacheEntry> getAll() {
    }

    @Override
    public int size() {
    }
}

```
**然后**，需要使用自定义的缓存替代原生的缓存：

```Java
configuration.setProcessDefinitionCache(new ProcessDefinitionRedisCache());

```
上一篇说过，即使实现了这个接口，但是由于BpmnModel和Process不能简单的序列化和反序列化，导致无法正常运行。

我试了很多办法，最后只能每次取缓存的时候调用ParsedDeployment来实现。

**首先**，加入缓存的时候，把Deployment加入缓存，因为Deployment支持序列化，并且可以一同将流程定义的XML加入缓存，就像这样：

```Java
@Override
public void add(String id, ProcessDefinitionCacheEntry processDefinitionCacheEntry) {
    redisTemplate.opsForValue().set(id + "_def", processDefinitionCacheEntry.getProcessDefinition());

    DeploymentEntity deployment = CommandContextUtil.getDeploymentEntityManager().findById(processDefinitionCacheEntry.getProcessDefinition().getDeploymentId());
    redisTemplate.opsForValue().set(id + "_dly", deployment);
}

```
**然后**，在取出缓存的时候，调用ParsedDeployment来获得BpmnModel和Process。

```Java
@Override
public ProcessDefinitionCacheEntry get(String id) {
    Object objectDef = redisTemplate.opsForValue().get(id + "_def");
    JSONObject jsonDef = (JSONObject) objectDef;
    Object objectDeployment = redisTemplate.opsForValue().get(id + "_dly");
    JSONObject jsonDeployment = (JSONObject) objectDeployment;

    ProcessDefinitionEntity processDefinitionEntity = jsonDef.toJavaObject(ProcessDefinitionEntityImpl.class);

    DeploymentEntity deployment = jsonDeployment.toJavaObject(DeploymentEntityImpl.class);

    ParsedDeploymentBuilderFactory factory = CommandContextUtil.getProcessEngineConfiguration().getParsedDeploymentBuilderFactory();
    ParsedDeployment parsedDeployment = factory
            .getBuilderForDeploymentAndSettings(deployment, null)
            .build();

    BpmnModel bpmnModel = parsedDeployment.getBpmnModelForProcessDefinition(parsedDeployment.getAllProcessDefinitions().get(0));
    Process process = parsedDeployment.getProcessModelForProcessDefinition(parsedDeployment.getAllProcessDefinitions().get(0));
    ProcessDefinitionCacheEntry cacheEntry = new ProcessDefinitionCacheEntry(processDefinitionEntity, bpmnModel, process);

    return cacheEntry;
}

```
这样的话，能够保证BpmnModel和Process是完整的。我测试了一下，倒是没发现什么问题。

但是，我**不建议**这么做，因为每一次都相当于重新解析XML，相当于重新执行流程发布时的耗时操作，失去了缓存的意义。

这一篇我觉得只是个总结，没有什么实际应用的价值，如有错误，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑