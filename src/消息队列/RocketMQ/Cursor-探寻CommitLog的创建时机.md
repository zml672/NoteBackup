# Cursor-探寻CommitLog的创建时机 | 字痕随行

事情的起源来自于我看一篇介绍RocketMQ CommitLog的文章。

大多数文章介绍这部分都会提到，CommitLog由多个磁盘文件组成，每个文件的最大容量为1G，当一个文件写满后就会创建一个新的文件继续写。

我在这里比较好奇的是，当一个新的消息到达的时候，它的处理规则是哪种：

1. 直接追加，当追加到满1G的时候，创建一个新的文件，然后在新的文件中写入剩余的消息。
2. 先判断，如果当前的文件容量已经不足以写入整个消息时，创建一个新的文件，将完整的消息写入新的文件。

我倾向于第二种，因为所有的介绍都会说“最大”、“至多”，而没有说“相等”。

但是，我查了一下资料，却没有找到特定的答案，所以只能去看源码了。

看源码很累，我又不想耗费很多的时间在这上面，所以我想让Cursor帮助我，看看能不能节省一些时间。

首先，我已经知道了CommitLog的位置，并且知道写入消息的方法：

```java
//CommitLog.java在store这个包内
public CompletableFuture<PutMessageResult> asyncPutMessage(final MessageExtBrokerInner msg) {}
```
然后，找到相关的代码：

```java
result = mappedFile.appendMessage(msg, this.appendMessageCallback, putMessageContext);
```
选中这段代码，打开chat窗口，询问这段代码的作用，以下是对这段代码的解释：

```Plain Text
所选代码块是rocketmq项目中CommitLog.java文件中的一行代码。具体来说，它在mappedFile对象上调用appendMessage方法，将msg、this.appendMessageCallback和putMessageContext作为参数传递。

根据前面提供的代码块中提供的上下文，似乎这段代码是处理将消息存储在提交日志中的更大方法的一部分。appendMessage方法可能负责实际将消息写入磁盘。

没有更多的上下文，很难说这行代码在做什么或其目的是什么。但是，它似乎是rocketmq项目中消息存储过程的重要部分。
```
然后，看看返回的result代表什么？

```Plain Text
根据代码中的switch语句，result.getStatus()可能返回以下值：

- APPENDOK：消息追加成功
- ENDOFFILE：当前MappedFile已满，需要创建新的MappedFile
- MESSAGESIZEEXCEEDED：消息大小超过了最大限制
- PROPERTIESSIZEEXCEEDED：消息属性大小超过了最大限制
- UNKNOWNERROR：未知错误发生
```
上文中的appendMessage()最终会调用：

```java
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb,
        PutMessageContext putMessageContext) {}
```
这个方法里面的关键点是：

```java
int currentPos = WROTE_POSITION_UPDATER.get(this);
```
对这段代码的解释如下：

```Plain Text
用户问题：  
int currentPos 这个变量有什么作用？  

回答：  
在所选代码块中，currentPos 变量用于跟踪映射文件中的当前写入位置。它通过调用 WROTE_POSITION_UPDATER 对象的 get 方法获得，该对象是一个 AtomicIntegerFieldUpdater，用于更新 DefaultMappedFile 类中的 wrotePosition 字段。  
当 currentPos 小于 fileSize 时，意味着映射文件中仍有空间可以写入更多数据。然后，appendMessage 方法创建一个 ByteBuffer 对象，该对象从 currentPos 开始，容量为 fileSize - currentPos。这个 ByteBuffer 与要写入的消息数据一起传递给 AppendMessageCallback 对象的 doAppend 方法。  
在消息数据写入映射文件后，使用 WROTE_POSITION_UPDATER 对象的 addAndGet 方法更新 wrotePosition 字段，并使用 doAppend 方法返回的时间戳更新 storeTimestamp 字段。最后，返回 AppendMessageResult 对象。
```
再看cb.append()方法里面的maxBlank参数，这个参数的赋值如下：

```java
this.fileSize - currentPos
```
相当于是当前CommitLog这个文件的剩余容量，之后在cb.append()内部执行的时候，就会看msg的大小是否在剩余容量之内：

```java
// Determines whether there is sufficient free space
if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
    this.msgStoreItemMemory.clear();
    // 1 TOTALSIZE
    this.msgStoreItemMemory.putInt(maxBlank);
    // 2 MAGICCODE
    this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
    // 3 The remaining space may be any value
    // Here the length of the specially set maxBlank
    final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
    byteBuffer.put(this.msgStoreItemMemory.array(), 0, 8);
    return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset,
        maxBlank, /* only wrote 8 bytes, but declare wrote maxBlank for compute write position */
        msgIdSupplier, msgInner.getStoreTimestamp(),
        queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
}
```
所以，最后得出的结论就是：如果msg的大小超出了当前CommitLog的剩余容量，就会重新建立一个文件，将msg完整的追加到新的文件内。

说一下体会吧，大概就是连自身的提炼总结都省了，直接划拉上代码开问，给出的答案足够指引自身获得目标结果，能够节省大量，可能会喜欢上读代码。

以上，如果有错误，欢迎探讨和指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑