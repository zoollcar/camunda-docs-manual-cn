---

title: '读取模型'
weight: 10

menu:
  main:
    identifier: "user-guide-bpmn-model-api-reading-a-model"
    parent: "user-guide-bpmn-model-api"

---


如果你已经创建了一个BPMN模型，并想通过BPMN模型API来处理它，你可以用以下方法导入它。

```java
// 从文件中读取一个模型
File file = new File("PATH/TO/MODEL.bpmn");
BpmnModelInstance modelInstance = Bpmn.readModelFromFile(file);

// 从流中读取一个模型
InputStream stream = [...]
BpmnModelInstance modelInstance = Bpmn.readModelFromStream(stream);
```

在导入你的模型后，你可以通过他们的id或元素的类型来搜索元素。

```java
// 通过id获取模型实例
StartEvent start = (StartEvent) modelInstance.getModelElementById("start");

// 找到所有task类型的元素
ModelElementType taskType = modelInstance.getModel().getType(Task.class);
Collection<ModelElementInstance> taskInstances = modelInstance.getModelElementsByType(taskType);
```

对于每个元素实例，你现在可以读取和编辑属性值。

你可以通过使用辅助方法或通用的XML模型API来实现。

如果你在BPMN元素中添加了自定义属性，你可以始终使用通用的XML模型API来访问它们。

```java
StartEvent start = (StartEvent) modelInstance.getModelElementById("start");

// 通过辅助方法读取属性
String id = start.getId();
String name = start.getName();

// 通过辅助方法编辑属性
start.setId("new-id");
start.setName("new name");

// 通过通用的XML模型API读取属性（有可选的命名空间）。
String custom1 = start.getAttributeValue("custom-attribute");
String custom2 = start.getAttributeValueNs("custom-attribute-2", "http://camunda.org/custom");

// 通过通用的XML模型API编辑属性（有可选的命名空间）。
start.setAttributeValue("custom-attribute", "new value");
start.setAttributeValueNs("custom-attribute", "http://camunda.org/custom", "new value");
```

你也可以访问一个元素的子元素或对其他元素的引用。

例如，一个序列流引用了一个源元素和一个目标元素，而一个流节点（如开始事件、任务等）有子元素用于传入和传出的序列流。

例如，下面的BPMN模型是由BPMN模型API创建的，作为一个简单流程的例子。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<definitions targetNamespace="http://camunda.org/examples" xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL">
  <process id="process-with-one-task">
    <startEvent id="start">
      <outgoing>start-task1</outgoing>
    </startEvent>
    <userTask id="task1">
      <incoming>start-task1</incoming>
      <outgoing>task1-end</outgoing>
    </userTask>
    <endEvent id="end">
      <incoming>task1-end</incoming>
    </endEvent>
    <sequenceFlow id="start-task1" sourceRef="start" targetRef="task1"/>
    <sequenceFlow id="task1-end" sourceRef="task1" targetRef="end"/>
  </process>
</definitions>
```

现在你可以使用BPMN模型API来获得ID为*start-task1*的序列流的源和目标流节点。

```java
// read bpmn model from file
BpmnModelInstance modelInstance = Bpmn.readModelFromFile(new File("/PATH/TO/MODEL.bpmn"));

// find sequence flow by id
SequenceFlow sequenceFlow = (SequenceFlow) modelInstance.getModelElementById("start-task1");

// get the source and target element
FlowNode source = sequenceFlow.getSource();
FlowNode target = sequenceFlow.getTarget();

// get all outgoing sequence flows of the source
Collection<SequenceFlow> outgoing = source.getOutgoing();
assert(outgoing.contains(sequenceFlow));
```

有了这些引用，你可以很容易地为不同的使用情况创建辅助方法。

例如，如果你想找到一个任务或网关的流程节点，你可以使用下面这样一个辅助方法。

```java
public Collection<FlowNode> getFlowingFlowNodes(FlowNode node) {
  Collection<FlowNode> followingFlowNodes = new ArrayList<FlowNode>();
  for (SequenceFlow sequenceFlow : node.getOutgoing()) {
    followingFlowNodes.add(sequenceFlow.getTarget());
  }
  return followingFlowNodes;
}
```

