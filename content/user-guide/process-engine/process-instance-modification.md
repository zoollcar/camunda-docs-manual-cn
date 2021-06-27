---

title: '修改流程实例'
weight: 50

menu:
  main:
    identifier: "user-guide-process-engine-process-instance-modification"
    parent: "user-guide-process-engine"

---

虽然流程定义模型必须包含以何种顺序执行活动的顺序流，但有时也需要灵活地再次启动一个活动或取消一个正在运行的活动。例如，当流程模型存在错误（例如顺序流条件写错了）并且运行中的流程实例需要被纠正时，则以下API可以提供帮助：

* 修复流程实例，其中一些步骤必须重复执行或跳过
* 将流程实例从流程定义的一个版本迁移到另一个版本
* 测试阶段：可以跳过或重复做一些活动，以便对个别流程段进行独立测试。

为了执行这样的操作，流程引擎提供 *流程实例修改API* ，包括 `RuntimeService.createProcessInstanceModification(..)`或 `RuntimeService.createModification(...)`。这些API允许通过使用流式的构造器在一次调用中指定多个 *修改指令* 。而且它还能做到：

* 在一个活动之前就开始执行
* 在离开活动的序列流上开始执行
* 取消一个正在运行的活动实例
* 取消一个给定活动的所有运行实例
* 设置变量

{{< note title="流程实例自己修改自己" class="warning"  >}}
不建议流程实例自己修改自己 ，试图修改它自己的流程实例的行为会导致不确定的结果，这应该被避免。
{{< /note >}}

{{< enterprise >}}
  Camunda企业版提供了一个用户界面，可以在[Camunda Cockpit]({{< ref "/webapps/cockpit/bpmn/process-instance-modification.md" >}})的BPMN图上直观地对流程实例的进行修改。
{{< /enterprise >}}

# 修改流程实例——案例

作为一个例子，请考虑以下流程模型。

<div data-bpmn-diagram="../bpmn/example1"></div>

该模型显示了一个处理贷款申请的简单流程。让我们假设一份贷款申请已经到达，贷款申请已经被评估，并被确定为需要拒绝。这意味着，该流程实例有以下活动实例状态。

```
ProcessInstance
  Decline Loan Application
```

现在，执行 *Decline Loan Application* 任务的工作人员认识到了评估结果是错误的，并得出结论，该申请还是应该被接受。虽然这种灵活性并不是流程的一部分，但对流程实例的修改允许纠正运行中的流程实例。使用下面的API调用就可以做到这一点：

```java
ProcessInstance processInstance = runtimeService.createProcessInstanceQuery().singleResult();
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("acceptLoanApplication")
  .cancelAllForActivity("declineLoanApplication")
  .execute();
```

这个命令首先在活动 *Accept Loan Application* 之前开始执行，直到达到一个等待状态--在这种情况下是用户任务的创建。之后，它取消了活动 *Decline Loan Application* 的所有运行实例。在工作者的任务列表中， *Decline* 任务已被删除，而 *Accept* 任务已出现。最后活动实例的状态是：

```
ProcessInstance
  Accept Loan Application
```

让我们假设在批准申请时，必须存在一个叫做 *approver* 的变量。这也是可以实现的，如下所示：

```java
ProcessInstance processInstance = runtimeService.createProcessInstanceQuery().singleResult();
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("acceptLoanApplication")
  .setVariable("approver", "joe")
  .cancelAllForActivity("declineLoanApplication")
  .execute();
```

增加的 "setVariable" 确保在调用开始活动之前，指定的变量会提交。

现在说一些更复杂的情况。假设申请再次不合格，活动 *Decline Loan Application* 处于活动状态。现在，工作人员认识到评估过程是错误的，并希望完全重新开始。下面的修改说明表示执行这项任务的修改请求。

可以启动子流程的活动：

```java
ProcessInstance processInstance = runtimeService.createProcessInstanceQuery().singleResult();
runtimeService.createProcessInstanceModification(processInstance.getId())
  .cancelAllForActivity("declineLoanApplication")
  .startBeforeActivity("assessCreditWorthiness")
  .startBeforeActivity("registerApplication")
  .execute();
```

也可以从子流程的启动事件开始：

```java
ProcessInstance processInstance = runtimeService.createProcessInstanceQuery().singleResult();
runtimeService.createProcessInstanceModification(processInstance.getId())
  .cancelAllForActivity("declineLoanApplication")
  .startBeforeActivity("subProcessStartEvent")
  .execute();
```

或者直接启动子流程本身:

```java
ProcessInstance processInstance = runtimeService.createProcessInstanceQuery().singleResult();
runtimeService.createProcessInstanceModification(processInstance.getId())
  .cancelAllForActivity("declineLoanApplication")
  .startBeforeActivity("evaluateLoanApplication")
  .execute();
```

或者启动重新启动流程开始节点:

```java
ProcessInstance processInstance = runtimeService.createProcessInstanceQuery().singleResult();
runtimeService.createProcessInstanceModification(processInstance.getId())
  .cancelAllForActivity("declineLoanApplication")
  .startBeforeActivity("processStartEvent")
  .execute();
```


## 在 JUnit 测试中进行流程实例修改

流程实例修改在JUnit测试中非常有用。你可以跳过冗长的部分，从你想开始测试的地方跑一遍流程，直接跳到要测试的活动或网关。

为此，你可以通过修改一个启动的流程实例，并将令牌直接放在流程实例内。
例如，你想跳过子流程 *Evaluate Loan Application* ，直接用你的流程变量测试网关 *Application OK?*，你可以用以下方式启动流程实例：

```java
ProcessInstance processInstance = runtimeService.createProcessInstanceByKey("Loan_Application")
  .startBeforeActivity("application_OK")
  .setVariable("approved", true)
  .execute();
```

现在在JUnit测试中，你可以断言流程实例正在 "Accept Loan Application" 处等待。

# 操作语义

这一章节规定了流程实例修改的确切语义，应阅读这些章节以了解不同情况下的修改效果。如果没有特别指出，下面的例子就是基于如下的流程模型来说明的。

<div data-bpmn-diagram="../bpmn/example1"></div>


## 修改指令

你可以提交下面这些流式修改流程实例的方法：

* `startBeforeActivity(String activityId)`
* `startBeforeActivity(String activityId, String ancestorActivityInstanceId)`
* `startAfterActivity(String activityId)`
* `startAfterActivity(String activityId, String ancestorActivityInstanceId)`
* `startTransition(String transitionId)`
* `startTransition(String transition, String ancestorActivityInstanceId)`
* `cancelActivityInstance(String activityInstanceId)`
* `cancelTransitionInstance(String transitionInstanceId)`
* `cancelAllForActivity(String activityId)`


### 在活动前启动

```java
ProcessInstanceModificationBuilder#startBeforeActivity(String activityId)
ProcessInstanceModificationBuilder#startBeforeActivity(String activityId, String ancestorActivityInstanceId)
```

通过`startBeforeActivity`在一个活动之前启动。该指令支持`asyncBefore`标志，意味着如果活动是`asyncBefore`，将创建一个job。一般来说，该指令从指定的活动开始执行流程模型，直到遇到一个等待状态。关于等待状态的详细信息，详情请参见[Transactions in Processes]({{< ref "/user-guide/process-engine/transactions-in-processes.md" >}})。


### 在活动后启动

```java
ProcessInstanceModificationBuilder#startAfterActivity(String activityId)
ProcessInstanceModificationBuilder#startAfterActivity(String activityId, String ancestorActivityInstanceId)
```

通过 `startAfterActivity` 在一个活动之后运行，意味着将从活动的下一节点开始执行。该指令不考虑活动设置的 `asyncAfter` 标志。如果有一个以上的下一节点或根本没有下一节点，则该指令将失败。如果成功，该指令从下一节点开始执行流程模型，直到进入等待状态。


### 启动一个过渡（Transition）

```java
ProcessInstanceModificationBuilder#startTransition(String transitionId)
ProcessInstanceModificationBuilder#startTransition(String transition, String ancestorActivityInstanceId)
```

通过 `startTransition` 启动一个过渡，就意味着在一个给定的序列流上开始执行。当有多个流出的序列流时，这可以与 `startAfterActivity` 一起使用。如果成功，该指令从序列流开始执行流程模型，直到达到等待状态。


### 取消一个活动实例

```java
ProcessInstanceModificationBuilder#cancelActivityInstance(String activityInstanceId)
```

可以通过`cancelActivityInstance`取消一个特定的活动实例。既可以是一个活动实例，如用户任务的实例，也可以是层次结构中更高范围的实例，如子流程的实例。参见[关于活动实例的详细信息]({{< relref "#activity-instance-based-api" >}})如何查询流程实例的活动实例。


### 取消一个过渡实例（Transition Instance）

```java
ProcessInstanceModificationBuilder#cancelTransitionInstance(String activityInstanceId)
```

过渡实例表示即将以异步延续的形式进入/离开一个活动的执行流。一个已经创建但尚未执行的异步延续作业被表示为一个过渡实例。这些实例可以通过`cancelTransitionInstance`来取消。参见[关于活动和过渡实例的细节]({{< relref "#activity-instance-based-api" >}})学习如何查询流程实例的过渡实例。


### 取消一个活动的所有活动实例

```java
ProcessInstanceModificationBuilder#cancelAllForActivity(String activityId)
```

为了方便起见，也可以通过指令 `cancelAllForActivity` 来取消一个特定活动的所有活动和过渡实例。


## 修改变量

每一个实例化活动 (i.e., `startBeforeActivity`, `startAfterActivity`, or `startTransition`), 都可以提交流程变量。
我们提供了下面这些API：

* `setVariable(String name, Object value)`
* `setVariables(Map<String, Object> variables)`
* `setVariableLocal(String name, Object value)`
* `setVariablesLocal(Map<String, Object> variables)`

变量会在 [创建了实例化的必要范围]({{< relref "#nested-instantiation" >}}) **之后**，在指定活动执行**之前**。 这意味着，在流程引擎历史记录中，这些变量不会出现，因为它们是在执行指定活动的 "startBefore" 和 "startAfter" 指令时设置的。局部变量设置在即将执行的活动上被设置，例如，进入活动等。

查看[本指南的变量一章]({{< ref "/user-guide/process-engine/variables.md" >}})，了解有关变量和范围的详细信息。


## 基于流程实例的API

流程实例修改API是基于 *活动实例* 的。一个流程实例的活动实例树可以通过以下方法来查询。

```java
ProcessInstance processInstance = ...;
ActivityInstance activityInstance = runtimeService.getActivityInstance(processInstance.getId());
```

`ActivityInstance`是一个递归的数据结构，由上述方法调用返回的流程实例代表流程实例。`ActivityInstance`对象的ID可用于[取消特定实例]({{< relref "#cancel-an-activity-instance" >}}) 或者 [在实例化中指定父级]({{< relref "#ancestor-selection-for-instantiation" >}})。

接口 "ActivityInstance" 有 "getChildActivityInstances" 和 "getChildTransitionInstances" 方法，可以在活动实例树中向下查询。例如，假设活动 *Assess Credit Worthiness* 和 *Register Application* 是活动的。那么活动实例树看起来如下。

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
    Register Application Request
```

在代码中， *Assess* 和 *Register* 可以通过对活动实例进行如下查询获取到:

```java
ProcessInstance processInstance = ...;
ActivityInstance activityInstance = runtimeService.getActivityInstance(processInstance.getId());
ActivityInstance subProcessInstance = activityInstance.getChildActivityInstances()[0];
ActivityInstance[] leafActivityInstances = subProcessInstance.getChildActivityInstances();
// leafActivityInstances has two elements; one for each activity
```

也可以直接查询一个特定活动的所有活动实例：

```java
ProcessInstance processInstance = ...;
ActivityInstance activityInstance = runtimeService.getActivityInstance(processInstance.getId());
ActivityInstance assessCreditWorthinessInstances = activityInstance.getActivityInstances("assessCreditWorthiness")[0];
```

与活动实例相比，*过渡实例* 不代表正在执行的活动，而是代表即将进入或即将离开的活动。当异步延续的job存在但尚未被执行时就是这种情况。对于一个活动实例，可以用`getChildTransitionInstances` 方法查询子过渡实例，过渡实例的API与活动实例的API类似。


## 嵌套实例化

假设上述示例流程的一个流程实例，其中活动 *Decline Loan Application* 是活动的。现在我们提交指令，在活动 *Assess Credit Worthiness* 之前启动。当应用这个指令后，流程引擎确保实例化所有尚未活动的父作用域。在这种情况下，在启动活动之前，流程引擎实例化了 *Evaluate Loan Application* 子流程。在以前，活动实例树是

```
ProcessInstance
  Decline Loan Application
```

修改后变为：

```
ProcessInstance
  Decline Loan Application
  Evaluate Loan Application
    Assess Credit Worthiness
```

除了实例化这些父作用域外，引擎还确保在这些作用域中注册事件的订阅和job。例如，考虑如下流程：

<div data-bpmn-diagram="../bpmn/example2"></div>

启动活动 *Assess Credit Worthiness* 也为消息边界事件 *Cancelation Notice Received* 注册一个事件订阅，这样就有可能以这种方式取消子流程。


## 实例化的父级选择

默认情况下，启动一个活动会实例化所有还没有被实例化的父级作用域。当活动实例树是如下的时候：

```
ProcessInstance
  Decline Loan Application
```

然后启动 *Assess Credit Worthiness* 会变为如下情况：

```
ProcessInstance
  Decline Loan Application
  Evaluate Loan Application
    Assess Credit Worthiness
```

子流程范围也被实例化了。现在假设子流程已经被实例化了，例如在下面的树中：

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
```

再次启动 *Assess Credit Worthiness* 将在现有子流程实例的上下文中启动它，产生的树将会是这样。

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
    Assess Credit Worthiness
```

如果你想避免这种行为，而想第二次实例化子流程，可以通过使用`startBeforeActivity(String activityId, String ancestorActivityInstanceId)`方法来提供一个父级活动实例的id--类似的方法存在于starting after an activity和启动过渡。参数`ancestorActivityInstanceId`取当前活动的活动实例的id，该实例属于要启动的活动的 *父级* 活动。如果一个活动包含要启动的活动（无论是直接的，还是间接的，中间是否有其他活动），它就是一个有效的父级。

有了给定的父级活动实例 ID，在父级活动和要启动的活动之间的所有作用域将被实例化，不管它们是否已经被实例化。在这个例子中，下面的代码以流程实例（是根活动实例）作为父级来启动活动 *Assess Credit Worthiness*。

```java
ProcessInstance processInstance = ...;
ActivityInstance activityInstanceTree = runtimeService.getActivityInstance(processInstance.getId());
runtimeService.createProcessInstanceModification(activityInstanceTree.getId())
  .startBeforeActivity("assessCreditWorthiness", processInstance.getId())
  .execute();
```

然后，产生的活动实例树如下所示：

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
  Evaluate Loan Application
    Assess Credit Worthiness
```

Evaluate Loan Application 子流程启动了两次。


## 取消传递

取消一个活动实例会传递到不包含其它活动实例的父活动实例。这种行为确保流程实例不被留在没有意义的执行状态中。这意味着，当单个活动在子流程中处于活动状态并且该活动实例被取消时，该子流程也被取消了。思考一下下面的活动实例树：

```
ProcessInstance
  Decline Loan Application
  Evaluate Loan Application
    Assess Credit Worthiness
```

取消 *Assess Credit Worthiness* 的活动实例后，树是。

```
ProcessInstance
  Decline Loan Application
```

如果所有的指令都被执行了，而且没有活动的活动实例了，整个流程实例就被取消了。在上面的例子中，如果两个活动实例都被取消，即 *Assess Credit Worthiness* 的活动实例和 *Decline Loan Application* 的活动实例，就会出现这种情况。

然而，只有在所有指令都被执行后，流程实例才会被取消。这意味着，如果流程实例在两条指令之间且没有活动的活动实例，那么流程实例就不会被立即取消。例如，假设活动 *Decline Loan Applicatio* 是活动的。活动实例树是：

```
ProcessInstance
  Decline Loan Application
```

尽管在取消指令被执行后，流程实例没有直接的活动实例，但是下面的修改操作还是可以成功执行的。

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .cancelAllForActivity("declineLoanApplication")
  .startBeforeActivity("acceptLoanApplication")
  .execute();
```


## 指令执行顺序

修改指令总是按照提交的顺序执行。因此，以不同的顺序执行相同的指令会产生不同的效果。考虑一下下面的活动实例树。

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
```

假设你的任务是取消 *Assess Credit Worthiness* 实例并开始 *Register Application* 的活动。这两条指令有两种顺序。要么先执行取消再实例化，要么先实例化再取消。在前一种情况下，代码看起来如下：

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .cancelAllForActivity("assesCreditWorthiness")
  .startBeforeActivity("registerApplication")
  .execute();
```

由于[取消传播]({{< relref "#cancelation-propagation" >}})，子流程实例在执行取消指令时被取消，只是在执行实例化指令时被重新实例化。这意味着，在修改被执行后，会产生一个不同的 *Evaluate Loan Application* 子流程的实例。任何与前一个实例相关的实体都已经被移除，例如变量或事件订阅。

与此相反，考虑先执行实例化的情况：

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("registerApplication")
  .cancelAllForActivity("assesCreditWorthiness")
  .execute();
```

由于在实例化过程中的[默认父级选择]({{< relref "#ancestor-selection-for-instantiation" >}})，以及在这种情况下取消不会传播到子流程实例，子流程实例在修改前后是一样的。像变量和事件订阅这样的相关实体会被保留下来。

## 启动带有中断／取消行为的活动

如果准备启动的活动存在中断或取消行为，流程实例的修改也会触发这些行为。尤其是，启动一个中断的边界事件或中断的事件子流程将取消／中断其他活动。考虑一下下面的流程。

<div data-bpmn-diagram="../bpmn/example3"></div>

假设活动 *Assess Credit Worthiness* 目前是活动的。事件子流程可以用以下代码启动。

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("cancelEvaluation")
  .execute();
```

由于 *Cancel Evaluation* 子流程的启动事件是中断的，它将取消 *Assess Credit Worthiness* 的运行实例。当事件子流程的启动事件通过以下方式启动时，也会发生同样的情况：

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("eventSubProcessStartEvent")
  .execute();
```

然而，当位于中断事件子流程中的活动被直接启动时，中断不会被执行。思考一下下面的代码：

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("notifyAccountant")
  .execute();
```

由此产生的活动实例树将是：

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
    Cancel Evaluation
      Notify Accountant
```


## 修改多实例活动实例

修改也适用于多实例活动。我们在下文中区分了 *多实例主体* 和 *内部活动* 。内部活动是真实的活动，它具有流程模型中声明的ID。多实例主体则是围绕这个活动的范围，它在流程模型中没有被表示为一个独立的元素。例如，对于 id 为 `anActivityId` 的活动，多实例主体的 id 是 `anActivityId#multiInstanceBody`。

有了这种区别，就可以启动整个多实例主体，也可以为正在运行的多实例活动启动单个内部活动实例。考虑一下下面的流程模型。

<div data-bpmn-diagram="../bpmn/example4"></div>

我们假设 `Contact Customer`活动是正在运行的的，有三个实例。

```
ProcessInstance
  Contact Customer - Multi-Instance Body
    Contact Customer
    Contact Customer
    Contact Customer
```

下面的修改在同一个多实例主体活动中启动了 *Contact Customer* 活动的第四个实例：

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("contactCustomer")
  .execute();
```

流程实例树现在变成了这样：

```
ProcessInstance
  Contact Customer - Multi-Instance Body
    Contact Customer
    Contact Customer
    Contact Customer
    Contact Customer
```

流程引擎可以确保正确更新与多实例相关的变量 `nrOfInstances` 、`nrOfActiveInstances` 和 `loopCounter`。如果多实例活动是基于集合配置的，那么当指令被执行时，集合不会被考虑，集合元素变量将不会为额外的实例填充。这种行为可以通过使用方法`#setVariableLocal`为实例化指令提供集合元素变量来实现：

现在考虑以下修改：

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("contactCustomer#multiInstanceBody")
  .execute();
```

这将第二次启动整个多实例主体，变成如下活动实例树：

```
ProcessInstance
  Contact Customer - Multi-Instance Body
    Contact Customer
    Contact Customer
    Contact Customer
    Contact Customer
  Contact Customer - Multi-Instance Body
    Contact Customer
    Contact Customer
    Contact Customer
```

## 异步修改流程实例

你可以异步地执行单个流程实例的[修改]({{< ref "/user-guide/process-engine/process-instance-modification.md#modification-instruction-types" >}}) 指令与同步修改相同，流式语法如下：

```java
Batch modificationBatch = runtimeService.createProcessInstanceModification(processInstanceId)
        .cancelActivityInstance("exampleActivityId:1")
        .startBeforeActivity("exampleActivityId:2")
        .executeAsync();
```
这将创建一个修改的[batch]({{< ref "/user-guide/process-engine/batch.md" >}})，然后以异步方式执行。
在执行单个流程实例的异步修改时，不支持修改变量。

## 修改多个流程实例

当有多个满足特定条件的流程实例时，可以使用`RuntimeService.createModification(...)`来一次性修改它们。这个方法允许指定修改命令和应该被修改的流程实例的ID。要求这些流程实例属于给定的流程定义。

流式修改生成器提供了以下要提交的代码：

* `startBeforeActivity(String activityId)`
* `startAfterActivity(String activityId)`
* `startTransition(String transitionId)`
* `cancelAllForActivity(String activityId)`

可以通过提供一组流程实例ID或流程实例查询的方式，批量选择流程实例进行修改。
也可以同时指定一个流程实例ID列表和一个查询。然后，要修改的流程实例将是所产生的集合的并集。

```java
ProcessInstanceQuery processInstanceQuery = runtimeService.createProcessInstanceQuery();

runtimeService.createModification("exampleProcessDefinitionId")
  .cancelAllForActivity("exampleActivityId:1")
  .startBeforeActivity("exampleActivityId:2")
  .processInstanceIds(processInstanceQuery)
  .processInstanceIds("processInstanceId:1", "processInstanceId:2")
  .execute();
```

对多个流程实例的修改可以同步或异步执行。
关于同步执行和异步执行的更多信息，请参考[用户指南]({{< ref "/user-guide/process-engine/process-instance-migration.md#executing-a-migration-plan" >}})的相关章节。

同步执行的一个例子：

```java
runtimeService.createModification("exampleProcessDefinitionId")
  .cancelAllForActivity("exampleActivityId:1")
  .startBeforeActivity("exampleActivityId:2")
  .processInstanceIds("processInstanceId:1", "processInstanceId:2")
  .execute();
```

异步执行的一个例子：

```java
Batch batch = runtimeService.createModification("exampleProcessDefinitionId")
  .cancelAllForActivity("exampleActivityId:1")
  .startBeforeActivity("exampleActivityId:2")
  .processInstanceIds("processInstanceId:1", "processInstanceId:2", "processInstanceId:100")
  .executeAsync();
```

## 跳过监听器、输入输出映射

可以跳过执行和job监听器的调用，以及执行修改的事务的输入/输出映射。当修改是在一个无法访问所涉及的流程应用部署及其包含的类的系统上时，这可能是有用的。可以通过使用修改构建器的方法 `execute(boolean skipCustomListeners, boolean skipIoMappings)` 跳过监听器和输入输出映射的调用。

## 备注

可以使用 `annotation` 选项来传递一个任意的文本备注，用于审计。

```java
runtimeService.createProcessInstanceModification(processInstanceId)
  .cancelAllForActivity("declineLoanApplication")
  .startBeforeActivity("processStartEvent")
  .annotation("Modified to resolve an error.")
  .execute();
```
它将在执行修改操作的[用户操作日志]({{< ref "/user-guide/process-engine/history.md#annotation-of-user-operation-logs" >}})中记录。

## 健全性检查

流程实例修改是一个非常强大的工具，允许随意启动和取消活动。因此，很容易产生正常流程执行所不能达到的情况。假设有以下流程模型：

<div data-bpmn-diagram="../bpmn/example5"></div>

假设活动 *Decline Loan Application* 是活动的。通过修改，活动 *Assess Credit Worthiness* 可以被启动。在该活动完成后，执行被卡在加入的并行网关上，因为没有令牌会从另一个输入流中流入使流程继续下去。这是流程实例不能继续执行的最明显的情况之一，当然还有许多其他情况，取决于具体的流程模型。

流程引擎是 **不能** 检测到修改是否会产生这样的结果。所以这取决于这个 API 的用户，他们要确保所做的修改不会使流程实例处于不希望的状态。然而如果出现这个情况，也可以使用流程实例的修改API也是修复:-)
