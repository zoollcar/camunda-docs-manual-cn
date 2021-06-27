---

title: '流程实例迁移'
weight: 115

menu:
  main:
    identifier: "user-guide-process-engine-process-instance-migration"
    parent: "user-guide-process-engine"

---

当部署流程定义的新版本时，在以前版本上运行的现有流程实例不受影响。这意味着，新的流程定义不会影响它们。如果流程实例应该在新的流程定义上继续执行，可以使用 *流程实例迁移* API。

迁移分为两部分：

1. 创建一个 *迁移计划* ，描述如何将流程实例从一个流程定义迁移到另一个流程定义。
2. 将迁移计划应用于流程实例

迁移计划由一组 *迁移指令* 组成的，这些指令实质上是两个流程定义活动之间的映射。具体地说，就是把 *源流程定义* 的活动（即流程实例被迁移出来的流程）映射到 *目标流程定义* 的活动（即流程实例被迁移到的流程）。迁移指令确保源活动的一个实例被迁移到目标活动的一个实例中。当所有的迁移来源都有对应的迁移指令时，迁移计划定义就完成了。

迁移指令的目的是为了映射语义上相等的活动。因此，迁移需要尽可能少地干扰活动实例的状态，以确保无缝过渡。
例如，被迁移的用户任务实例不会被重新分配。从受让人的角度来看，迁移基本上是透明的，所以迁移前开始的任务在迁移后可以顺利完成。
同样的原则也适用于其他BPMN元素类型。

对于活动在语义上不等同的情况。
我们建议将迁移与 [流程实例修改 API]({{< ref "/user-guide/process-engine/process-instance-modification.md" >}}) 结合起来。例如：在迁移前取消一个活动实例，在迁移后启动一个新的实例。

在本节的其余部分，除非另有说明，否则将使用以下流程模型来说明API和迁移的影响：

`exampleProcess:1`:

<div data-bpmn-diagram="../bpmn/process-instance-migration/example1"></div>

`exampleProcess:2`:

<div data-bpmn-diagram="../bpmn/process-instance-migration/example2"></div>

{{< enterprise >}}
  Camunda企业版的[Camunda Cockpit]({{< ref "/webapps/cockpit/bpmn/process-instance-migration.md" >}})提供了一个用于迁移流程实例的用户界面。
{{< /enterprise >}}

# 流程实例迁移案例

我们可以使用API接口 `RuntimeService#createMigrationPlan` 来定义一个迁移计划。
它返回一个流式构建器来创建一个迁移计划。对于上面的例子来说，代码如下：

```java
MigrationPlan migrationPlan = processEngine.getRuntimeService()
  .createMigrationPlan("exampleProcess:1", "exampleProcess:2")
  .mapActivities("assessCreditWorthiness", "assessCreditWorthiness")
  .mapActivities("validateAddress", "validatePostalAddress")
  .mapActivities("archiveApplication", "archiveApplication")
  .build();
```

`mapActivities` 调用各自指定了一个迁移指令，并表示第一个活动的实例应该成为第二个活动的实例。

让我们假设我们有一个流程实例处于以下活动实例状态：

```
ProcessInstance
├── Archive Application
└── Assess Credit Worthiness
    └── Validate Address
```

为了根据定义的迁移计划迁移这个流程实例，可以使用API方法`RuntimeService#newMigration`。

```java
MigrationPlan migrationPlan = ...;
List<String> processInstanceIds = ...;

runtimeService.newMigration(migrationPlan)
  .processInstanceIds(processInstanceIds)
  .execute();
```

运行后的活动实例状态是：

```
ProcessInstance
├── Handle Application Receipt
│   └── Archive Application
└── Assess Credit Worthiness
    └── Validate Postal Address
```

引擎做了以下事情：

* 创建了一个嵌入式子流程 *Handle Application Receipt* 的实例，以反映 "exampleProcess:2 "中的新子流程。
* *Archive Application* 、 *Assess Credit Worthiness* 和 *Validate Postal Address* 的活动实例被迁移。

第二点的具体含义是什么？因为这些活动实例有一个迁移指令，所以它们被迁移了。组成这个实例的实体被更新以引用新的活动和流程定义。除此之外，活动实例、任务实例和变量实例被保留下来。

在迁移之前，审核人员的任务列表中有一个任务实例，执行 *Validate Address* 活动。迁移后，同样的任务实例仍然存在，并且可以成功完成。它仍然有相同的属性，如受让人或名称。

从审核人员的角度来看，在执行任务时，迁移是完全透明的。


# API

下面给出了流程实例迁移的Java API的方法。请注意，这些操作也可以通过[REST]({{< ref "/reference/rest/migration/_index.md" >}})方式进行。

## 创建迁移计划

迁移计划可以通过API `RuntimeService#createMigrationPlan` 来创建。它定义了应该如何进行流程实例迁移。
迁移计划包含源和目标流程定义的ID以及迁移指令的列表。迁移指令是源流程定义中的活动与目标流程定义中的活动之间的映射。

例如，下面的代码创建了一个有效的迁移计划：

```Java
MigrationPlan migrationPlan = processEngine.getRuntimeService()
  .createMigrationPlan("exampleProcess:1", "exampleProcess:2")
  .mapActivities("assessCreditWorthiness", "assessCreditWorthiness")
  .mapActivities("validateAddress", "validatePostalAddress")
  .build();
```

所有的活动都可以被映射。迁移指令必须在相同类型的活动之间进行映射。

支持如下活动映射关系：

* 一对一关系

迁移计划在创建后要进行 *验证* ，以发现流程引擎不支持的迁移指令。详情请参见[创建时验证章节]({{< relref "#创建时验证章节" >}})。

此外，迁移计划在执行前也会被 *验证*，以确保它能被应用于特定的流程实例。例如，某些类型的迁移指令只支持过渡实例（即活动的异步延续），但不支持活动实例。详见[执行时验证章节]({{< relref "#执行时验证章节" >}})。

{{< note title="验证限制" class="warning" >}}
流程引擎只能验证流程模型是否可以被迁移。
但是用户还需要关心其他方面的问题。你可以在关于[验证未涵盖的方面](#验证未涵盖的方面)一节中阅读更多的内容。
{{< /note >}}

### 一对一映射操作说明

```java
MigrationPlanBuilder#mapActivities(String sourceActivityId, String targetActivityId)
```

一对一关系指令意味着源活动的一个实例会被迁移到目标活动的一个实例中。在执行迁移时，活动实例、任务实例和变量实例的状态将被保留下来。


### 更新活动触发器

迁移活动时，可以决定是否要更新相应的活动触发器。详见[BPMN的活动]({{< relref "#events" >}})。生成迁移计划时，可以通过使用 "updateEventTrigger" 方法，为包含 "timeout" 的任务监听器的[用户任务]({{< relref "#user-task" >}})和活动之间生成的指令定义这一设置。例如，下面的代码为一个边界活动生成了一个迁移指令，并在迁移过程中更新其活动触发器。

{{< note title="条件活动" class="info" >}}
对于有条件的活动，`#updateEventTrigger` 是强制性的。
{{< /note >}}

```java
MigrationPlan migrationPlan = processEngine.getRuntimeService()
  .createMigrationPlan("exampleProcess:1", "exampleProcess:2")
  .mapActivities("userTask", "userTask")
  .mapActivities("boundary", "boundary")
    .updateEventTrigger()
  .build();
```

### 为流程实例设置变量

有时，在将流程实例迁移到流程定义的新版本之后，有必要添加变量。例如，当新的流程模型有一个新的输入映射，需要一个特定的变量，而这个变量在迁移后的流程实例中还没有出现。这时就需要在迁移中设置变量。

请看下面如何调用Java API的案例：

```java
Map<String, Object> variables = Variables.putValue("my-variable", "my-value");
MigrationPlan migrationPlan = processEngine.getRuntimeService()
  .mapEqualActivities()
  .setVariables(variables)
  .build();
```

#### 已知的限制

目前，不可能异步设置瞬时变量。然而。
你可以同步地[设置瞬时变量][set transient variables]。

[set transient variables]: {{< ref "/user-guide/process-engine/variables.md#瞬时变量" >}}

## 生成迁移计划

除了手动指定所有迁移指令外，`MigrationPlanBuilder` 能够为源流程和目标流程定义中所有 *相等* 的活动生成迁移指令。这可以减少创建迁移的工作量，只需要创建那些不相等的活动即可。

活动相等需要满足如下条件：

* 它们属于相同的活动类型
* 它们有相同的 ID
* 它们属于同一个作用域，也就是说，根据这个定义，它们的父BPMN作用域是相等的。
  流程的定义总是相等的。

例如，考虑下面的代码片段：

```Java
MigrationPlan migrationPlan = processEngine.getRuntimeService()
  .createMigrationPlan("exampleProcess:1", "exampleProcess:2")
  .mapEqualActivities()
  .mapActivities("validateAddress", "validateProcessAddress")
  .build();
```

它为相同的活动 `assessCreditWorthiness` 自动生成了迁移指令。然后手动增加了一个从 `validateAddress` 到 `validateProcessAddress` 的额外映射。

### 更新活动触发器

和单个指令一样，可以通过使用 `updateEventTriggers` 方法为生成的迁移指令指定活动触发器更新标志。
这相当于对所有生成的活动迁移指令调用 `updateEventTrigger`。

```Java
MigrationPlan migrationPlan = processEngine.getRuntimeService()
  .createMigrationPlan("exampleProcess:1", "exampleProcess:2")
  .mapEqualActivities()
  .updateEventTriggers()
  .build();
```

## 执行迁移计划

迁移计划可以通过使用API方法 `RuntimeService#newMigration` 应用于一组流程实例。

迁移既可以同步执行（阻塞的），也可以使用[batch][batch]异步执行（非阻塞）。

下面是两者的对比：

- 以下情况应该使用同步迁移：
  - 流程实例的数量很少
  - 迁移应该是原子性的，也就是说，应该立即执行，如果至少有一个流程实例不能被迁移，就要失败。


- 以下情况应该使用异步迁移：
  - 流程实例的数量很大
  - 所有流程实例的迁移应该与其他实例解耦，也就是说，每个实例都在自己的事务中进行迁移
  - 迁移应该由另一个线程来执行，也就是说，应该由Job执行器执行。


### 选择要迁移的流程实例

可以通过提供一组流程实例ID或提供一个流程实例查询来选择流程实例进行迁移。也可以同时指定一个流程实例ID列表和一个查询。
被迁移的流程实例将是所产生的集合的并集。

#### 流程实例列表

迁移计划可以指定要迁移的流程实例ID列表：

```Java
MigrationPlan migrationPlan = ...;

List<String> processInstanceIds = ...;

runtimeService.newMigration(migrationPlan)
  .processInstanceIds(processInstanceIds)
  .execute();
```

对于静态数量的流程实例，有一个方便的可变参数方法：

```Java
MigrationPlan migrationPlan = ...;

ProcessInstance instance1 = ...;
ProcessInstance instance2 = ...;

runtimeService.newMigration(migrationPlan)
  .processInstanceIds(instance1.getId(), instance2.getId())
  .execute();
```

#### 流程实例查询

如果事先不知道要修改的实例，可以通过流程实例查询来选择流程实例：

```Java
MigrationPlan migrationPlan = ...;

ProcessInstanceQuery processInstanceQuery = runtimeService
  .createProcessInstanceQuery()
  .processDefinitionId(migrationPlan.getSourceProcessDefinitionId());

runtimeService.newMigration(migrationPlan)
  .processInstanceQuery(processInstanceQuery)
  .execute();
```


### 跳过监听器和输入输出映射

在迁移过程中，活动实例可能会结束或出现新的活动实例。
默认情况下，活动执行监听器和输入/输出映射会被调用。有时我们不希望它们被调用。

例如，如果一个执行监听器需要一个变量存在以正常工作，但该变量在源流程定义的实例中不存在，那么跳过监听器的调用可能是有用的。

在API中，方法 `#skipCustomListeners` 和 `#skipIoMappings` 可以分别跳过它们。

```Java
MigrationPlan migrationPlan = ...;
List<String> processInstanceIds = ...;

runtimeService.newMigration(migrationPlan)
  .processInstanceIds(processInstanceIds)
  .skipCustomListeners()
  .skipIoMappings()
  .execute();
```

### 同步执行迁移

同步执行迁移，需要使用`execute`方法。它将阻塞，直到迁移完成：

```Java
MigrationPlan migrationPlan = ...;
List<String> processInstanceIds = ...;

runtimeService.newMigration(migrationPlan)
  .processInstanceIds(processInstanceIds)
  .execute();
```

如果所有流程实例都能被迁移，则迁移成功。参考[验证章节]({{< relref "#验证" >}})，了解在执行迁移计划之前要进行哪些验证。


### 批量异步执行迁移

要异步执行迁移，需要使用 `executeAsync` 方法。它将立即返回一个对执行迁移批处理的引用。

```Java
MigrationPlan migrationPlan = ...;
List<String> processInstanceIds = ...;

Batch batch = runtimeService.newMigration(migrationPlan)
  .processInstanceIds(processInstanceIds)
  .executeAsync();
```

使用批处理，流程实例迁移被分割成几个Job，这些Job被异步执行。这些批处理Job是由Job执行器执行的。更多信息请参见[批处理][batch]部分。如果所有的批处理执行Job都成功完成，一个批处理就完成了。然而，与同步迁移相比，不能保证所有或没有流程实例被迁移。由于迁移被分割成几个独立的Job，每一个Job都可能失败或成功。

如果一个迁移Job失败了，Job执行者会重试，如果没有重试，就会产生一个事件。在这种情况下，需要手动操作来完成批量迁移。Job的重试次数可以增加，也可以删除该Job。删除会取消特定实例的迁移，但不会影响到此后的批次。

#### 异构集群中的批量迁移

正如用户指南的[job执行器][job executor]部分所描述的，流程引擎可用于异构集群，其中程序不均匀地部署在集群节点上。
*deployment-aware* 的Job执行器只执行在它那里注册的部署的Job。
在一个异构集群中，这避免了访问部署资源的问题。

因此，在执行迁移批处理时，批处理执行Job被限制在对源流程定义的部署有注册的Job执行器上。这就引入了一个要求，即源和目标部署要在同一个Job执行器上注册，否则在目标流程的上下文中执行自定义代码（例如执行监听器）时，迁移可能会失败。请注意，在迁移过程中也可以[跳过自定义代码的执行](#skipping-listeners and-input-output-mappings)。

# 针对BPMN的API和效果

根据一个流程模型所包含的活动类型，迁移具有不同的效果。

## 任务（Tasks）

### 用户任务

当一个用户任务被迁移时，除了流程定义id和任务定义key外，任务实例（即`org.camunda.bpm.engine.task.Task`）的所有属性都会保留下来。该任务不会被重新初始化。像assignee或name这样的属性不会改变。

#### 超时任务监听器

带有事件类型 "timeout" 附属任务监听器的用户任务，定义了一个持久的事件触发器，可以在迁移过程中更新或保留。
对于相关的定时器，[捕捉事件]({{< relref "#events" >}})的相关内容也适用于此。在用户任务的迁移中，适用以下语义：

* 如果根据超时任务监听器的 `id` 在源流程和目标流程定义中被找到，那么它的持久性事件触发器（即定时器）将被迁移。
* 如果根据源流程定义中的超时任务监听器的 `id` 在目标定义中找不到，那么它的事件触发器在迁移中会被删除。
* 如果目标定义中的一个超时任务监听器不是迁移指令的目标，那么在迁移过程中会初始化一个新的事件触发器。


### 接收任务

接收任务定义了一个持久的事件触发器，可以在迁移过程中更新或保留。
[捕捉事件]({{< relref "#events" >}})的相关内容也适用于此。

### 外部任务

当一个活动的[外部任务]({{< relref "external-tasks.md" >}})被迁移时，外部任务实例（即`org.camunda.bpm.engine.externaltask.ExternalTask`）的所有属性除了活动id、流程定义key和流程定义id外都被保留下来。特别是，像主题和锁定状态这样的属性不会改变。

可以把作为外部任务实现的活动相互映射，即使它们有不同的类型。例如，一个外部发送任务可以被映射到一个外部服务任务。

## 网关（Gateways）

### 包容 & 并行网关

包容网关和平行网关实例意味着网关需要等待之前的令牌才能激活。
它们可以通过提供迁移指令被迁移到目标流程中相同类型的网关。

此外，还需要满足以下条件：

* 目标网关的传入序列流数量需要等于或大于源网关相同数量的传入序列流。
* 网关所包含的作用域必须有一个有效的迁移指令
* 目标流程的每个网关最多只能映射一个源流程的网关。

### 基于事件的网关

要迁移一个基于事件的网关实例，需要在迁移计划中添加一个映射到目标基于事件的网关的迁移指令。
为了迁移网关的事件触发器（事件订阅、Jobs），这些事件触发器也能跟随网关迁移。
关于各种事件的指令的语法，请参见[事件章节]({{< relref "#事件" >}})。


## 事件

对于各种可以捕获的事件 (启动事件、中间事件、边界事件)，迁移指令支持持久化的事件触发器。 消息、条件、定时器和信号事件都是如此。

映射事件时，有两个可配置项：

1. **事件的触发器保持不变**。即使目标事件定义了不同的触发器（例如，改变了定时器的配置），被迁移的事件实例也是按照源定义来触发的。这是调用 `migrationBuilder.mapActivities("sourceTask", "targetTask")` 时的默认行为。
2. **事件触发器被更新**。被迁移的事件实例会根据目标定义被触发。
这种行为可以通过调用 `migrationBuilder.mapActivities("sourceTask", "targetTask").updateEventTrigger()` 来指定。

{{< note title="计时器事件" class="info" >}}
  使用 `#updateEventTrigger` 迁移定时器事件并不会考虑到在迁移之前已经过了一定的时间。
  因此，事件触发器根据目标事件被重置。

  考虑以下两个过程，其中边界事件的配置发生变化。

  `timerBoundary:1`:

  <div data-bpmn-diagram="../bpmn/process-instance-migration/example-boundary-timer1"></div>

  `timerBoundary:2`:

  <div data-bpmn-diagram="../bpmn/process-instance-migration/example-boundary-timer2"></div>

  指定指令 `migrationBuilder.mapActivities("timer", "timer").updateEventTrigger()` 会重新初始化定时器工作。
  实际上，边界事件在迁移后的十天就会触发。相反，如果不使用`updateEventTrigger`，那么定时器工作的配置将被保留。实际上，无论何时进行迁移，它都会在活动开始的五天后触发。
{{< /note >}}

{{< note title="条件事件" class="info" >}}
迁移条件事件时必须使用 `#updateEventTrigger` 。迁移后条件事件被新条件事件的条件所覆盖。
{{< /note >}}


### 边界事件

边界事件可以与它们所附着的活动一起从源流程定义映射到目标流程定义。以下情况适用：

* 如果边界事件被映射，它的持久性事件触发器（用于定时器、条件、消息和信号）被迁移。
* 如果源流程定义中的边界事件没有被映射，那么它的事件触发器在迁移期间被删除。
* 如果目标定义中的一个边界事件不是迁移指令的目标，那么在迁移过程中会初始化一个新的事件触发器


### 启动事件

事件子过程的启动事件可以从源头映射到目标，其语义与边界事件类似。特别是。

* 如果一个开始事件被映射，它的持久性事件触发器（用于定时器、条件反射、消息和信号）被迁移。
* 如果源流程定义中的一个开始事件没有被映射，那么它的事件触发器在迁移中被删除。
* 如果目标定义中的一个开始事件不是迁移指令的目标，那么在迁移过程中会初始化一个新的事件触发器。

### 中间捕获事件

如果一个流程实例在迁移过程中等待该事件，则必须映射中间的捕获事件。


### 补偿事件

#### 迁移补偿事件

当迁移具有活动补偿订阅的流程实例时，以下规则适用：

* 相应的补偿捕捉事件必须被映射出来
* 迁移后，可以从与迁移前相同的已迁移作用域或其最接近的已迁移父级触发补偿。
* 为了保留父作用域的变量快照，这些作用域也必须被映射。

具有活动补偿订阅的流程实例可以通过映射相应的捕捉补偿事件来进行迁移。
这告诉迁移 API 源流程模型的哪个补偿处理程序对应于目标流程模型中的哪个处理程序。

考虑一下这个源流程：

<div data-bpmn-diagram="../bpmn/process-instance-migration/example-compensation5"></div>

以及目标流程：

<div data-bpmn-diagram="../bpmn/process-instance-migration/example-compensation6"></div>

假设一个流程实例处于以下状态：

```
ProcessInstance
└── Assess Credit Worthiness
```

流程实例对 *Archive Application* 有一个补偿订阅。因此，迁移计划中必须包含一个补偿边界事件的映射。例如：

```java
MigrationPlan migrationPlan = processEngine.getRuntimeService()
  .createMigrationPlan("compensationProcess:1", "compensationProcess:2")
  .mapActivities("archiveApplication", "archiveApplication")
  .mapActivities("compensationBoundary", "compensationBoundary")
  .build();
```

在迁移后，可以从与迁移前相同的作用域（或者在该作用域被移除的情况下，可以从迁移的最接近的父级作用域）触发补偿。

为了说明问题，请考虑下面这个源流程：

<div data-bpmn-diagram="../bpmn/process-instance-migration/example-compensation3"></div>

以及目标流程：

<div data-bpmn-diagram="../bpmn/process-instance-migration/example-compensation4"></div>

当迁移与上例相同的流程实例状态时，在 *Archive Application* 活动上的内部补偿事件将 **不会触发** ，只会触发外部补偿事件。

{{< note title="积极补偿" class="info" >}}
  目前还不支持迁移具有活动补偿处理程序的流程实例。
{{< /note >}}


#### 增加的补偿事件

目标流程定义中包含的新补偿边界事件只对尚未开始或尚未完成的活动实例生效。

例如，考虑以下两个流程。

`compensation:1`:

<div data-bpmn-diagram="../bpmn/process-instance-migration/example-compensation1"></div>

`compensation:2`:

<div data-bpmn-diagram="../bpmn/process-instance-migration/example-compensation2"></div>

假设在迁移之前，一个流程实例处于以下状态：

```
ProcessInstance
└── Assess Credit Worthiness
```

如果这个流程实例被迁移（ *Assess Credit Worthiness* 被映射到它的等价物），迁移完成后是 **不会补偿** *Archive Application* 的。


## 子流程

如果为嵌入式/事件/过渡子流程设置了迁移指令，那么它们将被迁移到目标流程定义中的目标子流程。
这会保留子流程的状态，如变量。如果没有迁移指令，在执行迁移之前，子流程实例会被取消。
如果目标流程定义包含新的子流程，而现有的实例没有迁移到这些子流程的实例，那么这些子流程将在迁移过程中根据需要被实例化。

嵌入式/事件/过渡子流程可以互换地被映射。例如，可以将一个嵌入式子流程映射到一个事件子流程。

### 发起活动（Call Activity）

发起活动可以和其他活动一样被迁移。被发起的实例，无论是BPMN流程还是CMMN案例，都不会被改变。它可以单独迁移。


## Flow Node Markers

### 多实例

符合以下情况的多实例活动可以被迁移：

* 目标活动是同一类型的多实例（并行的或顺序的）。
* 目标活动不是多实例活动。

#### 迁移多实例活动

当把一个多实例活动的实例迁移到另一个多实例活动时，迁移计划需要包含两条指令。一条是针对 *内部活动* 的，也就是具有多实例循环特性的活动。另一条是针对 *多实例主体* 的。主体是包含内部活动的BPMN作用域，它没有直观地表示。按照惯例，它的id是 `<内部活动id>#multiInstanceBody` 。当迁移多实例体及其内部活动时，多实例的状态将被保留下来。这意味着，如果一个平行的多实例活动被迁移，五个实例中有两个处于活动状态，那么迁移后的状态也是一样的。

#### 移除多实例标记

如果目标活动不是多实例活动，那么只要有内部活动的指令就可以了。在迁移过程中，多实例变量 `nrOfInstances`、`nrOfActiveInstances`和`nrOfCompletedInstances`被删除。内部活动实例的数量被保留了。这意味着，如果在迁移前有五个活动实例中的两个，那么在迁移后将有两个目标活动的实例。此外，它们的`loopCounter`和集合元素变量会被保留。


### 异步延续

当一个异步延续是活动的，即相应的Job还没有被Job执行者完成，它以 *过渡实例* 的形式表示。例如，当Job执行失败，事件被创建时就是这种情况。对于过渡实例，映射指令就像对活动实例一样适用。这意味着，当有一个从活动`userTask`到活动`newUserTask`的指令时，所有代表`userTask`之前或之后异步延续的过渡实例被迁移到`newUserTask`。为了使之成功，目标活动也必须是异步的。

{{< note title="asyncAfter的限制" class="warning" >}}
当迁移表示异步延续的过渡实例时， asyncAfter，只有在以下限制成立的情况时，迁移才会成功：

* 如果源活动没有流出的序列流，那么目标活动不能有多于一个流出的序列流。
* 如果源活动有流出的序列流，那么目标活动必须有具有相同 ID 的序列流或必须有不超过一个流出的序列流。如果源活动有单一的序列流，这也适用。
{{< /note >}}


# 执行语义

下文描述了迁移的确切语义。建议阅读本节，以充分了解流程实例迁移的效果、能力和限制。

## 迁移程序

Migration of a process instance follows these steps:

1. Assignment of migration instructions to activity instances
2. Validation of the instruction assignment
3. Cancellation of unmapped activity instances and event handler entities
4. Migration of mapped activity instances and their dependent instances,
  instantiation of newly introduced BPMN scopes, and handler creation for newly introduced events


### 迁移指令的分配

In the first step, migration instructions are assigned to activity instances
of a process instance that is going to be migrated.


### 验证迁移指令

The created assignment must be executable by the migration logic which is
ensured by the validation step. In particular, the following conditions must hold:

* Exactly one instruction must apply to a leaf activity instance (e.g., user task)
* At most one instruction must apply to a non-leaf activity instance (e.g., embedded subprocess)
* The overall assignment must be executable. See the [validation chapter]({{< relref "#validation" >}}) for details.


### 取消未映射的活动实例和事件处理程序实例

Non-leaf activity instances to which no migration instructions apply are cancelled. Event handler entities
(e.g., message event subscriptions or timer jobs) are removed when their BPMN elements (e.g., boundary events)
are not migrated. Cancellation is performed before any migration instruction is applied,
so the process instance is still in the pre-migration state.

The semantics are:

* The activity instance tree is traversed in a bottom-up fashion and unmapped instances are cancelled
* Activity instance cancellation invokes the activity's end execution listeners and output variable mappings


### 迁移/创建活动实例

Finally, activity instances are migrated and new ones are created as needed.

The semantics are:

* The activity instance tree is traversed in a top-down fashion
* If an activity instance is migrated into a BPMN scope to which no parent activity instance
  is migrated, then a new activity instance is created
* Creation invokes the activity's start execution listeners and input variable mappings
* An activity instance is migrated according to its assigned migration instruction


#### 活动实例迁移

Migrating an activity instance updates the references to the activity and process definition
in the activity instance tree and its execution representation. Furthermore, it migrates or removes
*dependent* instances that belong to the activity instance. Dependent instances are:

* Variable instances
* Task instances (for user tasks)
* Event subscription instances


## 验证

A migration plan is validated at two points in time: When it is created, its
instructions are validated for static aspects. When it is applied to a
process instance, its instructions are matched to activity instances
and this assignment is validated.

Validation ensures that none of the limitations stated in this guide lead
to an inconsistent process instance state with undefined behavior after migration.


### 创建时间验证

For an instruction to be valid when a migration plan is created, it has to fulfill
the following requirements:

* It has to map activities of the same type
* It has to be a one-to-one mapping
* A migrated activity must remain a descendant of its closest migrating ancestor scope (**Hierarchy Preservation**)
* The migration plan adheres to [BPMN-element-specific considerations]({{< relref "#bpmn-specific-api-and-effects" >}})
* A set variable must not be of type `Object` **AND** its `serializationFormat` must not be `application/x-java-serialized-object`
  * Validation is skipped when the engine configuration flag `javaSerializationFormatEnabled` is set to `true`
  * Please see [Process Engine Configuration Reference]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#javaSerializationFormatEnabled" >}}) for more details

If validation reports errors, migration fails with a `MigrationPlanValidationException`
providing a `MigrationPlanValidationReport` object with details on the
validation errors.


#### 层次结构保存

An activity must stay a descendant of its closest ancestor scope that migrates (i.e., that is not cancelled during migration).

Consider the following migration plan for the example processes shown at the
[beginning of this chapter]({{<
ref "/user-guide/process-engine/process-instance-migration.md" >}}):

```java
MigrationPlan migrationPlan = processEngine.getRuntimeService()
  .createMigrationPlan("exampleProcess:1", "exampleProcess:2")
  .mapActivities("assessCreditWorthiness", "handleApplicationReceipt")
  .mapActivities("validateAddress", "validatePostalAddress")
  .build();
```

And a process instance in the following state:

```
ProcessInstance
└── Assess Credit Worthiness
    └── Validate Address
```

The migration plan cannot be applied, because the
hierarchy preservation requirement is violated: The activity *Validate
Address* is supposed to be migrated to *Validate Postal Address*. However, the
parent scope *Assess Credit Worthiness* is migrated to *Handle
Application Receipt*,  which does not contain *Validate Postal Address*.


### 执行时间验证

When a migration plan is applied to a process instance, it is validated beforehand
that the plan is applicable. In particular, the following aspects are checked:

* **Completeness**: There must be a migration instruction for every instance of leaf activities
  (i.e., activities that do not contain other activities)
* **Instruction Applicability**: For certain activity types, only transition instances but not
  activity instances can be migrated

If validation reports errors, migration fails with a `MigratingProcessInstanceValidationException`
providing a `MigratingProcessInstanceValidationReport` object with details on the
validation errors.

#### 完整性

Migration is only meaningful if a migration instruction applies to every instance of a leaf activity. Assume a migration plan as follows:

```java
MigrationPlan migrationPlan = processEngine.getRuntimeService()
  .createMigrationPlan("exampleProcess:1", "exampleProcess:2")
  .mapActivities("archiveApplication", "archiveApplication")
  .build();
```

Now consider a process instance in the following activity instance state:

```
ProcessInstance
└── Archive Application
```

The plan is complete with respect to this process instance because there is a migration instruction for the activity *Archive Application*.

Now consider another process instance:

```
ProcessInstance
├── Archive Application
└── Assess Credit Worthiness
    └── Validate Address
```

The migration plan is not valid with respect to this instance because there is no instruction that applies to the instance of *Validate Address*.


#### 指令适用性

Migration instructions are used to migrate activity instances as well as transition instances (i.e., active asynchronous continuations). Some
instructions can only be used to migrate transition instances but not activity instances. In general, activity instances can only be
migrated if they are instances of the following element types:

* Task
  * User Task
  * Receive Task
  * External Task
* Subprocess
  * Embedded Sub Process
  * Event Sub Process
  * Transaction Sub Process
  * Call Activity
* Gateways
  * Parallel Gateway
  * Inclusive Gateway
  * Event-based Gateway
* Events
  * Boundary Event
  * Intermediate Catch Event
* Misc
  * Multi-instance Body

Transition instances can be migrated for any activity type.

[batch]: {{< ref "/user-guide/process-engine/batch.md" >}}
[job executor]: {{< ref "/user-guide/process-engine/the-job-executor.md#job-execution-in-heterogeneous-clusters" >}}
[execution jobs]: {{< ref "/user-guide/process-engine/batch.md#execution-jobs" >}}


### 验证未涵盖的方面

#### 数据一致性

Process instances contain data such as variables that are specific to how a process is implemented.
Validation cannot ensure that such data is useful in the context of the target process definition.

#### 对象变量的反序列化

[Object type variables]({{< ref "/user-guide/process-engine/variables.md#supported-variable-values" >}}) represent Java objects. That means they have a serialized value along with a Java type name that is used to deserialize the value into a Java object. When migrating between processes of different process
applications, it may occur that an Object variable refers to a Java class that does not exist in the process
application of the target process.

This scenario is not prevented by validation. Accessing the deserialized value may therefore fail after migration.
If you end up with unusable Object variables, there are two ways to deal with
that situation:

* Add the missing classes to the target process application
* Convert the inconsistent variable into a variable for which the Java class is present based on its serialized value
