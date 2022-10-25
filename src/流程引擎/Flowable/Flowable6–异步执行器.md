# Flowable6.4 – 异步执行器 | 字痕随行
最近看了看Flowable的相关文档，我对一段说明比较感兴趣，这段说明的中文翻译如下：

*Flowable V5版本中，在之前的作业执行器(job executor)之外，还提供了异步执行器(async executor)。\_\_异步执行器已被许多Flowable的用户及我们自己的跑分证明，性能比老的作业执行器好。*

*从Flowable V6起，将只提供异步执行器。\_\_在V6中，对异步执行器进行了完全的重构，以提升性能及易用性。*

所以，我特地去看了一下这部分的源码，以下就是异步执行器的相关源码解析。

**首先，来看一下异步执行器的构造过程。**

先由ProcessEngineConfigurationImpl.init()方法入手，在此方法内初始化了异步执行器：

```java
public void initAsyncExecutor() {
    if (asyncExecutor == null) {
        //声明了一个默认的异步执行器
        DefaultAsyncJobExecutor defaultAsyncExecutor = new DefaultAsyncJobExecutor();
        //此处省略代码若干
    }
    asyncExecutor.setJobServiceConfiguration(jobServiceConfiguration);
    //需要注意这个属性，false时不会启动执行器
    asyncExecutor.setAutoActivate(asyncExecutorActivate);
    jobServiceConfiguration.setAsyncExecutor(asyncExecutor);
}

```
在new ProcessEngineImpl()会控制是否启动执行器：

```Java
//autoActivate为false时就不会启动
if (asyncExecutor != null && asyncExecutor.isAutoActivate()) {
    asyncExecutor.start();
}

```
在这个过程中，asyncExecutor.start()方法很重要，它会影响之后是否会使用异步控制器，主要是该方法内的一个参数：

```Java
/** Starts the async executor */
@Override
public void start() {
    //isActive这个参数会影响是否使用该执行器
    if (isActive) {
        return;
    }

    isActive = true;

    //以下省略代码若干
}

```
这里梳理一下：

如果asyncExecutorActivate等于false，isActive就等于false。

如果asyncExecutorActivate等于true，isActive就等于true。

而asyncExecutorActivate的默认值为false。

**然后，什么时候调用执行器。**

这个问题如果自己一点一点去读代码，就会很麻烦。其实官方文档有一段描述，中文翻译如下：

*在API调用成功后触发的事务监听器(transaction commit listener)，将会触发同一引擎中的异步执行器，让其执行该作业（因此可以保证数据库中已经保存了数据）。*

既然提示的这么明显，就可以去事务提交的地方去追踪代码，事务提交都在TransactionContext内，具体的以SpringTransactionContext作为示例，查找方法addTransactionListener()的调用，如下图：

其中jobAddedTransactionListener就是调用异步执行器的源头：

```Java
@Override
public void execute(CommandContext commandContext) {
    CommandConfig commandConfig = new CommandConfig(false, TransactionPropagation.REQUIRES_NEW);
    commandExecutor.execute(commandConfig, new Command<Void>() {
        @Override
        public Void execute(CommandContext commandContext) {
            if (LOGGER.isTraceEnabled()) {
                LOGGER.trace("notifying job executor of new job");
            }
            //调用异步执行
            asyncExecutor.executeAsyncJob(job);
            return null;
        }
    });
}

```
如果一层一层追踪上去，会发现一个方法：

```Java
protected void triggerExecutorIfNeeded(JobEntity jobEntity) {
    // When the async executor is activated, the job is directly passed on to the async executor thread
    //此处就是对于isActive的判断，如果为True才增加监听器
    if (isAsyncExecutorActive()) {
        hintAsyncExecutor(jobEntity);
    }
}

```
**最后，在什么位置开始调用。**  

这个位置比较明显，其实就在ContinueProcessOperation这个类中，这个类关系到整个流程的流转，当进入节点时，它就会被触发。

而它根据当前节点的Asynchronous属性，来决定当前节点的处理是同步还是异步执行。

**整个过程梳理一下：**

以下一节点为UserTask为例，在通过ContinueProcessOperation进入此节点时，会判断Asynchronous属性的值。

如果为True，则为TransactionListener增加异步执行器(需要注意全局是否开启异步处理)相关的监听，以便数据在Commit时能够调用异步执行器进行处理。

当数据Commit时，触发监听器，异步处理此节点数据。

需要注意的是，如果全局未开启异步处理，但是在节点上却将Asynchronous设置为True，那么到达此节点时，相关的作业会被写入ACT\_RU\_JOB，但是无法被执行。

以上，如有问题欢迎讨论，如有错误欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑