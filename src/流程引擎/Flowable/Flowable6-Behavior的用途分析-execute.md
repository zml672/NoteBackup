# Flowable6.4 - Behavior的用途分析(execute) | 字痕随行
原创 字痕随行 字痕随行

收录于话题

#流程引擎

52个

上一篇分析了一下Behavior的用途，不过只是追踪到leave()方法就结束了。后来在实际工作当中发现，其内部的execute也挺重要，所以本次就看一下execute()方法的主要作用。

在上一篇中，调用了super.leave()方法后，其实就会离开当前节点，主要是通过下面的代码实现的：

```Java
/**
 * Default way of leaving a BPMN 2.0 activity: evaluate the conditions on the outgoing sequence flow and take those that evaluate to true.
 */
public void leave(DelegateExecution execution) {
    bpmnActivityBehavior.performDefaultOutgoingBehavior((ExecutionEntity) execution);
}

```
这次代码跟踪，就由此方法开始，看看离开当前节点之后，都发生了什么。

首先，进入performDefaultOutgoingBehavior，一直跟踪下去，最终发现会调用TakeOutgoingSequenceFlowsOperation这个命令。

查看这个命令的run()方法，最终会看到一般的节点会调用leaveFlowNode()方法，这个方法很重要，因为它会按照条件挑选符合的流转路径(Sequence)，最终确定目标节点是哪个。

```Java
// Determine which sequence flows can be used for leaving
List<SequenceFlow> outgoingSequenceFlows = new ArrayList<>();
for (SequenceFlow sequenceFlow : flowNode.getOutgoingFlows()) {
    //省略代码若干，会按condition判断那个sequenceFlow可以使用
}

```
leaveFlowNode()这个方法的最后，会使当前的流程按照流转路径继续执行下去：

```Java
// Leave (only done when all executions have been made, since some queries depend on this)
for (ExecutionEntity outgoingExecution : outgoingExecutions) {
    agenda.planContinueProcessOperation(outgoingExecution);
}

```
planContinueProcessOperation()这个方法其实比较眼熟了，跟踪过去的话，最重要的命令类ContinueProcessOperation就会进入视野了，它的run()方法相当重要，这个方法会按照规则调用不同的方法保证流程运行。

```Java
@Override
public void run() {
    FlowElement currentFlowElement = getCurrentFlowElement(execution);
    if (currentFlowElement instanceof FlowNode) {
        continueThroughFlowNode((FlowNode) currentFlowElement);
    } else if (currentFlowElement instanceof SequenceFlow) {
        continueThroughSequenceFlow((SequenceFlow) currentFlowElement);
    } else {
        throw new FlowableException("Programmatic error: no current flow element found or invalid type: " + currentFlowElement + ". Halting.");
    }
}

```
基于前面的代码分析，着重关注continueThroughSequenceFlow()这个方法，继续一路跟踪下去，中间有一段关键代码需要关注一下：

```Java
//如果是流程节点，就去进入到目标节点内，否则继续重复之前的操作
if (targetFlowElement instanceof FlowNode) {
    continueThroughFlowNode((FlowNode) targetFlowElement);
} else {
    agenda.planContinueProcessOperation(execution);
}

```
最后会进入到executeActivityBehavior()这个方法内，于是一切都开始明朗了，最终会看到如下的代码：

```Java
try {
    activityBehavior.execute(execution);
} catch (RuntimeException e) {
    if (LogMDC.isMDCEnabled()) {
        LogMDC.putMDCExecution(execution);
    }
    throw e;
}

```
可以看到，进入流程节点时，就会调用Behavior的execute方法，如果看一下这个方法的代码，就会发现，这个方法其实主要是创建Task和触发Task事件，以普通的UserTask为例：

```Java
@Override
public void execute(DelegateExecution execution) {
    //创建Task
    TaskEntity task = taskService.createTask();

    //省略若干代码，为Task属性赋值

    // Handling assignments need to be done after the task is inserted, to have an id
    if (!skipUserTask) {
        //处理任务分派
        handleAssignments(taskService, activeTaskAssignee, activeTaskOwner,
                activeTaskCandidateUsers, activeTaskCandidateGroups, task, expressionManager, execution);
         //开始触发事件
        processEngineConfiguration.getListenerNotificationHelper().executeTaskListeners(task, TaskListener.EVENTNAME_CREATE);

        // All properties set, now firing 'create' events
        FlowableEventDispatcher eventDispatcher = CommandContextUtil.getTaskServiceConfiguration(commandContext).getEventDispatcher();
        if (eventDispatcher != null  && eventDispatcher.isEnabled()) {
            eventDispatcher.dispatchEvent(
                    FlowableTaskEventBuilder.createEntityEvent(FlowableEngineEventType.TASK_CREATED, task));
        }

    } else {

    }
}

```
所以Behavior不只是控制了是否leave当前节点，还控制了进入此节点时，所要执行的业务逻辑。

以上，就是本次的分析，欢迎指正和讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑