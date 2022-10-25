# Flowable6 - 事件 | 字痕随行
原创 字痕随行 字痕随行

收录于话题

#流程引擎

53个

貌似从来没有专门介绍过Flowable的事件，只是在流程设计器部分提到过，那么就总结一下吧。

Flowable所有的事件类型，可以参见枚举：

```Plain Text
org.flowable.common.engine.api.delegate.event.FlowableEngineEventType

```
比如最常用的：

```Plain Text
/**
 * A task has been created. This is thrown when task is fully initialized (before TaskListener.EVENTNAME_CREATE).
 */
TASK_CREATED,

/**
 * A task has been completed. Dispatched before the task entity is deleted ( #ENTITY_DELETED). If the task is part of a process, this event is dispatched before the process moves on, as a
 * result of the task completion. In that case, a #ACTIVITY_COMPLETED will be dispatched after an event of this type for the activity corresponding to the task.
 */
TASK_COMPLETED,

/**
 * A process instance has been started. Dispatched when starting a process instance previously created. The event PROCESS_STARTED is dispatched after the associated event ENTITY_INITIALIZED and
 * after the variables have been set.
 */
PROCESS_STARTED,

/**
 * A process has been completed. Dispatched after the last activity is ACTIVITY_COMPLETED. Process is completed when it reaches state in which process instance does not have any transition to
 * take.
 */
PROCESS_COMPLETED,

```
这些事件是**如何触发**的呢？在AbstractEngineConfiguration内初始化了事件的Dispatcher：

```Plain Text
public void initEventDispatcher() {
    if (this.eventDispatcher == null) {
        this.eventDispatcher = new FlowableEventDispatcherImpl();
    }

    //省略...
}

```
调用Dispatcher的dispatchEvent来触发事件，比如：

```Plain Text
eventDispatcher.dispatchEvent(
        FlowableTaskEventBuilder.createEntityEvent(
                FlowableEngineEventType.TASK_CREATED,
                task
        ),
        processEngineConfiguration.getEngineCfgKey()
);

```
如何**自定义监听**这些事件呢？有两个办法：

1. 像之前介绍的那样，在流程设计时，加入自定义的事件处理类。
2. 在初始化ProcessEngineConfiguration时定义自己的处理类。

简单说一下第二种方法，首先需要实现接口：

```Plain Text
public interface FlowableEventListener {
    void onEvent(FlowableEvent var1);

    boolean isFailOnException();

    boolean isFireOnTransactionLifecycleEvent();

    String getOnTransaction();
}

```
然后在实例化ProcessEngineConfiguration时引入实现类的集合：

```Plain Text
configuration.setEventListeners(new ArrayList<FlowableEventListener>(){
    {
        add(new FlowableEventListenerImpl());
    }
});

```
**最后**，想特别说一下：

```Plain Text
/**
 * A multi-instance activity has met its condition and completed successfully.
 */
MULTI_INSTANCE_ACTIVITY_COMPLETED_WITH_CONDITION

```
这个事件的触发条件：

1. 多实例情况下。
2. 达到了多实例节点的结束条件，也就是Completion condition的表达式为True。

这个事件比较有用，比如：流程的会签(多实例)节点达到结束条件时，可以清理一下冗余的数据，或者发送一条通知消息。

以上，就是关于事件的介绍，如有错误，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑