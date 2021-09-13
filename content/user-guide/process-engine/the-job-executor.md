---

title: 'Job 执行器'
weight: 170

menu:
  main:
    identifier: "user-guide-process-engine-the-job-executor"
    parent: "user-guide-process-engine"

---


Job是触发流程执行的任务的明确表示。当一个定时器事件或一个标记为异步执行的任务(见[事务边界]({{< ref "/user-guide/process-engine/transactions-in-processes.md" >}})接近时，就会创建一个Job。因此，Job处理可以被分成三个阶段。

* [Job 创建]({{< relref "#job-创建" >}})
* [Job 获取]({{< relref "#job-获取" >}})
* [Job 执行]({{< relref "#job-执行" >}})

虽然Job是在流程执行过程中创建的，但Job的获取和执行是Job执行器的责任。下图说明了这两个步骤。

{{< img src="../img/job-executor-basic-architecture.png" title="Basic Architecture" >}}


# Job执行器激活

当使用 **嵌入式流程引擎** 时，默认情况下，当流程引擎启动时，Job执行器不会被激活。

如果你希望在流程引擎时激活Job执行器。需要在流程引擎中配置如下配置项：

```xml
<property name="jobExecutorActivate" value="true" />
```

当使用 **shared process engine** 时，默认情况则相反：如果你没有在流程引擎配置上指定`jobExecutorActivate` 属性，Job执行器会自动启动。为了将其关闭，你必须明确地将该属性设置为false：

```xml
<property name="jobExecutorActivate" value="false" />
```

# 在单元测试中使用Job执行器

在单元测试中，使用这个后台组件是很麻烦的。因此，Java API提供了查询（`ManagementService.createJobQuery`）和 *手动* 执行Job（`ManagementService.executeJob`）的功能，这允许从单元测试中控制Job的执行。

# Job 创建

Job是由流程引擎为很多不同的目的而创建的。存在以下Job类型：

* 在流程中使用异步延续设置[事务边界]({{< ref "/user-guide/process-engine/transactions-in-processes.md" >}})
* 用于BPMN计时器事件的定时器Job
* BPMN事件的异步处理

在创建过程中，Job可以得到一个获取和执行的优先权。


## Job 优先级

在实践中，Job的处理量很少均匀地分布在一天中。相反，会有高负荷的高峰期，例如，当夜间运行批处理。在这种情况下，Job执行器可能会暂时过载的情况：数据库中的Job比Job执行器一次能处理的要多得多。 *Job优先级* 可以帮助应对这些情况，在一个明确的事项中，定义一个重要性的顺序，并按该顺序启用执行。

一般来说，有两种类型的用例可以用Job优先级来解决：

**在设计时预测优先级** 。在许多情况下，在设计一个流程模型时，可以预见到高负载的情况。在这些情况下，根据某些业务目标来确定Job执行的优先次序往往是很重要的。例如：
  * 一家零售店有休闲和VIP客户。在高负载的情况下，希望以更高的优先级处理VIP客户的订单，因为他们的满意度对公司的业务目标更重要。
  * 一家家具店使用以人为本的理念，他们为客户购买家具提供咨询，也有非时间优先的交付流程。优先排序可以用来确保咨询流程的快速响应时间，提高用户和客户的满意度。
**优先化作为对运行时条件的响应** 。 一些Job执行器高负荷的情况是由运行时的不可预见的条件造成的，这些条件在流程设计时无法处理。暂时覆盖优先级可以帮助优雅地处理这类情况。例如：
  * 一个服务任务访问一个网络服务来处理付款。支付服务遇到了过载，响应非常慢。为了避免因等待服务响应而占用Job执行器的所有资源，可以暂时降低各自Job的优先级。这样一来，不相关的流程实例和Job就不会被拖慢。在服务恢复后，可以再次清除优先级。


## Job优先权

优先级是Java `long` 值范围内的一个自然数。一个较高的数字代表一个较高的优先级。一旦被分配，优先级是静态的，这意味着流程引擎不会在未来的任何时候再次为该Job分配优先级。

Job优先级在流程执行过程中影响两个阶段：Job创建和Job获取。在Job创建期间，一个Job被分配了一个优先级。在Job获取过程中，流程引擎可以根据给定的Job优先级，相应地安排其执行。这意味着，Job是严格按照优先级的顺序来获取的。

{{< note title="关于Job饥饿问题的提示" class="info" >}}
在调度场景中，饥饿是一个典型的问题。当连续创建高优先级Job时，可能会发生永远无法获取低优先级Job的情况。

在性能方面，严格按优先级获取Job使Job执行器能够使用索引进行排序。 像 [aging](https://en.wikipedia.org/wiki/Aging_%28scheduling%29) 这样动态提高饥饿Job优先级的解决方案不能轻易地用索引来补充。

此外，在Job执行器永远无法赶上执行Job表中的所有Job以致无法在合理的时间内执行低优先级Job的环境中，可能存在资源过载的普遍问题。 在这种情况下，解决方案可能是根据 Job Executor 优先级范围分配工作负载（请参阅 [Job Executor priority range]({{< relref "#job-executor-priority-range" >}})）或增加 通过向集群添加新节点来分配Job执行器资源。
{{< /note >}}


## 为Job优先级配置流程引擎

本节解释了如何在配置中启用和禁用Job优先级。有两个相关的配置属性可以在流程引擎配置中设置。

`producePrioritizedJobs`。控制流程引擎是否为Job分配优先级。默认值是 "true"。如果不需要优先级，流程引擎的配置属性`producePrioritizedJobs`可以设置为`false`。在这种情况下，所有Job的优先级都是0。
关于如何指定Job优先级以及流程引擎如何分配优先级的细节，请参见下面的[指定Job优先级]一节（{{< relref "#specify-job-priorities" >}}）。

`jobExecutorAcquireByPriority`: 控制Job是否按照其优先级来获取。默认值是 "false"，这意味着需要明确地启用它。提示：当启用这个功能时，也应该创建额外的数据库索引。详见[Job获取顺序]({{< relref "#the-job-order-of-job-acquisition" >}})一节。


## 指定Job优先级

Job优先级可以在BPMN模型中指定，也可以通过API在运行时覆盖。


### 在 BPMN XML 模型文件中指定

Job优先级可以在流程或活动级别上分配。为了实现这一点，可以使用Camunda扩展属性`camunda:jobPriority`。

指定优先级，常量值和[表达式]({{< ref "/user-guide/process-engine/expression-language.md" >}})都是支持的。当使用常量值时，同一优先级被分配给流程或活动的所有实例。另一方面，表达式允许给流程或活动的每个实例分配不同的优先级。表达式必须评估为 Java `long`范围内的一个数字。
具体数值可以是复杂计算的结果，并基于用户提供的数据（来自任务表格或其他来源）。


#### 在流程级别设置优先级

在流程实例级别配置Job优先级，需要在bpmn的 `<process ...>` 元素上应用 `camunda:jobPriority` 属性。

```xml
<bpmn:process id="Process_1" isExecutable="true" camunda:jobPriority="8">
  ...
</bpmn:process>
```

其效果是，流程内的所有活动都继承相同的优先级（除非它被本地重写）。
也请参见。[Job优先级模式]({{< relref "#job-priority-precedence-schema" >}})。

上面的例子显示了如何使用一个常量值来设置优先级。这样一来，相同的优先级就会应用于流程的所有实例。如果不同的流程实例需要以不同的优先级执行，可以使用一个表达式。

```xml
<bpmn:process id="Process_1" isExecutable="true" camunda:jobPriority="${order.priority}">
  ...
</bpmn:process>
```

在上面的例子中，优先级是根据变量`order`的`priority`属性决定的。


#### 在活动级别设置优先级

在活动级别设置Job优先级, 需要在对应的 bpmn 元素上添加 `camunda:jobPriority` 属性:

```xml
<bpmn:serviceTask id="ServiceTask_1"
  name="Prepare Payment"
  camunda:asyncBefore="true"
  camunda:jobPriority="100" />
```

其效果是，该优先级会被应用于给定服务任务的所有实例。
该优先级会覆盖流程级的优先级。更多参见[Job优先级模式]({{< relref "#job-priority-precedence-schema" >}}).

当使用一个常量值时，如上面的例子所示，相同的优先级被应用于服务任务的所有实例。也可以使用一个表达式：

```xml
<bpmn:serviceTask id="ServiceTask_1"
  name="Schedule Delivery"
  camunda:asyncBefore="true"
  camunda:jobPriority="${customer.status == 'VIP' ? 10 : 0}" />
```

在上面的例子中，优先级是根据当前customer的属性`status`决定的。


#### 优先级表达式的执行环境

本节解释了在计算优先级表达式时，哪些环境变量和函数是可用的。
关于这方面的通用文档，请参见相应的[文档部分]({{< ref "/user-guide/process-engine/expression-language.md#availability-of-variables-and-functions-inside-expression-language" >}})。

所有的优先级表达式都是在当前 *执行* 的背景下进行计算的。这意味着变量 "execution" 是可用的，所有执行的变量也是可用的。

唯一的例外是会导致一个新的流程实例化的Job的优先级。
例如：

* Timer Start Event
* Asynchronous Signal Start Event


#### 向被调用流程实例传播优先级

当通过 Call Activity 启动一个流程实例时，你有时希望该流程实例 "继承" 调用流程实例的优先级。
实现这一目的的最简单的方法是用一个变量传递优先级，并在被调用的流程中用表达式来引用它。
如何通过 Call Activity 传递变量，参见[Call Activity 参数]({{< ref "/reference/bpmn20/subprocesses/call-activity.md#passing-variables" >}})章节。


### 通过ManagementService API设置job定义的优先级

有时需要在运行时改变Job的优先级，以处理特殊的情况。例如：考虑一个订单流程的服务任务 *Process Payment* ：该服务任务调用了一些外部支付服务，这些服务可能是繁忙的，因此响应很慢。因此，Job执行器被阻塞，等待响应。其他具有相同或更低优先级的并发Job不能继续进行，尽管在这种特殊情况下，这是合理的。


#### 通过Job定义覆盖优先级

有时需要在运行时改变Job的优先级，以处理特殊的情况。 因此，ManagementService API允许为一个Job定义临时覆盖设置优先级。下面的操作可以为一个给定Job定义的所有未来Job降低优先级。

```java
// 查询Job定义
JobDefinition jobDefinition = managementService
  .createJobDefinitionQuery()
  .activityIdIn("ServiceTask_1")
  .singleResult();

// 设置覆盖优先级
managementService.setOverridingJobPriorityForJobDefinition(jobDefinition.getId(), 0L);
```

设置覆盖优先级，可以确保根据这个定义创建的每个新Job都能获得给定的优先级。这个设置会覆盖BPMN XML中指定的任何优先级。

另外，通过使用`cascade`参数，覆盖的优先级可以分配于该定义的所有现有Job。

```java
managementService.setOverridingJobPriorityForJobDefinition(jobDefinition.getId(), 0L, true);
```

请注意，这不会导致当前执行的Job的抢占行为。


#### 通过Job定义重置优先级

当服务从繁忙情况中恢复后，可按以下方式清除覆盖的优先权。

```java
managementService.clearOverridingJobPriorityForJobDefinition(jobDefinition.getId());
```

从现在开始，所有新Job都会再次分配BPMN XML中指定的优先级。


### Job优先级来源

下图总结了在确定一项Job的优先权时，优先权来源的先后顺序：

{{< img src="../img/job-executor-priority-precedence.png" title="^Priority Precedence" >}}


### 通过ManagementService API设置Job优先级

ManagementService还提供了`ManagementService#setJobPriority(String jobId, long priority)`方法，可以改变单个Job的优先级。


# Job获取

Job获取是指从数据库中查询接下来要执行的Job的过程。因此，Job必须与决定Job是否可以执行的属性一起被持久化到数据库中。例如，为一个定时器事件创建的Job在定义的时间跨度过去之前可能不会被执行。


## 持久化

Job被持久化到数据库的`ACT_RU_JOB`表中。该数据库表主要有以下列：

```
ID_ | REV_ | LOCK_EXP_TIME_ | LOCK_OWNER_ | RETRIES_ | DUEDATE_
```

Job获取涉及到轮询这个数据库表和锁定Job。


## 可获取的Jobs

如果一项Job满足以下所有条件，它就是可获取的，也就是可以执行的候选Job。

* 它到期了，也就是`DUEDATE_`列中的值是过去的。
* 它没有被锁定，也就是`LOCK_EXP_TIME_`列中的值是过去的。
* 它的重试次数没有被耗尽，也就是`RETRIES_`列中的值大于零。

此外，流程引擎有一个Job暂停的概念。例如，当一个Job所属的流程实例被暂停时，它就会被暂停。一个Job只有在没有暂停的情况下才可以获取。

### Job获取性能优化

为了优化需要立即执行的Job的获取，"DUEDATE_"列没有设置（"null"），并添加一个（积极的）空检查作为获取的条件。

如果每个Job都必须设置 "DUEDATE_"，可以禁用优化。这可以通过设置[流程引擎配置标志]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#ensureJobDueDateNotNull" >}}) "ensureJobDueDateNotNull" 为 `true` 来实现。

然而，在禁用优化之前，任何 "DUEDATE_" 为 "null" 的Job将不会被Job获取阶段所获取，除非这些Job被明确地以{{< javadocref page="?org/camunda/bpm/engine/ManagementService.html#setJobDuedate-java.lang.String-java.util.Date-" text="Java" >}}/[Rest]({{< ref "/reference/rest/job/put-set-job-duedate.md" >}})的方式更新到期时间。 

## Job 获取的两个步骤

Job获取有两个阶段。第一阶段，Job执行器按照配置的数量查询的可获得的Job。如果至少能找到一个Job，就进入第二阶段，锁定Job。锁定是必要的，以确保Job被精确地执行一次。在一个集群场景中，通常会运行多个Job执行器实例（每个节点一个），它们都轮询同一个ACT&#95;RU&#95;JOB表。锁定一个Job可以确保它只被一个Job执行器实例获取。锁定一个Job意味着更新它在LOCK&#95;EXP&#95;TIME&#95;和LOCK&#95;OWNER&#95;列中的值。LOCK&#95;EXP&#95;TIME&#95;列被更新为一个时间戳，表示一个在未来的日期。这背后的意义是，我们想要锁定Job，直到达到该日期。LOCK&#95;OWNER&#95;列被更新为一个唯一标识当前Job执行器实例的值。在一个集群场景中，这是唯一可能识别当前集群节点的节点名称。

多个Job执行器实例试图同时锁定同一个Job的情况，可以通过使用乐观锁定来解决（见REV&#95;列）。

在锁定一个Job后，Job执行器实例已经有效地得到了执行Job的时间段：一旦达到写入LOCK&#95;EXP&#95;TIME&#95;列的日期，它将再次被传递到获取的Job队列中，对Job获取可见。


## 可获取Job的顺序

默认情况下，Job执行器并不附加一个可获取Job的顺序。这意味着Job获取的顺序取决于数据库和它的配置。这就是为什么Job获取被假定为随机的。这样做的目的是为了保持Job获取查询的简单和快速。

但这种获取Job的方法并不是在所有情况下都够用的，例如：

* **Job的优先次序**：当创建[优先考虑的Job]({{< relref "#job-prioritization" >}})时，Job执行器必须根据给定的优先级获取Job。
* **Job 饥饿**：在高负荷的情况下，当新Job创建的速度高于Job执行器所能处理的速度时，Job饥饿在理论上是可能的。
* **Preferred Handling of Timers**: 在高负荷的情况下，定时器的执行可能延迟到比实际到期日晚得多的时间点。虽然到期日不是保证Job执行的实时边界，但在某些情况下，一旦有Job可以执行，最好是立即获取定时器Job。

为了解决前面描述的问题，Job获取查询可以由流程引擎配置属性控制。目前，支持三个选项：

- `jobExecutorAcquireByPriority`. 如果设置为 `true`, Job执行器将获得具有最高优先级的Job。

- `jobExecutorPreferTimerJobs`. 如果设置为 `true`, Job执行器将在其他Job类型之前获取所有可获取的定时器Job。这并没有指定被获取的Job类型中的顺序。

- `jobExecutorAcquireByDueDate`. 如果设置为 `true`, Job执行器将按到期日升序获取Job。异步延续将其创建日期作为到期日，所以它是立即可执行的。

使用这些选项的组合会产生一个多级排序。选项的优先级层次与上述顺序相同。如果三个选项都被激活，那么优先级是首要的，Job类型是次要的，而到期日是第三级排序。这也表明，激活所有选项并不是解决优先级、饥饿和定时器处理等问题的最佳方案。例如，在这种情况下，定时器Job只在一个级别的优先级内被优先考虑。优先级较低的定时器是在所有优先级较高的Job被获取后才被获取的。建议根据具体的使用场景来决定激活哪些选项。

案例：

* 优先考虑Job执行， 只有 `jobExecutorAcquireByPriority` 应该设置为 true
* 尽快执行计时器Job, `jobExecutorPreferTimerJobs` 和
  `jobExecutorAcquireByDueDate` 两个选项应该被设置。Job执行器将首先获取定时器Job，之后是异步延续Job。而且还在类型内按到期日升序排列这些Job。

{{< note title="" class="warning" >}}
  所有这些选项默认值为 "false"，只有在使用场景需要时才应激活。这些选项改变了所使用的Job获取查询，并可能影响其性能。这就是为什么我们也建议在`ACT_RU_JOB`表的相应列上添加索引。
{{< /note >}}

<table class="table table-striped">
  <tr>
    <th>jobExecutorAcquireByPriority</th>
    <th>jobExecutorPreferTimerJobs</th>
    <th>jobExecutorAcquireByDueDate</th>
    <th>Recommended Index</th>
  </tr>
    <tr>
    <td><code>true</code></td>
    <td><code>false</code></td>
    <td><code>false</code></td>
    <td><code>ACT_RU_JOB(PRIORITY_ DESC)</code></td>
  </tr>
  <tr>
    <td><code>false</code></td>
    <td><code>true</code></td>
    <td><code>false</code></td>
    <td><code>ACT_RU_JOB(TYPE_ DESC)</code></td>
  </tr>
  <tr>
    <td><code>false</code></td>
    <td><code>false</code></td>
    <td><code>true</code></td>
    <td><code>ACT_RU_JOB(DUEDATE_ ASC)</code></td>
  </tr>
  <tr>
    <td><code>false</code></td>
    <td><code>true</code></td>
    <td><code>true</code></td>
    <td><code>ACT_RU_JOB(TYPE_ DESC, DUEDATE_ ASC)</code></td>
  </tr>
</table>

## Job Executor priority range

By default, the Job Executor executes all jobs regardless of their priorities. Some jobs might be more important to finish quicker than others, so we assign them priorities and set `jobExecutorAcquireByPriority` to `true` as described above. Depending on the workload, the Job Executor might be able to execute all jobs eventually. But if the load is high enough, we might face starvation where a Job Executor is always busy working on high-priority jobs and never manages to execute the lower priority jobs.

To prevent this, you can specify a priority range for the job executor by setting values for [`jobExecutorPriorityRangeMin`]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#jobExecutorPriorityRangeMin" >}}) or [`jobExecutorPriorityRangeMax`]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#jobExecutorPriorityRangeMax" >}}) (or both). The Job Executor will only acquire jobs that are inside its priority range (inclusive). Both properties are optional, so it is fine only to set one of them.

To avoid job starvation, make sure to have no gaps between Job Executor priority ranges. If, for example, Job Executor A has a priority range of 0 to 100 and Job Executor B executes jobs from priority 200 to `Long.MAX_VALUE` any job that receives a priority of 101 to 199 will never be executed. Job starvation can also occur with `batch jobs` and `history cleanup jobs` as both types of jobs also receive priorities (default: `0`). You can configure them via their respective properties: `batchJobPriority` and `historyCleanupJobPriority`.

This feature is particularly useful if you want to separate multiple types of jobs from each other. For example, short-running, urgent jobs with high priority and long-running jobs that are not urgent but should finish eventually. Setting up a Job Executor priority range for both types will ensure that long-running jobs can not block urgent ones.

## 规避策略
Job执行器使用一个规避策略来避免集群中的获取冲突，并在没有Job到期时减少数据库的负载。第二点可能会导致Job创建和Job执行之间的延迟，因为默认情况下，Job获取会将延迟时间翻倍到下一次获取运行。
默认的最大等待时间是60秒。你可以通过设置配置参数`maxWait`到一个低于60000毫秒的值来减少延迟。

# Job执行

## 线程池

获取的Job由线程池执行。线程池从获取的Job队列中消耗Job。获得的Job队列是一个具有固定容量的内存队列。当执行器开始执行一个Job时，它首先被从队列中移除。

在嵌入式流程引擎的情况下，该线程池的默认实现是 `java.util.concurrent.ThreadPoolExecutor` 。然而，这在Java EE环境中是不允许的。在那里，我们要与应用服务器的线程管理能力挂钩。关于如何实现，请参见[运行时容器集成]({{< ref "/user-guide/runtime-container-integration/_index.md" >}})部分中的特定平台信息。


## 失败的Job

当Job执行失败时，例如，服务任务调用抛出一个异常，一个Job将被重试若干次（默认为2，因此，该Job总共被试了三次）。
它不会被立即重试并加回获取队列，但RETRIES&#95;列的值会减少，执行器会解锁该Job。
流程引擎因此对失败的Job进行了记录。解锁还包括删除时间LOCK&#95;EXP&#95;TIME&#95;和锁的所有者LOCK&#95;OWNER&#95;，将这两个条目设置为`null`。随后，一旦Job被获取执行，失败的Job将自动被重试。一旦重试次数用尽（RETRIES&#95;列的值等于0），该Job就不再执行，引擎在该Job处停止，表示无法继续。

{{< note title="" class="info" >}}
虽然所有失败的Job都会被重试，但有一种情况下，Job的重试次数不会被递减。这就是，如果一个Job由于乐观的锁定异常而失败。乐观锁定是流程引擎解决资源更新冲突的机制，例如，当一个流程实例的两个Job并行执行时（见下面关于[并发Job执行]({{< relref "#concurrent-job-execution" >}})的章节）。由于从操作者的角度来看，乐观锁定异常并不是什么特殊情况，而且最终会解决，所以不会导致重试递减。
{{< /note >}}

如果启用了Job的事件创建，那么一旦Job重试耗尽，就会创建一个事件。 （查看 [事件的激活与冻结]({{< ref "/user-guide/process-engine/incidents.md#事件的激活与冻结" >}})）.
与Job相关的事件和历史事件可以通过类似如下的Java API请求访问：
```java
List<Incident> incidents = engineRule.getRuntimeService()
        .createIncidentQuery().configuration(jobId).list();

List<HistoricIncident> historicIncidents = engineRule.getHistoryService()
        .createHistoricIncidentQuery().configuration(jobId).list();
```

### 重试策略配置

默认情况下，一个失败的Job将被重试三次，重试是在失败后立即执行的。在日常业务中，配置重试策略可能是有用的，也就是说，通过设置重试Job的频率和引擎应该等待多长时间，直到它再次尝试执行Job。这种配置可以在流程引擎配置中全局指定：

```xml
<process-engine name="default">
  ...
  <properties>
    ...
    <property name="failedJobRetryTimeCycle">R5/PT5M</property>
  </properties>
</process-engine>
```

配置遵循 [ISO_8601 对重复时间间隔的规范](http://en.wikipedia.org/wiki/ISO_8601#Repeating_intervals)。例如，`R5/PT5M` 意味着最大重试次数是 5 (`R5`) 重试的延迟是5分钟 (`PT5M`).

Camunda引擎允许你为以下特定元素配置这一设置：

* [活动 (任务, call activities, 子流程)]({{< relref "#use-a-custom-job-retry-configuration-for-activities" >}})
* [事件]({{< relref "#use-a-custom-job-retry-configuration-for-events" >}})
* [多实例活动 ]({{< relref "#use-a-custom-job-retry-configuration-for-multi-instance-activities" >}})


#### 为活动使用自定义重试策略

一旦重试配置被启用，它就可以应用于任务、调用活动、嵌入式子流程和事务子流程。例如，任务中的Job重试可以在Camunda引擎中的BPMN 2.0 XML中配置如下。

```xml
<definitions xmlns:camunda="http://camunda.org/schema/1.0/bpmn">
  ...
  <serviceTask id="failingServiceTask" camunda:asyncBefore="true" camunda:class="org.mycompany.FailingDelegate">
    <extensionElements>
      <camunda:failedJobRetryTimeCycle>R5/PT5M</camunda:failedJobRetryTimeCycle>
    </extensionElements>
  </serviceTask>
  ...
</definitions>
```

你也可以在重试配置中设置表达式：

```xml
  <camunda:failedJobRetryTimeCycle>${retryCycle}</camunda:failedJobRetryTimeCycle>
```

LOCK&#95;EXP&#95;TIME&#95;用于定义何时可以再次执行Job，这意味着一旦LOCK&#95;EXP&#95;TIME&#95;日期过期，失败的Job将自动重试。

#### 为事件使用自定义重试策略

Job重试可以针对以下事件进行配置：

* Timer Start Event
* Boundary Timer Event
* Intermediate Timer Catch Event
* Intermediate Throw Event

与任务类似，重试可以作为事件的一个扩展元素来配置。下面的例子为一个边界定时器事件定义了三次重试，每次5秒：

```xml
<definitions xmlns:camunda="http://camunda.org/schema/1.0/bpmn">
  ...
  <boundaryEvent id="BoundaryEvent" name="BoundaryName" attachedToRef="MyActivity">
    <extensionElements>
      <camunda:failedJobRetryTimeCycle>R3/PT5S</camunda:failedJobRetryTimeCycle>
    </extensionElements>
    <outgoing>SequenceFlow_3</outgoing>
    <timerEventDefinition>
      <timeDuration>PT10S</timeDuration>
    </timerEventDefinition>
  </boundaryEvent>
  ...
</definitions>
```

提醒：如果在定时器之后的事务中出现任何故障，可能需要重试。

#### 为多实例活动使用自定义重试策略

如果为多实例活动设置了重试配置，那么该配置将应用于[多实例活动]({{< ref "/user-guide/process-engine/transactions-in-processes.md#asynchronous-continuations-of-multi-instance-activities" >}})。此外，内部活动的重试也可以使用扩展元素作为 "multiInstanceLoopCharacteristics" 元素的子元素进行配置。

下面的例子自定义了一个多实例服务主体及其内部活动的异步延续任务的重试。如果在五个并行实例中的一个发生了故障，那么故障实例的Job将被重试，最多3次，延迟5秒。如果所有的实例都成功结束，并且在延续任务的事务中发生了故障，那么该Job将被重试5次，延迟5分钟。

```xml
<definitions xmlns:camunda="http://camunda.org/schema/1.0/bpmn">
  ...
  <serviceTask id="failingServiceTask" camunda:asyncAfter="true" camunda:class="org.mycompany.FailingDelegate">
    <extensionElements>
      <!-- 配置多实例活动主体，例如活动后异步延续 -->
      <camunda:failedJobRetryTimeCycle>R5/PT5M</camunda:failedJobRetryTimeCycle>
    </extensionElements>
    <multiInstanceLoopCharacteristics isSequential="false" camunda:asyncBefore="true">
      <extensionElements>
        <!-- 配置内部活动的，例如在每个实例开始之前异步延续 -->
        <camunda:failedJobRetryTimeCycle>R3/PT5S</camunda:failedJobRetryTimeCycle>
      </extensionElements>
      <loopCardinality>5</loopCardinality>
    </multiInstanceLoopCharacteristics>
  </serviceTask>
  ...
</definitions>
```

### 重试间隔
属性重试时间周期允许定义重试的次数和失败的Job应该被重试的间隔。例如 R5/PT5M，时间间隔总是（至少）5分钟。当你需要没有静态间隔时，你可以在全局或特定Job配置中配置重试间隔列表（用逗号分隔）。本地配置具有优先权。
这是流程引擎配置的示例：
```xml
<process-engine name="default">
  ...
  <properties>
    ...
    <property name="failedJobRetryTimeCycle">PT10M,PT17M,PT20M</property>
  </properties>
</process-engine>
```
重试时间有三个，此示例的行为如下：

* Job首次失败 -> Job将在10分钟内重试 (PT10M)
* Job第二次失败 -> Job 将在17分钟内重试 (PT17M)
* Job第三次失败 -> Job 将在20分钟内重试 (PT20M)

如果用户在重试过程中决定将重试次数改为更高，那么列表的最后一个间隔将在新值和列表大小的差额内应用。此后，它将继续进行上述的正常流程。

### 自定义重试策略

你可以通过添加`customPostBPMNParseListeners`属性来配置一个自定义重试配置，并在流程引擎配置中指定你的自定义`FailedJobParseListener`。

```xml
<bean id="processEngineConfiguration" class="org.camunda.bpm.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration">
  <!-- 你定义的属性! -->
  ...
  <property name="customPostBPMNParseListeners">
    <list>
      <bean class="com.company.impl.bpmn.parser.CustomFailedJobParseListener" />
    </list>
  </property>
  ...
</bean>
```


# Job执行器的并发

Job执行器确保 **来自单个流程实例的Job不会被同时执行** 。为什么是这样呢？考虑一下下面的流程定义。

{{< img src="../img/job-executor-exclusive-jobs.png" title="Exclusive Jobs" >}}

我们有一个并行的网关，后面是三个服务任务，它们都执行[异步延续]({{< ref "/user-guide/process-engine/transactions-in-processes.md#asynchronous-continuations" >}})。 这样做的结果是，三个Job被添加到数据库中。一旦这样的Job出现在数据库中，它就可以被Job执行器处理。它获取Job并将其委托给实际处理Job的Job线程池。这意味着，使用异步延续，你可以将Job分配给这个线程池（在集群的情况下，甚至跨越集群中的多个线程池）。

这通常是一件好事。然而，它也存在着一个固有的问题：一致性。考虑一下服务任务之后的并行连接。当一个服务任务的执行完成后，我们到达并行连接处，需要决定是否等待其他执行，或者是否可以继续前进。这意味着，对于到达并行连接的每个分支，我们需要决定是否可以继续，或者是否需要等待其他分支的一个或多个执行。

这就要求各执行分支之间进行同步。流程引擎用乐观锁来解决这个问题。每当我们根据可能不是最新的数据做出决定时（因为另一个事务可能在我们提交之前修改它），我们要确保在两个事务中增加同一数据库行的修订版。这样一来，哪个事务先提交就赢了，其他事务就会因乐观锁定异常而失败。这就解决了上面讨论的流程的问题：如果多个执行同时到达并行连接，它们都会认为自己必须等待，增加其父执行（流程实例）的修订版，然后尝试提交。无论哪个执行是第一个，都可以提交，而其他的则会因为乐观锁异常而失败。由于这些执行是由Job触发的，Job执行器在等待一定时间后会重新尝试执行相同的Job，希望这次能通过同步网关。

然而，虽然从持久性和一致性的角度来看，这是一个完美的解决方案，但在更高层次上，这可能并不总是理想的行为，特别是如果执行有非交易的副作用，这不会被失败的事务回滚。例如，如果 *预订音乐会门票* 服务不与流程引擎共享同一事务，如果我们重试Job，我们可能会预订多张门票。这就是为什么同一流程实例的Job在默认情况下被 *排他* 处理。


## 排他处理的Job

具有排他性Job不能与同一流程实例的其他排他性Job同时执行。考虑上节所示的流程：如果对应于服务任务的Job被视为排他性的，Job执行器将尽量避免它们被并行执行。相反，它将确保每当它从某个流程实例中获取一个独占Job时，它也会从同一流程实例中获取所有其他独占Job，并将它们委托给同一个Job线程。这就强制了这些Job的顺序执行，并且在大多数情况下避免了乐观的锁定异常。然而，这种行为是启发式的，也就是说，Job执行器只能强制执行在 **查询时间内** 可用的Job的顺序执行。如果在这之后创建了一个潜在的冲突Job，目前正在运行或已经被安排执行，该Job可能被另一个Job执行线程并行处理。

**排他Job是默认配置** 。因此，所有的异步延续和定时器事件在默认情况下都是独占的。此外，如果你想让一个Job成为非排他性的，你可以使用 `camunda:exclusive="false"` 来配置它。例如，下面的服务任务将是异步的，但不是排他的。

```xml
<serviceTask id="service" camunda:expression="${myService.performBooking(hotel, dates)}" camunda:asyncBefore="true" camunda:exclusive="false" />
```

这是一个好的解决方案吗？我们有一些人怀疑。因为他们的担心，排他将使你无法 *并行* 地做事情，因此会成为一个性能问题。同样，有两件事必须考虑到：

* 如果你是专家，知道自己在做什么（并且已经理解了本节内容），就可以关闭它。除此以外，如果像异步连续和定时器这样的东西只是Job的话，对大多数用户来说是更直观的。注意：在并行网关处理 OptimisticLockingExceptions 的策略是将网关配置为使用异步连续。这样，Job执行器可以用来重试网关，直到异常解决。
* 这实际上不是一个性能问题。性能是重载下的一个问题。重载意味着Job执行器的所有Job线程都一直在忙碌。有了排他性Job，引擎会简单地以不同方式分配负载。独占Job意味着来自单个流程实例的Job由同一个线程按顺序执行。但是考虑到：你有不止一个单一的流程实例。来自其他流程实例的Job被委托给其他线程，并同时执行。这意味着，对于排他性Job，引擎不会并发地执行来自同一流程实例的Job，但它仍然会并发地执行多个实例。从整体吞吐量的角度来看，这在大多数情况下是可取的，因为它通常会让单个实例更快完成。


# Job执行器和多个流程引擎

在使用单一的、嵌入式应用程序的流程引擎的情况下，Job执行器的设置如下。

{{< img src="../img/job-executor-single-engine.png" title="Single Engine" >}}

有一个单一的Job表，引擎向其中添加Job，而Job获取操作则从中消耗Job。因此，创建第二个嵌入式引擎将创建另一个获取线程和执行线程池。

然而，在更大的部署中，这很快就会导致一个难以管理的局面。当在Tomcat或应用服务器上运行Camunda平台时，该平台允许声明多个流程引擎由多个流程应用程序共享。在Job执行方面，一个Job获取可以服务于多个Job表（以及它们的流程引擎），并且可以使用一个用于执行的线程池。

{{< img src="../img/job-executor-multiple-engines.png" title="Multiple Engines" >}}

**这种设置可以集中监控Job的获取与执行**
关于线程池在不同平台上的实现方式，请参见[运行时容器集成]({{< ref "/user-guide/runtime-container-integration/_index.md" >}}）章节的特定平台信息。

不同的Job获取也可以进行不同的配置，例如，为了满足像SLA这样的业务需求。当没有更多的可执行Job出现时，每个Job获取的超时可以被配置成不同的。

流程引擎被分配到哪个Job获取中，可以在引擎的声明中指定，因此可以在流程应用程序的 `processes.xml` 部署描述符或Camunda平台描述符中指定。下面是一个配置的例子，它声明了一个新的引擎，并将其分配给平台启动时创建的名为 "default" 的Job获取。

```xml
<process-engine name="newEngine">
  <job-acquisition>default</job-acquisition>
  ...
</process-engine>
```

Job的获取必须在Camunda平台的部署描述符中声明，详情见[容器特定的配置选项]({{< ref "/user-guide/runtime-container-integration/_index.md" >}})。


# 集群设置

在集群中运行Camunda平台时，有 *同构* 和 *异构* 的设置区分。我们将集群定义为一组网络节点，它们都针对同一个数据库运行Camunda平台（至少每个节点上有一个引擎）。在 *同构* 的情况下，相同的程序应用（以及像JavaDelegates这样的自定义类）被部署到所有的节点上，如下图所示：

{{< img src="../img/homogeneous-cluster.png" title="Homogeneous Cluster" >}}

在 *异构* 的情况下，有些是没有部署的，这意味着一些流程应用只被部署到一部分节点上：

{{< img src="../img/heterogeneous-cluster.png" title="Heterogenous Cluster" >}}


## 在异构集群中执行Job

如上所述的异构集群设置给Job执行器带来了额外的挑战。两个平台都声明使用相同的引擎，也就是说，它们针对相同的数据库运行。这意味着Job将被插入到同一个表中。然而，在默认配置中，节点1的Job获取线程将锁定该表的任何可执行Job，并将其提交给本地Job执行池。这意味着在流程应用B的上下文中创建的Job（所以在节点2上）可能会在节点1上执行，反之亦然。由于Job的执行可能涉及到属于B的部署的类，你可能会看到 "ClassNotFoundExeception" 或任何类似的异常。

为了防止节点1上的Job获取选择 *属于* 节点2的Job，可以通过在流程引擎配置中设置以下属性，将流程引擎配置为 *部署感知（deployment aware）* 。

```xml
<process-engine name="default">
  ...
  <properties>
    <property name="jobExecutorDeploymentAware">true</property>
    ...
  </properties>
</process-engine>
```

现在，节点1上的Job获取线程将只拾取属于该节点上的部署的Job，这就解决了问题。再深入一点，采集器将只采集那些属于在它所服务的引擎上 *注册过* 的部署的Job。每个部署都会被自动注册。此外，人们可以通过使用 `ManagementService` 方法 `registerDeploymentForJobExecutor(deploymentId)` 和`unregisterDeploymentForJobExecutor(deploymentId)` 明确地在引擎上注册和取消注册单个部署。它还提供了 `getRegisteredDeployments()` 方法来检查当前注册的部署。

由于这在引擎层面上是可配置的，你也可以在一个 *混合* 的设置中工作，当一些部署在所有节点之间共享，而一些不共享。你可以把全局共享的程序应用分配给一个不具备部署感知的引擎，而把其他程序分配给一个具备部署感知的引擎，可能两者都是针对同一个数据库运行。这样，在共享流程应用程序的上下文中创建的Job将在任何集群节点上执行，而其他Job只在其各自的节点上执行。
