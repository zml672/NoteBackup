# Flowable6.4 – 在事件中使用API接口 | 字痕随行
标题名称有点长，因为不太好描述今天所分享的内容。今天这篇的目的有两个：

1\. 在事件中，处于不同的阶段，使用不同的API进行数据操作。 

2. 从侧面验证上一篇文章《[Flowable6.4 - 事件，事务](http://mp.weixin.qq.com/s?__biz=MzI3NTE2NzczMQ==&mid=2650046179&idx=1&sn=42f99e228d54fe51e572da60b3aff646&chksm=f3083e7fc47fb76989dee2ec3ab4ad298bc43b1b1102aee96f1103a76231a8463d5469f699fa&scene=21#wechat_redirect)》的结论。

**如果事件包裹在事务内**，即：

```Java
@Override
public boolean isFireOnTransactionLifecycleEvent() {
    return false;
}

```
如果使用createXXXXQuery()来进行数据查询，是无法查找出正确的数据的，比如下面的语句：

```Java
runtimeService.createExecutionQuery().executionId(executionId).singleResult()

```
有时候，查找出的结果是null，可能的原因是：新生成的Execution还未被Commit，所以根本无法查到。

新的问题由此产生：在事务提交之前，该怎么来进行数据查询呢？

通过Flowable的源代码来看，会发现一个经常出现的工具类：

```Java
CommandContextUtil

```
比如，查找Execution，就可以使用下面的方法：

```Java
CommandContextUtil.getExecutionEntityManager().findById(executionId)

```
这也从侧面说明，如果isFireOnTransactionLifecycleEvent返回了False，其实是被包裹在事务内的。

也从另外一面说明为什么在《[Flowable6.4 - 设置流程分类](http://mp.weixin.qq.com/s?__biz=MzI3NTE2NzczMQ==&mid=2650046173&idx=1&sn=2a005108d5cfb027ea041da29fec003a&chksm=f3083e41c47fb757800d656e27d492fcf0cccf30cb316220d9dcc5cd34b32ac00c69ca96d8aa&scene=21#wechat_redirect)》中，只是做了以下操作，就可以改变Task的属性：

```Java
TaskEntityImpl.setCategory(deploymentEntity.getCategory());

```
**如果事件在Commit之后呢**？

就可以使用Flowable提供的API接口来进行数据访问了，比如：

```Java
runtimeService.createExecutionQuery().executionId(executionId).singleResult()

taskService.createTaskQuery().taskId(taskId).singleResult()

```
以上，如有问题，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑