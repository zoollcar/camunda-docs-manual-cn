---

title: '数据库配置'
weight: 20
menu:
  main:
    identifier: "user-guide-process-engine-database-configuraton"
    parent: "user-guide-process-engine-database"

---

有两种方法可以配置Camunda引擎使用的数据库。第一个方法是定义数据库的JDBC属性：

* `jdbcUrl`: 数据库的JDBC URL。
* `jdbcDriver`: 用于特定数据库的驱动程序实现。
* `jdbcUsername`: 连接到数据库的用户名。
* `jdbcPassword`: 连接到数据库的密码。

注意，引擎内部使用[Apache MyBatis](http://www.mybatis.org/)进行持久化。

基于提供的JDBC属性构建的数据源将具有默认的MyBatis连接池设置。以下属性可以选择性地设置，以调整该连接池（取自MyBatis文档）。

* `jdbcMaxActiveConnections`: 连接池在任何时间最多可以包含的活动连接数。缺省值是10。
* `jdbcMaxIdleConnections`: 连接池在任何时间最多可以包含的空闲连接数。
* `jdbcMaxCheckoutTime`: 连接被取出使用的最长时间，超过时间会被强制回收。默认值是20000（20秒）。
* `jdbcMaxWaitTime`: 这是一个底层配置，让连接池可以在长时间无法获得连接时，打印一条日志，并重新尝试获取一个连接。Default是20000（20秒）。
* `jdbcStatementTimeout`: JDBC驱动程序将等待从数据库的响应的时间内的时间量。默认值为null，这意味着没有超时。


## Jdbc 批处理

<a name="jdbcBatchProcessing"></a>另一个配置-- `jdbcBatchProcessing` --设置向数据库发送SQL语句时是否必须使用批处理模式。当关闭时，语句会被一个一个地执行。
可用值: `true` (默认), `false`.

批处理模式在流程引擎中的已知问题有：

* 批处理模式对早于12版本的Oracle不起作用。
* 当在MariaDB和DB2上使用批处理时，`jdbcStatementTimeout`配置会被忽略。


## 示例数据库配置

```xml
<property name="jdbcUrl" value="jdbc:h2:mem:camunda;DB_CLOSE_DELAY=1000" />
<property name="jdbcDriver" value="org.h2.Driver" />
<property name="jdbcUsername" value="sa" />
<property name="jdbcPassword" value="" />
```

或者, 也可以使用 `javax.sql.DataSource` (示例，来自Apache Commons的DBCP)：

```xml
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" >
  <property name="driverClassName" value="com.mysql.jdbc.Driver" />
  <property name="url" value="jdbc:mysql://localhost:3306/camunda?sendFractionalSeconds=false" />
  <property name="username" value="camunda" />
  <property name="password" value="camunda" />
  <property name="defaultAutoCommit" value="false" />
</bean>

<bean id="processEngineConfiguration" class="org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration">

    <property name="dataSource" ref="dataSource" />
    ...
```

请注意，Camunda并没有自带允许定义这种数据源的库。所以你必须确保这些库（例如来自DBCP）在你的classpath上。

无论你是使用JDBC还是数据源方法，都可以设置以下属性：

* `databaseType`: 通常没有必要指定这个属性，因为它可以从url中自动分析出来的。只有在自动分析失败的情况下才应该指定。可能的值有{h2, mysql, oracle, postgres, mssql, db2, mariadb}。这个设置将决定哪些创建/删除脚本和查询将被使用。参见 "支持的数据库" 一节，以了解支持哪些类型的数据库。</li>
* `databaseSchemaUpdate`: 允许设置策略来处理流程引擎启动和关闭时的数据库模式。
  * `true` (默认): 在建立流程引擎时，要检查数据库中是否存在Camunda表。如果它们不存在，就会被创建。除非正在执行[滚动更新]({{< ref "/update/rolling-update.md" >}})，否则必须确保数据库模式的版本与流程引擎库的版本一致。数据库模式的更新必须按照[更新和迁移指南]({{< ref "/update/_index.md" >}})中的描述手动完成。
  * `false`: 不执行任何检查，并假设数据库中存在Camunda表。除非正在执行[滚动更新]({{< ref "/update/rolling-update.md" >}})，否则必须确保数据库模式的版本与流程引擎库的版本一致。数据库模式的更新必须按照[更新和迁移指南]({{< ref "/update/_index.md" >}})中的描述手动完成。
  * `create-drop`: 在创建流程引擎时创建表，并在关闭流程引擎时删除表。

{{< note title="支持的数据库" class="info" >}}
  有关支持数据库的信息，请参阅[支持的环境]({{< ref "/introduction/supported-environments.md#databases" >}})
{{< /note >}}

以下是一些示例JDBC URL：

* H2: `jdbc:h2:tcp://localhost/camunda`
* MySQL: `jdbc:mysql://localhost:3306/camunda?autoReconnect=true&sendFractionalSeconds=false`
* Oracle: `jdbc:oracle:thin:@localhost:1521:xe`
* PostgreSQL: `jdbc:postgresql://localhost:5432/camunda`
* DB2: `jdbc:db2://localhost:50000/camunda`
* MSSQL: `jdbc:sqlserver://localhost:1433/camunda`
* MariaDB: `jdbc:mariadb://localhost:3306/camunda`


## 其他数据库架构配置

### Business Key

自从 Camunda Platform 7.0.0-alpha9 发布后，在运行时和历史表以及数据库模式创建和删除脚本中，业务键的唯一约束被删除。
如果你依赖该约束，你可以手动将以下sql语句应用到你的数据库中。

DB2

```sql
Runtime:
alter table ACT_RU_EXECUTION add UNI_BUSINESS_KEY varchar (255) not null generated always as (case when "BUSINESS_KEY_" is null then "ID_" else "BUSINESS_KEY_" end);
alter table ACT_RU_EXECUTION add UNI_PROC_DEF_ID varchar (64) not null generated always as (case when "PROC_DEF_ID_" is null then "ID_" else "PROC_DEF_ID_" end);
create unique index ACT_UNIQ_RU_BUS_KEY on ACT_RU_EXECUTION(UNI_PROC_DEF_ID, UNI_BUSINESS_KEY);

History:
alter table ACT_HI_PROCINST add UNI_BUSINESS_KEY varchar (255) not null generated always as (case when "BUSINESS_KEY_" is null then "ID_" else "BUSINESS_KEY_" end);
alter table ACT_HI_PROCINST add UNI_PROC_DEF_ID varchar (64) not null generated always as (case when "PROC_DEF_ID_" is null then "ID_" else "PROC_DEF_ID_" end);
create unique index ACT_UNIQ_HI_BUS_KEY on ACT_HI_PROCINST(UNI_PROC_DEF_ID, UNI_BUSINESS_KEY);
```

H2

```sql
Runtime:
alter table ACT_RU_EXECUTION add constraint ACT_UNIQ_RU_BUS_KEY unique(PROC_DEF_ID_, BUSINESS_KEY_);

History:
alter table ACT_HI_PROCINST add constraint ACT_UNIQ_HI_BUS_KEY unique(PROC_DEF_ID_, BUSINESS_KEY_);
```

MSSQL

```sql
Runtime:
create unique index ACT_UNIQ_RU_BUS_KEY on ACT_RU_EXECUTION (PROC_DEF_ID_, BUSINESS_KEY_) where BUSINESS_KEY_ is not null;

History:
create unique index ACT_UNIQ_HI_BUS_KEY on ACT_HI_PROCINST (PROC_DEF_ID_, BUSINESS_KEY_) where BUSINESS_KEY_ is not null;
```

MySQL

```sql
Runtime:
alter table ACT_RU_EXECUTION add constraint ACT_UNIQ_RU_BUS_KEY UNIQUE (PROC_DEF_ID_, BUSINESS_KEY_);

History:
alter table ACT_HI_PROCINST add constraint ACT_UNIQ_HI_BUS_KEY UNIQUE (PROC_DEF_ID_, BUSINESS_KEY_);
```

Oracle

```sql
Runtime:
create unique index ACT_UNIQ_RU_BUS_KEY on ACT_RU_EXECUTION
         (case when BUSINESS_KEY_ is null then null else PROC_DEF_ID_ end,
         case when BUSINESS_KEY_ is null then null else BUSINESS_KEY_ end);

History:
create unique index ACT_UNIQ_HI_BUS_KEY on ACT_HI_PROCINST
         (case when BUSINESS_KEY_ is null then null else PROC_DEF_ID_ end,
         case when BUSINESS_KEY_ is null then null else BUSINESS_KEY_ end);
```

PostgreSQL

```sql
Runtime:
alter table ACT_RU_EXECUTION add constraint ACT_UNIQ_RU_BUS_KEY UNIQUE (PROC_DEF_ID_, BUSINESS_KEY_);

History:
alter table ACT_HI_PROCINST add constraint ACT_UNIQ_HI_BUS_KEY UNIQUE (PROC_DEF_ID_, BUSINESS_KEY_);
```

## 隔离级别配置

大多数数据库管理系统提供了四种不同的隔离级别可供设置。例如，由ANSI/USO SQL定义的级别是（从低到高的隔离）有：

* 读未提交（READ UNCOMMITTED ）
* 不可重复读（READ COMMITTED）
* 可重复读（REPEATABLE READS）
* 串行化（SERIALIZABLE）

运行Camunda所需的隔离级别是 **不可重复读（READ COMMITTED）** ，根据你的数据库系统不同，它可能有不同的名称。将隔离级别设置为 REPEATABLE READS 会导致死锁，所以在改变隔离级别时需要谨慎。
