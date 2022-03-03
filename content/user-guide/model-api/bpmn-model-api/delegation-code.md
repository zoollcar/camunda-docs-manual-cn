---

title: '委托代码'
weight: 20

menu:
  main:
    identifier: "user-guide-bpmn-model-api-delegation"
    parent: "user-guide-bpmn-model-api"

---


如果你使用 [委托代码]({{< ref "/user-guide/process-engine/delegation-code.md" >}}), 你可以访问BPMN模型实例和当前执行流程的元素。如果访问BPMN模型，它将被缓存，以避免冗余的数据库查询。

# Java 代理类

如果你的类实现了`org.camunda.bpm.engine.delegate.JavaDelegate`接口，你可以访问BPMN模型实例和当前的流程元素。

在下面的例子中，`JavaDelegate`被添加到BPMN模型的一个服务任务中。因此，返回的流程元素可以被转换为`ServiceTask`。

```java
public class ExampleServiceTask implements JavaDelegate {

  public void execute(DelegateExecution execution) throws Exception {
    BpmnModelInstance modelInstance = execution.getBpmnModelInstance();
    ServiceTask serviceTask = (ServiceTask) execution.getBpmnModelElementInstance();
  }
}
```

# 执行监听器

如果你的类实现了`org.camunda.bpm.engine.delegate.ExecutionListener`接口，你可以访问BPMN模型实例和当前的流程元素。

由于一个执行监听器可以被添加到多个元素中，如流程、事件、任务、网关和序列流，因此不能保证流元素的类型。

```java
public class ExampleExecutionListener implements ExecutionListener {

  public void notify(DelegateExecution execution) throws Exception {
    BpmnModelInstance modelInstance = execution.getBpmnModelInstance();
    FlowElement flowElement = execution.getBpmnModelElementInstance();
  }
}
```

# 任务监听器

如果你的类实现了`org.camunda.bpm.engine.delegate.TaskListener`接口，你可以访问BPMN模型实例和当前的用户任务，因为任务监听器只能被添加到用户任务中。

```java
public class ExampleTaskListener implements TaskListener {

  public void notify(DelegateTask delegateTask) {
    BpmnModelInstance modelInstance = delegateTask.getBpmnModelInstance();
    UserTask userTask = delegateTask.getBpmnModelElementInstance();
  }
}
```
