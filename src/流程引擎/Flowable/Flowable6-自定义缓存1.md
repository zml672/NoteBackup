# Flowable6 - 自定义缓存(1) | 字痕随行
原创 字痕随行 字痕随行

收录于话题

#流程引擎

53个

我本来以为自定义缓存是个很容易的事情，毕竟接口是已经存在的，所以理论上只要实现接口，然后完成序列化和反序列化就可以了。

而实际上，折腾了一周多的时间，最后也不是真正意义上的成功。

先上结论：分析了一下源码，发现从Activiti6开始，缓存的类有了些许改变，加入了BpmnModel和Process。

```Java
public class ProcessDefinitionCacheEntry implements Serializable {

    private static final long serialVersionUID = 6833801933658529070L;

    protected ProcessDefinition processDefinition;
    protected BpmnModel bpmnModel;
    protected Process process;

    public ProcessDefinitionCacheEntry(ProcessDefinition processDefinition, BpmnModel bpmnModel, Process process) {
        this.processDefinition = processDefinition;
        this.bpmnModel = bpmnModel;
        this.process = process;
    }
}

```
问题就在这两个类上面，BpmnModel和Process并不支持序列化：

```Java
public class BpmnModel {
}

public class Process extends BaseElement implements FlowElementsContainer, HasExecutionListeners {
}

```
它们并没有实现Serializable接口，这就导致反序列化的时候会出现很多意想不到的问题，所以不可能愉快的使用Redis了。

在Github上已经有人问过这个问题，地址见：

```Plain Text
https://github.com/flowable/flowable-engine/issues/481

```
这个issues至今仍旧是open状态。所以我觉得，可以洗洗睡了。

上面就是结论了，如果想了解更多一点，可以继续往下看。

缓存是怎么写入的，其实可以看一下源码，关键在于BpmnDeployer这个类。

看这个类的源码，按照以下顺序来：

```Java
deploy(DeploymentEntity deployment, Map<String, Object> deploymentSettings)
->
bpmnParse.execute();
->
addProcessDefinitionToCache(processDefinition, bpmnModelMap, processEngineConfiguration, commandContext);

```
最开始的时候，我尝试着将流程定义的XML字符串缓存到Redis，然后再取出来，使用BpmnXMLConverter转换成BpmnModel对象，然后能够变相生成ProcessDefinitionCacheEntry。

很不幸的是，直接转换出的BpmnModel缺少很多的属性。如果去阅读源码就会发现，在bpmnParse.execute()中，不但convert出了BpmnModel，在之后还做了一些额外的工作：

```Java
bpmnModel.setSourceSystemId(sourceSystemId);
//这个对象不能为空，否则发布的时候报错
bpmnModel.setEventSupport(new FlowableEventSupport());
//这里用每一个流程节点的bpmnParserHandler进行了处理
transformProcessDefinitions();

```
最麻烦的就是它用了bpmnParserHandler去处理节点，然后回写给了BpmnModel的Process。

也就是，直接从XML字符串转换出来的BpmnModel天生缺点东西，当启动流程的时候会报异常，因为Process缺东西。

然后，我就死心了。除了上面的原因之外，另外一个原因就是，如果不使用进程外缓存，我觉得应该不会对分布式造成太大影响。

比如，部署了两个Flowable Engine，它们都是用的默认缓存，即一个HashMap。

当需要获得流程定义时，就会先去缓存中使用processDefinitionId获得，如果缓存中不存在该流程定义，则会从数据库中读取，然后同步缓存，而这个processDefinitionId其实是每次发布才会生成的。

这就是说：

1\. 在Flowable Engine A中进行了流程定义，并且发布。Flowable Engine B中的缓存不存在，如果这时候启动流程，会将数据库中新的流程定义同步到B的缓存中。

2\. 如果需要修改流程定义，直接读取的是act\_de\_model表中的信息，发布的时候会生成新的processDefinitionId，应该也不会造成差异。

所以，我觉得应该可以正常使用，不过这块我还**没时间测试**，建议完全测试后再上生产。

如果非得使用Redis作为缓存，可以看看下一章。在下一章里，我会变着法子实现一下，权当一个参考。

以上，就是本次的内容，如有错误，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑