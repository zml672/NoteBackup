# Flowable6.6-再试自定义缓存 | 字痕随行

很久之前，我写过两篇文章来介绍Flowable的自定义缓存，然后完败。

这次再折腾这个，起因是群里的大佬告诉我可以用编辑字节码的方式来实现，我就寻了个空又重新折腾了一下。

最开始尝试了字节码的方式搞不定，实在没时间折腾了，所以使用笨办法绕了绕，在这期间遇到的坑点如下：

1. 官方在一些属性上标识了@JsonIgnore，但是反序列化之后的对象缺需要这些属性不为Null。
2. 某些Behavior没有默认的构造函数，导致反序列化失败，比如UserTaskActivityBehavior。
3. 有很多循环引用，比如FlowElement，循环引用直接导致堆栈溢出。
4. getter、setter不规范，有的包含了业务逻辑，返回的类型和属性类型不一致。
5. 官方用Object类型来恶心人。

所谓的笨办法：

1. 把标识了@JsonIgnore但是却需要用到属性单独处理get、set。
2. 重新定义一个子类，给子类定义默认构造函数，使用子类替代父类。
3. 一般循环引用的官方都给了@JsonIgnore，所以使用Jackson来处理序列化和反序列化。
4. 不要使用getter、setter来进行序列化，只使用原始属性。
5. 对Object类型进行特殊处理。

关键的代码：

```java
public class ProcessDefinitionRedisCache implements DeploymentCache<ProcessDefinitionCacheEntry> {

    @Override
    public ProcessDefinitionCacheEntry get(String id) {
        RedisTemplate<String, Object> redisTemplate = BeanFactory.getBean("redisTemplate", RedisTemplate.class);
        if (null == redisTemplate) {
            return null;
        }
        Object objectDef = redisTemplate.opsForValue().get(id + "_def");
        Object objectProc = redisTemplate.opsForValue().get(id + "_proc");
        Object objectModel = redisTemplate.opsForValue().get(id + "_model");
        Object objectSupport = redisTemplate.opsForValue().get(id + "_support");
        Object objectBehavior = redisTemplate.opsForValue().get(id + "_behavior");
        Object objectSource = redisTemplate.opsForValue().get(id + "_source");
        Object objectTarget = redisTemplate.opsForValue().get(id + "_target");
        if (null == objectDef || null == objectProc
                || null == objectModel || null == objectSupport
                || null == objectBehavior || null == objectSource
                || null == objectTarget) {
            return null;
        }

        ProcessDefinitionEntity processDefinitionEntity = (ProcessDefinitionEntity) objectDef;
        BpmnModel bpmnModel = (BpmnModel) objectModel;
        Process process = (Process) objectProc;
        FlowableEventSupport eventSupport = (FlowableEventSupport) objectSupport;
        bpmnModel.setEventSupport(eventSupport);
        //处理循环引用那些属性
        setOther(objectBehavior, objectSource, objectTarget, process, bpmnModel);

        ProcessDefinitionCacheEntry cacheEntry = new ProcessDefinitionCacheEntry(processDefinitionEntity, bpmnModel, process);

        return cacheEntry;
    }

    @Override
    public boolean contains(String id) {
       
    }

    @Override
    public void add(String id, ProcessDefinitionCacheEntry processDefinitionCacheEntry) {
        RedisTemplate<String, Object> redisTemplate = BeanFactory.getBean("redisTemplate", RedisTemplate.class);
        if (null != redisTemplate) {
            redisTemplate.opsForValue().set(id + "_def", processDefinitionCacheEntry.getProcessDefinition());
            redisTemplate.opsForValue().set(id + "_proc", processDefinitionCacheEntry.getProcess());
            redisTemplate.opsForValue().set(id + "_model", processDefinitionCacheEntry.getBpmnModel());
            //下面都是需要特殊处理的属性
            FlowableEventSupport eventSupport = (FlowableEventSupport) processDefinitionCacheEntry.getBpmnModel().getEventSupport();
            redisTemplate.opsForValue().set(id + "_support", eventSupport);

            Map<String, Object> mapBehavior = new HashMap<>();
            Map<String, FlowElement> mapSource = new HashMap<>();
            Map<String, FlowElement> mapTarget = new HashMap<>();
            for (FlowElement flowElement : processDefinitionCacheEntry.getProcess().getFlowElements()) {
                if (flowElement instanceof FlowNode) {
                    FlowNode flowNode = (FlowNode) flowElement;
                    mapBehavior.put(flowElement.getId(), flowNode.getBehavior());
                }
                if (flowElement instanceof SequenceFlow) {
                    SequenceFlow sequenceFlow = (SequenceFlow) flowElement;
                    mapSource.put(flowElement.getId(), sequenceFlow.getSourceFlowElement());
                    mapTarget.put(flowElement.getId(), sequenceFlow.getTargetFlowElement());
                }
            }
            redisTemplate.opsForValue().set(id + "_behavior", mapBehavior);
            redisTemplate.opsForValue().set(id + "_source", mapSource);
            redisTemplate.opsForValue().set(id + "_target", mapTarget);
        }
    }

    @Override
    public void remove(String id) {
       
    }

    @Override
    public void clear() {
        
    }

    @Override
    public Collection<ProcessDefinitionCacheEntry> getAll() {
        
    }

    @Override
    public int size() {
        
    }

    /**
     * 要递归将循环引用的那些属性赋值，这是最麻烦的地方
     * 测试的是一个简单的流程，所以只有这些节点，复杂点的我也不知道行不行啊
    */
    private void setOther(Object objectBehavior, Object objectSource, Object objectTarget,
                          Process process, BpmnModel bpmnModel) {
        Map<String, Object> mapBehavior = (Map<String, Object>) objectBehavior;
        Map<String, FlowElement> mapSource = (Map<String, FlowElement>) objectSource;
        Map<String, FlowElement> mapTarget = (Map<String, FlowElement>) objectTarget;
        for (FlowElement flowElement : process.getFlowElements()) {
            setFlowElement(flowElement, mapBehavior, mapSource, mapTarget);
        }
        for (Process proc : bpmnModel.getProcesses()) {
            for (FlowElement flowElement : proc.getFlowElements()) {
                setFlowElement(flowElement, mapBehavior, mapSource, mapTarget);
            }
        }
        for (FlowElement flowElement : process.getFlowElementMap().values()) {
            setFlowElement(flowElement, mapBehavior, mapSource, mapTarget);
        }
        for (Process proc : bpmnModel.getProcesses()) {
            for (FlowElement flowElement : proc.getFlowElementMap().values()) {
                setFlowElement(flowElement, mapBehavior, mapSource, mapTarget);
            }
        }
        FlowNode flowNode = (FlowNode) process.getInitialFlowElement();
        flowNode.setBehavior(mapBehavior.get(flowNode.getId()));
        setFlowElement(flowNode, mapBehavior, mapSource, mapTarget);
    }

    private void setFlowElement(FlowElement flowElement,
                                Map<String, Object> mapBehavior,
                                Map<String, FlowElement> mapSource,
                                Map<String, FlowElement> mapTarget) {
        if (flowElement instanceof FlowNode) {
            FlowNode flowNode = (FlowNode) flowElement;
            flowNode.setBehavior(mapBehavior.get(flowElement.getId()));
        }
        if (flowElement instanceof StartEvent) {
            StartEvent startEvent = (StartEvent) flowElement;
            for (SequenceFlow sequenceFlow : startEvent.getOutgoingFlows()) {
                FlowElement source = mapSource.get(sequenceFlow.getId());
                FlowElement target = mapTarget.get(sequenceFlow.getId());
                sequenceFlow.setSourceFlowElement(source);
                sequenceFlow.setTargetFlowElement(target);
                if (null != target) {
                    setFlowElement(target, mapBehavior, mapSource, mapTarget);
                }
            }
        }
        if (flowElement instanceof UserTask) {
            UserTask userTask = (UserTask) flowElement;
            for (SequenceFlow sequenceFlow : userTask.getOutgoingFlows()) {
                FlowElement source = mapSource.get(sequenceFlow.getId());
                FlowElement target = mapTarget.get(sequenceFlow.getId());
                sequenceFlow.setSourceFlowElement(source);
                sequenceFlow.setTargetFlowElement(target);
                if (null != target) {
                    setFlowElement(target, mapBehavior, mapSource, mapTarget);
                }
            }
        }
        if (flowElement instanceof SequenceFlow) {
            SequenceFlow sequenceFlow = (SequenceFlow) flowElement;
            FlowElement source = mapSource.get(sequenceFlow.getId());
            FlowElement target = mapTarget.get(sequenceFlow.getId());
            sequenceFlow.setSourceFlowElement(source);
            sequenceFlow.setTargetFlowElement(target);
            if (null != target) {
                setFlowElement(target, mapBehavior, mapSource, mapTarget);
            }
        }
    }
}
```
```java
public class ExtUserTaskActivityBehavior extends UserTaskActivityBehavior {

    private static final long serialVersionUID = 7711531472879418236L;
    //要命的默认构造函数
    public ExtUserTaskActivityBehavior() {
        super(null);
    }

    public ExtUserTaskActivityBehavior(UserTask userTask) {
        super(userTask);
    }
}
```
但是，上面这一切都太麻烦了，并没有稳妥的解决问题，直到看到群友们一些关于序列化工具的讨论后，我尝试了一批序列化工具。

在先后尝试了Protostuff、FST、Kryo、Fury，尝试的结果如下：

1. Protostuff直接Stack Overflow，估计还是因为循环引用的问题。
2. FST能够进行序列化，但是有一个致命的问题就是Object类型，它无法反序列化。
3. Kryo能够成功，它和FST的原理差不多，区别在于Kryo可以处理Object类型，能够获取其真正的类型。
4. Fury能够成功，看官方的性能对比图要比Kryo和FST快，但是其版本还是Snapshot。

现在貌似能够进行序列化和反序列化了，但是还要尝试一下是否能够支撑流程运行，我要先去试验一下，本章只给出关键的代码，有兴趣的同学可以自行尝试下。

Kryo要进行一些设置：

```java
//不需要注册类，让kryo自己决定
kryo.setRegistrationRequired(false);
//无需构造函数
kryo.setInstantiatorStrategy(new DefaultInstantiatorStrategy(new StdInstantiatorStrategy()));
//启用引用追踪，这涉及到kryo的机制，可以避免StackOverflow
kryo.setReferences(true);
```


Fury也要进行一些设置：

```java
 Fury fury = Fury.builder().withLanguage(Language.JAVA)
                .withRefTracking(true)
                // Allow to deserialize objects unknown types,
                // more flexible but less secure.
                .withSecureMode(false)
                .build();
//不要直接用fury.deserializeJavaObject()
Object obj = fury.deserialize(bytes);
cacheEntry = (ProcessDefinitionCacheEntry) obj;
```


以上，就这样吧，耗尽了精力，如果有错误，欢迎探讨和指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑

