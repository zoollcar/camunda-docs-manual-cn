---

title: 'Script Task（脚本任务）'
weight: 50

menu:
  main:
    identifier: "bpmn-ref-tasks-script-task"
    parent: "bpmn-ref-tasks"
    pre: "Execute a Script."

---

脚本任务是自动活动。当进程执行到达脚本任务时，执行相应的脚本。

{{< bpmn-symbol type="script-task" >}}

通过指定脚本内容和脚本格式来定义脚本任务：

```xml
<scriptTask id="theScriptTask" name="Execute script" scriptFormat="groovy">
  <script>
    sum = 0
    for ( i in inputArray ) {
      sum += i
    }
  </script>
</scriptTask>
```

scriptFormat属性的值必须是一个与JSR-223（Java平台的脚本）兼容的名称。如果你想使用一个（与JSR-223兼容的）脚本引擎，你需要在classpath中添加相应的jar，并使用适当的名称。

脚本源代码必须作为`script`子元素的文本内容添加。
或者，源代码可以被指定为表达式或外部资源。更多
关于提供脚本源代码的可能方式的更多信息，请参见相应的
[用户指南][user-guide] 的 [脚本来源][script-source] 章节。

关于进程引擎中脚本的一般信息，请参见[用户指南][user-guide]中的[脚本]({{< ref "/user-guide/process-engine/scripting.md" >}})部分。

{{< note title="支持的脚本语言" class="info" >}}

Camunda平台应该可以与大多数JSR-223兼容的脚本引擎实现协同工作。我们测试了Groovy、JavaScript、JRuby和Jython的集成。请参阅<a href="{{< ref "/user-guide/_index.md" >}}">用户指南</a> 的 <a href="{{< ref "/introduction/third-party-libraries/_index.md#process-engine" >}}">第三方依赖性</a>部分了解更多细节。
{{< /note >}}

# 脚本中的变量

所有通过执行到达脚本任务中的进程变量都可以在脚本中使用。在下面的例子中，脚本变量`inputArray`实际上是一个流程变量（一个整数的列表）。

```xml
<script>
    sum = 0
    for ( i in inputArray ) {
      sum += i
    }
</script>
```

也可以在脚本中设置过程变量。变量可以通过使用`VariableScope`接口提供的`setVariable(..)`方法来设置。


```xml
<script>
    sum = 0
    for ( i in inputArray ) {
      sum += i
    }
    execution.setVariable("sum", sum);
</script>
```

## 启用自动存储脚本变量的功能

通过在流程引擎配置中设置属性`autoStoreScriptVariables`为true，流程引擎将自动把所有_global_脚本变量存储为流程变量。这是Camunda平台7.0和7.1中的默认行为，但它只可靠地适用于Groovy脚本语言（更多信息请参见[迁移指南]({{< ref "/update/_index.md" >}})中的[设置自动存储脚本变量][autostore-variables] 部分）。

要使用这一功能，你必须

* 在进程引擎配置中把 `autooStoreScriptVariables` 设置为 `true`。
* 用`def`关键字声明给所有不应该被存储为脚本变量的脚本变量：`def sum = 0`。在这种情况下，变量`sum`将不会被存储为进程变量。

{{< note title="只有Groovy支持" class="info" >}}
配置标志<code>autoStoreScriptVariables</code>只支持Groovy脚本任务。如果为其他脚本语言启用。
它不能保证哪些变量会被脚本引擎导出。例如
例如，Ruby根本就不会导出任何脚本变量。
{{< /note >}}

注意：以下名称是保留的，不能作为变量名使用：
`out`, `out:print`, `lang:import`, `context`, `elcontext`.


# 脚本返回结果

脚本任务的返回值可以分配给先前存在的或新的进程变量，方法是将进程变量名称作为脚本任务定义的`camunda:resultVariable`属性的字面值。特定过程变量的任何现有值将被脚本执行的结果值所覆盖。当没有指定结果变量名称时，脚本的结果值会被忽略。

```xml
<scriptTask id="theScriptTask" name="Execute script" scriptFormat="juel" camunda:resultVariable="myVar">
  <script>#{echo}</script>
</scriptTask>
```

在上面的例子中，脚本执行的结果（解析表达式`#{echo}`的值）在脚本完成后被设置到名为`myVar`的流程变量中。

{{< note title="结果变量和多实例" class="warning" >}}
请注意，当您在多实例构造中使用<code>camunda:resultVariable</code>时，例如在多实例子进程中，每次任务完成后，结果变量都会被覆盖，这可能会表现为随机行为。参见<a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#resultvariable" >}}">camunda:resultVariable</a>了解详情。
{{< /note >}}


# Camunda 扩展

<table class="table table-striped">
  <tr>
    <th>属性</th>
    <td>
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncbefore" >}}">camunda:asyncBefore</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncafter" >}}">camunda:asyncAfter</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#exclusive" >}}">camunda:exclusive</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#jobpriority" >}}">camunda:jobPriority</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#resultvariable" >}}">camunda:resultVariable</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#resource" >}}">camunda:resource</a>
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


[script-source]: {{< ref "/user-guide/process-engine/scripting.md#script-source" >}}
[user-guide]: {{< ref "/user-guide/_index.md" >}}
[autostore-variables]: {{< ref "/update/minor/71-to-72/_index.md#script-variable-storing" >}}
