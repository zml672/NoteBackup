# Flowable6.6-从任意节点启动 | 字痕随行

中国式流程有一些很变态的需求，比如流程重启。

我能想到流程重启的三种方式如下：

1. 调用startProcessInstanceByKey重新执行一遍流程。
2. 调用startProcessInstanceByKey重新启动流程，使用节点自由跳转的方式跳转到指定的UserTask。
3. 自定义Cmd，直接从指定的UserTask启动。

需要说明的是，这三种方式在与用户交互时，都需要通过businessKey进行个性化处理。

前两种方式的实现在之前的示例中都有所介绍，本篇主要介绍一下第三种方式。

如果注意过BpmnModel里面的Process，就会发现Process有一个属性：initialFlowElement。

去看一下startProcessInstanceByKey的代码，会发现：

```java
// Create the first execution that will visit all the process definition elements
ExecutionEntity execution = processEngineConfiguration.getExecutionEntityManager().createChildExecution(processInstance);
execution.setCurrentFlowElement(startInstanceBeforeContext.getInitialFlowElement());
```
这代码挺清晰的，设置了流程执行实例的当前节点元素。后面就会调用planContinueProcessOperation向下继续执行该节点的Behavior了。

所以，只需要定义一个Cmd就行了，只要这个Cmd里面能够改变initialFlowElement。

```java
public class CustomRestartProcessCmd extends StartProcessInstanceCmd<ProcessInstance> {

    ProcessInstanceHelper processInstanceHelper;
    //从哪个节点开始启动
    FlowElement restartFlowElement;

    public CustomRestartProcessCmd(String processDefinitionKey, String processDefinitionId, String businessKey, Map<String, Object> variables) {
        super(processDefinitionKey, processDefinitionId, businessKey, variables);
    }

    @Override
    public ProcessInstance execute(CommandContext commandContext) {
        ProcessEngineConfigurationImpl processEngineConfiguration = CommandContextUtil.getProcessEngineConfiguration(commandContext);
        processInstanceHelper = processEngineConfiguration.getProcessInstanceHelper();
        ProcessDefinition processDefinition = getProcessDefinition(processEngineConfiguration, commandContext);

        Process process = ProcessDefinitionUtil.getProcess(processDefinition.getId());

        ProcessInstance processInstance = processInstanceHelper.createAndStartProcessInstanceWithInitialFlowElement(
                processDefinition,
                businessKey,
                businessStatus,
                processInstanceName,
                restartFlowElement,
                process,
                variables,
                transientVariables,
                true);

        return processInstance;
    }

    public FlowElement getRestartFlowElement() {
        return restartFlowElement;
    }

    public void setRestartFlowElement(FlowElement restartFlowElement) {
        this.restartFlowElement = restartFlowElement;
    }
}
```
最后，测试的代码如下：

```java
@RequestMapping(value = "restart/{id}")
@ResponseBody
public void restartProcess(@PathVariable("id") String id) {
    HistoricActivityInstance historicActivityInstance = historyService.createHistoricActivityInstanceQuery().activityInstanceId(id).singleResult();
    BpmnModel bpmnModel = repositoryService.getBpmnModel(historicActivityInstance.getProcessDefinitionId());
    FlowElement flowElement = bpmnModel.getFlowElement(historicActivityInstance.getActivityId());
    CustomRestartProcessCmd cmd = new CustomRestartProcessCmd(
            null,
            historicActivityInstance.getProcessDefinitionId(),
            UUID.randomUUID().toString(),
            new HashMap<>()
    );
    cmd.setRestartFlowElement(flowElement);
    managementService.executeCommand(cmd);
}
```
以上代码，我简单的测试了一下：

1. 可以从任意UserTask启动。
2. 可以从任意SubProcess启动（它会触发子流程内的StartEvent，然后流转到UserTask停住）。

以上，如果有错误，欢迎探讨和指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑