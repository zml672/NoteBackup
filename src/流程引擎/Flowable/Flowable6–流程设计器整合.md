# 流程设计器整合 | 字痕随行
之前只有整合教程，而没有相关的整合代码。这周花了点时间，整理了一下，开源了两个项目。

**第一个项目**

基于SpringMVC整合了Activiti的流程设计器，开源地址如下：

[https://gitee.com/blackzs/activiti-designer](https://gitee.com/blackzs/activiti-designer)

相关的整合教程如下：

[整合Activiti6.0Web流程设计器](http://www.blackzs.com/archives/1217)

[整合Activiti6.0流程设计器-编辑保存](http://www.blackzs.com/archives/1244)

[整合Activiti6.0流程设计器-发布和运行](http://www.blackzs.com/archives/1248)

运行时说明如下：

```Plain Text
启动后的入口地址
http://domain:port/activiti/editor/index.html#/editor/

保存后修改流程的地址
http://domain:port/activiti/editor/index.html#/editor/{modelId}

启动一个流程
http://domain:port/flow/start/{modelId}

完成一个任务
http://domain:port/flow/complate/{taskId}

```
**第二个项目**

基于SpringBoot整合了Flowable6.4的流程设计器，这个整合不需要依赖Flowable的idm，开源地址如下：

[https://gitee.com/blackzs/flowable-designer](https://gitee.com/blackzs/flowable-designer)

相关的整合教程如下：

[SpringBoot整合Flowable6.4](http://www.blackzs.com/archives/1523)

[Flowable6.4 - 整合流程设计器](http://www.blackzs.com/archives/1557)

运行时说明如下：

```Plain Text
启动后的入口地址
http://domain:port/designer/editor/index.html#/editor/

保存后修改流程的地址
http://domain:port/designer/editor/index.html#/editor/{modelId}

启动一个流程
http://domain:port/flow/start/{modelId}

完成一个任务
http://domain:port/flow/complate/{taskId}

```
其它的相关教程可以参见：[流程引擎大杂烩](http://www.blackzs.com/archives/1306)

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑