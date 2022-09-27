# Camunda7 - 从completeTask看pvm | 字痕随行

其实我也没想好怎么起这个题目，然后这是心血来潮后的一篇记录。

背景就是：我没尝试过Camunda，生产环境也没有任何经验，只是久闻大名之后，突然心痒，所以看了一下源码，切入点集中在pvm而已。

我想看看Camunda里面的pvm到底起了什么作用，它的原理和Flowable到底有何不同，所以才有了这篇文章。

因为只耗费了数个小时而已，所以也不输出结论，以免以偏概全。另外，可能这只是一篇源码走读，稍显枯燥。

从TaskService的Complete入手，看看从一个UserTask是如何流转到另外一个UserTask的，当初看Flowable就这么入手的，所以这次依然这样了。

废话那么多，如果你还想继续看下去，那咱们就继续吧。

本文基于Camunda-7.15.0-alpha1。

总的来说，Flowable操作流程的跳转全靠Behavior，因为Flowable已经完全剔除了Pvm。

但是Camunda完全不同，虽然它也有Behavior，但是节点之间的流转控制完全交给了Pvm。

这里面需要特别注意一个类：PvmAtomicOperation，按这个类里面定义的顺序，已经可以了解整个Pvm的执行过程。

从TaskService的Complete入手：

```java
public void complete(String taskId) {
  complete(taskId, null);
}
```

整个的执行过程是：

```java
taskService.Complete
execution.signal
activityBehavior.signal
userTaskBehavior.signal -> leave //在这里触发了leave方法，一看就是离开节点
execution.dispatchDelayedEventsAndPerformOperation
ACTIVITY_LEAVE //pvm
behavior.doLeave
bpmnActivityBehavior.performDefaultOutgoingBehavior(execution)
TRANSITION_NOTIFY_LISTENER_END //pvm
TRANSITION_DESTROY_SCOPE //pvm
TRANSITION_NOTIFY_LISTENER_TAKE //pvm
TRANSITION_START_NOTIFY_LISTENER_TAKE //pvm
TRANSITION_CREATE_SCOPE //pvm
TRANSITION_NOTIFY_LISTENER_START //pvm
ACTIVITY_EXECUTE //pvm
activityBehavior.execute(execution) //进入了下一个节点
```

最后，进入下一个UserTask节点结束：

```java
@Override
public void performExecution(ActivityExecution execution) throws Exception {
  TaskEntity task = new TaskEntity((ExecutionEntity) execution);
  task.insert();

  // initialize task properties
  taskDecorator.decorate(task, execution);

  // fire lifecycle events after task is initialized
  task.transitionTo(TaskState.STATE_CREATED);
}
```

我一直再找流程节点间的连接线，找入口和出口，但是一直没有Sequence这个关键字出现，后来搜索了一下，才知道pvm中的Transition就是连接线的意思（接触Activiti太少了，相关知识储备不够啊）。

这样的话就可以看到代码中的过程是：pvm控制节点离开，然后再控制连接线寻找出口，最后再控制进入下一个节点(这里只讨论UserTask到UserTask)。

我目前的简单理解就是，流程在运行当中，pvm中的对象记载了流程元素在运行过程中的情况，并且加以控制。

但是，很模糊的点在于，什么时候能够使用pvm干什么事情，这个可能需要以后有机会再尝试了。

好了，记录完毕，如果文中有错误，欢迎指正。

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑
