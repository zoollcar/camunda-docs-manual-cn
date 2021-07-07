---

title: '时区'
weight: 255

menu:
  main:
    identifier: "time-zones"
    parent: "user-guide-process-engine"

---

## 流程引擎

Camunda 引擎在处理日期时使用 JVM 的默认时区：

* 从 BPMN XML 读取日期时间值时
* 在 REST responses 中
* 从/向数据库读取/写入 DateTime 值时

## 数据库

数据库时区和数据库会话时区超出了 Camunda 引擎的范围，必须明确配置。

Camunda 引擎中的时间戳列使用的是“时间戳[不带时区]（TIMESTAMP [WITHOUT TIME ZONE]）”数据类型（名称在不同的数据库服务器中有所不同）。
因此，不建议在数据库端更改时区，因为一旦设置，可能会导致 Camunda 引擎的错误操作。

{{< note title="夏令时" class="warning" >}}
时区信息不保存在时间戳中。 为了避免不明确的时间戳，建议使用像`UTC`这样的时区作为JVM的默认时区
它未针对“夏令时 (DST)”进行调整，因此不会产生不明确的时间戳。

如果这不是你的设置中的一个选项，请考虑在 DST 切换期间禁用 [Job执行器]({{< ref "/user-guide/process-engine/the-job-executor.md" >}})
以避免意外的Job执行。
{{< /note >}}

## Camunda Web应用

可以在不同时区使用 Camunda Web应用。 使用 UI 时，所有日期相关操作都会使用本地时区。

## 集群设置

如果流程引擎在[集群]({{< ref "/introduction/architecture.md#clustering-model" >}})中运行，则所有集群节点必须在同一个时区中运行。 如果集群节点存在于不同的时区，则无法保证使用 DateTime 值进行操作的正确性。