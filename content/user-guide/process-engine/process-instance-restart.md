---

title: '重启/恢复流程实例'
weight: 50

menu:
  main:
    identifier: "user-guide-process-engine-process-instance-restart"
    parent: "user-guide-process-engine"

---

在流程实例终止后，其历史数据仍然存在，并且可以被访问以重启流程实例，前提是历史级别被设置为FULL。
例如，当流程没有以期望的方式终止时，重启流程实例是有用的。这个API的使用的其他可能情况有：

* 恢复被错误地取消的流程实例的到最后状态
* 由于错误路由导致流程实例终止后，重启流程实例

为了执行这样的操作，进程引擎提供了 *流程实例重启API* `RuntimeService.restartProcessInstances(..)` 。该API允许通过使用流式构建器在一次调用中指定多个实例化指令。

请注意，这些操作也可以通过REST方式进行。[重启流程实例]({{< ref "/reference/rest/process-definition/post-restart-process-instance-sync.md" >}})和[重启流程实例（async）]({{< ref "/reference/rest/process-definition/post-restart-process-instance-async.md" >}})

# 流程实例重启的例子

考虑以下流程模型，其中红点标志着活动任务：

{{< img src="../img/variables-3.png" title="Running Process Instance" >}}

让我们假设流程实例已经被别人用以下代码从外部取消了：

```java
ProcessInstance processInstance = runtimeService.createProcessInstanceQuery().singleResult();
runtimeService.deleteProcessInstance(processInstance.getId(), "any reason");
```

之后，我们决定恢复该流程实例到最后状态：

```java
runtimeService.restartProcessInstance(processInstance.getProcessDefinitionId())
	.startBeforeActivity("receivePayment")
	.startBeforeActivity("shipOrder")
	.processInstanceIds(processInstance.getId())
	.execute();
```

流程实例已经用最后一组变量重启了。然而，在重启的流程实例中，只有全局变量被恢复了。
本地变量还需要调用 `RuntimeService.setVariableLocal(..)` 手动设置。

{{< note title="" class="info" >}}
  从技术上讲，创建的是一个新的流程实例。

  **请注意:**
  历史流程和重启的流程实例的id是不同的。
{{< /note >}}

# 操作语义

在下文中，将描述流程实例恢复功能的确切语义。建议阅读本节，以充分了解该功能的效果、能力和限制。

## 实例化指令类型

流式流程实例重启构建器提供了以下的方法，可以调用。

* `startBeforeActivity(String activityId)`
* `startAfterActivity(String activityId)`
* `startTransition(String transitionId)`

关于指令类型的信息，请参考类似的
 [修改指令类型]({{< ref "/user-guide/process-engine/process-instance-modification.md#modification-instruction-types" >}}) section.

## 选择要重启的流程实例

可以通过提供一组流程实例ID来选择流程实例进行重启
或者提供一个历史流程实例查询。也可以同时指定一个流程实例id列表和一个查询。
然后，要重启的流程实例将是所产生的集合的并集。

### 流程实例列表

应该被重启的进程实例可以是一个进程实例ID的列表。

```Java
ProcessDefinition processDefinition = ...;
List<String> processInstanceIds = ...;

runtimeService.restartProcessInstances(processDefinition.getId())
  .startBeforeActivity("activity")
  .processInstanceIds(processInstanceIds)
  .execute();
```

或者，对于固定数量的流程实例，有一个方便的可变参数方法。

```Java
ProcessDefinition processDefinition = ...;

HistoricProcessInstance instance1 = ...;
HistoricProcessInstance instance2 = ...;

runtimeService.restartProcessInstances(processDefinition.getId())
  .startBeforeActivity("activity")
  .processInstanceIds(instance1.getId(), instance2.getId())
  .execute();
```

### 历史流程实例查询

If the instances are not known beforehand, the process instances can be selected by a historic process instance query:

```Java
HistoricProcessInstanceQuery historicProcessInstanceQuery = historyService
  .createHistoricProcessInstanceQuery()
  .processDefinitionId(processDefinition.getId())
  .finished();

runtimeService.restartProcessInstances(processDefinition.getId())
  .startBeforeActivity("activity")
  .historicProcessInstanceQuery(historicProcessInstanceQuery)
  .execute();
```

## 跳过监听器、输入输出映射

It is possible to skip invocations of execution and task listeners as well as input/output mappings for the transaction that performs the restart. This can be useful when the restart is executed on a system that has no access to the involved process application deployments and their contained classes.

In the API, the two methods `#skipCustomListeners` and `#skipIoMappings`
can be used for this purpose:

```Java
ProcessDefinition processDefinition = ...;
List<String> processInstanceIds = ...;

runtimeService.restartProcessInstances(processDefinition.getId())
  .startBeforeActivity("activity")
  .processInstanceIds(processInstanceIds)
  .skipCustomListeners()
  .skipIoMappings()
  .execute();
```

## 使用初试变量集重启流程实例

默认情况下，进程实例会以最后一组变量重启。
如果要选择初始变量集，可以使用 `initialSetOfVariables` 方法。

This feature does not only copy the start variables, but will copy the first version of all process variables that have been set in the start activity of the old process instance.

```Java
ProcessDefinition processDefinition = ...;
List<String> processInstanceIds = ...;

runtimeService.restartProcessInstances(processDefinition.getId())
  .startBeforeActivity("activity")
  .processInstanceIds(processInstanceIds)
  .initialSetOfVariables()
  .execute();
```


The initial set of variables can not be set if the historic process instance has no unique start activity. In that case, no variables are taken over.

## 忽略历史进程实例的 Business Key

默认情况下，一个流程实例以与历史流程实例相同的Business Key重启。
通过使用方法`withoutBusinessKey`，重启的流程实例的Business Key不被设置。

```Java
ProcessDefinition processDefinition = ...;
List<String> processInstanceIds = ...;

runtimeService.restartProcessInstances(processDefinition.getId())
  .startBeforeActivity("activity")
  .processInstanceIds(processInstanceIds)
  .withoutBusinessKey()
  .execute();
```
## 执行

重启可以同步或通过使用[批处理]({{< ref "/user-guide/process-engine/batch.md" >}})功能异步执行

下面是两者适用的不同场景：

- 以下情况适用同步执行：
  - 流程实例的数量很小的情况
  - 重启操作必须是原子的，也就是重启必须立刻执行并且在任何一个重启不成功的情况下报错。


- 以下情况适用异步执行：
  - 流程实例的数量很大
  - 所有的进程实例都独立重启，也就是说所有实例都在自己的事务中重启
  - 重启需要由另一个线程执行，即job执行器处理执行

### 同步执行

To execute the restart synchronously, the `execute` method is used. It will
block until the restart is completed.
调用 `execute` 同步执行重启，调用后，直到重启完成的时间将阻塞。

```Java
ProcessDefinition processDefinition = ...;
List<String> processInstanceIds = ...;

runtimeService.restartProcessInstances(processDefinition.getId())
  .startBeforeActivity("activity")
  .processInstanceIds(processInstanceIds)
  .execute();
```

如果所有流程实例都能被重新启动，则重启成功。

### 异步批量执行

要异步执行重启，需要使用 `executeAsync` 方法。它将立即返回一个对执行重启的 `批处理` 的引用。

```Java
ProcessDefinition processDefinition = ...;
List<String> processInstanceIds = ...;

Batch batch = runtimeService.restartProcessInstances(processDefinition.getId())
  .startBeforeActivity("activity")
  .processInstanceIds(processInstanceIds)
  .executeAsync();
```

使用一个批处理，进程实例的重启被分割成几个job，这些job被job执行器异步执行。
更多信息请参见[批处理]({{< ref "/user-guide/process-engine/batch.md" >}}) 部分。
如果所有的批处理job都成功完成，那么这个批处理就完成了完成了。然而，与同步执行不同的是，它并不保证流程实例是否真的重启完成。由于重启被分割成几个独立的job，每一个job都可能失败或成功。

如果重启job失败，job执行者会重试，重试直到超出最大重试次数，然后就会创建一个事件。在这种情况下，可以手动操作来完成批量重启。那么job的重试次数会递增，也可以手动删除该job。删除会取消特定实例的重启，但是但不影响此后的批次。