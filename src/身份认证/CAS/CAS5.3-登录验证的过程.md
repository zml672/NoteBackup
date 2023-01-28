# CAS5.3-登录验证的过程 | 字痕随行

之前就已经介绍过，CAS Server是通过Spring Web Flow来处理请求的。

在本篇文章中，不会具体介绍Spring Web Flow的使用方式，基本概念的话需要各位自行了解，只会在必须说明的时候会带出来一些介绍。

我很好奇的是，如果我在地址栏输入：

```Plain Text
http://localhost:8080/cas/login
```
如果没有登录过，浏览器内会显示登录页面，如果已经登录过，浏览器会显示登录成功页面。

为什么同一个地址会显示两个页面，而这背后的处理过程是什么呢？

首先，我建议把CAS的源代码下载下来，这样便于跟踪阅读，当然编辑器的加载过程十分漫长，而且有可能会失败，甚至会内存溢出，做好心理准备。

然后，在cas-server.core.cas-server-core-webflow-api中找到类

```java
DefaultLoginWebflowConfigurer
```
至于为什么一上来就能找到这个类，那肯定是经过了无数次的反推分析的，这个过程我懒得记录了。

这个类就是Webflow的配置入口，从下面这段代码就可以看到Webflow的配置了：

```java
@Override
protected void doInitialize() {
    final Flow flow = getLoginFlow();

    if (flow != null) {
        createInitialFlowActions(flow);
        createDefaultGlobalExceptionHandlers(flow);
        //这里是结束的状态，包含了登录成功的页面
        createDefaultEndStates(flow);
        createDefaultDecisionStates(flow);
        createDefaultActionStates(flow);
        createDefaultViewStates(flow);
        createRememberMeAuthnWebflowConfig(flow);
        //这就是一切的开始，找到这个就找到了线头
        setStartState(flow, CasWebflowConstants.STATE_ID_INITIAL_AUTHN_REQUEST_VALIDATION_CHECK);
    }
}
```
这里面有一个知识点明白了可能就容易点。把Webflow想想成状态机，一个状态流转到另外一个状态，每个状态有一个处理的Action，Action返回的结果决定了目标状态。

了解了上面的知识点，就可以找到初始状态对应的Action：

```java
public class InitialAuthenticationRequestValidationAction extends AbstractAction {}
```
追一追doExecute()，就可以找到下面这段代码：

```java
@Override
public Set<Event> resolveInternal(final RequestContext context) {
    final String tgt = WebUtils.getTicketGrantingTicketId(context);
    final RegisteredService service = WebUtils.getRegisteredService(context);
    //service在本文讨论的场景下永远为空
    if (service == null) {
        LOGGER.debug("No service is available to determine event for principal");
        //这方法永远返回Success
        return resumeFlow();
    }
}
```
返回Success代表什么呢？代表刚才提到的知识点，看看结果是Success的时候，要到哪个状态去。

这直接看看代码，流转到：

```java
CasWebflowConstants.STATE_ID_TICKET_GRANTING_TICKET_CHECK
```
然后去看TicketGrantingTicketCheckAction的doExecute()方法：

```java
@Override
public Event doExecute(final RequestContext requestContext) {
    //获取cookie里面的tgt
    final String tgtId = WebUtils.getTicketGrantingTicketId(requestContext);
    if (StringUtils.isBlank(tgtId)) {
        //没有到这个状态去
        return new Event(this, CasWebflowConstants.TRANSITION_ID_TGT_NOT_EXISTS);
    }
    try {
        final Ticket ticket = this.centralAuthenticationService.getTicket(tgtId, Ticket.class);
        if (ticket != null && !ticket.isExpired()) {
            //有就到这个状态去
            return new Event(this, CasWebflowConstants.TRANSITION_ID_TGT_VALID);
        }
    } catch (final AbstractTicketException e) {
        LOGGER.trace("Could not retrieve ticket id [{}] from registry.", e.getMessage());
    }
    //最终到这个状态去
    return new Event(this, CasWebflowConstants.TRANSITION_ID_TGT_INVALID);
}
```
好了，老一套循环了，其实这就分开了，不同的状态跳转到不同的处理，返回不同的页面。

如果继续向下追到底的话，就是开篇问题的答案了。

以上，如有错误，欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑