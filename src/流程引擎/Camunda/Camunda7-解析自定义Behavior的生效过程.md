# Camunda7 - 解析自定义Behavior的生效过程 | 字痕随行
本篇内容源于我在Camunda里面为节点动态设置办理人。不同于之前在Flowable里面通过扩展Behavior实现，这里需要通过BpmnParseListener来重写设置。

在这里我产生了两个疑问：

1. 自定义的Behavior在这个过程中是如何生效的。
2. MultiInstanceActivityBehavior和UserTaskActivityBehavior的关系是什么。

这篇文章就是为了解答第一个问题而存在的。

这里有两个入手点，第一个是从ProcessEngineConfiguration入手，配置监听器生效：

```java
//增加监听器
configuration.setCustomPreBPMNParseListeners(new ArrayList<BpmnParseListener>() {{
    add(new MyBpmnParseListener());
}});
```
然后就可以去查找这个监听器的触发点了，这里可以看BpmnParse，以parseUserTask为例：

```java
//循环加入Listener
for (BpmnParseListener parseListener : parseListeners) {
  parseListener.parseUserTask(userTaskElement, scope, activity);
}
```
这里的parseListeners就是在初始化的：

```java
protected BpmnDeployer getBpmnDeployer() {
  //省略代码若干

  BpmnParser bpmnParser = new BpmnParser(expressionManager, bpmnParseFactory);

  if (preParseListeners != null) {
    bpmnParser.getParseListeners().addAll(preParseListeners);
  }
  bpmnParser.getParseListeners().addAll(getDefaultBPMNParseListeners());
  if (postParseListeners != null) {
    bpmnParser.getParseListeners().addAll(postParseListeners);
  }

  bpmnDeployer.setBpmnParser(bpmnParser);

  return bpmnDeployer;
}
```
所以，从这里可以看到自定义的Listener是如何被加载的。

第二个入手点，就是Behavior的调用时机。

直接看这段代码就可以了：

```java
public class PvmAtomicOperationActivityExecute implements PvmAtomicOperation {
  //此处省略代码若干

  public void execute(PvmExecutionImpl execution) {
    execution.activityInstanceStarted();

    execution.continueIfExecutionDoesNotAffectNextOperation(
        //此处省略代码若干

        ActivityBehavior activityBehavior = getActivityBehavior(execution);
        //此处省略代码若干
        activityBehavior.execute(execution);
        
        //此处省略代码若干
  }

  //此处省略代码若干
}
```
把上面的串起来看，就是整个生效过程：自定义MyBpmnParseListener -> 重写Activity的Behavior -> 加载自定义的Listener -> 调用Activity的Beahavior -> 触发自定义的Behavior。

如果有问题，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑
