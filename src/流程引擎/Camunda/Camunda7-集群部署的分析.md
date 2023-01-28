# Camunda7 - 集群部署的分析 | 字痕随行
**首先**，先看一下官方对于Camunda分布式部署的说明。可以参考它的官方文档：

```Plain Text
https://docs.camunda.org/manual/latest/introduction/architecture/
```
这是英文的，大概看一下Clustering Model这个章节。此章节确定了一点，如果多个流程引擎的实例连接的是同一个数据库，是可以在流程引擎层面实现集群部署的。

所以本文主要着眼于**两点**进行分析：

1. 每一次触发流程运行(调用taskComplete)，必从数据库获取最新的数据，执行完毕后，必将数据持久化到数据库，即：每一个流程引擎实例不会在运行中缓存数据，影响到下一次的触发。
2. 在任何一个流程引擎实例部署新的流程定义，在其它的流程引擎实例启动流程时，会使用最新的流程定义。

先分析**第一点**：

仍旧由taskComplete方法入手，很明显的一段查询代码如下：

```java
TaskManager taskManager = commandContext.getTaskManager();
TaskEntity task = taskManager.findTaskById(taskId);
```
看看最终是怎么查询task数据的：

```java
public <T extends DbEntity> T selectById(Class<T> entityClass, String id) {
  T persistentObject = dbEntityCache.get(entityClass, id);
  if (persistentObject!=null) {
    return persistentObject;
  }

  persistentObject = persistenceSession.selectById(entityClass, id);

  if (persistentObject==null) {
    return null;
  }
  // don't have to put object into the cache now. See onEntityLoaded() callback
  return persistentObject;
}
```
在上面的代码中，dbEntityCache一看就是从缓存查找，那persistenceSession就是从数据库查询了。

先看看dbEntityCache，这个地方其实就是从Map里面Get，仅从这里无法确定这数据是不是会被永久缓存。

其实，完全可以看看commandContext.getTaskManager()，由此可以看看commandContext是从哪里来的。

因为，我们都知道，Activiti、Flowable、Camunda都是基于命令链模式的，而且它们这块命令链的代码都差不多。

所以，直接看CommandContextInterceptor里面的：

```java
Context.getCommandContext()
```
就会发现，CommandContext被保存在当前线程里：

```java
protected static ThreadLocal<Deque<CommandContext>> commandContextThreadLocal = new ThreadLocal<Deque<CommandContext>>();
```
依照这个，再看其它相关的代码，就会发现Session、Cache都是保存在ThreadLocal的，这其实就可以保证数据不会扩散，不会影响到当前线程之外的其它处理结果。

在之前的Flowable相关文章里面也分析过，这些流程引擎数据写入数据库其实是在命令链结束时一次写入的，也就是在CommandContext的close方法内：

```java
public void close(CommandInvocationContext commandInvocationContext) {
}
```
至此，其实第一个问题已经分析完了，结论就是：每一个流程引擎实例不会在运行中缓存数据，影响到下一次的触发，而且每一次新的调用都会重新从数据库中初始化数据。

再来分析**第二点**：

直接由启动流程的代码入手

```java
@Override
public ProcessInstance startProcessInstanceByKey(String processDefinitionKey) {
  return createProcessInstanceByKey(processDefinitionKey)
      .execute();
}
```
依此定位到GetDeployedProcessDefinitionCmd的流程定义查询方法

```java
ProcessDefinitionEntity processDefinition = find(commandContext);
```
直接跟到最后的查询方法

```java
public T findDeployedLatestDefinitionByKey(String definitionKey) {
  T definition = getManager()
      .findLatestDefinitionByKey(definitionKey);
  checkInvalidDefinitionByKey(definitionKey, definition);
  definition = resolveDefinition(definition);
  return definition;
}
```
findLatestDefinitionByKey其实就是从数据库查询数据。到此，第二点也分析完了，结论就是：每次启动都从数据库查询最新的流程定义。

所以，只要是共享数据库，Camunda就是支持多流程引擎实例部署的，它运行时的所需的数据都是从数据库中获取最新的，并且运行完成后，会立即更新到数据库中。

但是，官方的文档中也指出了，负载均衡是需要单独支持的。举个例子，taskComplete并发了，并且两个请求打到了两个流程引擎实例上，这时候其实是有问题的，所以在更上层还是需要分布式锁来控制。

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑