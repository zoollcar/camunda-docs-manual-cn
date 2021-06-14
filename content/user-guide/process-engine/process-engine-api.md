---

title: '流程引擎 API'
weight: 20

menu:
  main:
    identifier: "user-guide-process-engine-api"
    parent: "user-guide-process-engine"

---


# 服务API

Java API是与引擎互动的最常见方式。中心起点是ProcessEngine，它可以通过几种方式创建，如配置部分所述。从ProcessEngine中，你可以获得包含工作流/BPM方法的各种服务。ProcessEngine和服务对象是线程安全的。所以你可以为整个服务器保留对其中一个对象的引用。

{{< img src="../img/api.services.png" title="API Services" >}}

```java
ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();

RepositoryService repositoryService = processEngine.getRepositoryService();
RuntimeService runtimeService = processEngine.getRuntimeService();
TaskService taskService = processEngine.getTaskService();
IdentityService identityService = processEngine.getIdentityService();
FormService formService = processEngine.getFormService();
HistoryService historyService = processEngine.getHistoryService();
ManagementService managementService = processEngine.getManagementService();
FilterService filterService = processEngine.getFilterService();
ExternalTaskService externalTaskService = processEngine.getExternalTaskService();
CaseService caseService = processEngine.getCaseService();
DecisionService decisionService = processEngine.getDecisionService();
```

`ProcessEngines.getDefaultProcessEngine()` 将在第一次调用时初始化并建立一个进程引擎，此后总是返回相同的进程引擎。所有进程引擎的正确创建和关闭可以通过`ProcessEngines.init()`和`ProcessEngines.destroy()`完成。

ProcessEngines类将扫描所有`camunda.cfg.xml`和`activiti.cfg.xml`文件。

对于所有`camunda.cfg.xml`文件，进程引擎将以典型的方式建立。

```java
ProcessEngineConfiguration
  .createProcessEngineConfigurationFromInputStream(inputStream)
  .buildProcessEngine()
```

对于所有的`activiti.cfg.xml`文件，进程引擎将以Spring的方式构建：首先创建Spring应用上下文，然后从该应用上下文中获得进程引擎。

所有的服务都是无状态的。这意味着你可以很容易地在一个集群的多个节点上运行Camunda平台，每个节点都去同一个数据库，而不必担心哪个机器实际执行了以前的调用。对任何服务的任何调用都是无状态的，无论它在哪里执行。

当使用Camunda引擎时，**仓库服务（RepositoryService）**可能是第一个需要的服务。这个服务提供了管理和操纵部署和流程定义的操作。流程定义是BPMN 2.0流程的Java对应物，这里就不多说了。它是一个流程的每个步骤的结构和行为的表示。部署是引擎中的包装单元。一个部署可以包含多个BPMN 2.0 XML文件和任何其他资源。一个部署中包含的内容的选择由开发者决定。它的范围可以从一个单一的流程BPMN 2.0 XML文件到整个流程和相关资源包（例如，部署 "hr-processes "可以包含与hr流程相关的一切内容）。RepositoryService允许部署这种包。进行一次部署意味着部署内容被上传到引擎，在那里所有的流程都被检查和解析，然后被存储在数据库中。从那时起，系统就知道该部署了，并且任何包含在部署中的进程现在都可以被启动。

此外，这项仓库服务（RepositoryService）允许：

*  查询引擎已知的部署和流程定义。
*  失效和激活流程定义。失效意味着不能对它们做进一步的操作，而激活则是相反的操作。
*  检索各种资源，如部署中包含的文件或由引擎自动生成的流程图。

RepositoryService是关于静态信息的（即，不改变的数据，或者至少不会改变很多的），而**RuntimeService**则完全相反。它处理的是启动流程定义的新流程实例。如上所述，一个流程定义定义了流程中不同步骤的结构和行为。一个流程实例是这样流程定义的一次执行。对于每个流程定义，通常有许多实例在同时运行。RuntimeService也是用于检索和存储[流程变量]({{< ref "/user-guide/process-engine/variables.md" >}})的服务。这是某个特定流程实例的数据，可以被流程中的各种构造使用（例如，排他网关经常使用流程变量来确定选择哪条路径继续流程）。RuntimeService还允许对进程实例和执行进行查询。执行（Executions）是BPMN 2.0中'token'的概念。一般来说，执行是一个指针，指向流程实例当前所在的位置。最后，当流程实例在等待外部触发并且流程需要继续时，RuntimeService就会被使用。流程实例可以有各种等待状态，这个服务包含各种操作，以 "通知" 实例已经收到外部触发，流程实例可以继续执行。

需要由系统的实际人类用户执行的任务是流程引擎的核心。围绕任务的一切都被归入**TaskService**，例如：

*  查询分配给用户或组的任务。
*  创建新的独立任务。这些是与流程实例无关的任务。
*  操纵一个任务被分配给哪个用户，或者哪个用户以某种方式参与到任务中。
*  声称并完成一项任务。声称意味着有人决定成为该任务的受让人，意味着这个用户将完成该任务。完成意味着 "完成任务的工作"。一般来说，这就是填写一个类似的表格。

身份服务（**IdentityService**）是非常简单的。它允许对组和用户进行管理（创建、更新、删除、查询...）。重要的是要理解，核心引擎实际上在运行时并不对用户进行任何检查。例如，一个任务可以被分配给任何用户，但引擎并不验证该用户是否为系统所知。这是因为该引擎也可以与LDAP、活动目录等服务结合使用。

表单服务（**FormService**）是一个可选的服务。这意味着没有它，Camunda引擎也可以完美地使用，不会缺少任何功能。这项服务引入了起始表单和任务表单的概念。开始表单是在流程实例启动前显示给用户的表单，而任务表单是当用户想完成一项任务时显示的表单。你可以在BPMN 2.0流程定义中定义这些表单。这个服务以一种简单的方式暴露了这些数据，以便于工作。但同样，这是可选的，因为表单不需要嵌入流程定义中。

历史服务（**HistoryService**）暴露了引擎收集的所有历史数据。当执行流程时，引擎可以保留很多数据（这是可配置的），如流程实例的开始时间、谁做了哪些任务、完成任务花了多长时间、每个流程实例遵循的路径等。该服务主要暴露了访问这些数据的查询功能。

管理服务（**ManagementService**）在编码自定义应用程序时通常不需要。它允许检索关于数据库表和表元数据的信息。此外，它暴露了查询功能和作业的管理操作。作业在引擎中被用于各种事情，如定时器、异步延续、延迟暂停/激活等。稍后，这些主题将被更详细地讨论。

过滤器服务（**FilterService**）允许创建和管理过滤器。过滤器是像任务查询一样的存储查询。例如，过滤器被任务列表用来[过滤用户任务]({{< ref "/webapps/tasklist/filters.md" >}})。

外部任务服务（**ExternalTaskService**）提供对[外部任务实例]({{< relref "external-tasks.md" >}})的访问。外部任务代表在外部处理的工作项目，独立于流程引擎。

案例服务（**CaseService**）与运行时服务（**RuntimeService**）类似，但用于案例实例。它处理启动案例定义的新案例实例并管理案例执行的生命周期。该服务也被用来检索和更新案例实例的过程变量。

决策服务 （**[DecisionService]({{< ref "/user-guide/process-engine/decisions/decision-service.md" >}})**） 允许评估部署在引擎中的决策。它是评估独立于流程定义的业务规则任务中的决策的一种选择。

{{< note title="Java Docs" class="warning" >}}
  关于服务操作和引擎API的更多详细信息，见 {{< javadocref page="" text="Java Docs" >}}.
{{< /note >}}


# 查询 API

要从引擎中查询数据，有多种可能性。

* Java查询API。流利的Java API可以查询引擎实体（如ProcessInstances, Tasks, ...）。
* REST查询API。REST API来查询引擎实体（如ProcessInstances、Tasks...）。
* 本地查询。提供自己的SQL查询，以检索引擎实体（如ProcessInstances，Tasks，...），如果查询API缺乏你需要的可能性（例如，OR条件）。
* 自定义查询。使用完全定制的查询和自己的MyBatis映射来检索自己的价值对象，或将引擎与领域数据连接起来。
* SQL查询。使用数据库的SQL查询，如报告。

推荐的方法是使用其中一个查询API。

Java查询API允许使用流畅的API来编写完全类型安全的查询。你可以在你的查询中添加各种条件（所有这些条件都作为一个逻辑AND一起应用），并精确地进行排序。下面的代码显示了一个例子：

```java
List<Task> tasks = taskService.createTaskQuery()
  .taskAssignee("kermit")
  .processVariableValueEquals("orderId", "0815")
  .orderByDueDate().asc()
  .list();
```

你可以在{{< javadocref page="" text="Java Docs" >}}中找到更多信息。

## 查询最大结果限制

在没有限制最大结果数量的情况下查询结果，或者查询大量的
的结果会导致高内存消耗，甚至出现内存不足的异常。借助于[查询最大结果限制]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#queryMaxResultsLimit" >}})的帮助下，你可以限制最大结果数。

此限制仅在以下情况下执行：

* 经过身份验证的用户执行查询
* 查询API被直接调用，例如通过REST API（在过程中没有通过[delegation code]({{< ref "/user-guide/process-engine/delegation-code.md" >}})强制执行）。


### 禁忌

* 使用<code>#list()</code>方法执行具有不受约束数量的结果的查询
* 执行一个[分页查询](#paginated-queries)，超过了配置的最大结果限制
* 执行基于查询的同步操作，获取的实例数量超过了最大结果的限制。（请使用[批处理操作]({{< ref "/user-guide/process-engine/batch-operations.md" >}})替代）

### 建议

* 使用<code>[Query#unlimitedList][javadocs-query-unlimited-list]</code>方法执行一个查询。
* 执行一个[分页查询](#paginated-queries)，其最大结果数小于或等于最大结果限制
* 执行一个[本地查询](#native-queries)，因为它不能通过REST API或Webapps来访问
 因此不太可能被利用
 
### 限制

* 通过REST API进行统计查询
* 通过Webapps（私有API）执行被调用的实例查询

### 自定义身份服务查询

当你提供...

1. 通过实现 `ReadOnlyIdentityProvider` 或 `WritableIdentityProvider` 接口，实现一个自定义的身份提供者
2. **和**一个专门的身份服务查询的实现（例如：`GroupQuery', `TenantQuery', `UserQuery')

在调用<code>[Query#unlimitedList][javadocs-query-unlimited-list]</code>时，确保不受任何限制地返回所有结果。
检索无限列表的可能性对于确保REST API的正常工作很重要，因为有几个接口依赖于检索无限的结果。

[javadocs-query-unlimited-list]: {{< javadocref_url page="?org/camunda/bpm/engine/query/Query.html#unlimitedList--" >}}

## 分页查询

分页允许配置一个查询所检索的最大结果数以及第一个索引的位置。

请参阅以下示例:
```java
List<Task> tasks = taskService.createTaskQuery()
  .taskAssignee("kermit")
  .processVariableValueEquals("orderId", "0815")
  .orderByDueDate().asc()
  .listPage(20, 50);
```

上面所示的查询检索了50个结果，从索引为20的结果开始。

## 或（OR）查询
查询API的默认行为是将过滤条件与一个AND表达式联系在一起。
OR查询可以建立查询，其中过滤条件与OR表达式联系在一起。

{{< note title="小心!" class="info" >}}
  - 这个功能只适用于任务和流程实例查询（runtime & history）
  - 以下方法不能应用于OR查询：orderBy...(), initializeFormKeys(),
  withCandidateGroups(), withoutCandidateGroups(), withCandidateUsers(), withoutCandidateUsers().
{{< /note >}}

在调用`or()`之后，可以有一个由几个过滤标准组成的链式查询。每个过滤条件都被连接在一起
用一个OR表达式连接起来。调用`endOr()`标志着OR查询的结束。调用这两个方法就相当于
就像把过滤条件放在括号里一样。

```java
List<Task> tasks = taskService.createTaskQuery()
  .taskAssignee("John Munda")
  .or()
    .taskName("Approve Invoice")
    .taskPriority(5)
  .endOr()
  .list();
```
上面的查询检索了所有分配给 "John Munda "的任务，这些任务同时被命名为 "Approve Invoice
或被赋予5级优先权(`assignee = "John Munda" AND (name = "Approve Invoice" OR priority = 5)`, <a href="https://en.wikipedia.org/wiki/Conjunctive_normal_form">Conjunctive Normal Form</a>).

在内部，该查询被翻译成如下SQL查询（稍微简化）：

```sql
SELECT DISTINCT *
FROM   act_ru_task RES
WHERE  RES.assignee_ = 'John Munda'
       AND ( Upper(RES.name_) = Upper('Approve Invoice')
             OR RES.priority_ = 5 );
```

一次可以使用任意数量的OR查询。当建立一个查询时，不仅包括一个单一的OR查询，还包括用AND表达式连接起来的过滤标准，OR查询被附加到标准链上作为一个AND表达式。一个与变量有关的过滤条件可以在同一个OR查询中多次应用。

```java
List<Task> tasks = taskService.createTaskQuery()
  .or()
    .processVariableValueEquals("orderId", "0815")
    .processVariableValueEquals("orderId", "4711")
    .processVariableValueEquals("orderId", "4712")
  .endOr()
  .list();
```

除了与变量有关的过滤标准外，这种行为是不同的。 每当一个**non-variable-filter-criterion**在查询中被使用多次时，只有最后应用的值才会被使用。

```java
List<Task> tasks = taskService.createTaskQuery()
  .or()
    .taskCandidateGroup("sales")
    .taskCandidateGroup("controlling")
  .endOr()
  .list();
```
{{< note title="小心!" class="info" >}}
在上面的查询中，过滤标准`taskCandidateGroup`的值 "sales"被替换为
"controlling"。为了避免这种行为，可以使用带有...**In**结尾的过滤条件，例如：

* taskCandidateGroup**In**()
* tenantId**In**()
* processDefinitionKey**In**()
{{< /note >}}

## REST Query API

java查询api也被暴露为REST服务，请参阅[REST文档]({{< ref "/reference/rest/_index.md" >}}) 。


## 本机查询

有时你需要更强大的查询，例如，使用OR运算符的查询或你无法使用查询API表达的限制。对于这些情况，我们引入了本地查询，它允许你编写你自己的SQL查询。返回类型由你使用的查询对象定义，数据被映射到正确的对象中，例如，任务、ProcessInstance、Execution等。由于查询将在数据库中进行，你必须使用[数据库模式]({{< ref "/user-guide/process-engine/database/database-schema.md" >}})中定义的表和列名。这需要一些关于内部数据结构的知识，建议谨慎使用本地查询。表名可以通过API检索，以保持尽可能小的依赖性。

```java
List<Task> tasks = taskService.createNativeTaskQuery()
  .sql("SELECT count(*) FROM " + managementService.getTableName(Task.class) + " T WHERE T.NAME_ = #{taskName}")
  .parameter("taskName", "aOpenTask")
  .list();

long count = taskService.createNativeTaskQuery()
  .sql("SELECT count(*) FROM " + managementService.getTableName(Task.class) + " T1, "
         + managementService.getTableName(VariableInstanceEntity.class) + " V1 WHERE V1.TASK_ID_ = T1.ID_")
  .count();
```


## 自定义查询

由于性能的原因，有时可能不希望查询引擎对象，而是查询一些自己的值或DTO对象，从不同的表中收集数据--也许包括你自己的类。

{{< note title="教程" class="info" >}}
  [使用自定义查询进行性能优化](http://blog.camunda.org/post/2017/12/custom-queries)。
{{< /note >}}


## SQL Queries

[表结构]({{< ref "/user-guide/process-engine/database/database-schema.md" >}})是非常直接的--我们专注于使其易于理解。因此，在报告等用例中进行SQL查询是可以的。只需确保你不会在不清楚自己在做什么的情况下，通过更新表来弄乱引擎的数据。
