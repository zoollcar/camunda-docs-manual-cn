---

title: '流程引擎中的决策服务类'
weight: 30

menu:
  main:
    name: "决策服务类"
    identifier: "user-guide-process-engine-decisions-service"
    parent: "user-guide-process-engine-decisions"
    pre: "Evaluate Decisions using the Services API"
---

决策服务是流程引擎的 [Services API][Services API] 的一部分。 它允许独立于 BPMN 和 CMMN 评估已部署的决策定义。

# 评估决策

要评估已部署的决策，请通过 id 或Key和版本的组合来引用它。 如果使用Key但未指定版本，则评估具有给定Key的决策定义的最新版本。

```java
DecisionService decisionService = processEngine.getDecisionService();

VariableMap variables = Variables.createVariables()
  .putValue("status", "bronze")
  .putValue("sum", 1000);

DmnDecisionResult decisionResult = decisionService
  .evaluateDecisionByKey("decision-key")
  .variables(variables)
  .evaluate(); 
  
// alternatively for decision tables only
DmnDecisionTableResult decisionResult = decisionService
  .evaluateDecisionTableByKey("decision-key")
  .variables(variables)
  .evaluate(); 
```

## 决策定义Key

决策定义Key由 DMN XML 中“decision”元素的“id”属性指定。不同的命名与[决策的版本化][Versioning of Decisions]有关。由于Key可以引用决策定义的多个版本，因此 id 仅指定一个版本。

## 传递数据

决策可能会引用一个或多个变量。 例如，可以在输入表达式或决策表的输入条目中引用变量。 变量作为键值对传递给决策服务。 每对指定变量的名称和值。

有关不同表达式的更多信息，请参阅[DMN 1.3 参考][DMN 1.3 参考]。

# 评估决策的授权

用户需要对资源`DECISION_DEFINITION` 的权限`CREATE_INSTANCE` 来评估决策。 授权的资源 id 是决策定义键。

有关授权的更多信息，请参阅 [授权服务][Authorization Service] 部分。

# 使用决策结果

评估的结果称为决策结果。 决策结果是一个类型为“DmnDecisionResult”的复杂对象。 将其视为键值对列表。

如果决策被实现为[决策表][decision table]，那么列表中的每个条目代表一个匹配的规则。 此规则的输出条目由键值对表示。 一对密钥由输出名称指定。

相反，如果决策被实现为 [决策文字表达式][decision literal expression]，那么该列表仅包含一个条目。 此条目表示表达式值并由变量名称映射。

决策结果提供了接口`List<Map<String, Object>>`的方法和一些方便的方法。

```java
DmnDecisionResult decisionResult = ...;

// 获取唯一匹配规则的单个条目的值
String singleEntry = decisionResult.getSingleResult().getSingleEntry();

// 获取唯一匹配规则名称为“result”的结果条目的值
String result = decisionResult.getSingleResult().getEntry("result");

// 获取第二个匹配规则的第一个条目的值
String firstValue = decisionResult.get(1).getFirstEntry();

// 获取所有匹配规则的输出名称为“result”的所有条目的列表
List<String> results = decisionResult.collectEntries("result");

// 获取单个规则结果的单个输出条目的快捷方式
// - 结合 getSingleResult() 和 getSingleEntry()
String result = decisionResult.getSingleEntry();
```

请注意，决策结果还提供了获取类型化输出条目的方法。
在 {{< javadocref page="?org/camunda/bpm/dmn/engine/DmnDecisionResult" text="Java Docs" >}} 中可以找到所有方法的完整列表。

如果决策被实施为[决策表][decision table]，那么它也可以使用{{< javadocref page="?org/camunda/bpm/engine/DecisionService.html##evaluateDecisionTableByKey(java.lang.String)" text="evaluateDecisionTable" >}}中的方法。 在这种情况下，结果返回一个 {{< javadocref page="?org/camunda/bpm/dmn/engine/DmnDecisionTableResult.html" text="DmnDecisionTableResult" >}}，它与` DmnDecisionResult` 在语义和方法上是相等的。

# 评估决策的历史

评估决策时，会创建一个新的历史记录条目，类型为“HistoricDecisionInstance”，其中包含决策的输入和输出。 历史可以通过历史服务进行查询。

```java
List<HistoricDecisionInstance> historicDecisions = processEngine
  .getHistoryService()
  .createHistoricDecisionInstanceQuery()
  .decisionDefinitionKey("decision-key")
  .includeInputs()
  .includeOutputs()
  .list();
```

有关这方面的更多信息，请参阅[DMN 决策的历史][History for DMN Decisions]。

[decision table]: {{< ref "/reference/dmn/decision-table/_index.md" >}}
[decision literal expression]: {{< ref "/reference/dmn/decision-literal-expression/_index.md" >}}
[Services API]: {{< ref "/user-guide/process-engine/process-engine-api.md#services-api" >}}
[DMN 1.3 reference]: {{< ref "/reference/dmn/decision-table/_index.md" >}}
[Versioning of Decisions]: {{< ref "/user-guide/process-engine/decisions/repository.md#versioning-of-decisions" >}}
[Authorization Service]: {{< ref "/user-guide/process-engine/authorization-service.md" >}}
[History for DMN Decisions]: {{< ref "/user-guide/process-engine/decisions/history.md" >}}
