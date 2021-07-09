---

title: '从流程和案例中调用决策'
weight: 40

menu:
  main:
    name: "BPMN 和 CMMN 中的决策"
    identifier: "user-guide-process-engine-decisions-bpmn"
    parent: "user-guide-process-engine-decisions"
    pre: "Invoke Decisions from BPMN Processes and CMMN Cases"
---


# BPMN 和 CMMN 集成

本节说明如何从 BPMN 和 CMMN 调用 DMN 决策。

## BPMN 业务规则任务

BPMN 业务规则任务可以引用 [部署的][deployed] 决策定义。 在执行任务时评估决策定义。

```xml
<definitions id="taskAssigneeExample"
  xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:camunda="http://camunda.org/schema/1.0/bpmn"
  targetNamespace="Examples">

  <process id="process">

    <!-- ... -->

    <businessRuleTask id="businessRuleTask"
                      camunda:decisionRef="myDecision"
                      camunda:mapDecisionResult="singleEntry"
                      camunda:resultVariable="result" />

    <!-- ... -->

  </process>
</definitions>
```

有关如何从业务规则任务中引用决策定义的更多信息，请参阅[BPMN 2.0 参考][business rule task]。

## DMN Decision Task

CMMN 决策任务引用了一个[部署的][deployed] 决策定义。
激活任务时调用决策定义。

```xml
<definitions id="definitions"
                  xmlns="http://www.omg.org/spec/CMMN/20151109/MODEL"
                  xmlns:camunda="http://camunda.org/schema/1.0/cmmn"
                  targetNamespace="Examples">
  <case id="case">
    <casePlanModel id="CasePlanModel_1">
      <planItem id="PI_DecisionTask_1" definitionRef="DecisionTask_1" />
      <decisionTask id="DecisionTask_1"
                    decisionRef="myDecision"
                    camunda:mapDecisionResult="singleEntry"
                    camunda:resultVariable="result">
      </decisionTask>
    </casePlanModel>
  </case>
</definitions>
```

有关如何从决策任务中引用决策定义的更多信息，请参阅 [CMMN 1.1 参考][decision task].

# 决策结果

决策的输出，也称为决策结果，是“DmnDecisionResult”类型的复杂对象。 通常，它是一个键值对列表。

如果决策被实现为[决策表][decision table]，那么列表中的每个条目代表一个匹配的规则。此规则的输出条目由键值对表示。Key由输出名称指定。

相反的，如果决策被实现为 [决策文字表达式][decision literal expression]，那么该列表仅包含一个条目。此条目表示表达式值并由变量名称映射组成。

`DmnDecisionResult` 类型除了提供来自 `List` 接口的方法还实现了一些方便的方法，如 `getSingleResult()` 或 `getFirstResult()` 来获取匹配规则的结果。 规则结果提供了来自`Map` 接口的方法以及类似`getSingleEntry()` 或`getFirstEntry()` 的便捷方法。

如果决策结果仅包含单个输出值（例如，评估决策文字表达式），则可以使用结合了 getSingleResult() 和 getSingleEntry() 的 getSingleEntry() 方法从结果中检索该值。

例如，以下代码返回名称为“result”的唯一匹配规则的输出条目。

```java
DmnDecisionResult decisionResult = ...;

Object value = decisionResult
  .getSingleResult()
  .getEntry("result");
```

它还提供了获取类型化输出条目的方法，例如`getSingleEntryTyped()`。 有关类型值的详细信息，请参阅[用户指南][Typed Value API]。 在 {{< javadocref page="?org/camunda/bpm/dmn/engine/DmnDecisionResult" text="Java Docs" >}} 中可以找到所有方法的完整列表。

决策结果在执行任务的本地范围内作为名为“decisionResult”的瞬态变量可用。 如有必要，可以使用决策结果的预定义或自定义映射将其传递到变量中。

## 决策结果的预定义映射

该引擎包括针对常见用例的决策结果的预定义映射。 该映射类似于[输出变量映射][output variable mapping]。 它从保存在流程/案例变量中的决策结果中提取一个值。 以下映射可用：

<table class="table table-striped">
  <tr>
    <th>映射器</th>
    <th>结果类型</th>
    <th>适用于</th>
  </tr>
  <tr>
    <td>singleEntry</td>
    <td>TypedValue</td>
    <td>decision literal expressions and <br/>
    decision tables with no more than one matching rule and only one output</td>
  </tr>
  <tr>
    <td>singleResult</td>
    <td>Map&lt;String, Object&gt;</td>
    <td>decision tables with no more than one matching rule</td>
  </tr>
  <tr>
    <td>collectEntries</td>
    <td>List&lt;Object&gt;</td>
    <td>decision tables with multiple matching rules and only one output</td>
  </tr>
  <tr>
    <td>resultList</td>
    <td>List&lt;Map&lt;String, Object&gt;&gt;</td>
    <td>decision tables with multiple matching rules and multiple outputs</td>
  </tr>
</table>

只有`singleEntry` 映射器返回一个[类型化值][Typed Value API]，它包装了输出条目的值和附加类型信息。 其他映射器将包含输出条目值的集合作为普通 Java 对象返回，而没有其他类型信息。

请注意，如果决策结果不合适，映射器会抛出异常。 例如，如果决策结果包含多个匹配的规则，`singleEntry` 映射器会抛出异常。

{{< note title="Limitations of Serialization" class="warning" >}}

如果你使用的是预定义映射器 `singleResult`、`collectEntries` 或 `resultList` 之一，那么你应该考虑 [序列化的限制]({{< relref "#limitations-of-the-serialization-of-the-mapping-result" >}}).

{{< /note >}}

要指定用于存储映射结果的流程（BPMN）/案例（CMMN）变量的名称，请使用 `camunda:resultVariable` 属性。

BPMN:
```xml
<businessRuleTask id="businessRuleTask"
                  camunda:decisionRef="myDecision"
                  camunda:mapDecisionResult="singleEntry"
                  camunda:resultVariable="result" />
```

CMMN:
```xml
<decisionTask id="DecisionTask_1"
              decisionRef="myDecision"
              camunda:mapDecisionResult="singleEntry"
              camunda:resultVariable="result">
```

{{< note title="结果变量的名称" class="warning" >}}

结果变量不应使用名称“decisionResult”，因为决策结果本身保存在具有此名称的变量中。保存结果为此名称时会抛出异常。

{{< /note >}}

## 决策结果的自定义映射

可以使用自定义决策结果映射来将决策结果传递给变量，而不是预定义的映射。

{{< note title="Limitations of Serialization" class="warning" >}}

如果将集合或复杂对象传递给变量，则应考虑 [序列化的限制]({{< relref "#limitations-of-the-serialization-of-the-mapping-result" >}}).


{{< /note >}}

### 自定义映射到流程变量

如果业务规则任务用于调用 BPMN 流程内部的决策，则可以使用 [输出变量映射][output variable mapping] 将决策结果传递到流程变量中。

例如，如果决策结果有多个输出值，这些输出值应该保存在单独的流程变量中，这可以通过在业务规则任务上定义输出映射来实现。

```xml
<businessRuleTask id="businessRuleTask" camunda:decisionRef="myDecision">
  <extensionElements>
    <camunda:inputOutput>
      <camunda:outputParameter name="result">
        ${decisionResult.getSingleResult().result}
      </camunda:outputParameter>
      <camunda:outputParameter name="reason">
        ${decisionResult.getSingleResult().reason}
      </camunda:outputParameter>
    </camunda:inputOutput>
  </extensionElements>
</businessRuleTask>
```

除了输出变量映射外，决策结果还可以由附加到业务规则任务的[执行侦听器][execution listener]处理。

```xml
<businessRuleTask id="businessRuleTask" camunda:decisionRef="myDecision">
  <extensionElements>
    <camunda:executionListener event="end"
      delegateExpression="${myDecisionResultListener}" />
  </extensionElements>
</businessRuleTask>
```

```java
public class MyDecisionResultListener implements ExecutionListener {

  @Override
  public void notify(DelegateExecution execution) throws Exception {
    DmnDecisionResult decisionResult = (DmnDecisionResult) execution.getVariable("decisionResult");
    String result = decisionResult.getSingleResult().get("result");
    String reason = decisionResult.getSingleResult().get("reason");
    // ...
  }

}
```

### 自定义映射到大小写变量

如果决策任务用于调用 CMMN 案例内的决策，则可以使用附加到决策任务的案例执行侦听器将决策结果传递给案例变量。

```xml
<decisionTask id="decisionTask" decisionRef="myDecision">
  <extensionElements>
    <camunda:caseExecutionListener event="complete"
      class="org.camunda.bpm.example.MyDecisionResultListener" />
  </extensionElements>
</decisionTask>
```

```java
public class MyDecisionResultListener implements CaseExecutionListener {

  @Override
  public void notify(DelegateCaseExecution caseExecution) throws Exception;
    DmnDecisionResult decisionResult = (DmnDecisionResult) caseExecution.getVariable("decisionResult");
    String result = decisionResult.getSingleResult().get("result");
    String reason = decisionResult.getSingleResult().get("reason");
    // ...
    caseExecution.setVariable("result", result);
    // ...
  }

}
```

## 映射结果序列化的限制

预定义的映射`singleResult`、`collectEntries` 和`resultList` 将决策结果映射到Java 集合。集合的实现取决于使用的 JDK 并包含无类型值作为对象。当一个集合被保存为流程/案例变量时，它会被序列化为对象值，因为没有合适的原始值类型。根据使用的[对象值序列化][object value serialization]，这可能会导致反序列化问题。

如果你使用默认的内置对象序列化，如果 JDK 更新或更改并且包含不兼容的集合类版本，则无法反序列化该变量。否则，如果你使用其他序列化（如 JSON），则应确保无类型值是可反序列化的。例如，不能使用 JSON 反序列化一组日期值，因为 JSON 默认没有注册日期映射器。

使用自定义输出变量映射可能会出现相同的问题，因为 DmnDecisionResult 具有返回与预定义映射器相同的集合的方法。此外，不建议将“DmnDecisionResult”或“DmnDecisionResultEntries”保存为流程/案例变量，因为在新版本的 Camunda 平台中底层实现可能会发生变化。

为了防止任何这些问题，你应该只使用原始变量。
或者，你可以使用你自己控制的自定义对象进行序列化。

# 从决策中访问变量

DMN 决策表和决策文字表达式包含多个将由 DMN 引擎评估的表达式。 有关决策表达式的更多信息，请参阅 [DMN 1.3 参考][decision table]。 这些表达式可以访问调用任务范围内可用的所有流程/案例变量。 这些变量是通过只读变量上下文提供的。

作为简写，可以在表达式中通过名称直接引用流程/案例变量。 例如，如果一个流程变量‘foo’存在，那么这个变量可以通过它的名字在决策表的输入表达式、输入条目和输出条目中使用。

```xml
<input id="input">
  <!--
    此输入表达式将返回流程/案例变量 `foo` 的值
  -->
  <inputExpression>
    <text>foo</text>
  </inputExpression>
</input>
```

表达式中流程/案例变量的返回值将是一个普通对象，而不是 [typed value][Typed Value API]。 如果要在表达式中使用类型化值，则必须从变量上下文中获取变量。 下面的代码片段与上面的示例相同。 它从变量上下文中获取变量 `foo` 并返回它的解包值。

```xml
<input id="input">
  <!--
    此输入表达式使用变量上下文来获取流程/案例变量 `foo` 的类型值
  -->
  <inputExpression>
    <text>
      variableContext.resolve("foo").getValue()
    </text>
  </inputExpression>
</input>
```

# 表达式语言集成

默认情况下，DMN 引擎使用 [FEEL] 作为输入表达式、输入条目、输出条目和文字表达式的表达式语言。 有关表达式语言的更多信息，请参阅 [DMN 引擎][expression languages] 指南。

## 访问 Bean

如果 DMN 引擎由 Camunda 平台调用，它使用与 Camunda 平台引擎相同的 JUEL 配置。 因此，也可以在决策中从 JUEL 表达式访问 Spring 和 CDI Beans。
有关此集成的更多信息，请参阅 [Spring] 和 [CDI] 指南中的相应部分。

{{< note title="小心！" class="info" >}}
使用 FEEL 作为表达式语言时，无法访问 Bean。
{{< /note >}}

## 其他的表达式语言

{{< note title="Use of Internal API" class="warning" >}}

这些 API **不是** [公共 API]({{< ref "/introduction/public-api.md" >}}) 的一部分，可能会在以后的版本中更改。

{{< /note >}}

可以添加可以在 JUEL 表达式中使用的自己的函数。
因此，必须实现一个新的 {{< javadocref page="?org/camunda/bpm/engine/impl/javax/el/FunctionMapper.html" text="FunctionMapper" >}}。 函数映射器必须在初始化后添加到流程引擎配置中。

```java
ProcessEngineConfigurationImpl processEngineConfiguration = (ProcessEngineConfigurationImpl) processEngine
  .getProcessEngineConfiguration();

processEngineConfiguration
  .getExpressionManager()
  .addFunctionMapper(new MyFunctionMapper());
```

例如，可以通过创建 [流程引擎插件][process engine plugin] 来完成。

请 **注意** 这些函数在平台中的所有 JUEL 表达式中都可用，而不仅仅是在 DMN 决策中。

[decision table]: {{< ref "/reference/dmn/decision-table/_index.md" >}}
[decision literal expression]: {{< ref "/reference/dmn/decision-literal-expression/_index.md" >}}
[deployed]: {{< ref "/user-guide/process-engine/decisions/repository.md" >}}
[business rule task]: {{< ref "/reference/bpmn20/tasks/business-rule-task.md" >}}
[decision task]: {{< ref "/reference/cmmn11/tasks/decision-task.md" >}}
[Typed Value API]: {{< ref "/user-guide/process-engine/variables.md#typed-value-api" >}}
[object value serialization]: {{< ref "/user-guide/process-engine/variables.md#object-value-serialization" >}}
[output variable mapping]: {{< ref "/user-guide/process-engine/variables.md#input-output-variable-mapping" >}}
[execution listener]: {{< ref "/user-guide/process-engine/delegation-code.md#execution-listener" >}}
[expression languages]: {{< ref "/user-guide/dmn-engine/expressions-and-scripts.md" >}}
[FEEL]: {{< ref "/reference/dmn/feel/_index.md" >}}
[Spring]: {{< ref "/user-guide/spring-framework-integration/_index.md#expression-resolving" >}}
[CDI]: {{< ref "/user-guide/cdi-java-ee-integration/expression-resolving.md" >}}
[process engine plugin]: {{< ref "/user-guide/process-engine/process-engine-plugins.md" >}}
