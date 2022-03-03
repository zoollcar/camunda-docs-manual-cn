---

title: '创建模型'
weight: 20

menu:
  main:
    identifier: "user-guide-bpmn-model-api-creating-a-model"
    parent: "user-guide-bpmn-model-api"

---


为了从头创建一个新的BPMN模型，你需要用以下方法创建一个空的BPMN模型实例。

```java
BpmnModelInstance modelInstance = Bpmn.createEmptyModel();
```

下一步是创建一个BPMN的定义元素（`bpmn:definitions`）。在定义上设置`targetNamespace`并将其添加到新创建的空模型实例。

```java
Definitions definitions = modelInstance.newInstance(Definitions.class);
definitions.setTargetNamespace("http://camunda.org/examples");
modelInstance.setDefinitions(definitions);
```
> 译者注：
>
> 生成出来的xml类似于此
>
> ```xml
> <?xml version="1.0" encoding="UTF-8" standalone="no"?>
> <definitions id="definitions_a94e86f0-3b8e-4aab-acdc-0baeba3d5dd0" targetNamespace="http://camunda.org/examples" xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"/>
> 
> ```



通常情况下，你想在模型中添加一个`process`过程。这与创建BPMN定义元素的3个步骤相同：

1. 创建一个BPMN元素的新实例
2. 设置元素实例的属性和子元素
3. 将新创建的元素实例添加到相应的父元素中

```java
Process process = modelInstance.newInstance(Process.class);
process.setId("process");
definitions.addChildElement(process);
```

> 译者注：
>
> 生成出来的xml类似于此
>
> ```xml
> <?xml version="1.0" encoding="UTF-8" standalone="no"?>
> <definitions id="definitions_2cc92107-3cde-45b2-86c8-11b0e88f5903" targetNamespace="http://camunda.org/examples" xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL">
>   <process id="process"/>
> </definitions>
> ```
>
> 如果不手动指定`process id`，会生成默认`process id` 类似于：`  <process id="process_a35ec973-16ad-4883-82be-c79de5d00b20"/>`

为了简化这个重复的程序，你可以使用一个像这样的辅助方法。

```java
protected <T extends BpmnModelElementInstance> T createElement(BpmnModelElementInstance parentElement, String id, Class<T> elementClass) {
  T element = parentElement.getModelInstance().newInstance(elementClass);
  element.setAttributeValue("id", id, true);
  parentElement.addChildElement(element);
  return element;
}
```

在你创建了流程元素，如开始事件、任务、网关和结束事件后，你必须用序列流去连接元素与序列流。

同样的，这也是按照创建元素的3个步骤进行的，可以通过以下的辅助方法来简化

```java
public SequenceFlow createSequenceFlow(Process process, FlowNode from, FlowNode to) {
  String identifier = from.getId() + "-" + to.getId();
  SequenceFlow sequenceFlow = createElement(process, identifier, SequenceFlow.class);
  process.addChildElement(sequenceFlow);
  sequenceFlow.setSource(from);
  from.getOutgoing().add(sequenceFlow);
  sequenceFlow.setTarget(to);
  to.getIncoming().add(sequenceFlow);
  return sequenceFlow;
}
```

根据BPMN 2.0规范验证模型，并将其转换成一个XML字符串或将其保存到一个文件或流中。

```java
// 验证模型
Bpmn.validateModel(modelInstance);	

// 转换成字符串
String xmlString = Bpmn.convertToString(modelInstance);

// 写到一个输入流中
OutputStream outputStream = new OutputStream(...);
Bpmn.writeModelToStream(outputStream, modelInstance);

// 写到一个文件中
File file = new File(...);
Bpmn.writeModelToFile(file, modelInstance);
```

# 示例 1: 用一个用户任务创建一个简单的流程

有了上面的辅助方法，创建简单的流程是非常容易和直接的。

首先，创建一个流程，包括一个开始事件、用户任务和一个结束事件。

{{< img src="../img/bpmn-model-api-simple-process.png" title="Single User Task Example" >}}

下面的代码使用上面的帮助方法创建了这个过程（没有DI元素）。

```java
// 创建一个空白模型
BpmnModelInstance modelInstance = Bpmn.createEmptyModel();
Definitions definitions = modelInstance.newInstance(Definitions.class);
definitions.setTargetNamespace("http://camunda.org/examples");
modelInstance.setDefinitions(definitions);

// 创建流程元素
Process process = createElement(definitions, "process-with-one-task", Process.class);

// 创建开始事件，用户任务，结束事件
StartEvent startEvent = createElement(process, "start", StartEvent.class);
UserTask task1 = createElement(process, "task1", UserTask.class);
task1.setName("User Task");
EndEvent endEvent = createElement(process, "end", EndEvent.class);

// 创建各个元素之间的连接
createSequenceFlow(process, startEvent, task1);
createSequenceFlow(process, task1, endEvent);

// 验证模型并写入到文件中
Bpmn.validateModel(modelInstance);
File file = File.createTempFile("bpmn-model-api-", ".bpmn");
Bpmn.writeModelToFile(file, modelInstance);
```


# 示例 2: 创建有两个并行任务的简单流程

更复杂的流程也可以通过标准的BPMN模型API用几行代码来创建。

{{< img src="../img/bpmn-model-api-parallel-gateway.png" title="Parallel Task Example" >}}

```java
// create an empty model
BpmnModelInstance modelInstance = Bpmn.createEmptyModel();
Definitions definitions = modelInstance.newInstance(Definitions.class);
definitions.setTargetNamespace("http://camunda.org/examples");
modelInstance.setDefinitions(definitions);

// create elements
StartEvent startEvent = createElement(process, "start", StartEvent.class);
ParallelGateway fork = createElement(process, "fork", ParallelGateway.class);
ServiceTask task1 = createElement(process, "task1", ServiceTask.class);
task1.setName("Service Task");
UserTask task2 = createElement(process, "task2", UserTask.class);
task2.setName("User Task");
ParallelGateway join = createElement(process, "join", ParallelGateway.class);
EndEvent endEvent = createElement(process, "end", EndEvent.class);

// create flows
createSequenceFlow(process, startEvent, fork);
createSequenceFlow(process, fork, task1);
createSequenceFlow(process, fork, task2);
createSequenceFlow(process, task1, join);
createSequenceFlow(process, task2, join);
createSequenceFlow(process, join, endEvent);

// validate and write model to file
Bpmn.validateModel(modelInstance);
File file = File.createTempFile("bpmn-model-api-", ".bpmn");
Bpmn.writeModelToFile(file, modelInstance);
```
