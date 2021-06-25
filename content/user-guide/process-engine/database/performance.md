---

title: '性能'
weight: 60
menu:
  main:
    identifier: "user-guide-process-engine-database-performance"
    parent: "user-guide-process-engine-database"

---

此页面介绍了数据库查询性能相关内容。而不是Camunda平台安装的一般性能分析或者优化提供工具及指导。

由于这里讨论的设置的影响在很大程度上取决于Camunda平台的设置和工作量级，这些建议对你的情况可能有帮助，也可能没有帮助。不能保证性能会提高。

# 任务查询

任务查询是流程引擎API中使用量最大、功能最强的查询之一。由于其丰富的功能集，它在SQL中也可能变得复杂，并可能表现得很糟糕。

## 禁用CMMN和独立任务（Standalone Tasks）

为了执行透明的访问检查，任务查询加入了授权表（`ACT_RU_AUTHORIZATION`）。对于任何一种与流程相关的过滤器，它加入流程定义表（`ACT_RE_PROCDEF`）。默认情况下，查询对这些操作使用左连接（left join）。如果不使用CMMN和独立任务（既不与BPMN流程相关，也不与CMMN案例相关的任务），引擎配置标志`cmmnEnabled`和`standaloneTasksEnabled`可以设置为`false`。然后，左联接被内联接取代，内联接在某些数据库中表现会更好。参见[配置属性参考]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#configuration-properties" >}})了解这些设置的详情。
