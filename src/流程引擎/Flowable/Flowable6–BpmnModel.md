# Flowable6.4 – BpmnModel | 字痕随行
​在Flowable的官方文档中，有一段这样的描述：

*在V6中，所有流程定义的信息都可以通过BpmnModel\_获取。这是一个BPMN 2.0 XML流程定义的Java表现形式（并对特定操作及搜索进行了增强）。*

这一次就看一看BpmnModel到底能够干什么。

**如何在一个已知的流程定义中获得BpmnModel呢？**

已有模型标识，获得BpmnModel：

```Java
byte[] modelEditorSource = repositoryService.getModelEditorSource(modelId);
JsonNode editorNode = new ObjectMapper().readTree(modelEditorSource);
BpmnJsonConverter jsonConverter = new BpmnJsonConverter();
BpmnModel bpmnModel = jsonConverter.convertToBpmnModel(editorNode);

```
最快的办法，通过流程定义ID获得BpmnModel:

```Java
BpmnModel bpmnModel = repositoryService.getBpmnModel(myProcessDefinitionId);

```
**获得BpmnModel后，可以做什么呢？**

发布流程：

```Java
BpmnModel model = new BpmnJsonConverter().convertToBpmnModel(modelNode);
repositoryService.createDeployment().name("test").addBpmnModel("test.bpmn20.xml", model).deploy();

```
导出流程定义：

```Java
BpmnXMLConverter xmlConverter = new BpmnXMLConverter();
byte[] exportBytes = xmlConverter.convertToXML(bpmnModel);

```
获得流程节点信息：

```Java
Process process = bpmnModel.getMainProcess();
Collection<FlowElement> flowElements = process.getFlowElements();
List<UserTask> userTasks = new ArrayList<>();
for (FlowElement flowElement : flowElements) {
    if (flowElement instanceof UserTask) {
        UserTask userTask = (UserTask)flowElement;
        System.out.println(userTask.getId() + ":" + userTask.getName());
    }
}

```
获得流程图坐标信息：

```Java
//获得流程节点信息
Map<String, GraphicInfo> locationMap = bpmnModel.getLocationMap();
//获得流程节点之间连线信息
Map<String, List<GraphicInfo>> flowLocationMap = bpmnModel.getFlowLocationMap();

```
以上就是BpmnModel的相关介绍，如有问题欢迎指正。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑