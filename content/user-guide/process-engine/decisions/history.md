---

title: 'History for DMN Decisions'
weight: 50

menu:
  main:
    name: "History"
    identifier: "user-guide-process-engine-decisions-history"
    parent: "user-guide-process-engine-decisions"
    pre: "Audit evaluated Decisions"
---

从 BPMN 流程、CMMN 案例或通过决策服务评估决策定义后，输入和输出将保存在平台历史记录中。 历史实体的类型为“HistoricDecisionInstance”，事件类型为“evaluate”。

有关历史机制的详细信息，请参阅[历史和审计事件日志][History and Audit Event Log]。

{{< note title="History Level" class="info" >}}

历史级别必需是 **FULL**。 否则，不会创建决策历史记录。

{{< /note >}}

# 查询评估决策

历史服务可用于查询“HistoricDecisionInstances”。 例如，使用以下查询获取决策定义的所有历史条目，键为“checkOrder”，按决策评估时间排序。

```java
List<HistoricDecisionInstance> historicDecisions = processEngine
  .getHistoryService()
  .createHistoricDecisionInstanceQuery()
  .decisionDefinitionKey("checkOrder")
  .orderByEvaluationTime()
  .asc()
  .list();
```

从 [BPMN 业务规则任务][BPMN business rule task] 评估的决策可以通过流程定义 ID 或键和流程实例 ID 进行过滤。

```java
HistoryService historyService = processEngine.getHistoryService();

List<HistoricDecisionInstance> historicDecisionInstances = historyService
  .createHistoricDecisionInstanceQuery()
  .processDefinitionId("processDefinitionId")
  .list();

historicDecisionInstances = historyService
  .createHistoricDecisionInstanceQuery()
  .processDefinitionKey("processDefinitionKey")
  .list();

historicDecisionInstances = historyService
  .createHistoricDecisionInstanceQuery()
  .processInstanceId("processInstanceId")
  .list();
```

从 [CMMN 决策任务][CMMN decision task] 评估的决策可以通过案例定义 ID 或Key和案例实例 ID 进行过滤。

```java
HistoryService historyService = processEngine.getHistoryService();

List<HistoricDecisionInstance> historicDecisionInstances = historyService
  .createHistoricDecisionInstanceQuery()
  .caseDefinitionId("caseDefinitionId")
  .list();

historicDecisionInstances = historyService
  .createHistoricDecisionInstanceQuery()
  .caseDefinitionKey("caseDefinitionKey")
  .list();

historicDecisionInstances = historyService
  .createHistoricDecisionInstanceQuery()
  .caseInstanceId("caseInstanceId")
  .list();
```

请注意，默认情况下，查询结果中不包含决策的输入和输出。 可以调用方法 `includeInputs()` 和 `includeOutputs()` 以从结果中查询输入和输出。

```java
List<HistoricDecisionInstance> historicDecisions = processEngine
  .getHistoryService()
  .createHistoricDecisionInstanceQuery()
  .decisionDefinitionKey("checkOrder")
  .includeInputs()
  .includeOutputs()
  .list();
```

# 历史决策实例

{{< javadocref page="?org/camunda/bpm/engine/history/HistoricDecisionInstance" text="HistoricDecisionInstance" >}} 包含有关决策的单个评估的信息。

```java
HistoricDecisionInstance historicDecision = ...;

// 决策定义的 id
String decisionDefinitionId = historicDecision.getDecisionDefinitionId();

// 决策定义的 Key
String decisionDefinitionKey = historicDecision.getDecisionDefinitionKey();

// 决策的 name
String decisionDefinitionName = historicDecision.getDecisionDefinitionName();

// 评估决定的时间
Date evaluationTime = historicDecision.getEvaluationTime();

// 决策的输入（如果在查询中指定了 includeInputs）
List<HistoricDecisionInputInstance> inputs = historicDecision.getInputs();

// 决策的输出（如果在查询中指定了 includeOutputs）
List<HistoricDecisionOutputInstance> outputs = historicDecision.getOutputs();
```

如果决策是从流程中评估的，流程定义、流程实例和活动的信息在“HistoricDecisionInstance”中设置。 这同样适用于从案例评估的决策，其中历史实例将引用相应的案例实例。

此外，如果决策是带有命中策略 `collect` 和聚合函数的决策表，那么聚合的结果可以通过 `getCollectResultValue()` 方法查询。

有关支持的命中策略的更多信息，请参阅 [DMN 1.3 参考][DMN 1.3 reference]。

## 历史决策的输入实例

{{< javadocref page="?org/camunda/bpm/engine/history/HistoricDecisionInputInstance" text="HistoricDecisionInputInstance" >}} 表示评估决策的一个输入（例如，决策表的输入子句）。 

```java
HistoricDecisionInputInstance input = ...;

// 输入子句的 id
String clauseId = input.getClauseId();

// 输入子句的 label
String clauseName = input.getClauseName();

// 输入表达式的计算值
Object value = input.getValue();

// 输入表达式的计算值作为包含类型信息的类型化值
TypedValue typedValue = input.getTypedValue();
```

请注意，如果输入指定了类型，则该值可能是类型转换的结果。

## 历史决策的输出实例

{{< javadocref page="?org/camunda/bpm/engine/history/HistoricDecisionOutputInstance" text="HistoricDecisionOutputInstance" >}} 表示评估决策的一个输出条目。 如果决策被实现为决策表，那么对于每个输出子句和匹配的规则，`HistoricDecisionInstance` 包含一个 `HistoricDecisionOutputInstance`。

```java
HistoricDecisionOutputInstance output = ...;

// 输出子句的 id
String clauseId = output.getClauseId();

// 输出子句的 label
String clauseName = output.getClauseName();

// 输出条目的评估值
Object value = output.getValue();

// 输出条目的评估值作为包含类型信息的类型化值
TypedValue typedValue = output.getTypedValue();

// 输出所属的匹配规则的 id
String ruleId = output.getRuleId();

// 规则在匹配规则列表中的位置
Integer ruleOrder = output.getRuleOrder();

// 用作输出变量标识符的输出子句的名称
String variableName = output.getVariableName();
```

请注意，如果输出指定类型，则该值可能是类型转换的结果。

# Cockpit

你可以在 [Cockpit][Cockpit] webapp 中审核评估的决策定义。



[Cockpit]: {{< ref "/webapps/cockpit/dmn/_index.md" >}}
[History and Audit Event Log]: {{< ref "/user-guide/process-engine/history.md" >}}
[DMN 1.3 reference]: {{< ref "/reference/dmn/decision-table/hit-policy.md" >}}
[BPMN business rule task]: {{< ref "/reference/bpmn20/tasks/business-rule-task.md#using-camunda-dmn-engine" >}}
[CMMN decision task]: {{< ref "/reference/cmmn11/tasks/decision-task.md" >}}
