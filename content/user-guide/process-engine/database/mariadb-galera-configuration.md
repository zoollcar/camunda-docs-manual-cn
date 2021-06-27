---

title: 'MariaDB Galera 数据库配置'
weight: 40
menu:
  main:
    identifier: "user-guide-process-engine-database-mariadb-galera-configuration"
    parent: "user-guide-process-engine-database"

---

本节记录了MariaDB所支持的Galera Cluster配置。服务器和客户端都需要进行正确的配置。请注意，在使用Galera集群时，有一些[已知的限制](#galera-cluster-的已知的限制)，详情见下文：

{{< note title="警告" class="warning" >}}
请注意，下面定义的服务器和客户端配置设置是唯一支持Galera Cluster的配置。其他配置都不支持。
{{</ note >}}

### 服务器配置

以下配置需要放入到每个服务器的 `my.cnf.d/server.cnf` 配置文件的 `[galera]` 部分。

```
[galera]
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
transaction-isolation=READ-COMMITTED
wsrep_on=ON
wsrep_causal_reads = 1
wsrep_sync_wait = 7
...
```

请注意，一些其他设置也可能会出现在这一部分，但是 "transaction-isolation"、"wsrep_on"、"wsrep_causal_reads" 和 "wsrep_sync_wait" 的设置必须设置，且具有 **完全相同** 的值。

### 客户端配置

只有 `failover` 和 `sequential` 配置是支持的。 **其他客户端模式，如 `replication:`, `loadbalance:`, `aurora:` 均不支持**

以下是数据源配置中jdbcUrl属性的必要格式。

```
jdbc:mariadb:[failover|sequential]://[host1:port],[host2:port],.../[data-base-name]
```

示例:

```
jdbc:mariadb:failover://192.168.1.1:32980,192.168.1.2:32980,192.168.1.3:32980/process-engine
jdbc:mariadb:sequential://192.168.1.1:32980,192.168.1.2:32980,192.168.1.3:32980/process-engine
```

重要提示：在群集中运行Camunda时，客户端配置需要在每个节点上相同。

### Galera Cluster 的已知的限制

使用Galera Cluster时，有以下已知限制：

1. 在数据库中需要悲观的读锁的API不能正确工作。受影响的API。独家消息相关（`.correlateExclusively()`）。参见({{< javadocref page="?org/camunda/bpm/engine/runtime/MessageCorrelationBuilder.html#correlateExclusively()" text="Javadocs" >}})。
另一个可能的负面效应。
 * 当从两个线程同时部署相同的资源时，会重复部署定义。
 * 当从两个线程同时调用 "HistoryService#cleanUpHistoryAsync" 时，会重复执行历史清理工作。
2. 如果资源同时部署在一个集群中，部署期间的重复检查不起作用。具体影响是：假设有一个Camunda流程引擎集群，它连接到同一个Galera集群。在部署一个新的流程应用时，流程引擎节点将检查流程应用所提供的BPMN流程是否已经部署，以避免重复部署。如果在多个流程引擎节点上同时进行部署，就会在数据库上获得一个独占的读锁（技术上来说，这意味着每个节点执行一个SQL "选择更新 "的查询。这在Galera Cluster上不起作用，可能导致同一进程的多个版本被部署。
3. `jdbcStatementTimeout` 配置项不起作用，不能使用。
