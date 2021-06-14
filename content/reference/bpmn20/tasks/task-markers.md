---

title: 'Task Markers 任务标记'
weight: 80

menu:
  main:
    identifier: "bpmn-ref-task-markers"
    parent: "bpmn-ref-tasks"
    pre: "Markers control operational aspects like repetition."

---

除了各种类型的任务外，我们还可以将任务标记为循环（loops）、多实例（Multiple Instance）或补偿（compensations）。标记可以与任务类型相结合使用。


# 多实例（Multiple Instance）

多实例活动是为业务流程中的某个步骤定义重复的一种方式。在编程概念中，多实例与 `for each ` 结构相匹配：它允许对给定集合中的每个项目按顺序或并行地执行某个步骤或甚至一个完整的子流程。

多实例是一个有额外属性（所谓的 "多实例特性"）的常规活动，它将导致该活动在运行时被多次执行。以下活动可以成为多实例活动。

* Service Task 服务任务
* Send Task 发送任务
* User Task 用户任务
* Business Rule Task 业务规则任务
* Script Task 脚本任务
* Receive Task 接收任务
* Manual Task 手动任务
* (Embedded) Sub-Process （嵌入）子流程
* Call Activity 发起活动
* Transaction Subprocess 事务子流程

网关或事件不能成为多实例。

如果一个活动是多实例的，这将由活动底部的三条短线表示。三条垂直线表示实例将以<strong>并行</strong>方式执行，而三条水平线表示**顺序**执行。

<div data-bpmn-diagram="../bpmn//multiple-instance"></div>

按照规范的要求，每个实例所创建的执行的每个父执行将有以下变量：

* **nrOfInstances**: 实例的总数量
* **nrOfActiveInstances**: 当前活动的，即尚未完成的实例的数量。对于一个连续的多实例，这将永远是1。
* **nrOfCompletedInstances**: 已经完成的实例的数量。

这些值可以通过调用 "execution.getVariable(x) "方法检索。

此外，每个创建的执行将有一个执行本地变量（即对其他执行不可见，也不存储在进程实例级别）。

* **loopCounter**: 表示该特定实例的`for each`循环中的索引

为了使一个活动成为多实例，活动xml元素必须有一个`multiInstanceLoopCharacteristics`子元素。

```xml
<multiInstanceLoopCharacteristics isSequential="false|true">
 ...
</multiInstanceLoopCharacteristics>
```

isSequential属性表示该活动的实例是按顺序执行还是并行执行。

实例的数量在进入活动时被计算一次。有几种方法可以配置这个。一种方法是通过使用`loopCardinality`子元素直接指定一个数字。

```xml
<multiInstanceLoopCharacteristics isSequential="false|true">
  <loopCardinality>5</loopCardinality>
</multiInstanceLoopCharacteristics>
```

值为正数的表达式也是可以的。

```xml
<multiInstanceLoopCharacteristics isSequential="false|true">
  <loopCardinality>${nrOfOrders-nrOfCancellations}</loopCardinality>
</multiInstanceLoopCharacteristics>
```

另一种定义实例数量的方法是指定一个过程变量的名称，该变量是一个使用`loopDataInputRef`子元素的集合。对于集合中的每个项目，将创建一个实例。可以选择使用inputDataItem子元素为实例设置集合中的那个特定项目。这在下面的XML例子中显示。

```xml
<userTask id="miTasks" name="My Task ${loopCounter}" camunda:assignee="${assignee}">
  <multiInstanceLoopCharacteristics isSequential="false">
    <loopDataInputRef>assigneeList</loopDataInputRef>
    <inputDataItem name="assignee" />
  </multiInstanceLoopCharacteristics>
</userTask>
```

假设变量assigneeList包含值[kermit, gonzo, foziee]。在上面的代码片段中，三个用户任务将被并行创建。每个执行都将有一个名为assignee的进程变量，包含集合的一个值，在这个例子中，它被用来分配用户任务。

`loopDataInputRef`和`inputDataItem`的缺点是：1）名字相当难记；2）由于BPMN 2.0模式的限制，它们不能包含表达式。我们通过在multiInstanceCharacteristics上提供collection和 elementVariable属性来解决这个问题。

```xml
<userTask id="miTasks" name="My Task" camunda:assignee="${assignee}">
  <multiInstanceLoopCharacteristics isSequential="true"
     camunda:collection="${myService.resolveUsersForTask()}" camunda:elementVariable="assignee" >
  </multiInstanceLoopCharacteristics>
</userTask>
```

多实例活动在所有实例都结束时结束。然而，我们可以指定一个表达式，它在每次一个实例结束时被计算。当这个表达式计算为真时，所有剩余的实例被销毁，多实例活动结束，继续这个过程。这样的表达式必须被定义在 completionCondition 子元素中。

```xml
<userTask id="miTasks" name="My Task" camunda:assignee="${assignee}">
  <multiInstanceLoopCharacteristics isSequential="false"
     camunda:collection="assigneeList" camunda:elementVariable="assignee" >
    <completionCondition>${nrOfCompletedInstances/nrOfInstances >= 0.6 }</completionCondition>
  </multiInstanceLoopCharacteristics>
</userTask>
```

在这个例子中，将为assigneeList集合的每个元素创建并行实例。然而，当60%的任务完成后，其他的任务被删除，这个过程继续进行


## Camunda 扩展

<table class="table table-striped">
  <tr>
    <th>属性</th>
    <td>
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#collection" >}}">camunda:collection</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#elementvariable" >}}">camunda:elementVariable</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncbefore" >}}">camunda:asyncBefore</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#asyncafter" >}}">camunda:asyncAfter</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#exclusive" >}}">camunda:exclusive</a>,
      <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#jobpriority" >}}">camunda:jobPriority</a>
    </td>
  </tr>
  <tr>
    <th>扩展元素</th>
    <td>&ndash;</td>
  </tr>
  <tr>
    <th>约束</th>
    <td>
      The <code>camunda:exclusive</code> attribute is only evaluated if the attribute
      <code>camunda:asyncBefore</code> or <code>camunda:asyncAfter</code> is set to <code>true</code>
    </td>
  </tr>
</table>


## 边界事件和多实例

由于多实例是一个常规的活动，所以有可能在其边界上定义一个边界事件。如果有一个中断的边界事件，当事件被捕获时，所有仍在活动的实例将被销毁。以下面这个多实进程为例：

<div data-bpmn-diagram="../bpmn/multiple-instance-boundary"></div>

在这里，子进程的所有实例将在定时器启动时被销毁，不管有多少实例，也不管哪些内部活动目前还没有完成。


## 循环（loops）

循环标记还没有被引擎原生支持。对于多实例来说，重复的次数是事先知道的--这使得它无论如何都不是循环的候选者，因为它定义了一个完成条件，在某些情况下这可能已经足够了。

为了绕过这个限制，解决方案是在你的BPMN流程中明确地模拟循环。

<div data-bpmn-diagram="../bpmn/loop-alternative"></div>

请放心，我们的代办事项中已经有了循环标记，以后将被添加到引擎中。

## JSON集合具有多实例集合

用[Camunda SPIN]({{< ref "/reference/spin/_index.md">}})创建的JSON数组可以作为多实例活动的集合。
参见下面这个初始化执行变量`collection`的JavaScript例子：

```javascript
var collection = S('{ "collection" : ["System 1", "System 3"] }');
execution.setVariable("collection", collection);
```

这个脚本可以通过几种方式注入模型中，例如使用[脚本任务]({{< ref "/reference/bpmn20/tasks/script-task.md">}}). 

我们现在可以在多实例活动的`camunda:collection`扩展元素中使用`collection`变量:

```xml
<multiInstanceLoopCharacteristics 
  camunda:collection="${collection.prop('collection').elements()}" 
  camunda:elementVariable="collectionElem" />
```

这使用SPIN的JSON`.prop()`和`.elements()`来返回JSON数组。 将多实例活动的`elementVariable`设置为一个变量名，该变量将包含数组项目。
将包含数组项目。为了访问元素的值，你可以在你的元素变量中使用`.value()`。

# 补偿（compensations）

如果一个活动被用来补偿另一个活动的影响，它可以被声明为补偿处理程序。补偿处理程序不包含在常规流程中，并且只在抛出补偿事件时被执行。

<div data-bpmn-diagram="../bpmn/compensation-marker"></div>

注意在 "cancel hotel reservation" 服务任务的底部中心区域的补偿处理程序图标。

补偿处理程序可能没有传入或传出的序列流。

补偿处理程序必须使用定向关联与补偿边界事件相关联。

为了声明一个活动是一个补偿处理程序，我们需要将属性isForCompensation设置为真：

```xml
<serviceTask id="undoBookHotel" isForCompensation="true" camunda:class="..." />
```


# 额外资料

* [Tasks](http://camunda.org/bpmn/reference.html#activities-task) in the [BPMN 2.0 Modeling Reference](http://camunda.org/bpmn/reference.html)
* [Transaction Subprocess]({{< ref "/reference/bpmn20/subprocesses/transaction-subprocess.md" >}})
