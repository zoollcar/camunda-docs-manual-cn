---

title: '指标'
weight: 205

menu:
  main:
    identifier: "user-guide-process-engine-metrics"
    parent: "user-guide-process-engine"

---

流程引擎会在数据库中记录运行时指标，这有助于得出关于Camunda平台的使用、负载和性能的结论。指标会被记录到表 `ACT_RU_METER_LOG` 和 `ACT_RU_TASK_METER_LOG`中。`ACT_RU_METER_LOG`中的单一指标由以下内容组成，某个时间段记录的在JAVA `long` 范围内的自然数，和标识这个记录的名字。在表`ACT_RU_TASK_METER_LOG` 中的任务指标记录包含固定长度，受让人的化名和受让的时间点，有一套默认报告的内置指标。

# 内置指标

下表包含内置指标和描述。所有内置度量的标识符都可以作为类 {{< javadocref page="?org/camunda/bpm/engine/management/Metrics.html" text="org.camunda.bpm.engine.management.Metrics" >}}的常量来使用。
{{< note title="小心!" class="warning" >}}
如果你是一个企业客户，你的许可协议可能要求你每年报告一些指标。请从 `ACT_RU_METER_LOG` 以及任务指标 `ACT_RU_TASK_METER_LOG`表中保留 `root-process-instance-start`, `activity-instance-start`, `executed-decision-instances` 和 `executed-decision-elements` 至少有18个月。
{{< /note >}}

<table class="table table-striped">
  <tr>
    <th>类别</th>
    <th>识别符</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><b>BPMN Execution</b></td>
    <td>root-process-instance-start*</td>
    <td>开始执行的根流程实例的数量。这也被称为<b>根流程实例（RPI）</b>。
    根流程实例没有父流程实例，即它是一个顶层执行。
    </td>
  </tr>
  <tr>
    <td></td>
    <td>activity-instance-start*</td>
    <td>活动实例的数量已启动。这也称为<b>流节点实例（FNI）</b>。</td>
  </tr>
  <tr>
    <td></td>
    <td>activity-instance-end</td>
    <td>结束的活动实例的数量。</td>
  </tr>
  <tr>
    <td><b>DMN Execution</b></td>
    <td>executed-decision-instances*</td>
    <td>评估决策实例的数量（EDI）。决策实例是DMN决策表或DMN文字表达式。</td>
  </tr>
  <tr>
    <td></td>
    <td>executed-decision-elements*</td>
    <td>在评估DMN决策表时执行的决策元素的数量。对于一个表，结果为子句数量和规则数量的乘积。</td>
  </tr>
  <tr>
    <td><b>Job Executor</b></td>
    <td>job-successful</td>
    <td>成功执行的Job数量。</td>
  </tr>
  <tr>
    <td></td>
    <td>job-failed</td>
    <td>未能执行的Job数量和提交重试的Job数量。计算每次尝试执行Job的尝试。</td>
  </tr>
  <tr>
    <td></td>
    <td>job-acquisition-attempt</td>
    <td>Job采集周期的数。</td>
  </tr>
  <tr>
    <td></td>
    <td>job-acquired-success</td>
    <td>获取和成功锁定执行的Job数量。</td>
  </tr>
  <tr>
    <td></td>
    <td>job-acquired-failure</td>
    <td>由于另一种Job执行程序锁定/执行Job而无法锁定，但无法锁定所获取但无法锁定的Job数量。</td>
  </tr>
  <tr>
    <td></td>
    <td>job-execution-rejected</td>
    <td>由于饱和执行资源而被拒绝的执行成功获取的Job的数量。这是执行线程池的Job队列已满的指标。</td>
  </tr>
  <tr>
    <td></td>
    <td>job-locked-exclusive</td>
    <td>立即锁定和执行的独占Job的数量。</td>
  </tr>
  <tr>
    <td><b>Task Metrics</b></td>
    <td>unique-task-workers*</td>
    <td>曾担任受让人的独特任务工作人员的数量。</td>
  </tr>
  <tr>
    <td><b>History Clean up</b></td>
    <td>history-cleanup-removed-process-instances</td>
    <td>历史清除后删除的流程实例数。</td>
  </tr>
  <tr>
    <td></td>
    <td>history-cleanup-removed-case-instances</td>
    <td>历史清除后删除的案例实例数。</td>
  </tr>
  <tr>
    <td></td>
    <td>history-cleanup-removed-decision-instances</td>
    <td>历史清除删除的决策实例数。</td>
  </tr>
  <tr>
    <td></td>
    <td>history-cleanup-removed-batch-operations</td>
    <td>历史清除删除的批处理操作数。</td>
  </tr>
  <tr>
    <td></td>
    <td>history-cleanup-removed-task-metrics</td>
    <td>历史清除后删除的任务指标数量。</td>
  </tr>
</table>

* 一些企业协议需要年度报告一些指标。请存储至少18个月的指标。

# 查询

可以构造一个`ManagementService`提供的 {{< javadocref page="?org/camunda/bpm/engine/management/MetricsQuery.html" text="MetricsQuery" >}} 来查询指标。下面的例子查询了整个报告历史中所有执行的活动实例的数量。

```java
long numCompletedActivityInstances = managementService
  .createMetricsQuery()
  .name(Metrics.ACTIVTY_INSTANCE_START)
  .sum();
```

指标查询提供了过滤器`#startDate(Date date)`和`#endDate(Date date)`来限制收集的指标到某个时间段。此外，通过使用过滤器`#reporter(String reporterId)`，可以将结果限制在由特定报告人收集的指标。当针对同一数据库配置多个引擎时，例如在集群设置中，这个选项可能很有用。

任务指标可以通过使用`ManagementService`提供的`getUniqueTaskWorkerCount`方法进行查询。这个方法接受可选 "Date" 值 "startTime" 和 "endTime" ，以便将指标限制在某个时间范围内。例如，下面的语句检索了到目前为止所有独特的任务工作者的数量。

```java
long numUniqueTaskWorkers = managementService.getUniqueTaskWorkerCount(null, null);
```

# 配置

## 指标记录器

流程引擎以15分钟的间隔将收集到的指标记录在运行时的数据库表。指标报告的行为可以通过替换流程引擎配置中的`dbMetricsReporter`实例来改变。例如，为了改变报告间隔，可以选择一个替换记录器的流程引擎插件。

```java
public class MetricsConfigurationPlugin implements ProcessEnginePlugin {

  public void preInit(ProcessEngineConfigurationImpl processEngineConfiguration) {
  }

  public void postInit(ProcessEngineConfigurationImpl processEngineConfiguration) {
    DbMetricsReporter metricsReporter = new DbMetricsReporter(processEngineConfiguration.getMetricsRegistry(),
        processEngineConfiguration.getCommandExecutorTxRequired());
    metricsReporter.setReportingIntervalInSeconds(5);

    processEngineConfiguration.setDbMetricsReporter(metricsReporter);
  }

  public void postProcessEngineBuild(ProcessEngine processEngine) {
  }

}
```

{{< note title="笔记" class="info" >}}
任务度量条目在用户任务的每一次分配中都会被创建。这种行为不能被修改，也不在指标记录器的责任范围内。
{{< /note >}}

## 记录器识别符

指标的记录有一个记录方的标识符。这个标识符允许在进行指标查询时将报告归属于单个引擎实例。例如，在一个集群中，负载指标与单个集群节点相关。默认情况下，流程引擎生成的报告者ID为 `<local IP>$<engine name>` 。可以通过实现接口 {{< javadocref page="?org/camunda/bpm/engine/impl/history/event/HostnameProvider.html" text="org.camunda.bpm.engine.impl.history.event.HostnameProvider" >}} 并将引擎属性 `hostnameProvider` 设置为该类的一个实例来定制生成。

{{< note title="小心!" class="info" >}}
推荐使用 {{< javadocref page="?org/camunda/bpm/engine/impl/metrics/MetricsReporterIdProvider.html" text="org.camunda.bpm.engine.impl.metrics.MetricsReporterIdProvider" >}}
接口和相应的 `metricsReporterIdProvider` 引擎属性。 
{{< /note >}}

## 禁用记录器

默认情况下，所有内置指标都被报告。通过XML文件（如standalone.xml或bpm-platform.xml）进行的配置，你可以通过添加属性来禁用报告。
```xml
<property name="metricsEnabled">false</property>
<property name="taskMetricsEnabled">false</property>
```

如果你直接访问Java API，你可以通过使用引擎配置标志`isMetricsEnabled`和`isTaskMetricsEnabled`并将其设置为`false`来禁用指标报告。
