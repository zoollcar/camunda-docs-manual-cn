---

title: '流程中的事务'
weight: 160

menu:
  main:
    identifier: "user-guide-process-engine-transactions-in-processes"
    parent: "user-guide-process-engine"

---


流程引擎是一段被动的Java代码，在客户端的线程中工作。例如，如果你有一个允许用户启动一个新的流程实例的Web应用，并且用户点击了相应的按钮，应用服务器的http线程池的一些线程将调用API方法`runtimeService.startProcessInstanceByKey(...)`，从而 *进入* 流程引擎并启动一个新的流程实例。我们把这称为 "借用客户线程"。

在任何这样的 *外部* 触发器上（即启动一个流程、完成一个任务、发出一个执行信号），引擎运行时将在流程中前进，直到它在某个活动的执行路径上到达等待状态。等待状态是指 *稍后* 执行的任务，这意味着引擎会将当前的执行持久化到数据库中，并等待再次被触发。例如，在用户任务的情况下，任务完成时的外部触发会导致运行时执行流程的下一个位，直到再次到达等待状态（或实例结束）。与用户任务不同的是，定时器事件不是由外部触发的。相反，它是由一个 *内部* 的触发器继续的。这就是为什么引擎还需要一个活跃的组件，[job执行器]({{< ref "/user-guide/process-engine/the-job-executor.md" >}})，它能够获取注册的Job并异步处理它们。


# 等待状态

我们谈到了作为事务边界的等待状态，在这里，流程状态被存储到数据库，线程返回到客户端，事务被提交。以下的BPMN元素始终是等待状态。


<div class="row"><div class="col-xs-12 col-md-6">
<div><a href="{{< ref "/reference/bpmn20/tasks/receive-task.md" >}}"><svg height="90" version="1.1" width="110" xmlns="http://www.w3.org/2000/svg" style="overflow: hidden; position: relative; top: -0.84375px;"><defs style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></defs><rect x="5" y="5" width="100" height="80" r="5" rx="5" ry="5" fill="#ffffff" stroke="#808080" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" id="svg_1"></rect><text x="55" y="45" text-anchor="middle" font="10px &quot;Arial&quot;" stroke="none" fill="#000000" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: middle; font-style: normal; font-variant: normal; font-weight: normal; font-stretch: normal; font-size: 12px; line-height: normal; font-family: Arial, Helvetica, sans-serif;" font-size="12px" font-family="Arial, Helvetica, sans-serif"><tspan style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" dy="4">Receive Task</tspan></text><path fill="#ffffff" stroke="#808080" d="M7,10L7,20L23,20L23,10ZM7,10L15,16L23,10" stroke-width="1" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" transform="matrix(1,0,0,1,5,5)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></path></svg></a></div>

<div><a href="{{< ref "/reference/bpmn20/tasks/user-task.md" >}}"><svg height="90" version="1.1" width="110" xmlns="http://www.w3.org/2000/svg" style="overflow: hidden; position: relative; top: -0.84375px;"><defs style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></defs><rect x="5" y="5" width="100" height="80" r="5" rx="5" ry="5" fill="#ffffff" stroke="#808080" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;" id="svg_1"></rect><text x="55" y="45" text-anchor="middle" font="10px &quot;Arial&quot;" stroke="none" fill="#000000" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: middle; font-style: normal; font-variant: normal; font-weight: normal; font-stretch: normal; font-size: 12px; line-height: normal; font-family: Arial, Helvetica, sans-serif;" font-size="12px" font-family="Arial, Helvetica, sans-serif"><tspan style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" dy="4">User Task</tspan></text><path fill="#f4f6f7" stroke="#808080" d="M6.0095,22.5169H22.8676V17.0338C22.8676,17.0338,21.2345,14.2919,17.9095,13.4169H11.434500000000002C8.342600000000001,14.35,5.951400000000001,17.4419,5.951400000000001,17.4419L6.009500000000001,22.5169Z" stroke-width="0.69999999" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" transform="matrix(1,0,0,1,5,5)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></path><path fill="none" stroke="#808080" d="M9.8,19.6L9.8,22.400000000000002" stroke-width="0.69999999" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" transform="matrix(1,0,0,1,5,5)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></path><path fill="#808080" stroke="#808080" d="M19.6,19.6L19.6,22.400000000000002" stroke-width="0.69999999" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" transform="matrix(1,0,0,1,5,5)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></path><circle cx="19.5" cy="13.5" r="5" fill="#808080" stroke="#808080" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" transform="matrix(0.75,0,0,0.75,4.875,3.375)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></circle><path fill="#f0eff0" stroke="#808080" d="M11.2301,10.5581C11.2301,10.5581,13.1999,8.8599,14.9933,9.293199999999999C16.7867,9.726499999999998,18.2301,8.8081,18.2301,8.8081C18.4051,9.9897,18.2595,11.4331,17.2095,12.716899999999999C17.2095,12.716899999999999,17.967599999999997,13.2419,17.967599999999997,13.7669C17.967599999999997,14.2919,18.055099999999996,15.0794,17.267599999999998,15.8669C16.480099999999997,16.6544,13.417599999999998,16.7419,12.542599999999998,15.8669C11.667599999999998,14.9919,11.667599999999998,14.5838,11.667599999999998,14C11.667599999999998,13.4162,12.075699999999998,13.125,12.542599999999998,12.6581C11.784499999999998,12.25,10.793299999999999,10.9956,11.230099999999998,10.5581Z" stroke-width="0.69999999" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" transform="matrix(1,0,0,1,5,5)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></path></svg></a></div>
</div>

<div class="col-xs-12 col-md-6">
<div><a href="{{< ref "/reference/bpmn20/events/message-events.md" >}}"><svg height="40" version="1.1" width="40" xmlns="http://www.w3.org/2000/svg" style="overflow: hidden; position: relative; top: -0.84375px;"><defs style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></defs><circle cx="20" cy="20" r="15" fill="#ffffff" stroke="#808080" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;" id="svg_1"></circle><circle cx="20" cy="20" r="12" fill="none" stroke="#808080" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></circle><path fill="#ffffff" stroke="#808080" d="M7,10L7,20L23,20L23,10ZM7,10L15,16L23,10" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" transform="matrix(0.9375,0,0,0.9375,5.9375,5.9375)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></path><text x="20" y="50" text-anchor="middle" font="10px &quot;Arial&quot;" stroke="none" fill="#000000" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: middle; font-style: normal; font-variant: normal; font-weight: normal; font-stretch: normal; font-size: 12px; line-height: normal; font-family: Arial, Helvetica, sans-serif;" font-size="12px" font-family="Arial, Helvetica, sans-serif"><tspan style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" dy="4">Message</tspan></text></svg> Message Event</a></div>

<div><a href="{{< ref "/reference/bpmn20/events/timer-events.md" >}}"><svg height="40" version="1.1" width="40" xmlns="http://www.w3.org/2000/svg" style="overflow: hidden; position: relative; top: -0.84375px;"><defs style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></defs><circle cx="20" cy="20" r="15" fill="#ffffff" stroke="#808080" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;" id="svg_1"></circle><circle cx="20" cy="20" r="12" fill="none" stroke="#808080" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></circle><circle cx="20" cy="20" r="10" fill="#ffffff" stroke="#808080" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></circle><path fill="#ffffff" stroke="#808080" d="M15,5L15,8M20,6L18.5,9M24,10L21,11.5M25,15L22,15M24,20L21,18.5M20,24L18.5,21M15,25L15,22M10,24L11.5,21M6,20L9,18.5M5,15L8,15M6,10L9,11.5M10,6L11.5,9M17,8L15,15L19,15" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" transform="matrix(1,0,0,1,5,5)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></path><text x="20" y="50" text-anchor="middle" font="10px &quot;Arial&quot;" stroke="none" fill="#000000" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: middle; font-style: normal; font-variant: normal; font-weight: normal; font-stretch: normal; font-size: 12px; line-height: normal; font-family: Arial, Helvetica, sans-serif;" font-size="12px" font-family="Arial, Helvetica, sans-serif"><tspan style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" dy="4">Timer</tspan></text></svg> Timer Event</a></div>

<div><a href="{{< ref "/reference/bpmn20/events/signal-events.md" >}}"><svg height="40" version="1.1" width="40" xmlns="http://www.w3.org/2000/svg" style="overflow: hidden; position: relative; top: -0.84375px;"><defs style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></defs><circle cx="20" cy="20" r="15" fill="#ffffff" stroke="#808080" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;" id="svg_1"></circle><circle cx="20" cy="20" r="12" fill="none" stroke="#808080" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></circle><path fill="#ffffff" stroke="#808080" d="M7.7124971,20.247342L22.333334,20.247342L15.022915000000001,7.575951200000001L7.7124971,20.247342Z" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" stroke-opacity="1" transform="matrix(0.9375,0,0,0.9375,5.9389,5.8695)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); stroke-linecap: round; stroke-linejoin: round; stroke-opacity: 1;"></path><text x="20" y="50" text-anchor="middle" font="10px &quot;Arial&quot;" stroke="none" fill="#000000" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: middle; font-style: normal; font-variant: normal; font-weight: normal; font-stretch: normal; font-size: 12px; line-height: normal; font-family: Arial, Helvetica, sans-serif;" font-size="12px" font-family="Arial, Helvetica, sans-serif"><tspan style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" dy="4">Signal</tspan></text></svg> Signal Event</a></div>
</div>
</div>


[基于事件的网关]({{< ref "/reference/bpmn20/gateways/event-based-gateway.md" >}}):

<div data-bpmn-diagram="../bpmn/event-based-gateway"></div>

一种特殊类型的[服务任务]({{< ref "/reference/bpmn20/tasks/service-task.md">}}): [外部任务]({{< ref "/user-guide/process-engine/external-tasks.md" >}})

<a href="../external-tasks">{{< bpmn-symbol type="service-task" >}}</a>

请记住，[异步延续]({{< relref "#asynchronous-continuations" >}})也可以为其他任务添加事务边界。


# 交易边界

从一个稳定状态到另一个稳定状态的转换总是单一事务的一部分，这意味着它作为一个整体成功，或者在执行过程中发生任何类型的异常时被回滚。这在下面的例子中得到了说明：

{{< img src="../img/transactions-1.png" title="Transaction Boundaries" >}}

我们看到一个BPMN流程的片段，有一个用户任务、一个服务任务和一个定时器事件。计时器事件标志着下一个等待状态。因此，完成用户任务和验证地址是同一个工作单元的一部分，所以它应该原子式地成功或失败。这意味着，如果服务任务抛出一个异常，我们要回滚当前的事务，这样执行就会追踪到用户任务，并且用户任务仍然存在于数据库中。这也是流程引擎的默认行为。

在 **1** ，一个应用程序或客户端线程完成了任务。在同一个线程中，引擎运行时现在正在执行服务任务，并一直推进到定时器事件的等待状态（ **2** ）。然后，它将控制权返回给调用者（ **3** ），可能会提交该事务（如果它是由引擎启动的）。


# 异步延续

## 为什么要异步延续?

在某些情况下，同步行为是不需要的。有时对流程中的事务边界进行自定义控制是很有用的。
最常见的动机是对 *逻辑工作单元* 的范围的要求。考虑一下下面的流程片段：

{{< img src="../img/transactions-2.png" title="Asynchronous Continuations" >}}

我们正在完成用户任务，生成发票，然后将发票发送给客户。可以说，发票的生成不属于同一个工作单元：如果生成发票失败，我们不希望回滚用户任务的完成。
理想情况下，流程引擎将完成用户任务（**1**），提交事务并将控制权返回给调用的应用程序（**2**）。在一个后台线程（**3**）中，它将生成发票。
这正是异步延续提供的行为：它们允许我们在流程中确定事务边界的范围。


## 配置异步延续

异步延续可以在活动 *之前* 和 *之后* 被配置。此外，流程实例本身也可以被配置为异步启动。

使用 `camunda:asyncBefore` 扩展属性可以启用活动前的异步延续：

```xml
<serviceTask id="service1" name="Generate Invoice" camunda:asyncBefore="true" camunda:class="my.custom.Delegate" />
```

使用`camunda:asyncAfter`扩展属性启用一个活动后的异步延续：

```xml
<serviceTask id="service1" name="Generate Invoice" camunda:asyncAfter="true" camunda:class="my.custom.Delegate" />
```

流程实例的异步实例化是使用流程级启动事件上的`camunda:asyncBefore`扩展属性启用的。
在实例化时，流程实例将被创建并持久化在数据库中，但执行将被推迟。另外，执行监听器将不会被同步调用。这在很多情况下是有帮助的，例如，在[异构集群]({{< ref "/user-guide/process-engine/the-job-executor.md#cluster-setups" >}})中执行监听器类在实例化流程的节点上不可用。

```xml
<startEvent id="theStart" name="Invoice Received" camunda:asyncBefore="true" />
```


## 异步延续多实例活动

[多实例活动]({{< ref "/reference/bpmn20/tasks/task-markers.md#multiple-instances" >}})可以像其他活动一样被配置为异步延续。声明多实例活动的异步延续使多实例体成为异步的，也就是说，在该活动的实例被创建之 *前* 或所有实例都被结束之 *后* ，该流程会异步地继续。

此外，内部活动也可以使用 "multiInstanceLoopCharacteristics" 元素上的 "camunda:asyncBefore" 和 "camunda:asyncAfter" 扩展属性配置为异步延续。

```xml
<serviceTask id="service1" name="Generate Invoice" camunda:class="my.custom.Delegate">
	<multiInstanceLoopCharacteristics isSequential="false" camunda:asyncBefore="true">
 		<loopCardinality>5</loopCardinality>
	</multiInstanceLoopCharacteristics>
</serviceTask>
```

声明内部活动的异步延续使得多实例活动的每个实例都是异步的。在上面的例子中，并行的多实例活动的所有实例将被创建，但是它们的执行将被推迟。这对于更多地控制多实例活动的事务边界或在并行的多实例活动中启用真正的并行性是很有用的。

## 理解异步延续

为了理解异步连续的工作方式，我们首先需要了解活动是如何被执行的：

{{< img src="../img/process-engine-activity-execution.png" title="Asynchronous Continuations" >}}

上面的图示显示了一个由序列流进入和离开的常规活动是如何执行的：

1. "TAKE" 监听器在进入活动的序列流上被调用。
2. "START" 监听器被活动本身调用。
3. 活动的行为被执行：实际的行为取决于活动的类型。
   如果是 "服务任务"，其行为包括调用[授权代码]({{< ref "/user-guide/process-engine/delegation-code.md" >}})，如果是 "用户任务"，其行为包括在任务列表中创建一个 "任务" 实例等等。
4. "END" 监听器在活动中被调用。
5. "TAKE" 监听器在离开序列流上被调用。

异步延续允许在序列流的执行和活动的执行之间设置保存点。

{{< img src="../img/process-engine-async.png" title="" >}}

上面的图例显示了不同类型的异步延续会在哪里保存流程执行。

* 在活动之前的异步延续打破了调用传入序列流的TAKE监听器和执行活动的START监听器之间的执行流程。
* 活动之后的异步延续打破了活动的END监听器的调用和流出的序列流的TAKE监听器之间的执行流程。

异步延续与事务边界直接相关：把异步延续放在一个活动之前或之后，在该活动之前或之后创建一个事务边界。

{{< img src="../img/process-engine-async-transactions.png" title="" >}}

更重要的是，异步延续总是由[Job执行器]({{< ref "/user-guide/process-engine/the-job-executor.md" >}})执行。


# 异常回滚

我们想强调的是，如果出现未处理的异常，当前事务会被回滚，流程实例回滚到最后的等待状态（保存点）。下面的图片直观地表明了这一点。

{{< img src="../img/transactions-3.png" title="Rollback" >}}

如果在调用 `startProcessInstanceByKey` 时发生异常，流程实例将根本不会被保存到数据库。


# 这样设计的理由

上述异常的解决方案通常会引起讨论，因为人们希望在任务引起异常的情况下，流程引擎能够停止。另外，其他BPM平台通常将每个任务实现为等待状态。然而，我们这种方法有以下几个 **优点** ：

 * 在测试案例中，你知道方法调用后引擎的确切状态，这使得对流程状态或服务调用结果的断言变得容易。
 * 在生产代码中也是如此；如果需要的话，允许你使用同步逻辑，例如因为你想在前端呈现一个同步的用户提示。
 * 执行是普通的Java计算，在优化性能方面非常有效。
 * 如果你需要不同的行为，你可以随时切换到 'asyncBefore/asyncAfter=true' 。

然而，有一些后果你应该记住：

 * 在出现异常的情况下，状态会回滚到流程实例的最后一个持久性等待状态。这甚至可能意味着流程实例将永远不会被创建! 你不能轻易地将异常追溯到流程中导致异常的节点。你必须在客户端处理这个异常。
 * 并行的流程路径不是以Java线程的方式并行执行的，不同的路径是按顺序执行的，因为我们只有并使用一个线程。
 * 计时器在事务提交到数据库之前不能启动。计时器在后面会有更详细的解释，但它们是由我们唯一使用单独线程的活动部分：Job执行器。它们在一个自己的线程中运行，从数据库中接收到期的计时器。然而，在数据库中，计时器在当前事务可见之前是不可见的。因此，下面的计时器将永远不会启动：

{{< img src="../img/NotWorkingTimerOnServiceTimeout.png" title="Not Working Timeout" >}}


# 事务集成

流程引擎既可以自己管理事务（"独立"事务管理），也可以与平台事务管理器集成。


## 独立的事务管理器

如果流程引擎被配置为执行独立的事务管理，它总是为每个被执行的命令打开一个新事务。要配置流程引擎使用独立的事务管理，请使用`org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration`。

```java
ProcessEngineConfiguration.createStandaloneProcessEngineConfiguration()
  ...
  .buildProcessEngine();
```

独立事务管理的用例是流程引擎不必与其他事务性资源（如二级数据源或消息系统）集成的情况。

{{< note title="" class="info" >}}
在Tomcat发行版中，流程引擎是使用独立的事务管理来配置的。
{{< /note >}}


## 事务管理器集成

流程引擎可以被配置为与事务管理器（或事务管理系统）集成。开箱即用，流程引擎支持与Spring和JTA事务管理的集成。更多信息可以在下面的章节中找到。

* [Section on Spring Transaction Management][tx-spring]
* [Section on JTA Transaction Management][tx-jta]

例如，当流程引擎需要与以下方面集成时，需要集成事务管理器：

* 注重事务的编程模型，如Java EE或Spring（Java EE中的事务范围JPA实体管理器）。
* 其他事务性资源，如二级数据源、消息传递系统或其他事务性中间件，如Web服务栈。
  
{{< note title="" class="warning" >}}
  当你配置一个事务管理器时，确保它实际管理着你为流程引擎配置的数据源。如果不是这样的话，数据源就会在自动提交模式下工作。 
  这可能导致数据库中的不一致，因为不再执行事务提交和回滚。
{{< /note >}}


[tx-spring]: {{< ref "/user-guide/spring-framework-integration/transactions.md" >}}
[tx-jta]: {{< ref "/user-guide/cdi-java-ee-integration/jta-transaction-integration.md" >}}

## 事务与流程引擎上下文

当一个流程引擎命令被执行时，引擎将创建一个流程引擎上下文。Context缓存了数据库实体，因此对同一实体的多次操作不会导致多次数据库查询。这也意味着对这些实体的改变会被累积起来，并在命令返回后立即被刷新到数据库。然而，应该注意的是，当前的事务可能会在以后的时间提交。

如果一个流程引擎命令被嵌套到另一个命令中，即一个命令在另一个命令中执行，默认行为是重复使用现有的流程引擎上下文。这意味着嵌套的命令将可以访问相同的缓存实体和对它们所做的更改。

当嵌套的命令要在一个新的事务中执行时，需要为其执行创建一个新的流程引擎上下文。在这种情况下，嵌套命令将为数据库实体使用一个新的缓存，独立于先前（外部）命令缓存。这意味着，一个命令的缓存中的变化对另一个命令是不可见的，反之亦然。当嵌套命令返回时，这些变化被刷入数据库，与外部命令的流程引擎上下文无关。

`ProcessEngineContext` 工具类可以用来向流程引擎声明，需要创建一个新的流程引擎上下文，以便在一个新的事务中分离嵌套流程引擎命令中的数据库操作。下面的`Java`代码例子显示了如何使用该类：

```java
try {

  // declare new Process Engine Context
  ProcessEngineContext.requiresNew();
  
  // call engine APIs
  execution.getProcessEngineServices()
    .getRuntimeService()
    .startProcessInstanceByKey("EXAMPLE_PROCESS");

} finally {
  // clear declaration for new Process Engine Context
  ProcessEngineContext.clear();
}
```

# 乐观锁

Camunda引擎可以用于多线程的应用中。在这样的环境中，当多个线程与流程引擎并发互动时，可能会发生这些线程试图对相同的数据做改变。例如：两个线程试图在同一时间（并发地）完成同一个用户任务。这种情况是一种冲突：任务只能完成一次。

Camunda引擎使用一种众所周知的技术 "乐观锁"（或乐观并发控制）来检测和解决这种情况。

本节的结构分为两部分。第一部分介绍了乐观锁的概念。如果你已经熟悉乐观锁的概念，可以跳过这一部分。第二部分解释了乐观锁定在Camunda中的应用。

## 什么是乐观锁？

乐观锁定（也称为乐观并发控制）是一种并发控制的方法，在基于事务的系统中使用。在数据被读取的频率高于数据被改变的频率的情况下，乐观锁是最有效的。许多线程可以在同一时间读取相同的数据对象而不互相排斥。在多个线程试图同时改变同一数据对象的情况下，通过检测冲突和防止更新来确保一致性。如果检测到这样的冲突，就可以确保只有一个更新成功，其他的都失败。

### 案例

假设我们有一个数据库表，其条目如下：

  <table border="1" width="400" align="center" class="table table-condensed">
    <tr>
      <th>Id</th>
      <th>Version</th>
      <th>Name</th>
      <th>Address</th>
      <th>...</th>      
    </tr>
    <tr>
      <td>8</td>
      <td>1</td>
      <td>Steve</td>
      <td>3, Workflow Boulevard, Token Town</td>
      <td>...</td>
    </tr>
    <tr>
     <td>...</td>
     <td>...</td>
     <td>...</td>
     <td>...</td>
     <td>...</td>
    </tr>
  </table>

上表显示的是持有用户数据的单行。该用户有一个唯一的ID（主键），一个版本，一个名字和一个当前地址。

我们现在设想这样一个情况，有两个事务试图更新这个条目，一个试图改变地址，另一个试图删除用户。预期的行为是，其中一个事务成功，另一个事务被中止，并出现一个错误，表明检测到并发冲突。然后，用户可以根据数据的最新状态决定是否重试该事务。

{{< img src="../img/optimisticLockingTransactions.png" title="Transactions with Optimistic Locking" >}}

正如你在上图中看到的，`Transaction 1` 读取用户数据，对数据做一些处理：删除用户，然后提交。
`Transaction 2` 在同一时间启动，读取相同的用户数据，并对数据进行处理。当 `Transaction 2` 试图更新用户地址时，发现有冲突（因为 `Transaction 1` 已经删除了用户）。

检测到冲突是因为当 `Transaction 2` 执行更新时，用户数据的当前状态被读取。在那个时候，并发的 `Transaction 1` 已经标记了要删除的行。数据库现在等待 `Transaction 1` 的结束。在它结束后，`Transaction 2` 可以继续执行。在这个时候，该行已经不存在了，更新成功，但是报告说已经改变了`0`行。应用程序可以对此做出反应，回滚`Transaction 2`，以防止该交易的其他更改生效。

应用程序（或使用它的用户）可以进一步决定是否应该重试 `Transaction 2`。在我们的例子中，该事务将不会找到用户数据，并报告说用户已被删除。

### 乐观锁 vs 悲观锁

悲观锁与读取锁一起使用。读取锁在读取时锁定一个数据对象，防止其他并发事务也读取它。这样，冲突就不会发生了。

在上面的例子中，`Transaction 1` 一旦读取用户数据，就会锁定它。当`Transaction 2`试图读取时，`Transaction 2`会被阻止，无法继续。一旦 `Transaction 1` 完成，`Transaction 2` 就可以继续并读取最新的状态。这种方式可以防止冲突，因为事务总是只在数据的最新状态上工作。

悲观锁在写和读一样频繁且竞争激烈的情况下是有效的。

然而，由于悲观锁是排他性的，并发性降低，性能下降。因此，乐观锁，检测冲突而不是防止冲突发生，在高并发水平和读比写更频繁的情况下，是更可取的。另外，悲观锁会很快导致死锁。

### 进一步学习

* [\[1\] 维基百科: 乐观锁](https://en.wikipedia.org/wiki/Optimistic_concurrency_control)
* [\[2\] Stackoverflow: 乐观锁 vs 悲观锁](http://stackoverflow.com/questions/129329/optimistic-vs-pessimistic-locking)

## Camunda的乐观锁

Camunda使用乐观锁来控制并发性。如果检测到并发性冲突，就会抛出一个异常，并回滚该事务。当 _UPDATE_ 或 _DELETE_ 语句被执行时，会检测冲突。
删除或更新语句的执行会返回一个受影响的行数。
如果这个计数等于0，表明该行以前被更新或删除。
在这种情况下，会检测到冲突并抛出一个 "OptimisticLockingException"。

### OptimisticLockingException 异常

`OptimisticLockingException` 可以通过API请求抛出。
考虑下面对 `completeTask(...)` 方法的调用：

```java
taskService.completeTask(aTaskId); // 可能会抛出 OptimisticLockingException
```
上述方法可能会抛出一个 "OptimisticLockingException"，如果执行该方法调用导致并发修改数据。

Job的执行也可能导致抛出 "OptimisticLockingException"。因为这是预料之中的，所以执行将被重试。

#### 处理乐观锁异常

如果当前的命令是由Job执行器触发的，"OptimisticLockingException" 会自动使用重试来处理。由于这种异常预计会发生，它不会减少重试次数。

如果当前命令是由外部API调用触发的，Camunda引擎会将当前事务回滚到最后的保存点（等待状态）。现在，用户必须决定如何处理这个异常，是否应该重试事务。还要考虑到，即使事务被回滚，它也可能有非事务的副作用，而这些副作用并没有被回滚。

为了控制事务的范围，可以使用异步延续在活动前后添加显式保存点。

### 抛出乐观锁异常的共同点

有一些常见的地方可以抛出 "OptimisticLockingException"。
例如：

* 竞争的外部请求：同时完成同一任务两次。
* 一个流程内部的同步点。例子有并行网关、多实例，等等。

下面的模型显示了一个并行网关，在这个网关上可能会发生`OptimisticLockingException`。

{{< img src="../img/optimisticLockingParallel.png" title="Optimistic Locking in parallel gateway" >}}

在打开并行网关后有两个用户任务。在两个用户任务完成后，会关闭并行网关，将执行的任务合并为一个。
在大多数情况下，其中一个用户任务将首先完成。然后执行在关闭的并行网关上等待，直到第二个用户任务完成。

然而，也有可能两个用户任务都是同时完成的。假设上面的用户任务已经完成。事务假定他是关闭的并行网关上的第一个。
下面的用户任务同时完成，事务也认为他是关闭的并行网关上的第一个。
两个事务都试图更新一行，这表明他们是关闭的并行网关上的第一个。在这种情况下，会抛出 "OptimisticLockingException"。其中一个事务被回滚，另一个成功地更新了该行。

### 乐观锁与非事务副作用

在发生 "OptimisticLockingException" 后，事务被回滚。任何事务性工作都将被撤销。
非事务性工作，如创建文件或调用非事务性网络服务的效果将不会被撤销。这可能会导致不一致的状态。

这个问题有几个解决方案，最常见的是在重试时合并。

### 内部实现细节

大多数Camunda引擎的数据库表都包含一个叫做 "REV_" 的列。这一列表示修订版本。
当读取一行时，数据是在给定的 "修订版" 下读取的。修改（UPDATEs和DELETEs）总是试图更新当前命令所读取的版本。更新会增加版本。在执行一个修改语句后，会检查受影响的行数。如果计数为`1`，则推断出在执行修改时读取的版本仍然是当前版本。如果受影响的行数是`0`，那么其他事务在这个事务运行时修改了相同的数据。这意味着检测到了一个并发冲突，这个事务不能被允许提交。随后，该事务被回滚（或标记为只回滚），并抛出一个`OptimisticLockingException`。
