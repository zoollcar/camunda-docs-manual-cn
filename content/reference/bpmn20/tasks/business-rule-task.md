---

title: 'Business Rule Task （业务规则任务）'
weight: 40

menu:
  main:
    identifier: "bpmn-ref-tasks-business-rule-task"
    parent: "bpmn-ref-tasks"
    pre: "Execute an automated business decision."

---

一个业务规则任务可以同步执行一个或多个规则，可以调用Java代码或外部工作者（external worker）来实现，或者以异步方式调用以Web服务形式实现的逻辑。

{{< bpmn-symbol type="business-rule-task" >}}


# 使用 Camunda DMN 引擎

你可以使用Camunda DMN引擎集成来进行一个DMN决策。你必须使用`camunda:decisionRef`属性
指定要评估的决策键。此外。
`camunda:decisionRefBinding`指定哪一个版本的决策应该被评估。
`camunda:decisionRefBinding`的有效值是：

* `deployment`, 这会评估deployment部署的决策版本，
* `latest` 这将始终评估最新的决策版本，
* `version` 这允许你通过使用`camunda:decisionRefVersion`属性指定执行的特定版本，
* `versionTag` 这允许你用`camunda:decisionRefVersionTag`属性指定一个特定的版本标签来执行。

```xml
<businessRuleTask id="businessRuleTask"
    camunda:decisionRef="myDecision"
    camunda:decisionRefBinding="version"
    camunda:decisionRefVersion="12" />
```

`camunda:decisionRefBinding`属性默认为`latest`。

```xml
<businessRuleTask id="businessRuleTask"
    camunda:decisionRef="myDecision" />
```

属性`camunda:decisionRef`、`camunda:decisionRefVersion`和`camunda:decisionRefVersionTag`可以被指定为
一个表达式，它将在任务的执行中被计算。

```xml
<businessRuleTask id="businessRuleTask"
    camunda:decisionRef="${decisionKey}"
    camunda:decisionRefBinding="version"
    camunda:decisionRefVersion="${decisionVersion}" />
```

决策的输出，也称为决策结果，不会自动保存为流程变量。它必须通过使用[预定义]({{< ref "/user-guide/process-engine/decisions/bpmn-cmmn.md#predefined-mapping-of-the-decision-result" >}})或者[定制]({{< ref "/user-guide/process-engine/decisions/bpmn-cmmn.md#custom-mapping-into-process-variables" >}})的方式传递到一个流程变量中。

如果是预定义的映射，`camunda:mapDecisionResult`属性会引用要使用的映射器。映射的结果被保存在由`camunda:resultVariable`属性指定的变量中。如果没有设置预定义的映射器，那么默认使用 `resultList` 映射器。

```xml
<businessRuleTask id="businessRuleTask"
    camunda:decisionRef="myDecision"
    camunda:mapDecisionResult="singleEntry"
    camunda:resultVariable="result" />
```

参见 [用户指南]({{< ref "/user-guide/process-engine/decisions/bpmn-cmmn.md#the-decision-result" >}}) 有关映射的详细信息。

{{< note title="结果变量的名称" class="warning" >}}
结果变量不应该使用 `decisionResult`这个名字，因为决策结果本身是保存在一个这个变量中。否则，在保存结果变量时就会出现异常。
{{< /note >}}

# 决策参考中的租户Id （Tenant Id）

当业务规则任务（Business Rule Task）解析要进行评估的决策定义时，必须考虑多租户的情况。

## 默认租户（Tenant）决定
默认情况下，调用流程定义的租户id 被用来评估决策定义。
也就是说，如果调用的流程定义没有租户 ID，那么业务规则任务将使用所提供的key和没有 租户ID 的决策定义绑定进行评估（租户 ID = null）。
如果调用的流程定义有租户ID，那么将评估具有所提供的key和相同租户ID 的决策定义。

注意，调用进程实例的租户ID在默认情况下是不被考虑的。

## 额外的租户（Tenant）决定

在某些情况下，覆盖默认行为并明确指定租户ID是有用的。

`camunda:decisionRefTenantId`属性允许明确指定租户ID。

```xml
<businessRuleTask id="businessRuleTask" decisionRef="myDecision"
  camunda:decisionRefTenantId="TENANT_1">
</businessRuleTask>
```

如果在设计流程时不知道租户ID，也可以使用一个表达式：

```xml
<businessRuleTask id="businessRuleTask" decisionRef="myDecision"
  camunda:decisionRefTenantId="${ myBean.calculateTenantId(variable) }">
</businessRuleTask>
```

表达式也允许使用调用流程实例的租户ID，而不是调用流程定义的：

```xml
<businessRuleTask id="businessRuleTask" decisionRef="myDecision"
  camunda:decisionRefTenantId="${ execution.tenantId }">
</businessRuleTask>
```

# 使用自定义规则引擎

你可以与其他规则引擎整合。要做到这一点，你必须实现你的规则任务（rule task），规则任务与服务任务（Service Task）实现的方式是相同的。

```xml
<businessRuleTask id="businessRuleTask"
    camunda:delegateExpression="${MyRuleServiceDelegate}" />
```


# 使用委托代码

另外，业务规则任务也可以像服务任务一样使用Java委托来实现。要了解更多相关信息，
请参见 [服务任务]({{< relref "service-task.md" >}}) 章节.


# 实现为外部任务

除上述内容外，还可以通过[外部任务]({{< ref "/user-guide/process-engine/external-tasks.md" >}})机制实现业务规则任务，
外部系统会轮询流程引擎的工作。参见[服务任务]({{< relref "service-task.md#external-tasks" >}})一节，
了解更多关于如何配置外部任务的信息。


# Chamunda 扩展

<table class="table table-striped">
  <tr>
    <th>属性</th>
    <td>
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncbefore" >}}">camunda:asyncBefore</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncafter" >}}">camunda:asyncAfter</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#class" >}}">camunda:class</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#decisionref" >}}">camunda:decisionRef</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#decisionrefbinding" >}}">camunda:decisionRefBinding</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#decisionreftenantid" >}}">camunda:decisionRefTenantId</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#decisionrefversion" >}}">camunda:decisionRefVersion</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#decisionrefversiontag" >}}">camunda:decisionRefVersionTag</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#delegateexpression" >}}">camunda:delegateExpression</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#exclusive" >}}">camunda:exclusive</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#expression" >}}">camunda:expression</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#jobpriority" >}}">camunda:jobPriority</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#mapdecisionresult" >}}">camunda:mapDecisionResult</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#resultvariable" >}}">camunda:resultVariable</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#topic" >}}">camunda:topic</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#type" >}}">camunda:type</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#taskpriority" >}}">camunda:taskPriority</a>
    </td>
  </tr>
  <tr>
    <th>扩展元素</th>
    <td>
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#failedjobretrytimecycle" >}}">camunda:failedJobRetryTimeCycle</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#field" >}}">camunda:field</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#connector" >}}">camunda:connector</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#inputoutput" >}}">camunda:inputOutput</a>
    </td>
  </tr>
  <tr>
    <th>约束</th>
    <td>
      One of the attributes <code>camunda:class</code>, <code>camunda:delegateExpression</code>, <code>camunda:decisionRef</code>,
      <code>camunda:type</code> or <code>camunda:expression</code> is mandatory
    </td>
  </tr>
  <tr>
    <td></td>
    <td>
      The attribute <code>camunda:resultVariable</code> can only be used in combination with the
      <code>camunda:decisionRef</code> or <code>camunda:expression</code> attribute
    </td>
  </tr>
  <tr>
    <td></td>
    <td>
      The <code>camunda:exclusive</code> attribute is only evaluated if the attribute
      <code>camunda:asyncBefore</code> or <code>camunda:asyncAfter</code> is set to <code>true</code>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>
      The attribute <code>camunda:topic</code> can only be used when the <code>camunda:type</code> attribute is set to <code>external</code>.
    </td>
  </tr>
  <tr>
    <td></td>
    <td>
      The attribute <code>camunda:taskPriority</code> can only be used when the <code>camunda:type</code> attribute is set to <code>external</code>.
    </td>
  </tr>
</table>


# 额外资料

* [决策]({{< ref "/user-guide/process-engine/decisions/_index.md" >}})
* [服务任务]({{< ref "/reference/bpmn20/tasks/service-task.md" >}})
* [Tasks](http://camunda.org/bpmn/reference.html#activities-task) in the [BPMN 2.0 Modeling Reference](http://camunda.org/bpmn/reference.html)
* [Demo using Drools on the Business Rule Task](https://github.com/camunda/camunda-consulting/tree/master/one-time-examples/order-confirmation-rules)
