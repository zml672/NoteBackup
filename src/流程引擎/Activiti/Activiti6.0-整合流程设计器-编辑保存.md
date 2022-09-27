# 整合Activiti6.0流程设计器-编辑保存 | 字痕随行
紧接着[上一篇](http://www.blackzs.com/archives/1217)，我们来看一下怎么能够保存和发布已经设计好的流程。

先注意一下两个即将用到的类：

1、RepositoryService：Activiti的七大接口之一，主要作用是管理流程仓库，例如部署，删除，读取流程资源等。

2、ObjectMapper：Jackson库的主要类。它提供一些功能，能够将Java对象转换成匹配的JSON结构，反之亦然。

运行一下上一篇已经构建好的工程，随便在设计器上画一个流程，点击保存按钮，输入Model的名称和关键字，然后提交，在开发者工具里面跟踪一下，会发现提交的地址：

```Plain Text
http://localhost:8080/activiti/app/rest/models/null/editor/json

```
再查看一下EditorController的源码，我们会发现需要注意以下路径：

1、创建时：

```Plain Text
/app/rest/models/

```
2、编辑时：

```Plain Text
GET: /app/rest/models/' + modelId + '/editor/json

```
3、保存时：

```Plain Text
POST: /app/rest/models/' + modelId + '/editor/json

```
相对应的，在ActivitiAppRest内也需要接收页面的请求。

创建时，所需要的Controller方法实现比较简单，参考第一篇内静态的Json就可以实现，这里注意的是需要使用RepositoryService提供的newModel()获得一个空的Model对象，代码如下：

```Java
public ObjectNode getModels() {
        Model model = repositoryService.newModel();
        ObjectNode modelNode = objectMapper.createObjectNode();
        modelNode.put("modelId", model.getId());
        modelNode.put("name", model.getName());
        modelNode.put("key", model.getKey());
        modelNode.put("description", "");
        modelNode.putPOJO("lastUpdated", model.getLastUpdateTime());
        ObjectNode editorJsonNode = objectMapper.createObjectNode();
        editorJsonNode.put("id", "canvas");
        editorJsonNode.put("resourceId", "canvas");
        ObjectNode stencilSetNode = objectMapper.createObjectNode();
        stencilSetNode.put("namespace", "http://b3mn.org/stencilset/bpmn2.0#");
        editorJsonNode.put("stencilset", stencilSetNode);
        editorJsonNode.put("modelType", "model");
        modelNode.put("model", editorJsonNode);
        return modelNode;
    }

```
编辑时，需要使用RepositoryService提供的getModel()获得已经存在的Model对象，并且使用getModelEditorSource()获得编辑器的内容。再使用ObjectMapper组装为合适的Json对象。

```Java
public ObjectNode getModelJSON(@PathVariable String modelId) {
        Model model = repositoryService.getModel(modelId);
        ObjectNode modelNode = objectMapper.createObjectNode();
        modelNode.put("modelId", model.getId());
        modelNode.put("name", model.getName());
        modelNode.put("key", model.getKey());
        modelNode.put("description", JSONObject.parseObject(model.getMetaInfo()).getString("description"));
        modelNode.putPOJO("lastUpdated", model.getLastUpdateTime());
        byte[] modelEditorSource = repositoryService.getModelEditorSource(modelId);
        if (null != modelEditorSource && modelEditorSource.length > 0) {
            try {
                ObjectNode editorJsonNode = (ObjectNode) objectMapper.readTree(modelEditorSource);
                editorJsonNode.put("modelType", "model");
                modelNode.put("model", editorJsonNode);
            } catch (Exception e) {
                e.printStackTrace();
            }

        }
        return modelNode;
    }

```
保存时，需要使用RepositoryService提供的saveModel()保存Model对象，并且使用addModelEditorSource()保存编辑器的内容。

```Java
public void saveModel(@PathVariable String modelId, @RequestBody MultiValueMap<String, String> values) {

        String json = values.getFirst("json_xml");
        String name = values.getFirst("name");
        String description = values.getFirst("description");
        String key = values.getFirst("key");

        Model modelData = repositoryService.getModel(modelId);
        if (null == modelData) {
            modelData = repositoryService.newModel();
        }

        ObjectNode modelNode = null;
        try {
            modelNode = (ObjectNode) new ObjectMapper().readTree(json);
        } catch (IOException e) {
            e.printStackTrace();
        }

        ObjectNode modelObjectNode = objectMapper.createObjectNode();
        modelObjectNode.put(ModelDataJsonConstants.MODEL_NAME, name);
        modelObjectNode.put(ModelDataJsonConstants.MODEL_REVISION, 1);
        description = StringUtils.defaultString(description);
        modelObjectNode.put(ModelDataJsonConstants.MODEL_DESCRIPTION, description);
        modelData.setMetaInfo(modelObjectNode.toString());
        modelData.setName(name);
        modelData.setKey(StringUtils.defaultString(key));

        repositoryService.saveModel(modelData);
        try {
            repositoryService.addModelEditorSource(modelData.getId(), modelNode.toString().getBytes("utf-8"));
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }

```
有个需要注意的地方，我们如何在最开始的时候，获得Json的格式？其实也挺简单的，只需要运行Activiti6.0提供的Release包，然后使用开发者工具截获即可。

如果有问题，欢迎指正讨论。

![image](../../images/公众号.jpg)

觉的不错？可以关注我的公众号↑↑↑