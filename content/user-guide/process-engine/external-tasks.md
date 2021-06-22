---

title: '外部任务'
weight: 98

menu:
  main:
    identifier: "user-guide-process-engine-external-tasks"
    parent: "user-guide-process-engine"

---

流程引擎支持两种执行服务任务的方式。

1. 内部服务任务：同步调用与进程应用程序一起部署的代码
2. 外部任务：在一个可以被工作者轮询的列表中提供一个工作单元

当代码被实现为[委托代码]({{< ref "/user-guide/process-engine/delegation-code.md" >}})或 [脚本]({{< ref "/user-guide/process-engine/scripting.md" >}})时，会使用第一个内部服务任务。外部（服务）任务的工作方式是：进程引擎将一个工作单元发布给外部工作者去完成。我们把这称为 *外部任务模式* 。

请注意，上面的区别并没有说实际的 "业务逻辑" 是在本地实现还是作为远程服务实现。内部服务任务调用的 Java代理类 也可以自己实现业务逻辑，也可以调用 Web/rest 服务，向另一个系统发送消息等等。对于外部工作者也是如此。外部工作者可以直接实现业务逻辑或再次委托给远程服务实现。

# 外部任务模式

执行外部任务的流程可以分为三个步骤，如下图所示：

{{< img src="../img/external-task-pattern.png" title="External Task Pattern" >}}

1. **流程引擎（Process Engine）**: 创建外部任务实例
2. **外部工作者（External Worker）**: 获取和锁定外部任务
3. **外部工作者（External Worker）和 流程引擎（Process Engine）**: 完整的外部任务实例

当流程引擎遇到一个被配置为外部处理的服务任务时，它会创建一个外部任务实例并将其添加到外部任务列表中（步骤1）。该任务实例接收一个主题（topic）*，该主题确定了要执行的工作的性质。在未来的某个时间，一个外部工作者可以为一组特定的主题获取并锁定任务（步骤2）。为了防止一个任务被多个工作者同时获取，一个任务有一个基于时间戳的锁，这个锁在任务被获取时被设置。只有当锁过期时，另一个工作者才能再次获取该任务。当外部工作者完成了所需的工作，它可以向进程引擎发出信号，然后流程引擎继续执行流程（步骤3）。

{{< note class="info" title="类比 **用户任务**" >}}
外部任务在概念上与用户任务非常相似。在第一次尝试理解外部任务模式时，将其与用户任务进行类比思考可能会有所帮助。
用户任务由流程引擎创建并添加到任务列表中。然后，流程引擎等待人类用户查询该列表，提出任务要求，然后完成它。外部任务是类似的。一个外部任务被创建，然后被添加到一个主题。然后，一个外部应用程序查询该主题并锁定该任务。在任务被锁定后，外部应用程序可以完成它。
{{< /note >}}

这种模式的本质是，执行实际工作的程序独立于流程引擎，并通过轮询流程引擎的API的方式接收工作项。这样做有以下好处：

* **解耦系统**: 外部工作者与流程引擎不需要在同一Java进程中、同一机器上、同一集群中或甚至在同一大陆上运行。只需要它能够访问进程引擎的API（通过REST或Java）。由于采用了轮询模式，外部工作者不需要暴露任何接口供进程引擎访问。
* **解耦技术选择**: 外部工作者不需要用Java实现。相反，可以使用任何最适合执行工作项的技术，并且可以用来访问流程引擎的API（通过REST或Java）。
* **外部工作者可以专注某个主题**: 外部工作者不需要是一个通用的应用程序。每个外部任务实例都会收到要执行的任务性质的主题名。外部工作者可以只轮询它们可以做到的任务主题。
* **细粒度扩展**: 如果某主题服务任务具有较高的负载，相应主题的外部工作者的数量可以独立于进程引擎来扩展。
* **独立维护**: 工作者可以独立于流程引擎进行维护，而不会破坏操作。例如，如果有某个特定主题的工作者停机一段时间（例如，由于更新停机），对进程引擎没有直接影响。这类工作者的外部任务的执行会优雅的存储在外部任务列表中，直到外部工作者恢复并运行。

# 使用外部任务

为了使用外部任务，它们必须在BPMN XML中声明。在运行时，可以通过Java和REST API访问外部任务实例。下面将解释API的概念，并重点介绍Java API。通常情况下，REST API在这种情况下更适合，特别是在不同地域，使用不同技术的外部工作者。

## BPMN

在流程定义的BPMN XML中，可以通过使用属性`camunda:type`和`camunda:topic`来声明服务任务由外部工作者执行。例如，服务任务 *Validate Address* 可以被配置为主题 `AddressValidation` ，如下所示：

```xml
<serviceTask id="validateAddressTask"
  name="Validate Address"
  camunda:type="external"
  camunda:topic="AddressValidation" />
```

也可以使用[表达式]({{< ref "/user-guide/process-engine/expression-language.md" >}})而不是固定值定义主题的名字。

此外，其他类似 *服务任务* 的元素，如发送任务、业务规则任务和抛出消息事件，都可以用外部任务模式来实现。更多细节参见[BPMN 2.0实现参考]({{< ref "/reference/bpmn20/_index.md" >}})

### 错误事件定义

外部任务允许定义错误事件，抛出一个指定的BPMN错误。这可以通过在任务定义中添加一个扩展元素 [camunda:errorEventDefinition]({{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#erroreventdefinition" >}})来完成。 与`bpmn:errorEventDefinition`相比，`camunda:errorEventDefinition`元素可以接受一个额外的`expression`属性，支持任何JUEL表达。在表达式内，你可以访问 {{< javadocref page="?org/camunda/bpm/engine/externaltask/ExternalTask.html" text="外部任务实体" >}} 对象，使用key "externalTask" 通过getter方法访问 "errorMessage"、"errorDetails"、"workerId"、"retries"。

该表达式在调用`ExternalTaskService#complete`和`ExternalTaskService#handleFailure`时被计算。
外部任务服务 `#handleFailure` 在调用时进行评估。如果表达式评估为 `true`，实际的方法执行将被取消，并被抛出相应的BPMN错误。这个错误可以被[错误边界事件]({{< ref "/reference/bpmn20/events/error-events.md#error-boundary-event" >}})所捕获。这意味着错误事件定义在成功和失败的情况下同样使用--即使任务成功完成，你仍然可以决定抛出一个BPMN错误。

```xml
<serviceTask id="validateAddressTask"
  name="Validate Address"
  camunda:type="external"
  camunda:topic="AddressValidation" >
  <extensionElements>
    <camunda:errorEventDefinition id="addressErrorDefinition" 
      errorRef="addressError" 
      expression="${externalTask.getErrorDetails().contains('address error found')}" />
  </extensionElements>
</serviceTask>
```

关于外部任务的错误事件定义的进一步信息可以在[表达式语言用户指南]({{< ref "/user-guide/process-engine/expression-language.md#external-task-error-handling" >}})中找到。在RPA协调场景的外部任务中的具体使用方法在[Camunda平台 RPA Bridge]({{< ref "/user-guide/camunda-bpm-rpa-bridge.md#error-handling" >}})。

## Rest API

关于如何通过HTTP访问API操作，请参见[REST API文档]({{< ref "/reference/rest/external-task/_index.md" >}})。

### 长轮询以获取并锁定外部任务

普通的HTTP请求会立即得到服务器的响应，无论所请求的信息是否可用。这不可避免地导致了这样一种情况：客户端必须执行多次重复的请求，直到信息可用（轮询）。这显然是十分消耗资源的。

{{< img src="../img/external-task-long-polling.png" alt="Long polling to fetch and lock external tasks" >}}

在长时间轮询的帮助下，如果没有外部任务，服务器会暂停请求。一旦有新的外部任务出现，请求就会重新被激活，并执行响应。暂停的时间可以通过 timeout 配置。

长轮询大大减少了请求的数量，使服务器和客户端都能更有效地利用资源。

参见 [REST API文档]({{< ref "/reference/rest/external-task/fetch.md" >}}).

{{< note title="小心!" class="info" >}}
该功能基于 JAX-RS 2.0，因此在 **IBM WebSphere Application Server 8.5** 上不可用。
{{< /note >}}

#### 唯一工作者请求（Unique Worker Request）
默认情况下，多个工作者可以使用同一个`workerId`。为了确保服务器端的 `workerId` 的唯一性，可以启用 `Unique Worker Request` 标志。这个配置标志只影响长轮询请求，而不是普通的 "获取并锁定" 请求。如果 "Unique Worker Request" 标志被启用，当收到一个新的请求时，具有相同 `workerId` 的请求会被取消。

为了启用 "Unique Worker Request" 标志，需要调整 *engine-rest* 组件中包含的 `engine-rest/WEB-INF/web.xml` 文件，将上下文参数`fetch-and-lock-unique-worker-request` 设置为 `true`。请考虑下面的配置片段：

```xml
<!-- ... -->

<context-param>
  <param-name>fetch-and-lock-unique-worker-request</param-name>
  <param-value>true</param-value>
</context-param>

<!-- ... -->
```

## Java API

外部任务的Java API的入口是 "ExternalTaskService"。它可以通过`processEngine.getExternalTaskService()`获取到。

下面是一个用于交互的例子，它获取了10个任务，在一个循环中处理这些任务，对于每个任务，要么完成任务，要么将其标记为失败：

```java
List<LockedExternalTask> tasks = externalTaskService.fetchAndLock(10, "externalWorkerId")
  .topic("AddressValidation", 60L * 1000L)
  .execute();

for (LockedExternalTask task : tasks) {
  try {
    String topic = task.getTopicName();

    // 业务代码
    ...

    // 如果执行成功，则设置任务成功
    if(success) {
      externalTaskService.complete(task.getId(), variables);
    }
    else {
      // 否则标记任务失败
      externalTaskService.handleFailure(
        task.getId(),
        "externalWorkerId",
        "Address could not be validated: Address database not reachable",
        1, 10L * 60L * 1000L);
    }
  }
  catch(Exception e) {
    //... 处理异常
  }
}
```

下面几节将更详细地讨论与 "外部任务服务" 的不同交互：

### 获取任务

实现轮询工作者，可以通过使用 `ExternalTaskService#fetchAndLock` 方法来执行获取操作。该方法返回一个流式构建器，允许定义一组主题来获取任务。请看下面的代码片段：

```java
List<LockedExternalTask> tasks = externalTaskService.fetchAndLock(10, "externalWorkerId")
  .topic("AddressValidation", 60L * 1000L)
  .topic("ShipmentScheduling", 120L * 1000L)
  .execute();

for (LockedExternalTask task : tasks) {
  String topic = task.getTopicName();

  // 业务代码
  ...
}
```

这段代码首先获取最多10个主题为 "AddressValidation" 和 "ShipmentScheduling" 的任务。获取后任务被锁定，只留给id为`externalWorkerId`的工作者。锁定意味着任务在一定的时间内被保留给这个工作者，从获取的时间开始到锁过期前防止其他工作者获得这个任务。如果锁过期了，同时任务还没有完成，那么另一个工作器则可以获取它，这样运行失败的工作器就不会无限期地阻塞执行。
确切的锁定持续时间在主题获取指令中给出 "AddressValidation" 的任务被锁定60秒（`60L * 1000L`毫秒），而 "ShipmentScheduling" 的任务被锁定120秒（`120L * 1000L`毫秒）。锁定到期时间不应短于预期执行时间。但它也不应该太高。

在获取任务时也可以获取执行任务所需的变量。例如，假设`AddressValidation`任务需要一个`address`变量。可以这样做：

```java
List<LockedExternalTask> tasks = externalTaskService.fetchAndLock(10, "externalWorkerId")
  .topic("AddressValidation", 60L * 1000L).variables("address")
  .execute();

for (LockedExternalTask task : tasks) {
  String topic = task.getTopicName();
  String address = (String) task.getVariables().get("address");

  // 业务代码
  ...
}
```

之后，产生的任务就包含了所请求的变量。注意，变量值是外部任务执行时在作用域中的。详情参见[变量作用域和变量可见性]({{< ref "/user-guide/process-engine/variables.md#variable-scopes-and-variable-visibility" >}})一章。

如果想要获取所有变量，不调用 `variables` 方法即可。

```java
List<LockedExternalTask> tasks = externalTaskService.fetchAndLock(10, "externalWorkerId")
  .topic("AddressValidation", 60L * 1000L)
  .execute();

for (LockedExternalTask task : tasks) {
  String topic = task.getTopicName();
  String address = (String) task.getVariables().get("address");

  // 业务代码
  ...
}
```

为了启用序列化变量值的反序列化（通常是存储自定义Java对象的变量），必须调用`enableCustomObjectDeserialization()`。否则，一旦从变量映射中检索到序列化的变量，就会抛出一个异常，即该对象没有被反序列化。

```java
List<LockedExternalTask> tasks = externalTaskService.fetchAndLock(10, "externalWorkerId")
  .topic("AddressValidation", 60L * 1000L)
  .variables("address")
  .enableCustomObjectDeserialization()
  .execute();

for (LockedExternalTask task : tasks) {
  String topic = task.getTopicName();
  MyAddressClass address = (MyAddressClass) task.getVariables().get("address");

  // 业务代码
  ...
}
```
 

### 外部任务的优先顺序
外部任务的优先级与Job的优先级相似。也存在同样的饥饿问题（一种锁的问题）需要考虑。
进一步的细节，见[Job优先级]({{< ref "/user-guide/process-engine/the-job-executor.md#the-job-priority" >}})一节。

### 外部任务的配置

本节解释了如何在配置中启用和禁用外部任务优先级。有两个相关的配置属性可以在流程引擎配置文件上配置。

`producePrioritizedExternalTasks`: 控制流程引擎是否为外部任务分配优先级。默认值是 `true`。
如果不需要优先级，流程引擎配置属性`producePrioritizedExternalTasks`可以设置为`false`。在这种情况下，所有外部任务的优先级都是0。
关于如何指定外部任务的优先级以及流程引擎如何分配这些优先级的细节，参见后面关于[指定外部任务优先级]({{< relref "#specify-external-task-priorities" >}})的部分。

### 指定外部任务的优先顺序

外部任务的优先级可以在BPMN模型中指定，也可以在运行时通过API重写。

#### BPMN XML 中指定优先级

外部任务的优先级可以在流程或活动级别分配。使用Camunda扩展属性`camunda:taskPriority`。

指定优先级使用常量值和[表达式]({{< ref "/user-guide/process-engine/expression-language.md" >}})都可以。
当使用常量值时，该流程或活动的所有实例都将使用相同的优先级。
表达式允许给流程或活动的每个实例分配不同的优先级。表达式返回类型必须为 Java `long` 范围内的一个数字。
具体数值可以是复杂计算的结果，并基于用户提供的数据（来自任务表格或其他来源）。

#### 在流程级别指定优先级

在流程实例级别配置外部任务优先级时，需要在bpmn的 `<process ...>` 元素中设置 `camunda:taskPriority` 属性


```xml
<bpmn:process id="Process_1" isExecutable="true" camunda:taskPriority="8">
  ...
</bpmn:process>
```

其效果是，流程内的所有外部任务都继承相同的优先级（除非它被本地重写）。
上面的例子显示了如何使用一个常量值来设置优先级。这样一来，相同的优先级就会应用于流程的所有实例。
如果不同的流程实例需要以不同的优先级执行，可以使用一个表达式：

```xml
<bpmn:process id="Process_1" isExecutable="true" camunda:taskPriority="${order.priority}">
  ...
</bpmn:process>
```

在上面的例子中，优先级是根据变量`order`的属性`priority`决定的。

#### 在服务任务级别指定优先级

在服务任务层面配置外部任务优先级时，需要将`camunda:taskPriority`属性应用于bpmn`<serviceTask ...>`元素。
服务任务必须是一个外部任务，属性为`camunda:type="external"`。

```xml
  ...
  <serviceTask id="externalTaskWithPrio" 
               camunda:type="external" 
			   camunda:topic="externalTaskTopic" 
			   camunda:taskPriority="8"/>
  ...
```

其效果是为定义的外部任务设置优先级（覆盖流程的taskPriority）。
上面的例子展示了如何使用一个常量值设置优先级。这样，在流程的不同实例中，所有外部任务都将使用相同的优先级。
如果不同的进程实例需要以不同的外部任务优先级来执行，可以使用一个表达式。

```xml
  ...
  <serviceTask id="externalTaskWithPrio" 
               camunda:type="external" 
			   camunda:topic="externalTaskTopic" 
			   camunda:taskPriority="${order.priority}"/>
  ...
```

在上面的例子中，优先级是根据变量`order`的属性`priority`决定的。



### 根据优先级查询外部任务

为了根据优先级查询外部任务，可以使用`ExternalTaskService#fetchAndLock`带有参数 "usePriority" 的重载方法。没有布尔参数的方法可以任意地返回外部任务。如果给了参数，返回的外部任务将按优先级降序排列。
请看下面的例子，它使用了外部任务的优先级查询：

```java
List<LockedExternalTask> tasks =
  externalTaskService.fetchAndLock(10, "externalWorkerId", true)
  .topic("AddressValidation", 60L * 1000L)
  .topic("ShipmentScheduling", 120L * 1000L)
  .execute();

for (LockedExternalTask task : tasks) {
  String topic = task.getTopicName();

  // 业务代码
  ...
}
```


### 完成任务

在获取并执行请求的工作后，工作者可以通过调用 "ExternalTaskService#complete" 方法完成外部任务。外部工作者只能完成它之前获取并锁定的任务。如果该任务在此期间被其他的工作器锁定，就会出现异常。

{{< note class="info" title="错误事件" >}}
外部任务可以包括[错误事件定义]({{< ref "/user-guide/process-engine/external-tasks.md#error-event-definitions" >}}) 在错误事件的表达式评估为 `true`的情况下，可以取消`#complete`的执行。如果错误事件的表达式评估产生了一个异常，对 `#complete` 的调用也会因为这个异常而失败。
{{< /note >}}

### 延长锁定时间

当外部任务被工作者锁定时，可以通过调用`ExternalTaskService#extendLock`方法来延长锁定时间。工作者可以指定更新超时的时间量（以毫秒为单位）。一个锁只能由拥有该锁的外部任务来延长。

### 报告任务执行失败

外部工作者可能并不总能够成功地完成任务。在这种情况下，它可以通过使用`ExternalTaskService#handleFailure`向流程引擎报告失败。与 "#complete" 一样，"#handleFailure" 只能由拥有最新任务锁的工作者调用。`#handleFailure`方法需要四个参数。 `错误信息（errorMessage）`，`错误细节（errorDetails）`，`重试（retries）`，`重试时间（retryTimeout）`。错误信息（errorMessage）可以包含对问题性质的描述，限制在666个字符。它可以在任务再次被获取或被查询时被访问。错误细节（errorDetails）可以包含完整的误差描述，长度不限。错误细节可以通过单独的方法`ExternalTaskService#getExternalTaskErrorDetails`访问，基于任务id参数。通过参数`重试（retries）`和`重试时间（retryTimeout）`，外部工作者可以指定一个重试策略。当设置`retries`的值大于0时，任务可以在`retryTimeout`到期后被再次获取。当设置retries为0时，任务不能再被提取，并为该任务创建一个事件。

参考下面的代码：

```java
List<LockedExternalTask> tasks = externalTaskService.fetchAndLock(10, "externalWorkerId")
  .topic("AddressValidation", 60L * 1000L).variables("address")
  .execute();

LockedExternalTask task = tasks.get(0);

// ... 处理任务失败

externalTaskService.handleFailure(
  task.getId(),
  "externalWorkerId",
  "Address could not be validated: Address database not reachable",     // errorMessage
  "Super long error details",                                           // errorDetails
  1,                                                                    // retries
  10L * 60L * 1000L);                                                   // retryTimeout

// ... 其他业务代码

externalTaskService.getExternalTaskErrorDetails(task.getId());
```

通过以上代码，任务会被报告为失败，这样它就可以在10分钟后再重试一次。流程引擎不会自己递减重试（参数retries）。可以通过在报告失败时将重试设置为`task.getRetries() - 1`来实现。

如果需要错误细节信息，可以使用单独的方法查询到。

{{< note class="info" title="错误事件" >}}
外部任务可以包括[错误事件定义]({{< ref "/user-guide/process-engine/external-tasks.md#error-event-definitions" >}}) 如果错误事件的表达式返回结果为 "true"，可以取消 "#handleFailure"的执行。如果错误事件的表达式执行引发异常，该表达式将被视为 "false"。
{{< /note >}}

### 报告BPMN错误

见[错误边界事件]({{< ref "/reference/bpmn20/events/error-events.md#error-boundary-event" >}})的一章。

由于某些原因，执行过程中会出现业务错误。在这种情况下，外部工作者可以通过使用`ExternalTaskService#handleBpmnError`向流程引擎报告一个BPMN错误。
与 "#complete"或 "#handleFailure"一样，它只能由拥有最新任务锁的工作者调用。
`#handleBpmnError`方法需要一个额外的参数：`errorCode`。
该错误代码确定了一个预定义的错误。如果给定的`errorCode`不存在或者没有定义边界事件。
当前的活动实例就会结束，错误不会被处理。

请看下面的例子：

```java
List<LockedExternalTask> tasks = externalTaskService.fetchAndLock(10, "externalWorkerId")
  .topic("AddressValidation", 60L * 1000L).variables("address")
  .execute();

LockedExternalTask task = tasks.get(0);

// ... 出现业务错误

externalTaskService.handleBpmnError(
  task.getId(),
  "externalWorkerId",
  "bpmn-error", // errorCode
  "Thrown BPMN Error during...", // errorMessage
  variables);
```

然后，一个带有错误代码`bpmn-error`的BPMN错误被传播。如果存在具有该错误代码的边界事件，BPMN错误将被捕获和处理。
错误信息和变量是可选的。它们可以为错误提供额外的信息。如果BPMN错误被捕获，这些变量将被传递给执行。

### 查询任务

可以通过`ExternalTaskService#createExternalTaskQuery`对外部任务进行查询。与`#fetchAndLock`相反，这是一个不设置任何锁的读取查询。

### 管理业务

其他的管理操作有`ExternalTaskService#unlock`、`ExternalTaskService#setRetries`和`ExternalTaskService#setPriority`来清除当前的锁，设置重试以及设置外部任务的优先级。
当一个任务的重试（retries ）为0，且必须手动恢复时，作为最后的手段，设置重试会很有用。优先级也可以设置，对于更重要的外部任务可以设置为较高的值，对于不那么重要的外部任务可以设置为较低的值。

还有操作`ExternalTaskService#setRetriesSync`和`ExternalTaskService#setRetriesAsync`可以同步或异步为多个外部任务设置重试。
