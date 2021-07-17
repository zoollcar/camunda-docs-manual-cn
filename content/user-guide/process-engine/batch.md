---

title: '批处理'
weight: 270

menu:
  main:
    identifier: "user-guide-process-engine-batch"
    parent: "user-guide-process-engine"

---

批处理的概念是将当前执行的工作卸载到后台的过程。这允许执行大量的异步流程引擎命令而不会阻塞实例。它还对单独命令调用进行了分离。

例如，[流程实例迁移][migration] 命令可以[使用批处理执行][batch-migration]。 这样可以异步迁移流程实例。 在同步流程实例迁移中，所有迁移都在单个事务中执行。 首先，这需要成功提交事务。 对于大量流程实例，事务可能变得太大，甚至无法提交到数据库。 随着使用批处理迁移，这情况发生了变化。 批处理以较小的块执行迁移，每个块使用单个事务。

优点：

- 异步（非阻塞）执行
- 可以使用多线程和Job执行器处理执行
- 执行解耦，即每个批处理执行Job使用自己的事务

缺点：

- 手动轮询以完成批处理
- 与流程引擎执行的其他Job的争用
- 当部分子集已经执行时，批处理可能会出现部分失败的情况，例如，某些流程实例被迁移而其他流程实例迁移失败

从技术上讲，批处理表示在流程引擎的上下文中执行命令的一组Job。

批处理利用流程引擎的[Job执行器][job executor]来执行批处理Job。单个批处理包含三种Job类型：

- 种子Job（Seed job）：创建所有批处理Job的Job
- 执行Job（Execution jobs）：批处理命令的实际执行，例如流程实例迁移
- 监控Job（Monitor job）：种子Job完成后，监控批处理执行和完成的进度

# API

下面概述了用于批处理的 Java API。

## 创建一个批处理

批处理是通过异步执行流程引擎命令创建的。

当前支持的批处理类型：

- [流程实例迁移][batch-migration]
- [取消运行的流程实例][process-instance-cancellation]
- [删除历史流程实例][process-instance-deletion]
- [修改流程实例暂停状态][process-instance-suspend]
- [设置与流程实例关联的Job的重试][set-job-retries]
- [为流程实例设置变量][set-variables]
- [流程实例修改][process-instance-modification]
- [流程实例重启][process-instance-restart]
- [设置外部任务的重试][set-external-tasks-retries]
- [为历史流程实例设置删除时间][set-removal-time]
- [为历史决策实例设置删除时间][set-removal-time]
- [为历史批处理设置删除时间][set-removal-time]

Java API 可用于创建批处理命令，具体使用示例请参考具体命令。

## 查询批处理

你可以通过 id 和类型查询正在运行的批处理，例如查询所有正在运行的流程实例迁移批处理：

```java
List<Batch> migrationBatches = processEngine.getManagementService()
  .createBatchQuery()
  .type(Batch.TYPE_PROCESS_INSTANCE_MIGRATION)
  .list();
```

## 批处理统计数据

你可以通过管理服务（ManagementService）查询批处理统计信息。
批处理统计信息将包含有关剩余的、已完成的和失败的批处理执行Job的信息。

```java
List<BatchStatistics> migrationBatches = processEngine.getManagementService()
  .createBatchStatisticsQuery()
  .type(Batch.TYPE_PROCESS_INSTANCE_MIGRATION)
  .list();
```

## 批处理的历史

当[历史等级][history level] 为 `FULL` 时，会创建一个历史批处理条目。你可以使用历史服务（HistoryService）查询它。

```java
HistoricBatch historicBatch = processEngine.getHistoryService()
  .createHistoricBatchQuery()
  .batchId(batch.getId())
  .singleResult();
```

历史记录还包含种子、监控和执行Job的Job日志条目。 你可以通过某个Job定义 id 查询相应的Job日志条目。

```java
HistoricBatch historicBatch = ...

List<HistoricJobLog> batchExecutionJobLogs = processEngine.getHistoryService()
  .createHistoricJobLogQuery()
  .jobDefinitionId(historicBatch.getBatchJobDefinitionId())
  .orderByTimestamp()
  .list();
```

你可以设置对已完成的历史批处理操作的[历史清理][history cleanup]配置。

## 暂停批处理

要暂停批处理和所有相应Job的执行，可以使用管理服务。

```java
processEngine.getManagementService()
  .suspendBatchById("myBatch");
```

然后可以再次激活暂停的批处理，同样使用管理服务。

```java
processEngine.getManagementService()
  .activateBatchById("myBatch");
```

## 删除批处理

可以使用管理服务删除正在运行的批处理。

```java
// 删除批处理，保留批处理的历史记录
processEngine.getManagementService()
  .deleteBatch("myBatch", false);

// 删除批处理包括批处理的历史记录
processEngine.getManagementService()
  .deleteBatch("myBatch", true);
```

可以使用历史服务删除历史批处理。

```java
processEngine.getHistoryService()
  .deleteHistoricBatch("myBatch");
```

{{< note title="" class="info" >}}
对于仍在执行Job的正在运行的批处理，建议在删除之前暂停该批处理。
有关详细信息，请参阅 [暂停批处理](#suspend-a-batch) 部分。
{{< /note >}}

## 批处理的优先级

由于所有批处理Job都是使用Job执行器执行的，因此可以使用 [Job优先级] [job prioritization] 功能来调整批处理Job的优先级。 默认批处理Job优先级由流程引擎配置项 `batchJobPriority` 设置。

可以使用管理服务调整特定批处理的 [Job定义][job-definition-priority] 甚至单个批处理 [Job][job-priority] 的优先级。

```java
Batch batch = ...;

String batchJobDefinitionId = batch.getBatchJobDefinitionId();

processEngine.getManagementService()
  .setOverridingJobPriorityForJobDefinition(batchJobDefinitionId, 100, true);
```

## 操作日志

请注意，用户操作日志仅针对 Batch 创建本身而写入，种子Job的执行以及执行操作的单个Job均由 Job执行器 执行，因此不被视为用户操作。

# Job 定义

## 种子Job

批处理最初会创建一个种子Job。 该种子将被重复执行以创建所有批处理执行Job。 例如，如果用户为 1000 个流程实例启动 [流程实例迁移批处理][batch-migration]。 使用默认流程引擎配置时，种子Job将在每次调用时创建 10 个批处理执行Job。 每个执行Job将迁移 1 个流程实例。 总之，种子Job将被调用 100 次，直到它创建了完成批处理所需的 1000 个执行Job。

可以使用 Java API 获取批处理种子Job的Job定义：

```java
Batch batch = ...;

JobDefinition seedJobDefinition = processEngine.getManagementService()
  .createJobDefinitionQuery()
  .jobDefinitionId(batch.getSeedJobDefinitionId())
  .singleResult();
```

要暂停创建更多批处理执行Job，可以使用管理服务暂停种子Job定义：

```java
processEngine.getManagementService()
  .suspendJobByJobDefinitionId(seedJobDefinition.getId());
```

## 执行Job

一个批处理的执行被分成几个执行Job。 具体Job数取决于批处理的总Job数和流程引擎配置（参见[种子Job][seed job]）。 每个执行Job针对给定数量的调用执行实际的批处理命令，例如，迁移多个流程实例。执行Job将由[Job执行器][job executor]执行。 它们的行为与其他Job一样，这意味着它们可能会失败并且Job执行器将 [重试][retry] 失败的批处理执行Job。 此外，对于没有重试的失败的批处理执行Job，将产生 [事件][incidents]。

Java API 可用于获取批处理的执行Job的Job定义，例如，对于 [流程实例迁移批处理][batch-migration]：

```java
Batch batch = ...;

JobDefinition executionJobDefinition = processEngine.getManagementService()
  .createJobDefinitionQuery()
  .jobDefinitionId(batch.getBatchJobDefinitionId())
  .singleResult();
```

要批量暂停批处理Job的执行，可以使用管理服务暂停批处理Job定义：

```java
processEngine.getManagementService()
  .suspendJobByJobDefinitionId(executionJobDefinition.getId());
```

## 监控Job

在 [种子job][seed job] 创建所有批处理执行Job后，将为批处理创建一个监控Job。 此Job定期轮询批处理是否已完成，即所有批处理执行Job是否已完成。 轮询间隔可以通过[流程引擎配置][process engine configuration]的`batchPollTime`（默认：30秒）属性进行配置。

Java API 可用于获取批处理的监控Job的Job定义：

```java
Batch batch = ...;

JobDefinition monitorJobDefinition = processEngine.getManagementService()
  .createJobDefinitionQuery()
  .jobDefinitionId(batch.getMonitorJobDefinitionId())
  .singleResult();
```

为防止批处理完成，即删除运行时数据，可以使用管理服务暂停监视Job定义：

```java
processEngine.getManagementService()
  .suspendJobByJobDefinitionId(monitorJobDefinition.getId());
```

## 配置 

You can configure the number of jobs created by every seed job invocation `batchJobsPerSeed` (default: 100) and the number of invocations per batch execution job `invocationsPerBatchJob` (default: 1) in the [process engine configuration][process engine configuration].
你可以在[流程引擎配置][process engine configuration]中配置每个种子Job的调用创建Job数量`batchJobsPerSeed`（默认：100）和每个批量执行Job的调用数量`invocationsPerBatchJob`（默认：1）

通过使用流程引擎配置项 [`invocationsPerBatchJobByBatchType`][invoc-per-batch-job-batch-type]，可以为每个批处理操作类型单独更改每个批处理执行Job的调用次数。 如果你没有按类型指定每个批处理Job的调用，配置将回退到通过 `invocationsPerBatchJob` 指定的全局配置。

你可以通过三种方式配置该属性：

1.  通过 [流程引擎插件][Process Engine Plugin] 编程
2.  在 Spring-based 环境中通过 [Spring XML 配置文件][spring-xml-config] 
    ```xml
    <bean id="processEngineConfiguration" 
          class="org.camunda.bpm.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration">
    
      <!-- ... -->
    
      <property name="invocationsPerBatchJobByBatchType">
        <map>
          <entry key="process-set-removal-time" value="10" />
          <entry key="historic-instance-deletion" value="3" />
        
          <!-- 在自定义批处理操作的情况下 -->
          <entry key="my-custom-operation" value="7" />
        </map>
      </property>
    </bean>
    ```
3.  在 Spring Boot 环境中通过 [`application.yaml`][spring-boot-config] 文件
    ```yaml
    camunda.bpm.generic-properties.properties:
      invocations-per-batch-job-by-batch-type:
        process-set-removal-time:     10
        historic-instance-deletion:   3
        my-custom-operation:          7  # 在自定义批处理操作的情况下
    ```

[process-instance-cancellation]: {{< ref "/user-guide/process-engine/batch-operations.md#cancellation-of-running-process-instances">}}
[process-instance-deletion]: {{< ref "/user-guide/process-engine/batch-operations.md#deletion-of-historic-process-instances">}}
[set-job-retries]: {{< ref "/user-guide/process-engine/batch-operations.md#setting-retries-of-jobs-associated-with-process-instances">}}
[set-variables]: {{< ref "/user-guide/process-engine/batch-operations.md#set-variables-to-process-instances">}}
[migration]: {{< ref "/user-guide/process-engine/process-instance-migration.md" >}}
[batch-migration]: {{< ref "/user-guide/process-engine/process-instance-migration.md#asynchronous-batch-migration-execution" >}}
[job executor]: {{< ref "/user-guide/process-engine/the-job-executor.md" >}}
[process engine configuration]: {{< ref "/user-guide/process-engine/process-engine-bootstrapping.md" >}}
[seed job]: #种子Job
[retry]: {{< ref "/user-guide/process-engine/the-job-executor.md#failed-jobs" >}}
[incidents]: {{< ref "/user-guide/process-engine/incidents.md" >}}
[history level]: {{< ref "/user-guide/process-engine/history.md#choose-a-history-level" >}}
[history cleanup]: {{< ref "/user-guide/process-engine/history.md#history-time-to-live-for-batch-operations" >}}
[job prioritization]: {{< ref "/user-guide/process-engine/the-job-executor.md#job-prioritization" >}}
[job-definition-priority]: {{< ref "/user-guide/process-engine/the-job-executor.md#override-priority-by-job-definition" >}}
[job-priority]: {{< ref "/user-guide/process-engine/the-job-executor.md#set-job-priorities-via-managementservice-api" >}}
[set-external-tasks-retries]: {{< ref "/user-guide/process-engine/batch-operations.md#setting-retries-of-external-tasks" >}}
[process-instance-restart]: {{< ref "/user-guide/process-engine/process-instance-restart.md#asynchronous-batch-execution" >}}
[process-instance-modification]: {{< ref "/user-guide/process-engine/process-instance-modification.md#modification-of-multiple-process-instances" >}}
[process-instance-suspend]: {{< ref "/user-guide/process-engine/batch-operations.md#update-suspend-state-of-process-instances">}}
[set-removal-time]: {{< ref "/user-guide/process-engine/batch-operations.md#set-a-removal-time">}}
[invoc-per-batch-job-batch-type]: {{< ref "/reference/deployment-descriptors/tags/process-engine.md#invocations-per-batch-job-by-batch-type" >}}
[Process Engine Plugin]: {{< ref "/user-guide/process-engine/process-engine-plugins.md" >}}
[spring-xml-config]: {{< ref "/user-guide/spring-framework-integration/configuration.md" >}}
[spring-boot-config]: {{< ref "/user-guide/spring-boot-integration/configuration.md#camunda-engine-properties" >}}