# Activiti执行监听器-启动和结束 | 字痕随行
按照Activiti的官方文档，流程的执行监听器可以捕获的事件有：

* 流程实例的启动和结束。
* 选中一条连线。
* 节点的开始和结束。
* 网关的开始和结束。
* 中间事件的开始和结束。
* 开始时间结束或结束事件开始。

在接下来的一段时间内，我会逐一尝试一下，并且通过Demo记录一下整个过程。

首先，我们来尝试捕获一下“流程实例的启动和结束”。下图是一个简单的流程图：

![image](../../images/Activiti6.0-执行监听器-启动和结束/985ac47f3bf42847e9c1185fa5ee2f68.jpg)

声明了两个类：MyStartListener和MyEndListener，各自实现了接口：

```Plain Text
org.activiti.engine.delegate.ExecutionListener

```
MyStartListener的代码如下：

```Java
import org.activiti.engine.delegate.DelegateExecution;
import org.activiti.engine.delegate.ExecutionListener;

public class MyStartListener implements ExecutionListener {

    @Override
    public void notify(DelegateExecution delegateExecution) {
        System.out.println("流程启动");
        System.out.println("EventName:" + delegateExecution.getEventName());
        System.out.println("ProcessDefinitionId:" + delegateExecution.getProcessDefinitionId());
        System.out.println("ProcessInstanceId:" + delegateExecution.getProcessInstanceId());
        System.out.println("=======");
    }
}

```
MyEndListener的代码如下：

```Java
import org.activiti.engine.delegate.DelegateExecution;
import org.activiti.engine.delegate.ExecutionListener;

public class MyEndListener implements ExecutionListener {

    @Override
    public void notify(DelegateExecution delegateExecution) {
        System.out.println("流程结束");
        System.out.println("EventName:" + delegateExecution.getEventName());
        System.out.println("ProcessDefinitionId:" + delegateExecution.getProcessDefinitionId());
        System.out.println("ProcessInstanceId:" + delegateExecution.getProcessInstanceId());
        System.out.println("=======");
    }
}

```
在流程设计器中进行相应的配置，如下图：

![image](../../images/Activiti6.0-执行监听器-启动和结束/97783c5306b01f8d931759e4cbb5a1c4.jpg)



*Listener配置*

![image](../../images/Activiti6.0-执行监听器-启动和结束/bcc3f261e1019ab6a109ecc21f4f7b6b.jpg)



*StartListener配置*

![image](../../images/Activiti6.0-执行监听器-启动和结束/a8d383c8133d4ece6f6b1e0eb37a1f03.jpg)



*EndListener配置*

启动这个流程，控制台会输出：

![image](../../images/Activiti6.0-执行监听器-启动和结束/9654760cb453c1f433be3340a2c637df.jpg)



*流程启动时StartEvent触发*

![image](../../images/Activiti6.0-执行监听器-启动和结束/231be366914cf2f8afe1d9c035d3f9d3.jpg)



*流程结束时EndEvent触发*

到此，流程的Start和End事件全部触发完毕，至于DelegateExecution内的方法都是什么含义，将会在之后带来。

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑