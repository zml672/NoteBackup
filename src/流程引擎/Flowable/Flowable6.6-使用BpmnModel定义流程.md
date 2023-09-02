# Flowable6.6-使用BpmnModel定义流程 | 字痕随行

基于之前的文章，可以得出一个结论：通过BpmnModel可以使用编程方式完成流程定义。

本篇就简单介绍一下如何实现。首先，创建一个的BpmnModel，包含完整的流程定义，代码如下：

```java
private BpmnModel createBpmnModel() {
    BpmnModel bpmnModel = new BpmnModel();
    //开始事件
    StartEvent startEvent = new StartEvent();
    startEvent.setId("startEvent");
    startEvent.setName("开始");
    //用户任务节点
    UserTask userTask = new UserTask();
    userTask.setId("userTask1");
    userTask.setName("用户任务1");
    userTask.setCategory("测试");
    //结束事件
    EndEvent endEvent = new EndEvent();
    endEvent.setId("endEvent");
    endEvent.setName("结束");
    //两条连接线
    SequenceFlow sequenceFlow1 = new SequenceFlow(startEvent.getId(), userTask.getId());
    sequenceFlow1.setId("sequenceFlow1");
    SequenceFlow sequenceFlow2 = new SequenceFlow(userTask.getId(), endEvent.getId());
    sequenceFlow2.setId("sequenceFlow2");
    //主流程
    Process process = new Process();
    process.setId("bpmnModelTestProcess");
    process.setName("bpmn模型测试流程");
    process.setExecutable(true);
    //把流程元素加入到主流程
    process.addFlowElement(startEvent);
    process.addFlowElement(userTask);
    process.addFlowElement(endEvent);
    process.addFlowElement(sequenceFlow1);
    process.addFlowElement(sequenceFlow2);
    //把主流程加入到BpmnModel中
    bpmnModel.addProcess(process);
    //使用自动排版功能排版，同时定义流程元素的BpmnDI
    new BpmnAutoLayout(bpmnModel).execute();

    return bpmnModel;
}
```
然后就是保存发布的代码了：

```java
//将bpmn转成xml文件格式
BpmnXMLConverter xmlConverter = new BpmnXMLConverter();
byte[] bytes = xmlConverter.convertToXML(bpmnModel);
//发布流程
Deployment deployment = repositoryService.createDeployment()
        .name(bpmnModel.getMainProcess().getName())
        .key(bpmnModel.getMainProcess().getId())
        .category("测试")
        .addString(bpmnModel.getMainProcess().getName() + ".bpmn20.xml", new String(bytes))
        .deploy();
//创建流程模型
Model model = repositoryService.newModel();
ObjectNode modelObjectNode = objectMapper.createObjectNode();
modelObjectNode.put(ModelDataJsonConstants.MODEL_NAME, bpmnModel.getMainProcess().getName());
modelObjectNode.put(ModelDataJsonConstants.MODEL_REVISION, 1);
modelObjectNode.put(ModelDataJsonConstants.MODEL_DESCRIPTION, "测试");
model.setMetaInfo(modelObjectNode.toString());
model.setName(bpmnModel.getMainProcess().getName());
model.setKey(bpmnModel.getMainProcess().getName());
model.setCategory("测试");
model.setDeploymentId(deployment.getId());
repositoryService.saveModel(model);
repositoryService.addModelEditorSource(model.getId(), bytes);
```
然后就可以用接口来测试了，复杂点的也是这样定义，换汤不换药。

以上，如果有错误，欢迎探讨和指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑

