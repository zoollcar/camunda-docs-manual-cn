---

title: '批处理操作'
weight: 280

menu:
  main:
    identifier: "user-guide-process-engine-batch-operations"
    parent: "user-guide-process-engine-batch"

---

以下操作可以异步执行：

- [流程实例迁移][batch-migration]
- [取消运行流程实例](#取消运行流程实例)
- [删除历史流程实例](#删除历史流程实例)
- [更新流程实例的暂停状态](#更新流程实例的暂停状态)
- [设置流程实例关联的Job的重试](#设置流程实例关联的Job的重试)
- [修改流程实例]({{< ref "/user-guide/process-engine/process-instance-modification.md#modification-of-multiple-process-instances" >}})
- [重启流程实例]({{< ref "/user-guide/process-engine/process-instance-restart.md#asynchronous-batch-execution" >}})
- [设置外部任务的重试](#设置外部任务的重试)
- [设置流程实例变量](#设置流程实例变量)
- [为历史流程实例设置移除时间](#历史流程实例)
- [为历史决策实例设置移除时间](#历史决策实例)
- [为历史批处理设置移除时间](#历史批处理)

所有批处理操作都依赖于提供同步操作实体列表的对应方法。参考[批处理][batch]文档以更好地了解创建过程。

可以基于特定实例列表以及提供实例结果列表的查询结果来执行异步操作。 如果同时提供实例列表和查询，则受影响实例的结果集将由这两个子集的并集组成。

所有列出的批处理操作，除了[为历史批处理设置删除时间](#historic-batches)，都是部署可知的。
特别是，这意味着种子Job和执行Job将收到一个 `deploymentId` 所以 [部署可知的Job执行者]({{< ref "/user-guide/process-engine/the-job-executor.md#job -execution-in-heterogeneous-clusters" >}}) 可以选择需要在其节点上执行的批处理Job。 种子Job的 deploymentId 是从涉及的部署列表中选择的。 此列表源自受影响实例的结果集。 执行Job仅包含相同部署的元素并绑定到该 deploymentId。

## 取消运行流程实例

可以使用以下 Java API 方法调用异步执行取消正在运行的流程实例：

```java
List<String> processInstanceIds = ...;
runtimeService.deleteProcessInstancesAsync(
        processInstanceIds, null, REASON);
```


## 删除历史流程实例

可以使用以下 Java API 方法调用异步执行历史流程实例的删除：

```java
List<String> historicProcessInstanceIds = ...;
historyService.deleteHistoricProcessInstancesAsync(
        historicProcessInstanceIds, TEST_REASON);
```

## 更新流程实例的暂停状态

使用以下 Java API 方法调用异步更新多个流程实例的挂起状态：

```java
List<String> processInstanceIds = ...;
runtimeService.updateProcessInstanceSuspensionState().byProcessInstanceIds(
  processInstanceIds).suspendAsync();
```

## 设置流程实例关联的Job的重试

可以使用以下 Java API 方法调用异步执行与流程实例关联的Job的重试设置：

```java
List<String> processInstanceIds = ...;
int retries = ...;
managementService.setJobRetriesAsync(
        processInstanceIds, null, retries);
```

## 设置外部任务的重试

可以使用以下 Java API 方法调用异步执行外部任务的设置重试：

```java
List<String> externalTaskIds = ...;
externalTaskService.setRetriesAsync(
        externalTaskIds, TEST_REASON);
```

## 设置流程实例变量

有时需要添加或更新已经运行的流程实例的变量。
例如，当用户在流程开始时输入不正确的数据时，需要即时更正数据时。

此批处理操作可帮助你将变量异步设置到流程实例的根范围。

你可以 (1) 使用 `HistoricProcessInstanceQuery` 或 `ProcessInstanceQuery` 过滤流程实例，或者 (2) 直接传递一组流程实例 ID。

请参阅下面如何调用 Java API：

```java
List<String> processInstanceIds = ...;
Map<String, Object> variables = Variables.putValue("my-variable", "my-value");
runtimeService.setVariablesAsync(processInstanceIds, variables);
```

### 已知的限制

目前，无法通过批处理操作设置瞬时变量。 但是，你可以同步[设置瞬时变量][set transient variables]。

## 设置移除时间

有时需要推迟甚至阻止删除某些历史实例。
移除时间可以与历史流程、决策和批处理异步设置。

可以选择以下模式：

* **绝对（Absolute）:** 将删除时间设置为任意日期
  * `.absoluteRemovalTime(Date removalTime)`
* **清除（Cleared）:** 重置移除时间（表示为 `null` 值）； 没有删除时间的实例不会被清理
  * `.clearedRemovalTime()`
* **计算（Calculated）:** 根据工作流引擎的设置（基准时间 + TTL）重新计算移除时间
  * `.calculatedRemovalTime()`

历史流程和决策实例可以是层次结构的一部分。 要为层次结构中的所有实例设置相同的删除时间，需要调用方法 `.hierarchical()`。

### 历史流程实例

```java
HistoricProcessInstanceQuery query = 
  historyService.createHistoricProcessInstanceQuery();

Batch batch = historyService.setRemovalTimeToHistoricProcessInstances()
  .absoluteRemovalTime(new Date()) // 设置绝对清除时间
   // .clearedRemovalTime()        // 将删除时间重置为null
   // .calculatedRemovalTime()     // 基于引擎配置计算
  .byQuery(query)
  .byIds("693206dd-11e9-b7cb-be5e0f7575b7", "...")
   // .hierarchical()              // 在层次结构上设置删除时间
  .executeAsync();
```

### 历史决策实例

```java
HistoricDecisionInstanceQuery query = 
  historyService.createHistoricDecisionInstanceQuery();

Batch batch = historyService.setRemovalTimeToHistoricDecisionInstances()
  .absoluteRemovalTime(new Date()) // 设置绝对清除时间
   // .clearedRemovalTime()        // 将删除时间重置为null
   // .calculatedRemovalTime()     // 基于引擎配置计算
  .byQuery(query)
  .byIds("693206dd-11e9-b7cb-be5e0f7575b7", "...")
   // .hierarchical()              // 在层次结构上设置删除时间
  .executeAsync();
```

{{< note title="已知的限制" class="info" >}}
决策实例批处理操作的 `.hierarchical()` 标志仅设置决策层次结构内的移除时间。 如果业务规则任务调用了决策，则不会更新调用流程实例（包括层次结构中存在的其他流程实例）。

要沿根流程实例（所有流程以及决策实例）的层次结构更新所有子实例，请对启用了 `.hierarchical()` 标志的流程实例使用批处理操作。
{{< /note >}}

### 历史批处理

```java
HistoricBatchQuery query = historyService.createHistoricBatchQuery();

Batch batch = historyService.setRemovalTimeToHistoricBatches()
  .absoluteRemovalTime(new Date()) // 设置绝对清除时间
   // .clearedRemovalTime()        // 将删除时间重置为null
   // .calculatedRemovalTime()     // 基于引擎配置计算
  .byQuery(query)
  .byIds("693206dd-11e9-b7cb-be5e0f7575b7", "...")
  .executeAsync();
```

[batch-migration]: {{< ref "/user-guide/process-engine/process-instance-migration.md#asynchronous-batch-migration-execution" >}}
[batch]: {{< ref "/user-guide/process-engine/batch.md" >}}
[set transient variables]: {{< ref "/user-guide/process-engine/variables.md#transient-variables" >}}
