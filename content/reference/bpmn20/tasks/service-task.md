---

title: 'Service Task （服务任务）'
weight: 10

menu:
  main:
    identifier: "bpmn-ref-tasks-service-task"
    parent: "bpmn-ref-tasks"
    pre: "Invoke or execute business logic."

---



服务任务用于调用服务。在Camunda中，这是通过调用Java代码来完成的，或者为外部工作者提供的工作项来完成以异步或直接调用Web服务的形式实现逻辑。

{{< bpmn-symbol type="service-task" >}}

# 调用Java Code.

有四种方式配置如何调用Java逻辑：

* 指定实现Java委托或活动行为的类
* 计算解析为委派对象的表达式
* 调用方法表达式
* 计算表达式

要指定在进程执行期间调用的类，所需的完全限定类名需要由`camunda:class`属性提供。

```xml
<serviceTask id="javaService"
             name="My Java Service Task"
             camunda:class="org.camunda.bpm.MyJavaDelegate" />
```

请参阅[User Guide]({{< ref "/user-guide/_index.md" >}}) 中的 [Java 委托类]({{< ref "/user-guide/process-engine/delegation-code.md#java-delegate" >}}) 章节，了解有关如何实现Java委托的详细信息。

还可以使用解析为对象的表达式。这个对象必须遵循使用与`Camunda：Class`属性时创建的对象相同的规则。

```xml
<serviceTask id="beanService"
             name="My Bean Service Task"
             camunda:delegateExpression="${myDelegateBean}" />
```

或调用方法或解析值的表达式。

```xml
<serviceTask id="expressionService"
             name="My Expression Service Task"
             camunda:expression="${myBean.doWork()}" />
```

有关表达式语言作为委派代码的详细信息，请参阅相应的
[User Guide]({{< ref "/user-guide/_index.md" >}}) 的
[section]({{< ref "/user-guide/process-engine/expression-language.md#use-expression-language-as-delegation-code" >}}) 章节。

也可以调用以Web服务形式实现的逻辑。 `camunda:connector` 是一个扩展，允许直接从工作流程调用REST/SOAP API。
有关使用连接器的更多信息，请参阅相应的信息 [User Guide]({{< ref "/user-guide/_index.md" >}}) 中的 [section]({{< ref "/user-guide/process-engine/connectors.md#use-connectors" >}}) 章节。


## 通过 Java 委托类 & 参数注入

您可以轻松地编写通用Java委托类，该类可以通过BPMN 2稍后配置.  0 XML在服务任务中。请参考 [User Guide]({{< ref "/user-guide/_index.md" >}}) 中的 [参数注入]({{< ref "/user-guide/process-engine/delegation-code.md#field-injection" >}}) 章节。


## 服务任务结果

服务执行的返回值（对于专门使用表达式的服务任务）可以通过将 `camunda:resultVariable` 指定为流程变量名来分配给已存在的或新进程变量。特定进程变量的任何现有值将被服务执行的结果值覆盖。未指定结果变量名时，忽略服务执行结果值。

```xml
<serviceTask id="aMethodExpressionServiceTask"
           camunda:expression="#{myService.doSomething()}"
           camunda:resultVariable="myVar" />
```

在上面的示例中，服务执行的结果 (返回值的 `doSomething()`对象上的方法调用 `myService`) 在服务执行完成后，将被设置为命名的流程变量 `myVar` 。

{{< note title="结果变量和多实例" class="warning" >}}
请注意，当您使用时 <code>camunda:resultVariable</code> 在多实例构造中，例如在多实例子处理中，每次任务完成时都会覆盖结果变量，这可能显示为随机行为。看 <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#resultvariable" >}}">camunda:resultVariable</a> 有关详细信息。
{{< /note >}}

# 外部任务

与调用Java代码相比，进程引擎同步调用Java逻辑，可以以外部任务的形式实现流程引擎边界之外的服务任务。当服务任务声明为外部时，流程引擎为工人提供工作项，该工人独立地研究了发动机以进行工作。这使得从过程引擎执行任务的实施，并允许跨系统和技术边界。查阅 [外部任务的用户指南]({{< ref "/user-guide/process-engine/external-tasks.md" >}}) 了解相关概念和相关API的详细信息。

要声明要在外部处理的服务任务，属性 `camunda:type` 可以设置为 `external` 和属性 `camunda:topic` 指定外部任务的主题。例如，以下XML片段定义了具有主题的外部服务任务`ShipmentProcessing`:

```xml
<serviceTask id="anExternalServiceTask"
           camunda:type="external"
           camunda:topic="ShipmentProcessing" />
```

# Camunda 扩展

<table class="table table-striped">
  <tr>
    <th>属性</th>
    <td>
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncbefore" >}}">camunda:asyncBefore</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncafter" >}}">camunda:asyncAfter</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#class" >}}">camunda:class</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#delegateexpression" >}}">camunda:delegateExpression</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#exclusive" >}}">camunda:exclusive</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#expression" >}}">camunda:expression</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#jobpriority" >}}">camunda:jobPriority</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#resultvariable" >}}">camunda:resultVariable</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#topic" >}}">camunda:topic</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#type" >}}">camunda:type</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#taskpriority" >}}">camunda:taskPriority</a>
    </td>
  </tr>
  <tr>
    <th>扩展元素</th>
    <td>
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#erroreventdefinition" >}}">camunda:errorEventDefinition</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#failedjobretrytimecycle" >}}">camunda:failedJobRetryTimeCycle</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#field" >}}">camunda:field</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#connector" >}}">camunda:connector</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#inputoutput" >}}">camunda:inputOutput</a>
    </td>
  </tr>
  <tr>
    <th>约束</th>
    <td>
      One of the attributes <code>camunda:class</code>, <code>camunda:delegateExpression</code>,
      <code>camunda:type</code> or <code>camunda:expression</code> is mandatory
    </td>
  </tr>
  <tr>
    <td></td>
    <td>
      The attribute <code>camunda:resultVariable</code> can only be used in combination with the
      <code>camunda:expression</code> attribute
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
  <tr>
    <td></td>
    <td>
      The element <code>camunda:errorEventDefinition</code> can only be used as a child of <code>serviceTask</code> when the <code>camunda:type</code> attribute is set to <code>external</code>.
    </td>
  </tr>
</table>


# 相关资源

* [BPMN Modeling Reference](http://camunda.org/bpmn/reference.html) 的 [Tasks](http://camunda.org/bpmn/reference.html#activities-task) 章节。
* [How to call a Webservice from BPMN](http://www.bpm-guide.de/2010/12/09/how-to-call-a-webservice-from-bpmn/)。请注意，本文已过时。但是，关于如何使用流程引擎调用Web服务的方式仍然有效。
