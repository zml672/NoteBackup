# Lync开发-Conversation | 字痕随行

本文将介绍Lync Client SDK中的Conversation类。类Conversation属于命名空间Microsoft.Lync.Model.Conversation。它描述了会话，包括了一些详细信息，如会话参与者、收发模式（即时信息，音频/视频）、状态等。并且实现了合并、终止等其它会话动作。它的重要成员如下：

| |名称|说明|
| ----- | ----- | ----- |
|方法|AddParticipant(Contact)|将Contact添加到该会话中。|
|方法|BeginSetProperty|设置此Conversation的属性。|
|方法|RemoveParticipant|移除某一个参与者。|
|方法|End|终止在本地端点上的会话。如果会话是一个会议，对其他参与者该会议将继续有效。|
|属性|ConversationManager|获取此会话的父级会话管理器。|
|属性|Modalities|获取会话模式的集合。例如，即时消息模式或音频/视频模式。|
|属性|Participants|获取参与者集合。|
|属性|SelfParticipant|获取作为参与者的当前登录用户。|
|属性|State|获取作为参与者的当前登录用户。|
|事件|ParticipantAdded|获取作为参与者的当前登录用户。|
|事件|ParticipantRemoved|当参与者从会话中被移除时发生。|

## 如何创建会话

可以通过Microsoft Lync SDK提供的API接口创建会话，主要的步骤如下：

1. 登录Lync客户端，并且获取LyncClient实例。
2. 为LyncClient实例注册事件ConversationAdded。
3. 通过读取LyncClient对象的属性ConversationManager获取ConversationManager实例。
4. 调用方法AddConversation。

简单的示例代码如下：
```c#
lyncClient = LyncClient.GetClient();
lyncClient. ConversationAdded += ConversationManager_ConversationAdded
conversation = lyncClient. ConversationManager.AddConversation();
void ConversationManager_ConversationAdded(object sender, ConversationManagerEventArgs e)
{
    //新增会话后触发事件，可在此添加会话参与者
}
```

当会话创建之后，会话中只有创建者本人，一个完整的会话至少还需要一名参与者，添加其他参与者的示例代码如下：
```c#
contact = lyncClient. ContactManager.GetContactByUri(strSIP);
conversation.ParticipantAdded += Conversation_ParticipantAdded;
conversation.AddParticipant(contact);
void Conversation_ParticipantAdded(object sender, ParticipantCollectionChangedEventArgs e)
{
}
```

会话创建完毕后，其他参与者还暂时无法知道此会话的存在，需要发送一条消息通知其他参与者，其他参与者在收到这条消息时，就会触发自身的ConversationAdded事件。

MSDN参考资料：[Get started with Lync conversations](http://msdn.microsoft.com/en-us/library/jj933207(v=office.15).aspx "Get started with Lync conversations")

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑