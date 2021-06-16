---

title: '流程引擎概念'
weight: 30

menu:
  main:
    identifier: "user-guide-process-engine-concepts"
    parent: "user-guide-process-engine"

---


本节介绍了一些核心过程引擎概念，用于流程引擎API和内部流程引擎实现。了解这些基础知识可以更容易的额使用流程引擎API。


# 流程定义

流程定义定义了一个流程的结构。你可以说，流程定义**就是**流程。Camunda平台使用 [BPMN 2.0](http://camunda.org/bpmn/tutorial.html) 作为其主要的流程建模语言，用于建模过程的定义。

{{< note title="BPMN 2.0 参考" class="info" >}}
  Camunda平台配有两个BPMN 2.0参考资料：

* [BPMN 2.0 建模参考](http://camunda.org/bpmn/reference.html#!/reference) 介绍了BPMN 2.0的基本原理，帮助你开始对流程进行建模。(请确保同时阅读[教程](http://camunda.org/bpmn/tutorial.html)。)
* [BPMN 2.0 实现参考]({{< ref "/reference/bpmn20/_index.md" >}}) 涵盖了Camunda平台中各个BPMN 2.0构造的实现。如果你想实现和执行BPMN流程，你应该阅读这个参考资料。
{{< /note >}}

在Camunda平台中，您可以将流程以BPMN 2.0 XML格式部署到流程引擎中。XML文件被解析并转化为一个流程定义图结构。该图结构由流程引擎执行。

## 流程定义的查询

你可以使用Java API和通过`RepositoryService`提供的`ProcessDefinitionQuery`来查询所有部署的流程定义。例如：

```java
List<ProcessDefinition> processDefinitions = repositoryService.createProcessDefinitionQuery()
    .processDefinitionKey("invoice")
    .orderByProcessDefinitionVersion()
    .asc()
    .list();
```

上面的查询返回所有已部署的流程定义，关键是`invoice`，按其`version`属性排序。

你也可以[使用REST API查询流程定义]({{< ref "/reference/rest/process-definition/get-query.md" >}}).


## Key和版本

流程定义的**key**（上面例子中的 "invoice"）是流程的逻辑标识符。它在整个API中被使用，最典型的是用于启动流程实例，（见[流程实例部分]({{< relref "#process-instances" >}})）。流程定义的关键是使用BPMN2.0 XML文件中相应的`<process ... >`元素的`id`属性来定义的。

```xml
<process id="invoice" name="invoice receipt" isExecutable="true">
  ...
</process>
```

如果你用同一个key部署了多个流程，流程引擎会将它们视为同一流程定义的不同版本。请参考[流程版本管理]({{< ref "/user-guide/process-engine/process-versioning.md" >}})了解详情。


## 禁用流程定义

禁用一个流程定义会使其暂时禁止实例化。`RuntimeService` Java API可以暂停一个流程定义。同样地，你可以激活一个流程定义来撤销这个效果。


# 流程实例

流程实例是一个流程定义的单独执行。流程实例与流程定义的关系与面向对象编程中**对象**和**类**的关系类似（流程实例扮演对象的角色，流程定义扮演类的角色）。

流程引擎负责创建流程实例并管理其状态。如果你启动一个包含等待状态的流程实例，例如一个[用户任务]({{< ref "/reference/bpmn20/tasks/user-task.md" >}})，流程引擎必须确保流程实例的状态被捕获并存储在数据库内，直到离开等待状态（用户任务被完成）。

## 启动流程实例

启动一个流程实例的最简单方法是使用RuntimeService提供的`startProcessInstanceByKey(..)`方法。

    ProcessInstance instance = runtimeService.startProcessInstanceByKey("invoice");

你可以选择性地传入几个变量：

    Map<String, Object> variables = new HashMap<String,Object>();
    variables.put("creditor", "Nice Pizza Inc.");
    ProcessInstance instance = runtimeService.startProcessInstanceByKey("invoice", variables);

流程变量对流程实例中的所有任务来说都是可用的，并且在流程实例到达等待状态时自动持久化到数据库。

也可以[使用REST API启动一个流程实例]({{< ref "/reference/rest/process-definition/post-start-process-instance.md" >}}).

### 通过 Tasklist 创建流程实例

你可以使用 [Tasklist]({{< ref "/webapps/tasklist/working-with-tasklist.md#start-a-process" >}}) 创建流程实例，
[`startableInTasklist`]({{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#isstartableintasklist" >}}) 选项的存在是为了指定哪些流程是可以被用户启动的。

例如，如果只应启动父流程而不启动子流程，请按以下方式调整子流程xml文件是明智的（*.bpmn）。

```xml
<process id="subProcess"
         name="请直接发起父流程"
         isExecutable="true"
         camunda:isStartableInTasklist="false">
...
</process>
```

## 在任何一组活动中启动一个过程实例

`startProcessInstanceByKey`和`startProcessInstanceById`方法在其默认的初始活动处启动流程实例，这通常是流程定义的单一空白启动事件。通过使用流程实例的**fluent builder**，也可以在流程实例的任何地方启动。可以通过RuntimeService的方法`createProcessInstanceByKey`和`createProcessInstanceById`来访问fluent builder。

下面是在活动 "SendInvoiceReceiptTask "和嵌入式子流程 "DeliverPizzaSubProcess "之前启动一个流程实例。

```java
ProcessInstance instance = runtimeService.createProcessInstanceByKey("invoice")
  .startBeforeActivity("SendInvoiceReceiptTask") // 发送发票收据任务
  .setVariable("creditor", "Nice Pizza Inc.")
  .startBeforeActivity("DeliverPizzaSubProcess") // 提供披萨子流程
  .setVariableLocal("destination", "12 High Street")
  .execute();
```

fluent builder 允许提交任何数量的实例化指令。当调用`执行'时，流程引擎按照指定的顺序执行这些指令。在上面的例子中，引擎首先启动任务*SendInvoiceReceiptTask*并执行该流程，直到它达到等待状态，然后启动*DeliverPizzaTask*并做同样的事情。在这两条指令之后，"执行 "调用返回。

### 启动时返回的变量

要访问流程实例在执行过程中使用的最新变量，可以使用 "executeWithVariablesInReturn"，而不是 "execute "方法。
请看下面的例子：

```java
ProcessInstanceWithVariables instance = runtimeService.createProcessInstanceByKey("invoice")
  .startBeforeActivity("SendInvoiceReceiptTask")
  .setVariable("creditor", "Nice Pizza Inc.")
  .startBeforeActivity("DeliverPizzaSubProcess")
  .setVariableLocal("destination", "12 High Street")
  .executeWithVariablesInReturn();
```

如果流程实例结束或达到等待状态， `executeWithVariablesInReturn` 会返回。返回的`ProcessInstanceWithVariables`对象包含流程实例的信息和最新的变量。

## 查询流程实例

你可以使用`RuntimeService`提供的`ProcessInstanceQuery`查询所有当前运行的流程实例：

    runtimeService.createProcessInstanceQuery()
        .processDefinitionKey("invoice")
        .variableValueEquals("creditor", "Nice Pizza Inc.")
        .list();

上面的查询将选择 `invoice` 流程的所有流程实例，其中 `creditor` 是 `Nice Pizza Inc.`。

您还可以[使用REST API查询流程实例]({{< ref "/reference/rest/process-instance/get-query.md" >}}).


## 流程变量的交互

一旦你对一个特定的过程实例（或过程实例的列表）进行了查询，你可能想与它进行交互。有多种可能性与一个流程实例进行交互，最典型的是：

  * 触发它（使其继续执行）：
      * 通过 [消息事件]({{< ref "/reference/bpmn20/events/message-events.md" >}})
      * 通过 [信号事件]({{< ref "/reference/bpmn20/events/signal-events.md" >}})
  * 取消它：
      * 使用 `RuntimeService.deleteProcessInstance(...)` 方法.
  * 启动/取消任何活动：
      * 使用 [process instance modification feature]({{< ref "/user-guide/process-engine/process-instance-modification.md" >}})

如果你的流程包含用户任务，你也可以使用TaskService API与流程实例交互。

## 禁用流程实例

禁用流程实例是有帮助的，如果你想确保它不被进一步执行。例如，如果流程变量处于不合理的状态，你可以暂停该实例并“安全”地改变变量。

详细来说，暂停意味着不允许所有改变实例的“token”状态（即当前执行的活动）的行动。例如，不可能为暂停的流程实例发出事件信号或完成用户任务，因为这些操作随后会继续执行流程实例。尽管如此，像设置或删除变量这样的操作仍然是允许的，因为它们不会改变token状态。

另外，当暂停一个流程实例时，属于它的所有任务将被暂停。因此，将不再可能调用对任务的生命周期有影响的操作（即用户分配、任务委托、任务完成...）。然而，任何不涉及生命周期的操作，如设置变量或添加注释，仍是可以执行的。

一个流程实例可以通过使用`RuntimeService`的`suspendProcessInstanceById(...)`方法来禁用。同样，它也可以被重新激活。

如果你想禁用某个流程定义的所有流程实例，你可以使用`RepositoryService`的`suspendProcessDefinitionById(...)`方法并指定`suspendProcessInstances`选项。


# 执行（Executions）

如果你的流程实例包含多个执行路径（例如[并行网关]({{< ref "/reference/bpmn20/gateways/parallel-gateway.md" >}})），你必须能够区分流程实例内当前的活动路径。在下面的例子中，两个用户任务“receive payment”和“ship order”可以同时执行。

{{< img src="../img/parallel-gw.png" title="并行网关" >}}

在内部实现中，流程引擎在流程实例遇到并行网关时会创建两个并发的执行，每个并发的执行都有一个执行路径。
或者“范围”也会创建新的执行，例如，如果流程引擎到达一个[嵌入式子流程]({{< ref "/reference/bpmn20/subprocesses/embedded-subprocess.md" >}})，就会创建执行。
或者[多实例]({{< ref "/reference/bpmn20/tasks/task-markers.md" >}})也会创建新的执行。

执行是分层次的，一个流程实例内的所有执行组成一棵树，流程实例是树上的根节点。注意：流程实例本身就是一个执行。执行属于[变量范围]({{< ref "/user-guide/process-engine/variables.md" >}}), 意味着动态数据可以与之关联。

## 执行的查询

你可以使用`RuntimeService`提供的`ExecutionQuery`来查询执行情况。

```java
runtimeService.createExecutionQuery()
    .processInstanceId(someId)
    .list();
```

上述查询返回一个给定进程实例的所有执行情况。

你也可以[使用REST API查询执行情况]({{< ref "/reference/rest/execution/get.md" >}}).


# 活动实例（Activity Instances）

活动实例的概念与执行的概念相似，但采取了不同的视角。执行可以被想象成在流程中移动的 “token”，而活动实例则代表一个活动（任务、子流程...）的单个实例。因此，活动实例的概念更加倾向“面向状态（state-oriented）”。

活动实例也构成一棵树，遵循 BPMN 2.0 所提供的范围标准。处于 "同一层次的子流程"（即，同一范围的一部分，包含在同一子流程中）的活动，其活动实例会处于树的同一层次。

例如，活动实例被用来进行[流程实例的修改]({{< ref "/user-guide/process-engine/process-instance-modification.md" >}}) 和查看 [Cockpit中的活动实例树]({{< ref "/webapps/cockpit/bpmn/process-instance-view.md#activity-instance-tree" >}}).

例如:

* 并行网关后有两个并行用户任务的进程：

<div data-bpmn-diagram="guides/user-guide/process-engine/activity-instances/parallelGateway_two_userTasks"></div>

```
ProcessInstance
  receive payment
  ship order
```

* 在并行网关后有两个并行的多实例用户任务的进程：

<div data-bpmn-diagram="guides/user-guide/process-engine/activity-instances/parallelGateway_two_multiInstance_userTasks"></div>

```
ProcessInstance
  receive payment - Multi-Instance Body
    receive payment
    receive payment
  ship order - Multi-Instance Body
    ship order
```

注意：多实例活动由多实例主体和内部活动组成。多实例主体包含所有内部活动，并收集内部活动的活动实例。

* 嵌入子进程内的用户任务：

<div data-bpmn-diagram="guides/user-guide/process-engine/activity-instances/userTask_inside_embeddedSubprocess"></div>

```
ProcessInstance
  Subprocess
    receive payment
```

* 在用户任务之后，用抛出的补偿事件进行处理。

<div data-bpmn-diagram="guides/user-guide/process-engine/activity-instances/compensation_userTask"></div>

```
ProcessInstance
  cancel order
  cancel shipping
```

## 查询活动实例

目前，活动实例只能通过流程实例检索。

```java
ActivityInstance rootActivityInstance = runtimeService.getActivityInstance(processInstance.getProcessInstanceId());
```

你也可以[使用REST API检索活动实例树]({{< ref "/reference/rest/process-instance/get-activity-instances.md" >}})。


## 身份 与 唯一性

每个活动实例被分配一个唯一的ID。这个ID是确定的，如果你多次调用这个方法，相同的活动实例ID将被返回相同的活动实例。然而，可能会分配不同的执行，见下文：

## 与执行相关的内容

流程引擎中的执行概念与活动实例概念并不完全一致，因为执行树通常与BPMN中的活动/范围概念不一致。一般来说，执行和活动实例之间是n对1的关系，也就是说，在一个给定的时间点上，一个活动实例可以与多个执行相关联。此外，不能保证启动某个活动实例的执行也能结束它。流程引擎为了对执行树压缩进行了一些内部优化，这可能导致执行被重新排序和裁剪。这可能会导致这样的情况：一个给定的执行启动了一个活动实例，但另一个执行却结束了它。另一种特殊情况是：如果流程实例正在执行流程定义范围下面的非范围活动（例如用户任务），它将被root活动实例和用户任务活动实例所引用。

注意：如果你需要用 BPMN 流程模型来解释流程实例的状态，通常，使用活动实例树将比执行树更容易。


# jobs 和 job 的定义

Camunda流程引擎包括一个名为“Job执行器（Job Executor）”的组件。Job执行器是一个调度组件，负责执行异步的后台Job。用定时器事件的例子：每当流程引擎到达定时器事件时，它将停止执行，将当前状态持久化到数据库，并创建一个Job以在未来恢复执行。一个Job有一个到期时间，这个到期时间是使用BPMN XML中提供的定时器表达式计算的。

当一个流程被部署时，流程引擎为流程中的每个活动创建一个Job定义，该定义将在运行时创建Job。这允许你在流程中查询关于计时器和异步继续的信息。


## 查询jobs

使用managementService，你可以查询Job。以下是选择所有在某一日期后到期的Job。

```java
managementService.createJobQuery()
  .duedateHigherThan(someDate)
  .list()
```

也可以使用REST API查询Job。


## 查询 Job 定义

使用managementService，你也可以查询Job的定义。以下是选择特定流程中所有Job的定义：

```java
managementService.createJobDefinitionQuery()
  .processDefinitionKey("orderProcess")
  .list()
```

返回结果中将包含目标流程中所有定时器和异步继续的信息。

也可以使用REST API查询Job定义。


## 禁用和启用Job执行器

禁用Job可以防止Job被执行。暂停Job的执行可以在不同的层面上进行控制。

* Job实例级别：单个Job可以通过`managementService.suspendJob(..)` API直接暂停，或者在暂停一个进程实例或Job定义时过渡性地暂停。
* Job定义级别：某个定时器或活动的所有实例可以被暂停。

通过对Job定义的暂停Job允许你暂停某个定时器或异步继续的所有实例。直观地说，这允许你暂停进程中的某个活动，其方式是所有的进程实例都会执行到它们达到这个活动，然后就因为活动被暂停而不再继续了。

让我们假设有一个以 "orderProcess" 为关键字部署的进程，它包含一个名为 "processPayment" 的服务任务。该服务任务配置了一个异步继续，导致它被Job执行器执行。下面的例子显示了如何防止 "processPayment"服务被执行：

```java
List<JobDefinition> jobDefinitions = managementService.createJobDefinitionQuery()
        .processDefinitionKey("orderProcess")
        .activityIdIn("processPayment")
        .list();

for (JobDefinition jobDefinition : jobDefinitions) {
  managementService.suspendJobDefinitionById(jobDefinition.getId(), true);
}
```
