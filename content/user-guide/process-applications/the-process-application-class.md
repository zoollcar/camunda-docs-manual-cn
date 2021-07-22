---

title: 'Process Application 类'
weight: 10

menu:
  main:
    identifier: "user-guide-process-applications-class"
    parent: "user-guide-process-applications"

---

你可以将流程引擎和流程部署的引导委托给 Process Application 类。基本的 ProcessApplication 功能由 `org.camunda.bpm.application.AbstractProcessApplication` 基类提供。 在这个类的基础上，有一组特定于环境的子类，它们实现了特定环境中的集成：

* **ServletProcessApplication**: 用于 Apache Tomcat 等 Servlet 容器中的 Process Application 。
* **EjbProcessApplication**: 用于 Java EE 应用服务器，如 JBoss 或 IBM WebSphere Application Server。
* **EmbeddedProcessApplication**: 用于在普通 Java SE 应用程序中嵌入流程引擎中使用。
* **SpringProcessApplication**: 用于从 Spring 应用程序上下文引导 Process Application 。

在下一节中，我们将介绍不同的实现，并讨论可以在何处以及如何使用它们。


# ServletProcessApplication

**支持:** Apache Tomcat, JBoss/Wildfly. 所有容器都支持 Servlet Process Application。请参阅[关于 Servlet Process Application 和 EJB/Java EE 容器的说明]({{< relref "#在-jboss-等-ejb-java-ee-容器中使用-servletprocessapplication" >}})

**打包格式**: WAR (或 EAR 中的 嵌入式 WAR)

`ServletProcessApplication` 类是基于 Servlet 规范（Java Web 应用程序）开发 Process Application 的基类。 servlet  Process Application 实现了`javax.servlet.ServletContextListener` 接口，该接口允许它参与你的 Web 应用程序的部署生命周期

以下是 Servlet Process Application 的示例：

```java
package org.camunda.bpm.example.loanapproval;

import org.camunda.bpm.application.ProcessApplication;
import org.camunda.bpm.application.impl.ServletProcessApplication;

@ProcessApplication("Loan Approval App")
public class LoanApprovalApplication extends ServletProcessApplication {
  // 空实现
}
```

注意 `@ProcessApplication` 注释。 这个注解有两个作用：

  * **provide the name of the ProcessApplication**: 你可以使用注释为 Process Application 提供自定义名称：`@ProcessApplication("Loan Approval App")`。 如果未提供名称，则会自动检测名称。 如果是 ServletProcessApplication，则使用 ServletContext 的名称。
  * **trigger auto-deployment**. 在 Servlet 3.0 容器中，注释足以确保 Process Application 被 servlet 容器自动选取并作为 ServletContextListener 自动添加到 Servlet 容器部署中。此功能由位于 camunda-engine 模块中的名为“org.camunda.bpm.application.impl.ServletProcessApplicationDeployer”的“javax.servlet.ServletContainerInitializer”实现实现。该实现既适用于将 camunda-engine.jar 作为 Web 应用程序库嵌入到 WAR 文件的“WEB-INF/lib”文件夹中，也适用于将 camunda-engine.jar 作为共享库部署在应用服务器的共享库（例如，Apache Tomcat 全局`lib/` 文件夹）目录。 Servlet 3.0 规范预见了两种部署方案。如果是嵌入式部署，则在部署 Web 应用程序时会通知 `ServletProcessApplicationDeployer` 一次。在部署为共享库的情况下，对于每个包含用“@ProcessApplication”注释的类的 WAR 文件，都会通知“ServletProcessApplicationDeployer”（根据 Servlet 3.0 规范的要求）。

这意味着，如果你部署到符合 Servlet 3.0 的容器（例如 Apache Tomcat），使用 `@ProcessApplication` 注释你的类就足够了。

{{< note title="" class="info" >}}
  有一个 名为 ```camunda-archetype-servlet-war``` 的 [Maven 模板]({{< ref "/user-guide/process-applications/maven-archetypes.md" >}})， 它为你提供了一个基于 ServletProcessApplication 的完整运行项目。
{{< /note >}}


## 在 JBoss 等 EJB/Java EE 容器中使用 ServletProcessApplication

你可以在 EJB/Java EE 容器（例如 JBoss）中使用 ServletProcessApplication。 Process Application 引导和部署将以相同的方式工作。 但是，你将无法在运行时使用所有 Java EE 功能。 与 `EjbProcessApplication`（见下一节）相比，`ServletProcessApplication` 没有执行正确的 Java EE 跨应用程序上下文切换。 当流程引擎从你的应用程序调用 Java Delegates 时，只有当前线程的 Context Class Loader 被设置为你的应用程序的类加载器。 这确实允许流程引擎从你的应用程序解析 Java Delegate 实现，但容器不会对你的应用程序执行 EE 上下文切换。 因此，如果你在 Java EE 容器内使用 ServletProcessApplciation，你将无法使用以下功能：

  * 将 CDI bean 和 EJB 作为 JavaDelegate 实现与 Job Executor 结合使用
  * 将 @RequestScoped CDI Beans 与 Job Executor 一起使用
  * 从应用程序的命名范围查找 JNDI 资源

如果你的应用程序不使用此类功能，则在 EE 容器内使用 ServletProcessApplication 完全没问题。 在这种情况下，你只能获得 servlet 规范保证。


# EjbProcessApplication

**支持:** JBoss/Wildfly. Java EE 6 容器或更高版本支持 EjbProcessApplication。 它在 Apache Tomcat 等 Servlet 容器上不受支持。 它可能适用于在 Java EE 5 Containers 中工作。

**打包格式:** JAR, WAR, EAR

EjbProcessApplication 是用于开发基于 Java EE 的 Process Application 的基类。 Ejb Process Application 类本身必须部署为 EJB。

要将 Ejb Process Application 添加到你的 Java 应用程序，你有两个选择：

  * **绑定 camunda-ejb-client**: 我们提供了一个通用的、可重用的 EjbProcessApplication 实现（名为 `org.camunda.bpm.application.impl.ejb.DefaultEjbProcessApplication`），捆绑为一个 maven 工件。 最简单的可能性是将此实现作为 maven 依赖项添加到你的应用程序中。
  * **实现自定义 EjbProcessApplication**: 如果你想自定义 EjbProcessApplication 的行为，你可以编写 EjbProcessApplication 类的自定义子类并将其添加到你的应用程序中。

下面将更详细地解释这两个选项。


## 绑定 camunda-ejb-client Jar

将 Process Application 部署到 Ejb 容器的最方便的选择是将以下 maven 依赖项添加到你的 maven 项目中：

{{< note title="" class="info" >}}
  请导入 [Camunda BOM](/get-started/apache-maven/) 以确保每个 Camunda 项目的版本正确。
{{< /note >}}

```xml
<dependency>
  <groupId>org.camunda.bpm.javaee</groupId>
  <artifactId>camunda-ejb-client</artifactId>
</dependency>
```

camunda-ejb-client 包含 EjbProcessApplication 的可重用默认实现，作为具有自动激活功能的单例会话 Bean。

此部署选项要求你的项目是复合部署（例如 WAR 或 EAR），因为你需要添加库 JAR 文件。 你当然可以使用 maven shade 插件之类的东西将包含在 camunda-ejb-client 工件中的类添加到基于 JAR 的部署中。

{{< note title="" class="info" >}}
  我们始终建议使用 camunda-ejb-client 而非部署自定义 EjbProcessApplication 类，除非你想自定义 EjbProcessApplication 的行为。

  有一个 名为 ```camunda-archetype-servlet-war``` 的 [Maven 模板]({{< ref "/user-guide/process-applications/maven-archetypes.md" >}})， 它为你提供了一个基于 ServletProcessApplication 的完整运行项目。
{{< /note >}}


## 部署自定义 EjbProcessApplication 类

如果你想自定义 EjbProcessApplication 类的行为，你可以选择编写自定义 EjbProcessApplication 类。 以下是此类实现的示例：

```java
@Singleton
@Startup
@ConcurrencyManagement(ConcurrencyManagementType.BEAN)
@TransactionAttribute(TransactionAttributeType.REQUIRED)
@ProcessApplication
@Local(ProcessApplicationInterface.class)
public class MyEjbProcessApplication extends EjbProcessApplication {

  @PostConstruct
  public void start() {
    deploy();
  }

  @PreDestroy
  public void stop() {
    undeploy();
  }

}
```


## 使用自定义 EjbProcessApplication 公开 Servlet 上下文路径

如果你的应用程序是一个 `WAR` （或者 `EAR` 中的 `WAR`） 并且你想在 [Tasklist]({{< ref "/webapps/tasklist/_index.md" >}}) 中使用 [嵌入式表单]({{< ref "/user-guide/task-forms/_index.md#embedded-task-forms" >}}) 或 [外部任务表单]({{< ref "/user-guide/task-forms/_index.md#external-task-forms" >}})，那么你自定义的 EjbProcessApplication 必须将应用程序的 servlet 上下文路径公开为属性。 这样 Tasklist 才能够解析嵌入或外部任务表单的路径。

因此，你的自定义 EjbProcessApplication 必须通过 `Map` 和该 `Map` 的getter 方法进行扩展，如下所示：

```java
@Singleton
@Startup
@ConcurrencyManagement(ConcurrencyManagementType.BEAN)
@TransactionAttribute(TransactionAttributeType.REQUIRED)
@ProcessApplication
@Local(ProcessApplicationInterface.class)
public class MyEjbProcessApplication extends EjbProcessApplication {

  protected Map<String, String> properties = new HashMap<String, String>();

  @PostConstruct
  public void start() {
    deploy();
  }

  @PreDestroy
  public void stop() {
    undeploy();
  }

  public Map<String, String> getProperties() {
    return properties;
  }

}
```

此外，要提供 servlet 上下文路径，必须将自定义 `javax.servlet.ServletContextListener` 添加到你的应用程序中。 在 `ServletContextListener` 的自定义实现中，你需要：

* 使用 `@EJB` 注解注入你自定义的 EjbProcessApplication
* 解析servlet上下文路径
* 通过自定义 EjbProcessApplication 中的 `ProcessApplicationInfo#PROP_SERVLET_CONTEXT_PATH` 属性公开 servlet 上下文路径。

这可以按如下方式实现：

```java
public class ProcessArchiveServletContextListener implements ServletContextListener {

  @EJB
  private ProcessApplicationInterface processApplication;

  public void contextInitialized(ServletContextEvent contextEvent) {

    String contextPath = contextEvent.getServletContext().getContextPath();

    Map<String, String> properties = processApplication.getProperties();
    properties.put(ProcessApplicationInfo.PROP_SERVLET_CONTEXT_PATH, contextPath);
  }

  public void contextDestroyed(ServletContextEvent arg0) {
  }

}
```
最后，自定义的 `ProcessArchiveServletContextListener` 必须添加到你的 `WEB-INF/web.xml` 文件中：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">

  <listener>
    <listener-class>org.my.project.ProcessArchiveServletContextListener</listener-class>
  </listener>

  ...

</web-app>
```


## EjbProcessApplication 的调用语义

The fact that the EjbProcessApplication exposes itself as a Session Bean Component inside the EJB container determines

 * the invocation semantics when invoking code from the process application and
 * the nature of the `ProcessApplicationReference` held by the process engine.

When the process engine invokes the Ejb process application, it gets EJB invocation semantics. For example, if your process application provides a `JavaDelegate` implementation, the process engine will call the EjbProcessApplication's `execute(Callable)` method and from that method invoke the `JavaDelegate`. This makes sure that

  * the call is intercepted by the EJB container and "enters" the process application legally.
  * the `JavaDelegate` may take advantage of the EjbProcessApplication's invocation context and resolve resources from the component's environment (such as a `java:comp/BeanManager`).

<pre>
                   Big pile of EJB interceptors
                                |
                                |  +--------------------+
                                |  |Process Application |
                  invoke        v  |                    |
 ProcessEngine ----------------OOOOO--> Java Delegate   |
                                   |                    |
                                   |                    |
                                   +--------------------+
</pre>

The EjbProcessApplication allows to hook into the invocation by overriding the `execute(Callable callable, InvocationContext invocationContext)` method. It provides the context of the current invocation (e.g., the execution) and can be used to execute custom code, for example initialize the security context before a service task is invoked.

```java
public class MyEjbProcessApplication extends EjbProcessApplication {

  @Override
  public <T> T execute(Callable<T> callable, InvocationContext invocationContext) {

    if(invocationContext != null) {
      // execute custom code (e.g. initialize the security context)
    }

    return execute(callable);
  }
}
```

When the EjbProcessApplication registers with a process engine (see `ManagementService#registerProcessApplication(String, ProcessApplicationReference)`, the process application passes a reference to itself to the process engine. This reference allows the process engine to reference the process application. The EjbProcessApplication takes advantage of the Ejb Containers naming context and passes a reference containing the EJBProcessApplication's Component Name to the process engine. Whenever the process engine needs access to process application, the actual component instance is looked up and invoked.

# EmbeddedProcessApplication

**支持：** JVM, Apache Tomcat, JBoss/Wildfly

**打包格式：** JAR, WAR, EAR

`org.camunda.bpm.application.impl.EmbeddedProcessApplication` 只能与嵌入式流程引擎结合使用。 不支持与共享流程引擎结合使用，因为该类在运行时不执行 Process Application 上下文切换。

嵌入式 Process Application 也不提供自启动。 你需要手动调用 Process Application 的 deploy 方法：

```java
// 实例化 process application
MyProcessApplication processApplication = new MyProcessApplication();

// 部署 process application
processApplication.deploy();

// 与流程引擎交互
ProcessEngine processEngine = BpmPlatform.getDefaultProcessEngine();
processEngine.getRuntimeService().startProcessInstanceByKey(...);

// 取消部署 process application
processApplication.undeploy();
```

“MyProcessApplication”类可能如下所示：

```java
@ProcessApplication(
    name="my-app",
    deploymentDescriptors={"path/to/my/processes.xml"}
)
public class MyProcessApplication extends EmbeddedProcessApplication {

}
```

请记住，要使手动管理的 EmbeddedProcessApplication 生效，你必须在 RuntimContainer 上注册你的 ProcessEngine：

```java
RuntimeContainerDelegate runtimeContainerDelegate = RuntimeContainerDelegate.INSTANCE.get();
runtimeContainerDelegate.registerProcessEngine(processEngine);
```


# SpringProcessApplication

**支持：** JVM, Apache Tomcat. The Spring process application is currently not supported on JBoss EAP 6

**打包类型：** JAR, WAR, EAR

`org.camunda.bpm.engine.spring.application.SpringProcessApplication` 类允许通过 Spring 应用程序上下文引导 Process Application 。 你可以从基于 XML 的应用程序上下文配置文件中引用 SpringProcessApplication 类，也可以使用基于注释的设置。

如果你的应用程序是 Web 应用程序，则应使用“org.camunda.bpm.engine.spring.application.SpringServletProcessApplication”，因为它支持通过“ProcessApplicationInfo#PROP_SERVLET_CONTEXT_PATH”属性公开 servlet 上下文路径。

我们建议始终使用 SpringServletProcessApplication，除非部署不是 Web 应用程序。 使用这个类需要 ```org.springframework:spring-web``` 模块在类路径上。



## 配置Spring Process应用程序

下面显示了如何在 Spring 应用程序上下文 XML 文件中引导 SpringProcessApplication 的示例：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="invoicePa" class="org.camunda.bpm.engine.spring.application.SpringServletProcessApplication" />

</beans>
```

请记住，你还需要一个“META-INF/processes.xml”文件。

> 如果你手动管理 processEngine，则必须按照 EmbeddedProcessEngine 部分中的说明在 RuntimeContainerDelegate 上注册它。


## Process Application 名称

SpringProcessApplication 将使用 bean 名称（上例中的`id="invoicePa"`）作为 Process Application 的自动检测名称。 确保在此处提供唯一的 Process Application 名称（在部署在单个应用程序服务器实例上的所有 Process Application 中唯一）。 作为替代方案，你可以提供 SpringProcessApplication（或 SpringServletProcessApplication）的自定义子类并覆盖 `getName()` 方法。


## 使用 Spring 配置托管流程引擎

如果你使用 Spring Process Application ，你可能希望在 Spring 应用程序上下文 Xml 文件（而不是 processes.xml 文件）中配置你的流程引擎。 在这种情况下，你必须使用 `org.camunda.bpm.engine.spring.container.ManagedProcessEngineFactoryBean` 类来创建流程引擎对象实例。 除了创建流程引擎对象之外，该实现还将流程引擎注册到 Camunda 平台基础架构，以便流程引擎由“ProcessEngineService”返回。 以下是如何使用 Spring 配置托管流程引擎的示例。

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="dataSource" class="org.springframework.jdbc.datasource.TransactionAwareDataSourceProxy">
        <property name="targetDataSource">
            <bean class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
                <property name="driverClass" value="org.h2.Driver"/>
                <property name="url" value="jdbc:h2:mem:camunda;DB_CLOSE_DELAY=1000"/>
                <property name="username" value="sa"/>
                <property name="password" value=""/>
            </bean>
        </property>
    </bean>

    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <bean id="processEngineConfiguration" class="org.camunda.bpm.engine.spring.SpringProcessEngineConfiguration">
        <property name="processEngineName" value="default" />
        <property name="dataSource" ref="dataSource"/>
        <property name="transactionManager" ref="transactionManager"/>
        <property name="databaseSchemaUpdate" value="true"/>
        <property name="jobExecutorActivate" value="false"/>
    </bean>

    <!-- 使用 ManagedProcessEngineFactoryBean 允许使用 BpmPlatform 注册 ProcessEngine -->
    <bean id="processEngine" class="org.camunda.bpm.engine.spring.container.ManagedProcessEngineFactoryBean">
        <property name="processEngineConfiguration" ref="processEngineConfiguration"/>
    </bean>

    <bean id="repositoryService" factory-bean="processEngine" factory-method="getRepositoryService"/>
    <bean id="runtimeService" factory-bean="processEngine" factory-method="getRuntimeService"/>
    <bean id="taskService" factory-bean="processEngine" factory-method="getTaskService"/>
    <bean id="historyService" factory-bean="processEngine" factory-method="getHistoryService"/>
    <bean id="managementService" factory-bean="processEngine" factory-method="getManagementService"/>

</beans>
```
