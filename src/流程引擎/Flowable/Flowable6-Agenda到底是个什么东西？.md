# Flowable6.4 - Agenda到底是个什么东西？ | 字痕随行
原创 字痕随行 字痕随行

收录于合集

#流程引擎 58 个

#Flowable 2 个

之前梳理过命令和责任链大概是个什么东西，简单的说过Flowable内部运行的过程？

详细可以戳这里：

[命令和责任链模式](http://mp.weixin.qq.com/s?__biz=MzI3NTE2NzczMQ==&mid=2650045912&idx=1&sn=713d3f14dd8b1b26aa1096937f06192d&chksm=f3083d44c47fb452adf9f7ce33f9002fe3ff75183e043627de14780d1ca71fec73b8a96dca8a&scene=21#wechat_redirect)

[Activiti和Flowable源码解析](http://mp.weixin.qq.com/s?__biz=MzI3NTE2NzczMQ==&mid=2650045917&idx=1&sn=5df6ba652eac5dd823f03c27d8cd447d&chksm=f3083d41c47fb45764f2ce567494361823110bbd72918773aa5bfc2df2341711589a7bdb183f&scene=21#wechat_redirect)

今天主要说两点：

1.  更详细的说说Flowable的命令链。
2.  更详细的说说Agenda到底是个什么？

**首先**，从初始化入手：

```Java
org.flowable.engine.impl.cfg.ProcessEngineConfigurationImpl

```
有一段函数执行了Command相关的初始化方法：

```Java
@Override
public void initCommandExecutors() {
    initDefaultCommandConfig();
    initSchemaCommandConfig();
    initCommandInvoker();
    initCommandInterceptors();
    initCommandExecutor();
}

```
从最后的initCommandExecutor()入手，看看调用Api接口时，传入的Command到底是怎么运行的：

```Java
public void initCommandExecutor() {
    if (commandExecutor == null) {
        //先找头，这就是执行的开始
        CommandInterceptor first = initInterceptorChain(commandInterceptors);
        commandExecutor = new CommandExecutorImpl(getDefaultCommandConfig(), first);
    }
}

```
first是什么？其实就是commandInterceptors里面的第一个执行器，如果这中间没有自定义过CommandInterceptor，那第一个执行的就是：

```Java
 public Collection<? extends CommandInterceptor> getDefaultCommandInterceptors() {
     if (defaultCommandInterceptors == null) {
         List<CommandInterceptor> interceptors = new ArrayList<>();
         //这就是那第一个
         interceptors.add(new LogInterceptor());
 
         //省略代码若干...
     }
     return defaultCommandInterceptors;
}

```
而最后一个就是：

```Java
 public void initCommandInterceptors() {
     if (commandInterceptors == null) {
         commandInterceptors = new ArrayList<>();
         if (customPreCommandInterceptors != null) {
             commandInterceptors.addAll(customPreCommandInterceptors);
         }
         //这里决定的是第一个
         commandInterceptors.addAll(getDefaultCommandInterceptors());
         if (customPostCommandInterceptors != null) {
            commandInterceptors.addAll(customPostCommandInterceptors);
        }
        //这里就是最后一个
        commandInterceptors.add(commandInvoker);
    }
}

```
从名字上大概来看，除了最后一个之外，都是用来做辅助的。而最后一个Invoker才是真正的执行器。

**接下来**，通过CommandInvoker就可以看一看一个Command到底是怎么执行的。同时，也可以看一看Agenda到底是个什么。

直接上内部代码：

```Java
 @Override
 @SuppressWarnings("unchecked")
 public <T> T execute(final CommandConfig config, final Command<T> command, CommandExecutor commandExecutor) {
     final CommandContext commandContext = Context.getCommandContext();
 
     FlowableEngineAgenda agenda = CommandContextUtil.getAgenda(commandContext);
     if (commandContext.isReused() && !agenda.isEmpty()) { // there is already an agenda loop being executed
         return (T) command.execute(commandContext);
 
    } else {

        // Execute the command.
        // This will produce operations that will be put on the agenda.
        agenda.planOperation(new Runnable() {

            @Override
            public void run() {
                commandContext.setResult(command.execute(commandContext));
            }
        });

        // Run loop for agenda
        executeOperations(commandContext);

        //此处省略代码若干...
        //只看上面核心的

        return (T) commandContext.getResult();
    }
}

```
直接看else就行了，未特殊设置情况下isReused是false。

可以看到，所有的命令最终都是交给Agenda去执行的，在创建一个Agenda执行计划之后，又不停的去循环执行Agenda里面的执行计划。

所以，最终就落到题目那个问题：Agenda到底是个什么东西？

**最后**，看看Agenda到底是个什么东西。

结论，Agenda就是个链表：

```Java
protected LinkedList<Runnable> operations = new LinkedList<>();

```
所以，理论上来说，Flowable的执行，就是遍历这张链表，执行存储于这张链表内的Runnable对象。

这算扒完了整个过程，**但是**我还是有一些疑问的：

本来我以为它这样设计，可能是因为能够异步执行，这样可以有序执行，不过代码显示的是同步执行，所以肯定不是这个原因。

那只可能是为了封装和解耦了，毕竟把一个一个链条封装好，然后在需要复用的时候，直接压入链表，然后它就会按照设计有序执行，最终得到需要的结果，就很符合它的设计思想。

好了，以上的个人观点，如有错误，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑