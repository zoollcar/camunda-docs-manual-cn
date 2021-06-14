---

title: 'Manual Task 手动任务'
weight: 70

menu:
  main:
    identifier: "bpmn-ref-tasks-manual-task"
    parent: "bpmn-ref-tasks"
    pre: "A task which is performed externally."

---

手动任务定义了一个对BPM引擎来说是外部的任务。它被用来模拟由引擎不需要知道的人所做的工作，而且没有已知的系统或UI接口。对于引擎来说，手工任务被当作一个传递节点来处理，在流程执行到达的那一刻将自动继续流程。

{{< bpmn-symbol type="manual-task" >}}

```xml
<manualTask id="myManualTask" name="Manual Task" />
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
