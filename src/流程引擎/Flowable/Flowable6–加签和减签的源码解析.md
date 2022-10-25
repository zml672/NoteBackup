# Flowable6.4 – 加签和减签的源码解析 | 字痕随行
[上一篇](http://www.blackzs.com/archives/1464)简单实现了一下加签和减签的操作，这次主要是看看Flowable是如何实现加签和减签的。

**首先，加签。**

Flowable实现加签主要是通过下面的方法实现的：

```java
runtimeService.addMultiInstanceExecution(String activityId, String parentExecutionId, Map<String, Object> executionVariables)

```
跟踪代码进入其方法体，发现执行了下面这个命令：

```java
AddMultiInstanceExecutionCmd(activityId, parentExecutionId, executionVariables)

```
这里的三个参数所代表的的意义是：

activityId：流程节点的标识。

parentExecutionId：流程执行实例标识，proInstId。

executionVariables：所要传入的参数。

查看AddMultiInstanceExecutionCmd这个类的源码，主要关注方法execute()，此方法就是加签操作实现的关键。

关键的代码加注释，如下：

```java
@Override
public Execution execute(CommandContext commandContext) {
    ExecutionEntityManager executionEntityManager = CommandContextUtil.getExecutionEntityManager();
    //获得multi instance execution，即IS_MI_ROOT
    ExecutionEntity miExecution = searchForMultiInstanceActivity(activityId, parentExecutionId, executionEntityManager);

    if (miExecution == null) {
        throw new FlowableException("No multi instance execution found for activity id " + activityId);
    }

    if (Flowable5Util.isFlowable5ProcessDefinitionId(commandContext, miExecution.getProcessDefinitionId())) {
        throw new FlowableException("Flowable 5 process definitions are not supported");
    }
    //创建新的流程执行实例
    ExecutionEntity childExecution = executionEntityManager.createChildExecution(miExecution);
    childExecution.setCurrentFlowElement(miExecution.getCurrentFlowElement());
    //获得BPMN模型中节点的配置信息
    BpmnModel bpmnModel = ProcessDefinitionUtil.getBpmnModel(miExecution.getProcessDefinitionId());
    Activity miActivityElement = (Activity) bpmnModel.getFlowElement(miExecution.getActivityId());
    MultiInstanceLoopCharacteristics multiInstanceLoopCharacteristics = miActivityElement.getLoopCharacteristics();
    //设置流程参数nrOfInstances
    Integer currentNumberOfInstances = (Integer) miExecution.getVariable(NUMBER_OF_INSTANCES);
    miExecution.setVariableLocal(NUMBER_OF_INSTANCES, currentNumberOfInstances + 1);
    //设置子流程执行实例的参数
    if (executionVariables != null) {
        childExecution.setVariablesLocal(executionVariables);
    }
    //如果是并行，需要执行操作，生成Task记录
    if (!multiInstanceLoopCharacteristics.isSequential()) {
        miExecution.setActive(true);
        miExecution.setScope(false);

        childExecution.setCurrentFlowElement(miActivityElement);
        CommandContextUtil.getAgenda().planContinueMultiInstanceOperation(childExecution, miExecution, currentNumberOfInstances);
    }

    return childExecution;
}

```
需要注意的就是最后部分的操作，因为节点有并行和串行的区分，所以需要不同的处理。

**再看，减签。**

Flowable实现加签主要是通过下面的方法实现的：

```java
runtimeService.deleteMultiInstanceExecution(String executionId, boolean executionIsCompleted)

```
跟踪代码进入其方法体，发现执行了下面这个命令：

```java
DeleteMultiInstanceExecutionCmd(executionId, executionIsCompleted)

```
这里的两个参数所代表的的意义是：executionId：需要删除的流程执行实例标识。

executionIsCompleted：是否完成此流程执行实例。查看

DeleteMultiInstanceExecutionCmd这个类的源码，主要关注方法execute()，此方法就是加签操作实现的关键。

关键的代码加注释，如下：

```java
@Override
public Void execute(CommandContext commandContext) {
    ExecutionEntityManager executionEntityManager = CommandContextUtil.getExecutionEntityManager();
    ExecutionEntity execution = executionEntityManager.findById(executionId);
    //获得BPMN模型中节点的配置信息
    BpmnModel bpmnModel = ProcessDefinitionUtil.getBpmnModel(execution.getProcessDefinitionId());
    Activity miActivityElement = (Activity) bpmnModel.getFlowElement(execution.getActivityId());
    MultiInstanceLoopCharacteristics multiInstanceLoopCharacteristics = miActivityElement.getLoopCharacteristics();

    if (miActivityElement.getLoopCharacteristics() == null) {
        throw new FlowableException("No multi instance execution found for execution id " + executionId);
    }

    if (!(miActivityElement.getBehavior() instanceof MultiInstanceActivityBehavior)) {
        throw new FlowableException("No multi instance behavior found for execution id " + executionId);
    }

    if (Flowable5Util.isFlowable5ProcessDefinitionId(commandContext, execution.getProcessDefinitionId())) {
        throw new FlowableException("Flowable 5 process definitions are not supported");
    }
    //删除指定的流程执行实例和与其关联的数据
    ExecutionEntity miExecution = getMultiInstanceRootExecution(execution);
    executionEntityManager.deleteChildExecutions(execution, "Delete MI execution", false);
    executionEntityManager.deleteExecutionAndRelatedData(execution, "Delete MI execution", false);
    //获得循环的索引值，以便之后重新设置
    int loopCounter = 0;
    if (multiInstanceLoopCharacteristics.isSequential()) {
        //如果是串行，则获得当前的索引值
        SequentialMultiInstanceBehavior miBehavior = (SequentialMultiInstanceBehavior) miActivityElement.getBehavior();
        loopCounter = miBehavior.getLoopVariable(execution, miBehavior.getCollectionElementIndexVariable());
    }
    //如果设置为流程执行实例已经完成，则已完成数量+1，并且索引值也+1
    //如果设置为流程执行实例未完成，则流程实例数量-1，索引值不变
    if (executionIsCompleted) {
        Integer numberOfCompletedInstances = (Integer) miExecution.getVariable(NUMBER_OF_COMPLETED_INSTANCES);
        miExecution.setVariableLocal(NUMBER_OF_COMPLETED_INSTANCES, numberOfCompletedInstances + 1);
        loopCounter++;

    } else {
        Integer currentNumberOfInstances = (Integer) miExecution.getVariable(NUMBER_OF_INSTANCES);
        miExecution.setVariableLocal(NUMBER_OF_INSTANCES, currentNumberOfInstances - 1);
    }
    //生成一个新的流程执行实例(个人觉得这个是专为串行准备的)
    ExecutionEntity childExecution = executionEntityManager.createChildExecution(miExecution);
    childExecution.setCurrentFlowElement(miExecution.getCurrentFlowElement());
    //如果是串行，需要执行一次生成Task，并且设置正确的loopCounter
    if (multiInstanceLoopCharacteristics.isSequential()) {
        SequentialMultiInstanceBehavior miBehavior = (SequentialMultiInstanceBehavior) miActivityElement.getBehavior();
        miBehavior.continueSequentialMultiInstance(childExecution, loopCounter, childExecution);
    }

    return null;
}

```
所以，通过分析上述源码，如果是并行的多实例节点，并且删除了最后一个流程执行实例，会发现没有了Task，导致整个流程中断。

以上，就是对于Flowable6.4的加签和减签的源码分析，如果有错误，请指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑