# Flowable6.6-自定义Cmd动态增加节点
上一篇介绍了如何通过使用Flowable原生的SDK动态增加节点。但是，原生的SDK有瑕疵，无法满足实际的生产需求。本篇，就介绍一下如何通过自定义Cmd动态增加节点。

**首先**，来看一下InjectUserTaskInProcessInstanceCmd这个类是怎么实现的。这个类的内部结构如下：

```java
public class InjectUserTaskInProcessInstanceCmd extends AbstractDynamicInjectionCmd implements Command<Void> {

    protected String processInstanceId;
    protected DynamicUserTaskBuilder dynamicUserTaskBuilder;

    public InjectUserTaskInProcessInstanceCmd(String processInstanceId, DynamicUserTaskBuilder dynamicUserTaskBuilder) {
        this.processInstanceId = processInstanceId;
        this.dynamicUserTaskBuilder = dynamicUserTaskBuilder;
    }

    @Override
    public Void execute(CommandContext commandContext) {
        createDerivedProcessDefinitionForProcessInstance(commandContext, processInstanceId);
        return null;
    }

    //更新流程定义
    @Override
    protected void updateBpmnProcess(CommandContext commandContext, Process process,
            BpmnModel bpmnModel, ProcessDefinitionEntity originalProcessDefinitionEntity, DeploymentEntity newDeploymentEntity) {
    
    }

    //后置更新流程执行所需要的数据
    @Override
    protected void updateExecutions(CommandContext commandContext, ProcessDefinitionEntity processDefinitionEntity, 
            ExecutionEntity processInstance, List<ExecutionEntity> childExecutions) {
    
    }

}
```
这个类有两个方法是需要关注的：

1. updateBpmnProcess这个方法主要由两大块功能组成。一部分是动态创建节点和连线，一部分是绘制新的流程图。
2. updateExecutions这个方法是用来更新流程执行所需数据的，原生的方法里面是创建了一个新的流程执行实例，并且置为当前活动的任务。

所以，直接照葫芦画瓢就完事了。

**接下来**，先改造updateBpmnProcess这个方法，直接上代码了。

```java
@Override
protected void updateBpmnProcess(CommandContext commandContext, Process process,
                                 BpmnModel bpmnModel, ProcessDefinitionEntity originalProcessDefinitionEntity, DeploymentEntity newDeploymentEntity) {

    if (null == currentElement) {
        super.updateBpmnProcess(commandContext, process, bpmnModel, originalProcessDefinitionEntity, newDeploymentEntity);
        return;
    }
    if (!(this.currentElement instanceof UserTask)) {
        return;
    }
    StartEvent startEvent = getStartEvent(process);
    if (null == startEvent) {
        return;
    }
    UserTask currentUserTask = (UserTask) this.currentElement;
    if (currentUserTask.getOutgoingFlows().size() <= 0) {
        return;
    }
    SequenceFlow currentOutgoingFlow = currentUserTask.getOutgoingFlows().get(0);
    FlowElement targetFlowElement = currentOutgoingFlow.getTargetFlowElement();
    //创建新的任务节点和两条连线
    UserTask newUserTask = createUserTask(process);
    SequenceFlow newSequenceFlow1 = createSequenceFlow(currentUserTask, newUserTask);
    SequenceFlow newSequenceFlow2 = createSequenceFlow(newUserTask, targetFlowElement);
    //加到流程内
    process.addFlowElement(newUserTask);
    process.addFlowElement(newSequenceFlow1);
    process.addFlowElement(newSequenceFlow2);
    process.removeFlowElement(currentOutgoingFlow.getId());

    //绘制新的流程图
    GraphicInfo elementGraphicInfo = bpmnModel.getGraphicInfo(currentUserTask.getId());
    if (elementGraphicInfo != null) {
        double yDiff = 0;
        double xDiff = 80;
        if (elementGraphicInfo.getY() < 173) {
            yDiff = 173 - elementGraphicInfo.getY();
            elementGraphicInfo.setY(173);
        }

        Map<String, GraphicInfo> locationMap = bpmnModel.getLocationMap();
        for (String locationId : locationMap.keySet()) {
            if (startEvent.getId().equals(locationId)) {
                continue;
            }

            GraphicInfo locationGraphicInfo = locationMap.get(locationId);
            locationGraphicInfo.setX(locationGraphicInfo.getX() + xDiff);
            locationGraphicInfo.setY(locationGraphicInfo.getY() + yDiff);
        }

        Map<String, List<GraphicInfo>> flowLocationMap = bpmnModel.getFlowLocationMap();
        for (String flowId : flowLocationMap.keySet()) {
            List<GraphicInfo> flowGraphicInfoList = flowLocationMap.get(flowId);
            for (GraphicInfo flowGraphicInfo : flowGraphicInfoList) {
                flowGraphicInfo.setX(flowGraphicInfo.getX() + xDiff);
                flowGraphicInfo.setY(flowGraphicInfo.getY() + yDiff);
            }
        }
        //把之前的连线给移除了
        bpmnModel.removeFlowGraphicInfoList(currentOutgoingFlow.getId());
        //重新排版，这里需要引入flowable-bpmn-layout
        new BpmnAutoLayout(bpmnModel).execute();
    }
    
    BaseDynamicSubProcessInjectUtil.processFlowElements(commandContext, process, bpmnModel, originalProcessDefinitionEntity, newDeploymentEntity);
}
```
这时候，流程图已经变正常了。上面的代码演示的是在当前任务节点之后增加了一个新的任务节点，如果需要在之前增加，可以自行改造一下。

现在，流程图虽然已经变正常了，但是execution现在还是两个，task也是两个。

如果只是想增加个任务节点，而不需要增加execution和task，那么直接将updateExecutions方法置空就行了，如下：

```java
@Override
protected void updateExecutions(CommandContext commandContext, ProcessDefinitionEntity processDefinitionEntity,
                                ExecutionEntity processInstance, List<ExecutionEntity> childExecutions) {

}
```
**最后**，把这个类定义为CustomInjectUserTaskCmd吧，然后就可以试一下了：

```java
public class CustomInjectUserTaskCmd extends InjectUserTaskInProcessInstanceCmd {}
```
本篇只局限于在主流程内增加一个用户任务节点，接下来我还打算试试在子流程内增加一个任务节点。

以上，如有问题，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑