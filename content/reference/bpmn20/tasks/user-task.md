---

title: 'User Task 人工任务'
weight: 30

menu:
  main:
    identifier: "bpmn-ref-tasks-user-task"
    parent: "bpmn-ref-tasks"
    pre: "A task performed by a human participant."

---

用户任务用于模拟人类演员需要完成的工作。当流程执行到达这样的用户任务时，在分配给该任务的用户或组的任务列表中创建一个新任务。

{{< bpmn-symbol type="user-task" >}}

用户任务在XML中定义如下。 id属性是必需的，而name属性是可选的。

```xml
<userTask id="theTask" name="Important task" />
```


# 描述

用户任务还可以具有描述。事实上，任何BPMN 2.0元素可以有描述。通过添加 documentation 元素来定义描述。

```xml
<userTask id="theTask" name="Schedule meeting" >
  <documentation>
      Schedule an engineering meeting for next week with the new hire.
  </documentation>
```

可以从标准Ja​​va方式中的任务中检索 documentation 文本：

```java
task.getDescription();
```


# 特性

## 截止日期

每个任务都有一个字段，指示该任务的截止日期。查询API可用于查询到某个日期之前或之后截止的任务。

有一个活动扩展名，允许您在任务定义中指定表达式，以在创建时设置任务的初始截止日期。表达式应该始终解析为 java.util.Date, java.util.String ([ISO8601](http://en.wikipedia.org/wiki/ISO_8601) 格式化) 或 null。 使用ISO8601格式化字符串时，您可以指定相对于创建任务的时间的时间或时间段。例如，您可以使用在此过程中以先前表单输入的日期，或者在以前的服务任务中计算。

```xml
<userTask id="theTask" name="Important task" camunda:dueDate="${dateVariable}"/>
```

任务的截止日期也可以使用任务服务或任务侦听器中使用传递的委托任务更改。

## 跟进日期

每个任务都有一个字段，指示该任务的跟进日期。查询API可用于查询需要在特定日期之前或之后进行操作的任务。

有一个活动扩展名，允许您在任务定义中指定表达式，以在创建时设置任务的初始后续后续日期。表达式应该始终解析为 java.util.Date, java.util.String ([ISO8601](http://en.wikipedia.org/wiki/ISO_8601) 格式化) 或 null。 使用ISO8601格式化字符串时，您可以指定相对于创建任务的时间的时间或时间段。例如，您可以使用在此过程中以先前表单输入的日期，或者在以前的服务任务中计算。

```xml
<userTask id="theTask" name="Important task" camunda:followUpDate="${dateVariable}"/>
```

# 用户分配

用户任务可以直接分配给单个用户，用户列表或组列表。

## 使用BPMN资源分配分配

BPMN定义了一些可以在Camunda中使用的本机分配概念。
作为一种更强大的替代方案，Camunda还定义了一组自定义扩展元素（见下文）。

### Human Performer

这是通过定义人类执行者的子元素来完成的。这种人类执行者定义需要实际定义用户的资源分配表达式。目前，只有 formalExpressions 支持.

```xml
<process ... >
  ...
  <userTask id='theTask' name='important task' >
    <humanPerformer>
      <resourceAssignmentExpression>
        <formalExpression>kermit</formalExpression>
      </resourceAssignmentExpression>
    </humanPerformer>
  </userTask>
```

只有一个用户可以分配给任务作为人类执行者。在引擎术语中，此用户称为assignee（受让人）。在其他用户的任务列表中，具有assignee（受让人）的任务在其他用户的任务列表中不可见，只能在受让人的个人任务列表中找到。

可以通过任务服务检索直接分配给用户的任务，如下所示：

```java
List<Task> tasks = taskService.createTaskQuery().taskAssignee("kermit").list();
```

### Potential Owner

任务也可以放在所谓的候选人任务列表中。在这种情况下，必须使用潜在的所有者构造。使用类似于人类执行者构造。请注意，对于formalExpressions的每个人选，需要特别定义它是用户或组（引擎无法猜到）。

```xml
<process ... >
  ...
  <userTask id='theTask' name='important task' >
    <potentialOwner>
      <resourceAssignmentExpression>
        <formalExpression>user(kermit), group(management)</formalExpression>
      </resourceAssignmentExpression>
    </potentialOwner>
  </userTask>
```

可以使用潜在所有者构造定义的任务，如下所示（或者可以使用类似的任务查询，例如用于具有受让人的任务）：

```java
List<Task> tasks = taskService.createTaskQuery().taskCandidateUser("kermit");
```

这将检索所有`kermit`是候选用户的任务，也就是说，formalExpression 包含用户`kermit`。这也将检索所有分配给`kermit`为成员的组的任务（`group(management)`，如果kermit是该组的成员并且使用了身份组件）。
用户的组在运行时被解析，这些可以通过IdentityService来管理。

如果没有给出给定文本字符串是用户或组的细节，则引擎默认为组。所以以下两个写法是相同的：

```xml
<formalExpression>accountancy</formalExpression>
<formalExpression>group(accountancy)</formalExpression>
```

## 使用Camunda扩展进行用户分配

很明显，用户和组分配非常麻烦，使用分配更复杂。为避免这些复杂性，可以进行用户任务的自定义扩展。

### Assignee（受让人）

`assignee` 属性: 这个自定义扩展允许将用户任务直接分配给一个给定的用户。

```xml
<userTask id="theTask" name="my task" camunda:assignee="kermit" />
```
这与使用上面定义的humanPerformer结构完全相同。

### Candidate Users （候选人）

`candidateUsers` 属性: 这个自定义扩展允许你使用户成为一项任务的候选人。

```xml
<userTask id="theTask" name="my task" camunda:candidateUsers="kermit, gonzo" />
```

这与使用上面定义的潜在所有者构造完全相同。请注意，不需要使用 `user(kermit)` 声明与潜在所有者构造的情况一样，因为此属性只能用于用户。

### Candidate Groups （候选组）

`candidateGroups` 属性: 这个自定义扩展允许你使一个组成为一个任务的候选人。

```xml
<userTask id="theTask" name="my task" camunda:candidateGroups="management, accountancy" />
```

这与使用上面定义的潜在所有者构造完全相同。请注意，不需要使用 `group(management)` 声明与潜在所有者构造的情况一样，因为此属性只能用于组。

### 结合使用候选用户和组

`candidateUsers` 和 `candidateGroups` 可以同时为一个用户任务定义。


## 基于数据和服务逻辑的赋值

在上面的例子中， 使用了类似 `kermit` 或 `management` 的常量值。 但是如果在设计时不知道被指派人或候选组的确切名称呢？ 如果被指派者不是一个常量值，而是取决于数据例如 _"The person who started the process"_ 应该怎么办? 有时赋值逻辑也更复杂，甚至需要访问外部数据源，例如LDAP以实现诸如 _"The manager of the employee who started the process"_ 的查询。

可以使用赋值表达式或任务侦听器来实现这样的功能。

### 赋值表达

赋值表达式允许访问流程变量或调用beans和services。

#### 使用流程变量

流程变量对于基于已经收集或计算的数据的分配非常有用。

以下示例显示了如何将用户任务分配给启动该过程的人：

```xml
<startEvent id="startEvent" camunda:initiator="starter" />
...
<userTask id="task" name="Clarify Invoice" camunda:assignee="${ starter }"/>
...
```

首先, `camunda:initiator` 参数用于绑定启动的人的用户标识 (_"initiated"_) 变量的过程 `starter`. 然后是表达式 `${ starter }` 检索该值并将其用作任务的assignee（受让人）。

可以使用所有[可见]({{< ref "/user-guide/process-engine/variables.md#variable-scopes-and-variable-visibility" >}})的过程变量用于用户任务的表达式。

#### 调用 Service / Bean

当使用Spring或CDI时，有可能委托给Bean或Service实现。这样就有可能调用复杂的赋值逻辑，而不需要在过程中把它建模为一个显式的服务任务，然后产生一个用于赋值的变量。

在下面的例子中，assignee（受让人）将通过调用名为`ldapService`的Spring/CDI bean上的`findManagerOfEmployee()`方法来设置。输出的`emp`参数是一个流程变量。

```xml
<userTask id="task" name="My Task" camunda:assignee="${ldapService.findManagerForEmployee(emp)}"/>
```

这也适用于候选用户和组的类似方式工作：

```xml
<userTask id="task" name="My Task" camunda:candidateUsers="${ldapService.findAllSales()}"/>
```

请注意，调用方法的返回类型只能是String(候选用户)或Collection\<String\>（组）：

```java
public class FakeLdapService {

  public String findManagerForEmployee(String employee) {
    return "Kermit The Frog";
  }

  public List<String> findAllSales() {
    return Arrays.asList("kermit", "gonzo", "fozzie");
  }
}
```

### 在任务侦听器（task listener）中进行分配

也可以使用[任务侦听器]({{< ref "/user-guide/process-engine/delegation-code.md#task-listener" >}}) 处理任务。以下示例演示了“创建”事件的任务侦听器：

```xml
<userTask id="task1" name="My task" >
  <extensionElements>
    <camunda:taskListener event="create" class="org.camunda.bpm.MyAssignmentHandler" />
  </extensionElements>
</userTask>
```

传递给任务侦听器实现的委托任务允许您设置受让人和候选用户/组：

```java
public class MyAssignmentHandler implements TaskListener {
  public void notify(DelegateTask delegateTask) {
    // Execute custom identity lookups here
    // and then for example call following methods:
    delegateTask.setAssignee("kermit");
    delegateTask.addCandidateUser("fozzie");
    delegateTask.addCandidateGroup("management");
    ...
  }
}
```

{{< note title="提示" class="info" >}}
通过TaskListener分配任务或设置任何其他属性，将不会触发 `assignment` 或 `update` 事件。使用 `TaskService` 则会触发。
这是故意设计的，用来避免产生事件循环。
{{< /note >}}

## 在身份服务（Identity Service） 中进行分配

尽管Camunda引擎提供了一个身份管理组件，该组件通过IdentityService访问，但它并不检查所提供的用户是否被 identity component 所知。这提供了当引擎被嵌入到一个应用程序中时，可以与现有的身份管理解决方案进行整合的能力。

然而，如果这对你有帮助的话，你可以在一个 service/bean或监听器中使用身份服务来查询你的用户库。

你可以在身份服务的帮助下查询用户。请看下面的例子。

```java
ProcessEngine processEngine = delegateTask.getProcessEngine();
IdentityService identityService = processEngine.getIdentityService();

List<User> managementUsers = identityService.createUserQuery()
    .memberOfGroup("management")
    .list();

User kermit = identityService.createUserQuery()
    .userFirstName("kermit")
    .singleResult();
```

# 报告BPMN错误

查看[错误边界事件(boundary event)]({{< ref "/reference/bpmn20/events/error-events.md#error-boundary-event" >}})的文档。

要报告用户任务操作期间的业务错误，请使用`TaskService#handleBpmnError`。它只能在任务处于活动状态时被调用。
`#handleBpmnError`方法需要一个强制参数：`errorCode`。
`errorCode` 确定了一个预定义的错误。如果给定的`errorCode`不存在或者没有定义边界事件(boundary event)。
当前的活动实例就会结束，错误不会被处理。

请看下面的例子：

```java

Task task = taskService.createTaskQuery().taskName("Perform check").singleResult();

// ... 出现业务错误

taskService.handleBpmnError(
  task.getId(),
  "bpmn-error-543", // errorCode
  "Thrown BPMN Error during...", // 错误信息
  variables);
```

一个错误代码为`bpmn-error-543`的BPMN错误将被传播。如果存在处理该错误代码的边界事件，BPMN错误将被捕获和处理。
错误信息和变量是可选的。它们可以为错误提供额外的信息。如果BPMN错误被捕获，这些变量将被传递给执行。

# 报告BPMN升级

查看[捕获升级事件]({{< ref "/reference/bpmn20/events/escalation-events.md#catching-escalation-events" >}})的文档。

在用户任务执行过程中报告升级可以通过`TaskService#handleEscalation`实现。调用时用户任务应处于活动状态，`escalationCode`是调用升级的必要参数，这个代码标识了一个预定义的升级。如果给定的`escalationCode`不存在，将抛出流程引擎异常。请看下面的例子：

```java
taskService.handleEscalation(
  taskId,
  "escalation-432", // escalationCode
  variables);
```

这样，升级将被传播，升级代码为`escalation-432`。如果存在处理该升级代码的边界事件，升级将被捕捉和处理。
这些变量是可选的。如果升级被捕获，它们将被传递给执行。

# 完成任务

完成状态是[任务生命周期]({{< ref "/webapps/tasklist/task-lifecycle.md" >}})的一部分。 通过传递变量来完成一个任务，同时也能查询流程变量：

```java
taskService.complete(taskId, variables);

// 或完成任务并查询流程变量
VariableMap processVariables = taskService
  .completeWithVariablesInReturn(taskId, variables, shouldDeserializeValues);
```

# 表单（Forms）

可以通过使用`camunda:formKey`来渲染用户任务表单。
参考：

```xml
<userTask id="someTask" camunda:formKey="someForm.html">
  ...
</userTask>
```

这里的 form key 是一个在 BPMN XML 文件中定义的符号值， 在运行时会被流程引擎用于查询

如果用户任务表单显示在Camunda任务列表内，formKey的格式必须遵循规则。 [有关详细信息，请参阅“用户指南”中的相应部分]({{< ref "/user-guide/task-forms/_index.md" >}}).

在自定义应用程序中，form key 属性的值可以自由设置。根据选择的特定UI技术不同，可以是一个 HTML 文件或者 JSF / Facelets 模板，Vaadin / GWT 视图，..

## 使用表单服务（formService）检索form key。

```java
String formKey = formService.getTaskFormData(someTaskId).getFormKey();
```

## 使用任务服务（TaskService）检索表单

执行任务查询时，也可以检索form key。这是很有用的
如果需要检索form key以获取完整的任务列表：

```java
List<Task> tasks = TaskService.createTaskQuery()
  .assignee("jonny")
  .initializeFormKeys() // 这句必须被调用
  .list();

for(Task task : tasks) {
  String formKey = task.getFormKey();
}
```

请注意，需要调用`initializeFormKeys()`在查询方法之前，确保初始化form key。

## 表格提交

提交表单时，可以获取流程变量以返回：

```java
VariableMap processVariables = formService
  .submitTaskFormWithVariablesInReturn(taskId, properties, shouldDeserializeValues);

// or avoid unnecessary variable access
formService.submitTaskForm(taskId, properties);
```

# Camunda 扩展

<table class="table table-striped">
  <tr>
    <th>属性</th>
    <td>
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#assignee" >}}">camunda:assignee</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncbefore" >}}">camunda:asyncBefore</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncafter" >}}">camunda:asyncAfter</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#candidategroups" >}}">camunda:candidateGroups</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#candidateusers" >}}">camunda:candidateUsers</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#duedate" >}}">camunda:dueDate</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#exclusive" >}}">camunda:exclusive</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#formhandlerclass" >}}">camunda:formHandlerClass</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#formkey" >}}">camunda:formKey</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#jobpriority" >}}">camunda:jobPriority</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#priority" >}}">camunda:priority</a>
    </td>
  </tr>
  <tr>
    <th>扩展元素</th>
    <td>
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#formdata" >}}">camunda:formData</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#formproperty" >}}">camunda:formProperty</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#tasklistener" >}}">camunda:taskListener</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#failedjobretrytimecycle" >}}">camunda:failedJobRetryTimeCycle</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#inputoutput" >}}">camunda:inputOutput</a>
    </td>
  </tr>
  <tr>
    <th>约束</th>
    <td>
      The attribute <code>camunda:assignee</code> cannot be used simultaneously with the <code>humanPerformer</code>
      element
    </td>
  </tr>
  <tr>
    <td></td>
    <td>
      Only one <code>camunda:formData</code> extension element is allowed
    </td>
  </tr>
  <tr>
    <td></td>
    <td>
      The <code>camunda:exclusive</code> attribute is only evaluated if the attribute
      <code>camunda:asyncBefore</code> or <code>camunda:asyncAfter</code> is set to <code>true</code>
    </td>
  </tr>
</table>
