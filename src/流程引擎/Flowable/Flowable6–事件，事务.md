# Flowable6.4 – 事件，事务 | 字痕随行
最近有个需求，假设流程节点都是同步的，在UserTask创建完成后，推送一条企业微信消息。

需求很简单，实现上也没有什么难度，但是在实现FlowableEventListener这个接口的时候，发现和事务有所联系。

然后，很自然的想到一个问题：这个事件的触发到底是在事务提交之后，还是在事务提交之前。如果在事务提交之前触发了事件，事务提交时又失败回滚，这条发出的消息岂不是无用的。

所以，我又去翻了一遍代码。以TaskService.complete(String taskId)作为切入点，跟踪下去，很容易找到：

```Java
TaskHelper.completeTask(task, variables, transientVariables, localScope, commandContext);

```
然后在上面这个方法内，就会找到事件的触发点：

```Java
FlowableEventDispatcher eventDispatcher = CommandContextUtil.getProcessEngineConfiguration(commandContext).getEventDispatcher();
if (eventDispatcher != null && eventDispatcher.isEnabled()) {
    if (variables != null) {
        eventDispatcher.dispatchEvent(FlowableEventBuilder.createEntityWithVariablesEvent(
                FlowableEngineEventType.TASK_COMPLETED, taskEntity, variables, localScope));
    } else {
        eventDispatcher.dispatchEvent(
                FlowableEventBuilder.createEntityEvent(FlowableEngineEventType.TASK_COMPLETED, taskEntity));
    }
}

```
如果继续跟踪下去，就会找到最终的触发代码：

```Java
protected void dispatchEvent(FlowableEvent event, FlowableEventListener listener) {
    if (listener.isFireOnTransactionLifecycleEvent()) {
        //与事务有关的事件
        dispatchTransactionEventListener(event, listener);
    } else {
        //一般的事件
        dispatchNormalEventListener(event, listener);
    }
}

```
然后就会看到if...else...判断，而这个判断的条件恰好是在实现FlowableEventListener这个接口时需要实现的方法。

从字面意思上来看，也比较容易理解，一个是触发与事务有关的事件，一个是触发正常的事件。

如果触发与事务有关的事件，可以看到代码是如下运行的：

```Java
protected void dispatchTransactionEventListener(FlowableEvent event, FlowableEventListener listener) {
    //此处省略代码若干

    ExecuteEventListenerTransactionListener transactionListener = new ExecuteEventListenerTransactionListener(listener, event); 
    //注意这个listener.getOnTransaction()
    if (listener.getOnTransaction().equalsIgnoreCase(TransactionState.COMMITTING.name())) {
        transactionContext.addTransactionListener(TransactionState.COMMITTING, transactionListener);

    } 

    //此处省略代码若干
}

```
上面代码中的getOnTransaction()，正好也是在实现FlowableEventListener这个接口时需要实现的方法。

所以结论就是：

1\. 在实现FlowableEventListener这个接口时，如果返回了False，事件就会包裹在事务内。

```Java
@Override
public boolean isFireOnTransactionLifecycleEvent() {
    return false;
}

```
2\. 在实现FlowableEventListener这个接口时，如果返回了True，事件会按照TransactionState的值来触发，与事务的关系也会不同。

```Java
@Override
public boolean isFireOnTransactionLifecycleEvent() {
    return true;
}

@Override
public String getOnTransaction() {
    //事务提交后触发
    return TransactionState.COMMITTED.name();
}

```
3\. 所以开头那个问题，最好是设置一下TransactionState，并且在事务提交后触发，可以保证发送的消息是有效的。当然为了保证消息能够可靠送达，还需要一些其它的手段。

以上，如有问题，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑