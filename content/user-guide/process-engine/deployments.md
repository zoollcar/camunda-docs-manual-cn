---

title: '部署'
weight: 245

menu:
  main:
    identifier: "user-guide-process-engine-deployments"
    parent: "user-guide-process-engine"

---

在流程引擎可以执行流程（或案例或决策）之前，必须对其进行部署。 部署是将部署在一起的多组资源的逻辑实体。 可以通过 Java API 或 [REST API]({{< ref "/reference/rest/deployment/post-deployment.md" >}}) 以编程方式进行部署，或者针对 [流程应用]({{ < ref "/user-guide/process-applications/_index.md" >}})使用声明方式。本节介绍高级部署概念。

# 集群场景中的部署

在流程引擎开始执行部署之前，它会尝试获取对“ACT_GE_PROPERTY”表中一行的排他锁。当流程引擎能够成功获取锁时，它开始部署并持有排它锁，只要部署执行发生。

如果在集群场景中同时在多个节点上部署相同的资源，则获取的排他锁确保重复过滤按预期工作。否则，并行部署可能会导致同一流程定义的多个版本。

此外，排他锁确保具有相同密钥的多个定义（例如，流程定义）在同时部署时不会获得相同的版本，这可能导致失败和意外行为。请注意，数据库中没有检查定义唯一性的唯一约束。

因此，排他锁强制执行顺序部署。

默认情况下，排他锁获取是开启的。如果不需要，可以通过将名为 `deploymentLockUsed` 的流程引擎配置标志设置为 false 来禁用它。

{{< note class="warning" title="H2 数据库" >}}
请注意，集群方案不支持 H2 数据库。流程引擎不会创建排他锁，因为 H2 默认使用表级锁，如果执行部署时部署命令需要使用 DbIdGenerator 获取新 Id，这可能会导致死锁。
{{</note >}}
