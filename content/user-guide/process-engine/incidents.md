---

title: '事件'
weight: 210

menu:
  main:
    identifier: "user-guide-process-engine-incidents"
    parent: "user-guide-process-engine"

---


事件是发生在流程引擎中的值得注意的事件。这种事件通常表明与流程执行有关的某种问题。这类事件的例子可能是一个失败Job，其重试次数已耗尽（retries=0），表明执行被卡住了，必须采取人工管理行动来修复流程实例。如果出现这样的事件，流程引擎会触发一个内部事件，该事件可以被可配置的事件处理程序处理。

在默认配置中，流程引擎将事件写到流程引擎数据库中。然后你可以使用`RuntimeService`的`IncidentQuery`查询数据库中不同类型和种类的事件。

```java
runtimeService.createIncidentQuery()
  .processDefinitionId("someDefinition")
  .list();
```

事件被储存在 ACT_RU_INCIDENT 数据表中。

如果你想自定义事件处理行为，则可以替换流程引擎配置中的默认事件处理程序，并提供自定义的实现（见下文）。


# 事件类型

目前，流程引擎支持以下事件不同类型的事件：

**failedJob**：当一项工作的自动重试（定时器或异步延续）被耗尽时，会被引发。该事件表明，相应的执行被卡住了，不会自动继续。管理行动是必要的。当Job被手动执行或相应job的重试次数被设置为>0时，该事件将被解决。
**failedExternalTask**：当[外部任务]({{< ref "/user-guide/process-engine/external-tasks.md" >}})的工作者报告失败，并且给定的重试次数被设置为<=0时，会引发该事件，表明相应的外部任务被卡住，不会被工作者获取。需要采取管理措施来重置重试。

可以用Java API创建任何类型的自定义事件。

# 创建和解决自定义事件

通过调用 `RuntimeService#createIncident` 可以创建任何类型的事件 ...

```java
runtimeService.createIncident("someType", "someExecution", "someConfiguration", "someMessage");
```

... 或直接使用 `DelegateExecution#createIncident`：
```java
delegateExecution.createIncident("someType", "someConfiguration", "someMessage");
```

自定义事件必须总是与现有的执行有关。

除了**failedJob**和**failedExternalTask**外，任何类型的事件都可以通过调用`RuntimeService#resolveIncident`来解决。

事件也可以通过REST API[创建]({{<ref "/reference/rest/execution/post-create-incident.md">}})和[解决]({{<ref "/reference/rest/incident/resolve-incident.md">}})报告失败，并且给定的重试值被设置为<=0，该事件表明相应的外部任务被卡住，不会被外部工作者取走。


# 事件的激活与冻结


流程引擎允许你根据事件类型来配置是否应该提出某些事件。
以下属性在`org.camunda.bpm.engine.ProcessEngineConfiguration`类中可用。

  * `createIncidentOnFailedJobEnabled`：表示是否应该提出失败的Job事件。


# 实现自定义事件处理程序

事件处理者负责处理某一类型的事件（见[事件类型]({{< relref "#incident-types" >}})）。

一个事件处理程序需要实现以下接口：

```java
public interface IncidentHandler {

  String getIncidentHandlerType();

  Incident handleIncident(IncidentContext context, String message);

  void resolveIncident(IncidentContext context);

  void deleteIncident(IncidentContext context);

}
```

当创建一个新的事件时，会调用 "handleIncident" 方法。当一个事件被解决时，`resolveIncident`方法被调用。如果你想提供一个自定义的事件处理程序实现，你可以使用以下方法替换一个或多个事件处理程序。

```java
org.camunda.bpm.engine.impl.cfg.ProcessEngineConfigurationImpl.setCustomIncidentHandlers(List<IncidentHandler>)
```

自定义事件处理程序的一个例子是，只要发生 "failedJob" 类型的事件，就向管理员发送电子邮件，从而扩展默认行为。然而，仅仅添加自定义事件处理程序就会用自定义事件处理程序的行为覆盖默认行为。因此，默认事件处理程序不再被执行。如果默认行为也应该被执行，那么自定义事件处理程序也需要调用默认事件处理程序，这包括使用内部 API。

{{< note title="使用内部API" class="warning" >}}

请注意，该API **不是** [公共API]({{< ref "/introduction/public-api.md" >}})的一部分，在以后的版本中可能会发生变化。

{{< /note >}}