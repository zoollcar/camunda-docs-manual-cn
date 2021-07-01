---

title: '流程引擎插件'
weight: 220

menu:
  main:
    identifier: "user-guide-process-engine-plugins"
    parent: "user-guide-process-engine"

---


流程引擎配置可以通过流程引擎插件扩展。流程引擎插件是[流程引擎配置]({{< ref "/user-guide/process-engine/process-engine-bootstrapping.md" >}})的扩展。

插件必须提供实现{{< javadocref page="?org/camunda/bpm/engine/impl/cfg/ProcessEnginePlugin.html" text="ProcessEnginePlugin" >}}接口。


# 配置流程引擎插件

可以在下列位置配置流程引擎插件：

* 在 [Camunda平台 部署描述符]({{< ref "/reference/deployment-descriptors/_index.md" >}}) (bpm-platform.xml/processes.xml),
* 在 [JBoss Application Server 7/Wildfly 配置文件]({{< ref "/user-guide/runtime-container-integration/jboss.md" >}}) (standalone.xml/domain.xml),
* 在 [Spring Beans XML]({{< ref "/user-guide/spring-framework-integration/_index.md#configure-a-process-engine-plugin-in-spring" >}}),
* 编程配置。

下面是如何在bpm-platform.xml文件中配置一个流程引擎插件的例子。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bpm-platform xmlns="http://www.camunda.org/schema/1.0/BpmPlatform"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.camunda.org/schema/1.0/BpmPlatform http://www.camunda.org/schema/1.0/BpmPlatform ">

  <job-executor>
    <job-acquisition name="default" />
  </job-executor>

  <process-engine name="default">
    <job-acquisition>default</job-acquisition>
    <configuration>org.camunda.bpm.engine.impl.cfg.JtaProcessEngineConfiguration</configuration>
    <datasource>jdbc/ProcessEngine</datasource>

    <plugins>
      <plugin>
        <class>org.camunda.bpm.engine.MyCustomProcessEnginePlugin</class>
        <properties>
          <property name="boost">10</property>
          <property name="maxPerformance">true</property>
          <property name="actors">akka</property>
        </properties>
      </plugin>
    </plugins>
  </process-engine>

</bpm-platform>
```

流程引擎插件类必须对加载流程引擎类的类加载器可见。


# 内置流程引擎插件列表

下面是内置的流程引擎插件的列表：

* [LDAP Identity Service插件]({{< ref "/user-guide/process-engine/identity-service.md#the-ldap-identity-service" >}})
* [Administrator Authorization 插件]({{< ref "/user-guide/process-engine/authorization-service.md#the-administrator-authorization-plugin" >}})
* [Process Application 事件监听器插件]({{< ref "/user-guide/process-applications/process-application-event-listeners.md" >}})
