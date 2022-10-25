# 命令和责任链模式 | 字痕随行
之前关于Activiti和Flowable的示例，都会实现Command接口，通过命令和责任链模式执行。

下面是命令和责任链模式的解释：

命令模式（Command Pattern）是一种数据驱动的设计模式，它属于行为型模式。请求以命令的形式包裹在对象中，并传给调用对象。调用对象寻找可以处理该命令的合适的对象，并把该命令传给相应的对象，该对象执行命令。

责任链模式（Chain of Responsibility Pattern）为请求创建了一个接收者对象的链。这种模式给予请求的类型，对请求的发送者和接收者进行解耦。这种类型的设计模式属于行为型模式。在这种模式中，通常每个接收者都包含对另一个接收者的引用。如果一个对象不能处理该请求，那么它会把相同的请求传给下一个接收者，依此类推。

读起来有些拗口，我个人理解就是把一段业务逻辑封装在一个类内，然后按照预设的顺序执行这些类的对象，从而实现更复杂的业务逻辑。

优点自然是解耦和可扩展。  

下面是一段简单的示例代码：

首先，声明一个接口：

```java
public interface Command {

    void excute();

    void setNext(Command command);

    Command getNext();
}

```
接着，使用一个抽象类实现此接口，并实现一些通用的方法：

```java
public abstract class AbstractCommand implements Command {

    Command nextCommand;

    @Override
    public void setNext(Command command) {
        this.nextCommand = command;
    }

    @Override
    public Command getNext() {
        return this.nextCommand;
    }
}

```
声明一个日志类，实现此抽象类，模拟日志输出：

```java
public class LogCommandImpl extends AbstractCommand {

    @Override
    public void excute() {
        System.out.println("开始执行命令");
        if (null != this.nextCommand) {
            this.nextCommand.excute();
        }
    }
}

```
在声明一个业务类，也实现此抽象类，模拟业务逻辑的执行：

```java
public class ServiceCommand extends AbstractCommand {

    @Override
    public void excute() {
        System.out.println("执行业务逻辑1+1=2");
        if (null != this.nextCommand) {
            this.nextCommand.excute();
        }
    }
}

```
最后，组装这两个命令，让它们按规则运行，比如：

```java
public class CommandTest {

    public static void main(String[] args) {
        Command serviceCmd = new ServiceCommand();
        Command logCmd = new LogCommandImpl();
        logCmd.setNext(serviceCmd);
        logCmd.excute();
    }
}

```
输出的结果为：

```Plain Text
Connected to the target VM, address: '127.0.0.1:55261', transport: 'socket'
开始执行命令
执行业务逻辑1+1=2
Disconnected from the target VM, address: '127.0.0.1:55261', transport: 'socket'

```
如果想要前后都加上日志输出，可以这样：

```java
public class CommandTest {

    public static void main(String[] args) {
        Command serviceCmd = new ServiceCommand();
        Command logCmd1 = new LogCommandImpl();
        Command logCmd2 = new LogCommandImpl();

        logCmd1.setNext(serviceCmd);
        serviceCmd.setNext(logCmd2);

        logCmd1.excute();
    }
}

```
输出的结果变成了：

```Plain Text
Connected to the target VM, address: '127.0.0.1:55279', transport: 'socket'
开始执行命令
执行业务逻辑1+1=2
开始执行命令
Disconnected from the target VM, address: '127.0.0.1:55279', transport: 'socket'

```
本文简单的说明和试验一下命令和职责链模式，便于接下来解读Activiti和Flowable的源码。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑