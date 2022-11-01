# Camunda7 - 解析多实例UserTask节点Behavior执行过程 | 字痕随行
本篇内容源于我在Camunda里面为节点动态设置办理人。不同于之前在Flowable里面通过扩展Behavior实现，这里需要通过BpmnParseListener来重写设置。

在这里我产生了两个疑问：

1. 自定义的Behavior在这个过程中是如何生效的。
2. MultiInstanceActivityBehavior和UserTaskActivityBehavior的关系是什么。

这篇文章就是为了解答第二个问题而存在的。

要解答这个问题，其实只需要看一看多实例UserTask是否仍旧触发UserTaskActivityBehavior，然后再看看MultiInstanceActivityBehavior触发的时机即可。

仍旧可以从taskService.complete()方法入手，大概看一下多实例任务执行的过程：

```java
taskService.complete() ->
CompleteTaskCmd().execute() ->
completeTask() ->
task.complete() ->
execution.signal() ->
//UserTaskActivityBehavior，触发leave方法，处理离开节点逻辑
activityBehavior.signal() ->
PvmAtomicOperation.ACTIVITY_LEAVE ->
FlowNodeActivityBehavior.doLeave() ->
//关键点逻辑开始
BpmnActivityBehavior.performOutgoingBehavior() ->
execution.end(true) ->
PvmAtomicOperation.ACTIVITY_NOTIFY_LISTENER_END ->
//最终的逻辑在这里，是要看的地方了
PvmAtomicOperation.ACTIVITY_END
```
上面的一系列过程走完后，只需要看ACTIVITY\_END里面的execute()，这里面有部分判断，关键的判断是：

```java
PvmScope flowScope = activity.getFlowScope();
if(flowScope == activity.getProcessDefinition()) {

} else {

}
```
所以，这个flowScope是什么对象，决定了最终要执行的逻辑。

这时候就要去看activity的flowScope是在何时赋值的，找一下就可以在BpmnParse里面发现它的处理过程了。

直接看parseActivity这个方法，这个方法的参数scopeElement就是activity里面flowScope的值。

```java
protected ActivityImpl parseActivity(Element activityElement, Element parentElement, ScopeImpl scopeElement) {}
```
向上找到入参的地方，可以发现scopeElement正常情况下传递的是processDefinition，而多实例情况下，就会变成miBody。

```java
boolean isMultiInstance = false;
ScopeImpl miBody = parseMultiInstanceLoopCharacteristics(activityElement, scopeElement);
if (miBody != null) {
  scopeElement = miBody;
  isMultiInstance = true;
}
```
这里miBody就会被赋予多实例的Behavior:

```java
MultiInstanceActivityBehavior behavior = null;
if (isSequential) {
  behavior = new SequentialMultiInstanceActivityBehavior();
} else {
  behavior = new ParallelMultiInstanceActivityBehavior();
}
miBodyScope.setActivityBehavior(behavior);
```
到这里，再看之前的PvmAtomicOperation.ACTIVITY\_END.execute()，就能够理解它的逻辑处理过程了。

很明显的，多实例节点的flowScope不是processDefinition，这时候就会执行：

```java
PvmActivity flowScopeActivity = (PvmActivity) flowScope;
ActivityBehavior activityBehavior = flowScopeActivity.getActivityBehavior();
if (activityBehavior instanceof CompositeActivityBehavior) {
  CompositeActivityBehavior compositeActivityBehavior = (CompositeActivityBehavior) activityBehavior;
  //此处省略代码若干
}
```
CompositeActivityBehavior这个类，其实就是MultiInstanceActivityBehavior了，至于是并行的还是串行的，就看实际的设置了。

最后总结一下就是，UserTask节点，先执行UserTaskActivityBehavior，如果是多实例节点，就会再执行MultiInstanceActivityBehavior。

如果有问题，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑