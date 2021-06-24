---

title: 'Microsoft Sql Server 数据库配置'
weight: 30
menu:
  main:
    identifier: "user-guide-process-engine-database-mssql-configuration"
    parent: "user-guide-process-engine-database"

---

Microsoft SQL Server实现的隔离级别 `READ_COMMITTED` 与大多数数据库不同，与流程引擎的[乐观锁]({{< ref "/user-guide/process-engine/transactions-in-processes.md#optimistic-lock" >}}方案不能很好地兼容。
因此，当把流程引擎放在高负荷下时，可能会遇到死锁。

如果你在MSSQL安装中遇到死锁，则必须执行以下语句以启用 SNAPSHOT 隔离：

```sql
ALTER DATABASE [process-engine]
SET ALLOW_SNAPSHOT_ISOLATION ON

ALTER DATABASE [process-engine]
SET READ_COMMITTED_SNAPSHOT ON
```
其中 `[process-engine]` 是数据库的名称。