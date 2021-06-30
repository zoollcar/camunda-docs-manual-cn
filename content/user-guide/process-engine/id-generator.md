---

title: 'Id 生成器'
weight: 190

menu:
  main:
    identifier: "user-guide-process-engine-id-generator"
    parent: "user-guide-process-engine"

---


由流程引擎管理的所有持久性实体（流程实例、任务......）都有唯一的Ids。这些Ids唯一地标识一个单独的任务、流程实例等。当这些实体被持久化到数据库时，这些ID被用作相应数据库表中的主键。

开箱即用，流程引擎提供了两种Id生成器的实现：


# 数据库ID生成器

数据库标识生成器是在`ACT_GE_PROPERTY`表的基础上使用序列生成器实现的。

这个ID生成器有利于调试和测试，因为它生成了人类可读的ID。

{{< note title="" class="warning" >}}
  数据库标识生成器 **不应该** 在生产环境中使用，因为它不具有高并发性。
{{< /note >}}


# UUID生成器

StrongUuidGenerator使用一个UUID生成器，它在内部使用[Java UUID Generator(JUG)][1]库。

{{< note title="" class="warning" >}}
  在生产环境始终使用UUID生成器。
{{< /note >}}

在[Camunda Platform Full Distributions][2]中，StrongUuidGenerator被预设为流程引擎所使用的默认Id 生成器。

如果你使用嵌入式流程引擎配置并使用Spring配置流程引擎，你需要在Spring配置中添加以下几行，以启用 "StrongUuidGenerator"。

```xml
<bean id="processEngineConfiguration" class="org.camunda.bpm.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration">

  [...]

  <property name="idGenerator">
    <bean class="org.camunda.bpm.engine.impl.persistence.StrongUuidGenerator" />
  </property>

</bean>
```

另外，您需要以下Maven依赖项：

```xml
<dependency>
  <groupId>com.fasterxml.uuid</groupId>
  <artifactId>java-uuid-generator</artifactId>
  <scope>provided</scope>
  <version>3.1.2</version>
</dependency>
```

[1]: https://mvnrepository.com/artifact/com.fasterxml.uuid/java-uuid-generator
[2]: {{< ref "/introduction/downloading-camunda.md#full-distribution" >}}
