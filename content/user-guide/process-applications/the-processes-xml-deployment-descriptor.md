---

title: 'processes.xml 部署配置文件'
weight: 20

menu:
  main:
    identifier: "user-guide-process-application-descriptor"
    parent: "user-guide-process-applications"

---


processes.xml 部署描述符包含流程应用程序的部署元数据。 以下示例是一个简单的 `processes.xml` 部署描述符示例：

```xml
<process-application
  xmlns="http://www.camunda.org/schema/1.0/ProcessApplication"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <process-archive name="loan-approval">
    <process-engine>default</process-engine>
    <properties>
      <property name="isDeleteUponUndeploy">false</property>
      <property name="isScanForProcessDefinitions">true</property>
    </properties>
  </process-archive>

</process-application>
```

声明了单个部署（流程档案）。 流程档案名为 *loan-approval*，并以名称 *default* 部署到流程引擎。 指定了两个附加属性：

  * `isDeleteUponUndeploy`: 此属性控制流程应用程序的取消部署是否需要从数据库中删除流程引擎部署。 默认设置为假。 如果此属性设置为 true，则取消部署流程应用程序会导致从数据库中删除部署（包括流程实例）。
  * `isScanForProcessDefinitions`: 如果此属性设置为 true，则会自动扫描流程应用程序的类路径以查找可部署资源。 可部署的资源必须以 `.bpmn20.xml`、`.bpmn`、`.cmmn11.xml`、`.cmmn`、`.dmn11.xml` 或 `.dmn` 结尾。

参阅 [部署描述符参考文档]({{< ref "/reference/deployment-descriptors/descriptors/processes-xml.md" >}}) 了解有关“processes.xml”文件的语法。


# 空 processes.xml

processes.xml 可以选择为空（留空）。 在这种情况下，使用默认值。 空的processes.xml对应如下配置：

```xml
<process-application
  xmlns="http://www.camunda.org/schema/1.0/ProcessApplication"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <process-archive>
    <properties>
      <property name="isDeleteUponUndeploy">false</property>
      <property name="isScanForProcessDefinitions">true</property>
    </properties>
  </process-archive>

</process-application>
```

空的 processes.xml 将扫描流程定义并对默认流程引擎执行单一部署。


# processes.xml 文件的位置

processes.xml 文件的默认位置是`META-INF/processes.xml`。 Camunda 平台将解析和处理流程应用程序类路径上的所有 processes.xml 文件。 复合流程应用程序（WAR / EAR）可以携带多个子部署，提供一个 META-INF/processes.xml 文件。

在基于 apache maven 的项目中，将 processes.xml 文件添加到 `src/main/resources/META-INF` 文件夹。


# processes.xml 文件的自定义位置

如果要为 processes.xml 文件指定自定义位置，则需要使用 `@ProcessApplication` 注释的 `deploymentDescriptors` 属性：

```java
@ProcessApplication(
    name="my-app",
    deploymentDescriptors={"path/to/my/processes.xml"}
)
public class MyProcessApp extends ServletProcessApplication {

}
```

提供的路径必须可通过流程应用程序的 `AbstractProcessApplication#getProcessApplicationClassloader()` 方法返回的类加载器的 `ClassLoader#getResourceAsStream(String)` 方法解析。

支持多个不同的位置。


# 在 processes.xml 文件中配置流程引擎

processes.xml 文件也可用于配置一个或多个流程引擎。 以下是流程引擎在 processes.xml 文件中的配置示例：

```xml
<process-application
xmlns="http://www.camunda.org/schema/1.0/ProcessApplication"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <process-engine name="my-engine">
    <configuration>org.camunda.bpm.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration</configuration>
  </process-engine>

  <process-archive name="loan-approval">
    <process-engine>my-engine</process-engine>
    <properties>
      <property name="isDeleteUponUndeploy">false</property>
      <property name="isScanForProcessDefinitions">true</property>
    </properties>
  </process-archive>

</process-application>
```

`<configuration>...</configuration>` 属性允许指定在构建流程引擎时要使用的流程引擎配置类的名称。

# 在 processes.xml 文件中为流程档案指定 租户ID

对于[带租户标识符的多租户]({{< ref "/user-guide/process-engine/multi-tenancy.md#single-process-engine-with-tenant-identifiers" >}})，你可以 通过设置属性“tenantId”来指定流程档案的租户ID。 如果设置了租户ID，则将为给定的租户ID 部署所有包含资源。 以下是一个 processes.xml 文件的示例，其中包含一个具有租户ID 的流程档案：

```xml
<process-application
xmlns="http://www.camunda.org/schema/1.0/ProcessApplication"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <process-archive name="loan-approval" tenantId="tenant1">
    <process-engine>default</process-engine>
    <properties>
      <property name="isDeleteUponUndeploy">false</property>
      <property name="isScanForProcessDefinitions">false</property>
    </properties>
  </process-archive>

</process-application>
```

请注意，processes.xml 文件可以包含具有不同 租户ID 的多个流程档案。

# 流程应用部署

BPMN 2.0 文件部署到流程引擎时，会创建流程部署。通过将流程部署到引擎数据库，可以在流程引擎停止和重新启动时，从数据库中恢复流程定义并继续执行。当流程应用程序执行部署时，除了数据库部署之外，还将为流程引擎创建此部署的注册。如下图所示：


{{< img src="../img/process-application-deployment.png" title="Process Application Deployment" >}}

流程应用程序“invoice.war”的部署显示在左侧：

1. 流程应用程序 “invoice.war” 将 invoice.bpmn 文件部署到流程引擎。
2. 流程引擎检查数据库中是否有之前的部署。如果不存在此类数据库部署。因此，为流程定义创建了一个新的数据库部署 `deployment-1`。
3. 流程申请注册为 `deployment-1` 并返回注册。

取消部署流程应用程序时，会删除部署注册（请参见上图的右侧）。清除注册后，部署仍然存在于数据库中。

注册允许流程引擎在执行流程时从流程应用程序加载额外的 Java 类和资源。与可以在流程引擎重新启动时恢复的数据库部署相比，流程应用程序的注册保持在内存中状态。这种内存状态对于单个集群节点来说是本地的，允许我们在特定集群节点上取消部署或重新部署流程应用程序，而不会影响其他节点，也无需重新启动流程引擎。如果 Job Executor 具有部署感知能力，则此流程应用程序创建的Job的Job执行也将停止。但是，因此，在重新启动应用程序服务器时，也需要重新创建注册。如果流程应用程序参与应用程序服务器部署生命周期，这将自动发生。例如，ServletProcessApplications 被部署为 ServletContextListeners，当 servlet 上下文启动时，它会创建部署并注册到流程引擎。下图说明了重新部署过程：

{{< img src="../img/process-application-redeployment.png" title="Process Application Redeployment" >}}

(a) 左边：invoice.bpmn 没有改变：

1. 流程应用程序“invoice.war”将 invoice.bpmn 文件部署到流程引擎。
2. 流程引擎检查数据库中是否有先前的部署。 由于`deployment-1` 仍然存在于数据库中，流程引擎将数据库部署的xml 内容与流程应用程序中的 bpmn20.xml 文件进行比较。 在这种情况下，两个 xml 文档是相同的，这意味着可以恢复现有部署。
3. 流程应用程序为现有部署 `deployment-1` 注册。

(b) 右边: invoice.bpmn 改变了:

1. 流程应用程序“invoice.war”将invoice.bpmn 文件部署到流程引擎。
2. 流程引擎检查数据库中是否有先前的部署。 由于`deployment-1` 仍然存在于数据库中，流程引擎将数据库部署的xml 内容与流程应用程序中的invoice.bpmn 文件进行比较。 在这种情况下，会检测到更改，这意味着必须创建新部署。
3. 流程引擎创建一个新的部署 `deployment-2`，其中包含更新的 invoice.bpmn 流程。
3. 流程应用程序为新部署`deployment-2` 和现有部署`deployment-1` 注册。

恢复之前的部署（deployment-1）是一个名为 `resumePreviousVersions` 的特性，默认是激活的。 如何恢复以前的部署有两种不同的可能性。

第一个是默认方式，是根据流程定义键解析先前的部署。 根据你使用流程应用程序部署的流程，所有包含具有相同密钥的流程定义的部署都将恢复。

第二个选项是根据部署名称（更准确地说是流程存档的`name` 属性的值）恢复部署。 这样，你可以删除新部署中的流程，但流程应用程序将为以前的部署注册自己，因此也会为已删除的流程注册。 这使得已删除流程的正在运行的流程实例可以为该流程应用程序继续运行。

要激活此行为，你已将属性 `isResumePreviousVersions` 设置为 true，并将属性 `resumePreviousBy` 设置为 `deployment-name`：

```xml
<process-application
  xmlns="http://www.camunda.org/schema/1.0/ProcessApplication"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <process-archive name="loan-approval">
    ...
    <properties>
      ...
      <property name="isResumePreviousVersions">true</property>
      <property name="resumePreviousBy">deployment-name</property>
    </properties>
  </process-archive>

</process-application>
```

如果要停用此功能，则必须在 processes.xml 文件中将该属性设置为 `false`：

```xml
<process-application
  xmlns="http://www.camunda.org/schema/1.0/ProcessApplication"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <process-archive name="loan-approval">
    ...
    <properties>
      ...
      <property name="isResumePreviousVersions">false</property>
    </properties>
  </process-archive>

</process-application>
```
