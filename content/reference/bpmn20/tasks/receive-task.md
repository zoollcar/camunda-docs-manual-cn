---

title: 'Receive Task 接收消息任务'
weight: 60

menu:
  main:
    identifier: "bpmn-ref-tasks-receive-task"
    parent: "bpmn-ref-tasks"
    pre: "Wait for a message to arrive."

---

一个接收任务是一个简单的任务，它等待某个消息的到来。当进程的执行到达一个接收任务时，进程的状态被提交到持久性存储中。这意味着进程将一直处于这种等待状态，直到引擎收到一个特定的消息，从而触发进程在接收任务之外的继续执行。

{{< bpmn-symbol type="receive-task" >}}

一个有消息参考的接收任务可以像普通事件一样被触发。

```xml
<definitions ...>
  <message id="newInvoice" name="newInvoiceMessage"/>
  <process ...>
    <receiveTask id="waitState" name="wait" messageRef="newInvoice">
  ...
```

然后你可以将它和消息关联起来。

```java
// 关联消息
runtimeService.createMessageCorrelation(subscription.getEventName())
  .processInstanceBusinessKey("AB-123")
  .correlate();
```

或者明确地查询订阅并触发它：

```java
ProcessInstance pi = runtimeService.startProcessInstanceByKey("processWaitingInReceiveTask");

EventSubscription subscription = runtimeService.createEventSubscriptionQuery()
  .processInstanceId(pi.getId()).eventType("message").singleResult();

runtimeService.messageEventReceived(subscription.getEventName(), subscription.getExecutionId());
```

{{< note title="" class="warning" >}}
并行多实例的相关性是不可能的，因为订阅不能被明确地识别。
{{< /note >}}

要继续一个当前在接收任务中等待的流程实例，而没有消息参考，可以调用`runtimeService.signal(executionId)`，使用接收任务中的执行的id。

```xml
<receiveTask id="waitState" name="wait" />
```

以下代码段显示在实践中如何运作：

```java
ProcessInstance pi = runtimeService.startProcessInstanceByKey("receiveTask");
Execution execution = runtimeService.createExecutionQuery()
  .processInstanceId(pi.getId()).activityId("waitState").singleResult();

runtimeService.signal(execution.getId());
```


# Camunda 扩展

<table class="table table-striped">
  <tr>
    <th>属性</th>
    <td>
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncbefore" >}}">camunda:asyncBefore</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncafter" >}}">camunda:asyncAfter</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#exclusive" >}}">camunda:exclusive</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#jobpriority" >}}">camunda:jobPriority</a>
    </td>
  </tr>
  <tr>
    <th>扩展元素</th>
    <td>
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#failedjobretrytimecycle" >}}">camunda:failedJobRetryTimeCycle</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#inputoutput" >}}">camunda:inputOutput</a>
    </td>
  </tr>
  <tr>
    <th>约束</th>
    <td>
      The <code>camunda:exclusive</code> attribute is only evaluated if the attribute
      <code>camunda:asyncBefore</code> or <code>camunda:asyncAfter</code> is set to <code>true</code>
    </td>
  </tr>
</table>


# 额外资料

* [Tasks](http://camunda.org/bpmn/reference.html#activities-task) in the [BPMN 2.0 Modeling Reference](http://camunda.org/bpmn/reference.html)
* [Message Receive Events]({{< ref "/reference/bpmn20/events/message-events.md" >}})
* [Trigger a subscription via REST]({{< ref "/reference/rest/execution/post-signal.md" >}})
