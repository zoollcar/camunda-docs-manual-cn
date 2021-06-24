---

title: 'MySQL 数据库配置'
weight: 50
menu:
  main:
    identifier: "user-guide-process-engine-database-mysql-configuration"
    parent: "user-guide-process-engine-database"

---

本节描述了如何进行 MySQL 配置。 

## 数据库表结构

引擎的MySQL数据库模式不支持列类型 `TIMESTAMP` 和 `DATETIME` 的毫秒精度。
也就是会四舍五入到下一秒或上一秒，例如，`2021-01-01 15:00:46.731`被四舍五入到`2021-01-01 15:00:47`。

{{< note title="小心!" class="info" >}}
日期/时间值缺失的毫秒精度会影响流程引擎的行为。
请阅读[如何配置MySQL JDBC驱动程序]({{<ref "#jdbc驱动配置" >}})以确保正确处理日期/时间值。
{{< /note >}}

## JDBC驱动配置

在这里你可以找到MySQL JDBC驱动程序的配置条件，以确保与流程引擎兼容。

### 为 date/time 类型禁用发送毫秒精度

{{< note title="小心!" class="info" >}}
这个配置是必要的，以避免在用MySQL操作流程引擎时出现意外行为。
{{< /note >}}

将date/time值发送到数据库的任何SQL语句的一部分时, 当 MySQL JDBC驱动 >= 5.1.23 时会发送毫秒。 
在使用MySQL数据库时，这个行为会导致问题。 [表结构不支持带有毫秒的 date/time 类型值][mysql-schema-milliseconds]。

在发送date/time值时确保流程引擎的正确行为， 确保将MySQL JDBC驱动程序更新为版本> = 5.1.37。
在这些版本中，你可以通过设置[`sendFractionalSeconds=false`][mysql-fract-secs]来避免向MySQL服务器发送毫秒。
在你的JDBC连接URL中设置[sendFractionalSeconds=false][mysql-fract-secs]。

下面是在没有配置 "sendFractionalSeconds=false" 标志的情况下发生的问题行为的例子：

*当用户以`due date == 2021-01-01 15:00:46.731`执行任务查询时，该查询返回的结果等于`2021-01-01 15:00:46.731`。然而，由于引擎的[数据库模式不存储毫秒][mysql-schema-milliseconds]，没有返回结果。
* 当用户为任务设置due date时，该值被四舍五入到下一秒或前一秒，例如，`2021-01-01 15:00:46.731`被四舍五入为`2021-01-01 15:00:47`。也请参见官方[MySQL文档](https://dev.mysql.com/doc/refman/5.6/en/fractional-seconds.html)。

[mysql-schema-milliseconds]: #database-schema
[mysql-fract-secs]: https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-connp-props-datetime-types-processing.html#cj-conn-prop_sendFractionalSeconds
