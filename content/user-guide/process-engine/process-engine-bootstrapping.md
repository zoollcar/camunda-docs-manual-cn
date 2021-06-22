---

title: '流程引擎集成'
weight: 10

menu:
  main:
    identifier: "user-guide-process-engine-bootstrapping"
    parent: "user-guide-process-engine"

---


你有许多选项来配置并创建一个流程引擎，这取决于你打算使用一个应用管理流程引擎的还是一个共享的、容器管理流程引擎。


# 应用程序管理流程引擎

你将流程引擎为应用程序的一部分。可以配置以下方式来配置它：

* [通过Java API以编程方式管理]({{< relref "#bootstrap-a-process-engine-using-the-java-api" >}})
* [通过XML配置]({{< relref "#configure-process-engine-using-camunda-cfg-xml" >}})
* [通过 Spring]({{< ref "/user-guide/spring-framework-integration/_index.md" >}})


# 共享的，容器化管理流程引擎

你选择的容器（例如Tomcat、JBoss或IBM WebSphere）为你管理流程引擎。配置是以一种特定的容器方式进行的，详情见[运行时容器集成]({{< ref "/user-guide/runtime-container-integration/_index.md" >}}).


## ProcessEngineConfiguration Bean （流程引擎配置Bean）

Camunda引擎使用 {{< javadocref page="?org/camunda/bpm/engine/ProcessEngineConfiguration.html" text="ProcessEngineConfiguration bean" >}} 来配置和构建一个独立的流程引擎。它有多个可用的子类，可以用来定义流程引擎的配置。这些类代表不同的环境，并相应地设置默认值。最好的做法是选择与你的环境相匹配（大部分）的类，以尽量减少配置引擎所需的属性数量。目前有以下几个类。

* `org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration`  
流程引擎是以独立的方式使用的。引擎本身将负责处理事务。默认情况下，只有在引擎启动时才会检查数据库（如果没有数据库模式或模式版本不正确，会抛出一个异常）。
* `org.camunda.bpm.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration`  
这是一个用于单元测试的工具类。引擎本身将负责处理事务。默认使用H2内存数据库。该数据库将在引擎启动和关闭时被创建和删除。当使用这个时，可能不需要额外的配置（除了，当使用作业执行器（job executor）或邮件功能时）。
* `org.camunda.bpm.engine.spring.SpringProcessEngineConfiguration`  
当流程引擎被用于Spring环境时使用。更多信息请参见Spring集成部分。
* `org.camunda.bpm.engine.impl.cfg.JtaProcessEngineConfiguration`  
当引擎以独立模式运行时，使用JTA事务。


## 使用Java API启动流程引擎

你可以通过创建正确的ProcessEngineConfiguration对象或使用一些预定义的对象，以编程方式配置流程引擎。

```java
ProcessEngineConfiguration.createStandaloneProcessEngineConfiguration();
ProcessEngineConfiguration.createStandaloneInMemProcessEngineConfiguration();
```

现在你可以调用`buildProcessEngine()`操作来创建一个流程引擎。

```java
ProcessEngine processEngine = ProcessEngineConfiguration.createStandaloneInMemProcessEngineConfiguration()
  .setDatabaseSchemaUpdate(ProcessEngineConfiguration.DB_SCHEMA_UPDATE_FALSE)
  .setJdbcUrl("jdbc:h2:mem:my-own-db;DB_CLOSE_DELAY=1000")
  .setJobExecutorActivate(true)
  .buildProcessEngine();
```


## 使用camunda cfg XML配置流程引擎

配置你的流程引擎的最简单的方法是通过一个叫做`camunda.cfg.xml`的XML文件。使用这个文件，你可以简单这样做:

```java
ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine()
```

`camunda.cfg.xml`必须包含一个id为`processEngineConfiguration`的bean，选择最适合你需求的`ProcessEngineConfiguration`类。

```xml
<bean id="processEngineConfiguration" class="org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration">
```

这将在classpath上寻找一个`camunda.cfg.xml`文件，并根据该文件中的配置构建一个引擎。下面的片段显示了一个配置的例子：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="processEngineConfiguration" class="org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration">

    <property name="jdbcUrl" value="jdbc:h2:mem:camunda;DB_CLOSE_DELAY=1000" />
    <property name="jdbcDriver" value="org.h2.Driver" />
    <property name="jdbcUsername" value="sa" />
    <property name="jdbcPassword" value="" />

    <property name="databaseSchemaUpdate" value="true" />

    <property name="jobExecutorActivate" value="false" />

    <property name="mailServerHost" value="mail.my-corp.com" />
    <property name="mailServerPort" value="5025" />
  </bean>

</beans>
```
如果没有找到`camunda.cfg.xml`资源，默认引擎将搜索`activiti.cfg.xml`文件作为备用。如果两者都缺失，引擎就会停止运行，并打印出关于缺失配置资源的错误信息。

请注意，配置XML实际上是一个Spring配置。这并不意味着Camunda引擎只能在Spring环境中使用。我们只是在内部利用Spring的解析和依赖注入功能来建立引擎。

ProcessEngineConfiguration对象也可以使用配置文件以编程方式创建。也可以使用不同的bean id。

```java
ProcessEngineConfiguration.createProcessEngineConfigurationFromResourceDefault();
ProcessEngineConfiguration.createProcessEngineConfigurationFromResource(String resource);
ProcessEngineConfiguration.createProcessEngineConfigurationFromResource(String resource, String beanName);
ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream(InputStream inputStream);
ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream(InputStream inputStream, String beanName);
```

也可以不使用配置文件，而根据默认值创建配置（更多信息见不同的支持类）。

```java
ProcessEngineConfiguration.createStandaloneProcessEngineConfiguration();
ProcessEngineConfiguration.createStandaloneInMemProcessEngineConfiguration();
```

所有这些`ProcessEngineConfiguration.createXXX()`方法都返回一个`ProcessEngineConfiguration`，如果需要可以进一步调整。在调用`buildProcessEngine()`操作后，一个`ProcessEngine`被创建，如上所述。


## 在bpm-platform.xml中配置流程引擎

`bpm-platform.xml`文件用于配置下列发行版中的Camunda平台。

* Apache Tomcat
* IBM WebSphere Application Server
* Oracle WebLogic Application Server

`<process-engine ... />` xml标签允许你定义一个流程引擎。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bpm-platform xmlns="http://www.camunda.org/schema/1.0/BpmPlatform"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://www.camunda.org/schema/1.0/BpmPlatform http://www.camunda.org/schema/1.0/BpmPlatform">

  <job-executor>
    <job-acquisition name="default" />
  </job-executor>

  <process-engine name="default">
    <job-acquisition>default</job-acquisition>
    <configuration>org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration</configuration>
    <datasource>java:jdbc/ProcessEngine</datasource>

    <properties>
      <property name="history">full</property>
      <property name="databaseSchemaUpdate">true</property>
      <property name="authorizationEnabled">true</property>
    </properties>

  </process-engine>
</bpm-platform>
```

见[部署描述符参考]({{< ref "/reference/deployment-descriptors/descriptors/bpm-platform-xml.md" >}}) 以获得关于`bpm-platform.xml`文件语法的完整文档。


## 在 processes.xml中配置流程引擎

流程引擎也可以使用`META-INF/processes.xml`文件进行配置和引导。 见[processes.xml 文件]({{< ref "/user-guide/process-applications/the-processes-xml-deployment-descriptor.md" >}}) 了解详情。

见 [部署描述符参考]({{< ref "/reference/deployment-descriptors/descriptors/processes-xml.md" >}}) 以获得关于`processes.xml`文件语法的完整文档。
