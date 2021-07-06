---

title: '多租户'
weight: 180

menu:
  main:
    identifier: "user-guide-process-engine-multi-tenancy"
    parent: "user-guide-process-engine"

---


*多租户* 是指一个单一的Camunda应用需要为多个的租户服务的情况。对于每个租户来说，应该有某些隔离的保证。例如，一个租户的流程实例不应干扰另一个租户的流程实例。

多租户可以通过两种不同的方式实现。一种是使用[每个租户一个流程引擎]({{< relref "#每个租户使用一个流程引擎" >}})。另一种方式是只使用一个流程引擎，并将数据与[租户标识符]({{< relref "#使用租户标识符的单一流程引擎" >}})相关联。这两种方式在数据隔离程度、维护工作和可扩展性方面各有不同。两种方式的组合也是可能的。

# 使用租户标识符的单一流程引擎

多租户可以使用租户标识符（即tenant-ids）的流程引擎来实现。所有租户的数据都存储在一个表中（同一数据库和表结构）。通过存储在列中的租户标识符来提供隔离。

{{< img src="../img/multi-tenancy-tenant-identifiers.png" title="One Process Engine with Tenant-Identifiers Architecture" >}}

租户标识符是在部署中指定的，并传播到从部署中创建的所有数据（例如，流程定义、流程实例、任务等）。为了访问特定租户的数据，流程引擎允许通过租户标识符过滤查询，或为命令（例如，创建流程实例）指定租户标识符。此外，流程引擎与身份服务相结合，提供透明的访问限制，允许省略租户标识符。

请注意，并非所有的API都实现了透明的租户分离。例如，通过部署API，一个租户可以为另一个租户部署一个流程。因此，直接向租户暴露这样的API端点并不是一个支持的用例。相反，应该在Camunda API的基础上建立自定义的访问检查逻辑。

所有租户也可以共享相同的流程和决策定义，而不需要为每个租户部署这些定义。在租户数量较多的情况下，共享定义可以简化对部署的管理。

{{< note title="案例" class="info" >}}
查看[GitHub上的案例](https://github.com/camunda/camunda-bpm-examples)查看如何使用租户标识符

* [Embedded Process Engine](https://github.com/camunda/camunda-bpm-examples/tree/master/multi-tenancy/tenant-identifier-embedded)
* [Shared Process Engine](https://github.com/camunda/camunda-bpm-examples/tree/master/multi-tenancy/tenant-identifier-shared)
{{< /note >}}


## 为租户部署定义

要为为租户部署定义，必须在部署中设置租户标识符。给定的标识符被传播到部署的所有定义，以便确认它们属于租户。

如果没有设置租户标识符，那么部署和它的定义属于所有租户。在这种情况下，所有租户都可以访问该部署和定义。参见[这一节]({{< relref "#所有租户共享的定义" >}})以阅读更多关于如何使用共享定义的信息。

### 通过Java API指定租户标识符

当使用存储库服务创建一个部署时，租户标识符可以在{{< javadocref page="?org/camunda/bpm/engine/repository/DeploymentBuilder.html" text="DeploymentBuilder" >}}中设置。

```java
repositoryService
  .createDeployment()
  .tenantId("tenant1")
  .addZipInputStream(inputStream)
  .deploy()
```

### 通过部署描述符指定租户标识符

如果一个流程应用程序，部署是由文件 [processes.xml]({{< ref "/user-guide/process-applications/the-processes-xml-deployment-descriptor.md" >}}) 定义的。由于描述符可以包含多个流程-档案（即部署），可以在每个流程-档案上设置`tenantId`租户标识符属性。

```xml
<process-application
  xmlns="http://www.camunda.org/schema/1.0/ProcessApplication"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <process-archive tenantId="tenant1">
    <process-engine>default</process-engine>
    <properties>
      <property name="isDeleteUponUndeploy">false</property>
      <property name="isScanForProcessDefinitions">true</property>
    </properties>
  </process-archive>

</process-application>
```

### 通过 Spring配置 指定租户标识符

如果使用Spring框架集成并使用[自动资源部署]({{< ref "/user-guide/spring-framework-integration/deployment.md" >}})时，租户标识符可以在流程引擎配置中指定为`deploymentTenantId`属性。

```xml
<bean id="processEngineConfiguration" class="org.camunda.bpm.engine.spring.SpringProcessEngineConfiguration">
  <property name="deploymentResources">
    <array>
      <value>classpath*:/org/camunda/bpm/engine/spring/test/autodeployment/autodeploy.*.cmmn</value>
      <value>classpath*:/org/camunda/bpm/engine/spring/test/autodeployment/autodeploy.*.bpmn20.xml</value>
    </array>
  </property>
  <property name="deploymentTenantId" value="tenant1" />
</bean>
```

### 租户特定定义的版本管理

当一个定义被部署到一个租户时，它被分配了一个版本，与其他租户的定义无关。例如，如果一个新的流程定义是为两个租户部署的，那么这两个定义都被分配为`1`版本。一个租户内的版本管理与不属于任何租户的[定义的版本管理]({{< ref "/user-guide/process-engine/process-versioning.md" >}})一样。

## 查询租户数据

流程引擎对租户特定数据的查询（例如，部署查询、流程定义查询）允许通过一个或多个租户标识符进行过滤。如果没有设置标识符，则结果包含所有租户的数据。

请注意，租户的[透明访问限制]({< relref "#transparent-access-restrictions-for-tenants" >}})可以影响查询的结果，如果用户不允许看到某个租户的数据。

### 查询租户的部署

要找到特定租户的部署，必须将租户标识符传递给`DeploymentQuery`的`tenantIdIn`。

```java
List<Deployment> deployments = repositoryService
  .createDeploymentQuery()
  .tenantIdIn("tenant1", "tenant2")
  .orderByTenantId()
  .asc()
  .list();
```

在[共享定义]({{< relref "#shared-definitions-for-all-tenants" >}})的情况下，通过调用`withoutTenantId()`来查询不属于任何租户的部署。

```java
List<Deployment> deployments = repositoryService
  .createDeploymentQuery()
  .withoutTenantId()
  .list();
```

也可以通过调用`includeDeploymentsWithoutTenantId()`来查询属于特定租户或不属于租户的部署。

```java
List<Deployment> deployments = repositoryService
  .createDeploymentQuery()
  .tenantIdIn("tenant1")
  .includeDeploymentsWithoutTenantId()
  .list();
```

### 查询租户的定义

与 "部署查询" 类似，定义查询允许通过一个或多个租户和不属于任何租户的定义进行过滤。

```java
List<ProcessDefinition> processDefinitions = repositoryService
  .createProcessDefinitionQuery()
  .tenantIdIn("tenant1")
  .includeProcessDefinitionsWithoutTenantId();
  .list();
```

## 使用租户身份执行命令

当一个定义被部署到多个租户时，一个命令可能是模棱两可的（例如，通过key启动一个流程实例）。如果这样的命令被执行，就会抛出一个 "ProcessEngineException"。为了成功运行该命令，必须将租户标识符传递给该命令。

注意，如果用户只被允许看到其中一个定义，租户的[透明访问限制]({{< relref "#租户的透明访问限制" >}})可以省略租户标识符。

### 创建一个流程实例

通过key创建一个为多租户部署的流程定义的实例，必须在{{< javadocref page="?org/camunda/bpm/engine/runtime/ProcessInstantiationBuilder.html" text="ProcessInstantiationBuilder" >}}中传递租户标识符 。

```java
runtimeService
  .createProcessInstanceByKey("key")
  .processDefinitionTenantId("tenant1")
  .execute();
```

### 关联消息

[message API]({{< ref "/reference/bpmn20/events/message-events.md#message-api" >}})可用于将消息与一个或所有租户相关联。如果消息可以与多个租户的定义或执行相关，则必须将租户标识传递给 {{< javadocref page="?org/camunda/bpm/engine/runtime/MessageCorrelationBuilder.html" text="MessageCorrelationBuilder" >}}。否则会抛出 `MismatchingMessageCorrelationException`。

```java
runtimeService
  .createMessageCorrelation("messageName")
  .tenantId("tenant1")
  .correlate();
```

要将一条消息与所有租户相关联，不需要向构建器传递租户标识符，直接调用`correlateAll()`。

```java
runtimeService
  .createMessageCorrelation("messageName")
  .correlateAll();
```

### 发送信号

[Signal API]({{< ref "/reference/bpmn20/events/signal-events.md#signal-api" >}}) 可用于向一个或所有租户发送信号。可以将租户标识传递给 {{< javadocref page="?org/camunda/bpm/engine/runtime/SignalEventReceivedBuilder.html" text="SignalEventReceivedBuilder" >}} 用该来将信号传递给特定的租户。如果没有传递标识符，则信号将传递给所有租户。

```java
runtimeService
  .createSignalEvent("signalName")
  .tenantId("tenant1")
  .send();
```

当一个信号在流程中被抛出时（即中间信号事件或信号结束事件），那么该信号将被传递给与调用执行属于同一租户或无租户的定义和执行中。

### 创建一个案例实例

要通过部署为多个租户的案例定义的密钥来创建实例，必须将租户标识符传递给 {{< javadocref page="?org/camunda/bpm/engine/runtime/CaseInstanceBuilder.html" text="CaseInstanceBuilder" >}}。

```java
caseService
  .withCaseDefinitionByKey("key")
  .caseDefinitionTenantId("tenant1")
  .execute();
```

### 评估决策表

要通过部署为多个租户的密钥来评估决策表，必须将租户标识符传递给 {{< javadocref page="?org/camunda/bpm/engine/dmn/DecisionEvaluationBuilder.html" text="DecisionEvaluationBuilder" >}}。

```java
decisionService
  .evaluateDecisionTableByKey("key")
  .decisionDefinitionTenantId("tenant1")
  .evaluate();
```

## 租户的透明访问限制

当把Camunda集成到一个应用程序中时，在每个camunda API调用中传递租户ID可能是很麻烦的。由于这样的应用程序通常也有一个 "认证用户" 的概念，因此可以在设置认证时设置租户ID的列表。

```java
try {
  identityService.setAuthentication("mary", asList("accounting"), asList("tenant1"));

  // 此处执行的所有API调用的租户ID都会是`tenant1`

}
finally {
  identityService.clearAuthentication();
}
```

在上面的例子中，在 "setAuthentication(...) " 和 "clearAuthentication()" 之间执行的所有API调用都是根据租户ID列表以透明方式执行的.

### 查询案例

下面的查询：

```java
try {
  identityService.setAuthentication("mary", asList("accounting"), asList("tenant1"));

  repositoryService.createProcessDefinitionQuery().list();
}
finally {
  identityService.clearAuthentication();
}
```

相当于：

```java
repositoryService.createProcessDefinitionQuery()
  .tenantIdIn("tenant1")
  .includeProcessDefinitionsWithoutTenantId()
  .list();
```

### 任务（Task）访问示例

对于像 `complete()` 这样的其他命令，透明的访问检查确保被认证的用户不会访问其他租户的资源：

```java
try {
  identityService.setAuthentication("mary", asList("accounting"), asList("tenant1"));

  // 如果租户ID不是 "tenant1" ，会抛出异常
  taskService.complete("someTaskId");
}
finally {
  identityService.clearAuthentication();
}
```

### 从身份服务（Identity Service）获取用户的租户ID

流程引擎的身份服务可以用来管理用户、组和租户以及他们的关系。
下面的例子显示了如何为一个给定的用户查询组和租户的列表，然后在设置认证时使用这些列表。

```java
List<Tenant> groups = identityService.createGroupQuery()
  .userMember(userId)
  .list();

List<Tenant> tenants = identityService.createTenantQuery()
  .userMember(userId)
  .includingGroupsOfUser(true)
  .list();

try {
  identityService.setAuthentication(userId, groups, tenants);

  // 获取用户可见的所有任务。
  taskService.createTaskQuery().list();
  
}
finally {
  identityService.clearAuthentication();
}
```

{{< note title="LDAP身份服务" class="info" >}}
上面的示例仅适用于[数据库身份服务]({{< ref "/user-guide/process-engine/identity-service.md#the-database-identity-service" >}}) (即, 默认实现). [LDAP身份服务]({{< ref "/user-guide/process-engine/identity-service.md#the-ldap-identity-service" >}})不支持租户。
{{< /note >}}  


### Camunda REST API和Web应用程序

Camunda [Rest API]({{< ref "/reference/rest/_index.md" >}})和Web应用程序Cockpit和Tasklist支持透明访问限制。当一个用户登录时，他只能看到也只能访问属于他的一个租户的数据（例如，流程定义）。

租户及其成员资格可以在[Admin]({{< ref "/webapps/admin/tenant-management.md" >}})网络应用中管理。

### 禁用透明访问限制

默认情况下，透明访问限制是启用的。要禁用这些限制，请在[ProcessEngineConfiguration]({{< ref "/user-guide/process-engine/process-engine-bootstrapping.md#processengineconfiguration-bean" >}})中设置`tenantCheckEnabled`属性为`false`。

此外，也可以禁用单个命令的限制（例如，为维护任务）。使用`CommandContext`来禁用和启用对当前命令的限制。

```java
commandContext.disableTenantCheck();

// 例如，为所有租户做维护工作

commandContext.enableTenantCheck();
```

注意，如果在 "ProcessEngineConfiguration" 中禁用了这些限制，就不能为一个命令启用限制了。

### 作为管理员访问所有租户

作为 "camunda-admin" 组成员的用户可以访问所有租户的数据，即使他们不属于这些租户。这对一个多租户应用程序的管理员很有用，因为他必须管理所有租户的数据。

## 所有租户共享的定义

在[为租户部署定义](#为租户部署定义)一节中，解释了如何为某一租户部署流程定义或决策定义。其结果是，该定义只对其被部署的租户可见，而对其他租户不可见。如果租户有不同的流程和决策，这很有用。然而，也有许多情况下，所有租户应该共享相同的定义。在这种情况下，最好是只部署一次定义，使其对所有租户可见。
然后，当某一租户创建一个新的实例时，它应该只对该租户（当然还有管理员）可见。
这可以通过一种我们称之为 "共享定义" 的使用模式来实现。
我们所说的 *使用模式* 是指它不是Camunda本身的一个功能，而是使用它来实现所需行为的特定方式。

{{< note title="案例" class="info" >}}
你可以查看在[GitHub](https://github.com/camunda/camunda-bpm-examples)上的[案例](https://github.com/camunda/camunda-bpm-examples/tree/master/multi-tenancy/tenant-identifier-shared-definitions) 学习如何使用共享定义。
{{< /note >}}

### 部署共享定义

部署一个共享定义只是一个 "常规" 的部署，而不用给部署分配一个租户身份。

```java
repositoryService
  .createDeployment()
  .addClasspathResource("processes/default/mainProcess.bpmn")
  .addClasspathResource("processes/default/subProcess.bpmn")
  .deploy();
```

### 在查询中包含共享的定义

在一个应用程序中，我们经常希望向用户提供一个 "可用" 的流程定义的列表。
在一个共享资源的多租户环境中，我们希望这个列表包括具有以下属性的定义。

* 租户ID是当前用户的租户ID。
* 租户id为`null` = 流程是一个共享资源。

为了通过查询实现这一点，查询需要对用户的租户ID列表进行限制（通过调用`tenantIdIn(..)`），并包括没有租户ID的定义（`includeProcessDefinitionsWithoutTenantId()`）。或者，反过来看：排除所有租户ID与当前用户的租户ID不同的定义。

实例:

```java
repositoryService.createProcessDefinitionQuery()
  .tenantIdIn("someTenantId")
  .includeProcessDefinitionsWithoutTenantId()
  .list();
```

### 实例化共享定义

当创建（启动）一个新的流程实例时，流程定义的租户ID被传播到流程实例中。
共享资源没有租户ID，这意味着没有租户ID被自动传播。为了将启动流程实例的用户的租户ID分配给流程实例，需要提供{{< javadocref page="?org/camunda/bpm/engine/impl/cfg/multitenancy/TenantIdProvider.html" text="TenantIdProvider" >}}SPI的实现。

当一个流程定义、案例定义或决策定义的实例被创建时，`TenantIdProvider` 会收到一个回调。然后它可以为新创建的实例分配一个租户ID（或者不分配）。

下面的例子显示了如何根据当前的认证给一个实例分配租户ID：

```java
public class CustomTenantIdProvider implements TenantIdProvider {

  @Override
  public String provideTenantIdForProcessInstance(TenantIdProviderProcessInstanceContext ctx) {
    return getTenantIdOfCurrentAuthentication();
  }

  @Override
  public String provideTenantIdForCaseInstance(TenantIdProviderCaseInstanceContext ctx) {
    return getTenantIdOfCurrentAuthentication();
  }

  @Override
  public String provideTenantIdForHistoricDecisionInstance(TenantIdProviderHistoricDecisionInstanceContext ctx) {
    return getTenantIdOfCurrentAuthentication();
  }

  protected String getTenantIdOfCurrentAuthentication() {

    IdentityService identityService = Context.getProcessEngineConfiguration().getIdentityService();
    Authentication currentAuthentication = identityService.getCurrentAuthentication();

    if (currentAuthentication != null) {

      List<String> tenantIds = currentAuthentication.getTenantIds();
      if (tenantIds.size() == 1) {
        return tenantIds.get(0);

      } else if (tenantIds.isEmpty()) {
        throw new IllegalStateException("no authenticated tenant");

      } else {
        throw new IllegalStateException("more than one authenticated tenant");
      }

    } else {
      throw new IllegalStateException("no authentication");
    }
  }

}
```

使用 `TenantIdProvider`，必须在流程引擎配置中设置，例如使用 `camunda.cfg.xml`:

```xml
<beans>
  <bean id="processEngineConfiguration" class="org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration">
    <!-- ... -->
    
    <property name="tenantIdProvider" ref="tenantIdProvider" />
  </bean>
  
  <bean id="tenantIdProvider" class="org.camunda.bpm.CustomTenantIdProvider">
</beans>
```

如果是共享的流程引擎，提供者可以通过[流程引擎插件]({{< ref "/user-guide/process-engine/process-engine-plugins.md" >}})来设置。

### 租户的特定行为与Call Activities

到目前为止，我们已经看到，如果租户有相同的流程定义，共享资源是一种有用的模式。这样做的好处是，我们不必为每个租户部署一次相同的流程定义。然而，在许多现实世界的应用中，情况有些介于两者之间：租户共享 *大程度* 上相同的流程定义，但有一些租户有具体变化。

处理这种情况的常见方法是，在一个单独的流程中提取租户特定的行为，然后使用Call Activities来调用。使用业务规则任务的特定租户决策逻辑（即决策表）也很常见。

为了实现这一点，Call Activities或业务规则任务需要根据当前流程实例的租户ID 选择要调用的正确定义。[共享资源示例](https://github.com/camunda/camunda-bpm-examples/tree/master/multi-tenancy/tenant-identifier-shared-definitions)展示了如何实现这一点。

也可以看看：

* [共享资源示例](https://github.com/camunda/camunda-bpm-examples/tree/master/multi-tenancy/tenant-identifier-shared-definitions)
* [Called Element Tenant Id]({{< ref "/reference/bpmn20/subprocesses/call-activity.md#calledelement-tenant-id" >}})
* [案例租户ID]({{< ref "/reference/bpmn20/subprocesses/call-activity.md#case-tenant-id" >}}) for call activities.
* [Decision Ref Tenant Id]({{< ref "/reference/bpmn20/tasks/business-rule-task.md#decisionref-tenant-id" >}}) for business rule tasks.

# 每个租户使用一个流程引擎

多租户可以通过为每个租户提供一个流程引擎来实现。每个流程引擎被配置为使用不同的数据源，连接租户的数据。租户的数据可以存储在不同的数据库中，也可以存储在具有不同schema的一个数据库中，或者存储在具有不同表的一个schema中。

{{< img src="../../../introduction/img/multi-tenancy-process-engine.png" title="One Process Engine per Tenant Architecture" >}}

流程引擎可以在同一台服务器上运行，这样所有的流程引擎都可以共享相同的计算资源，如数据源（通过schema或表进行隔离时）或用于异步Job执行的线程池。

{{< note title="教程" class="info" >}}
  你可以查看 [案例](https://github.com/camunda/camunda-bpm-examples/tree/master/multi-tenancy/schema-isolation) 了解如何通过schema实现多租户。
{{< /note >}}

## 配置流程引擎

流程引擎可以在配置文件中或通过 Java API 进行配置。每个引擎都应该有一个与租户有关的名字，这样就可以根据租户来识别它。例如，每个引擎可以用它所服务的租户来命名。详情见[流程引擎自启动]({{< ref "/user-guide/process-engine/process-engine-bootstrapping.md" >}})章节。

### 数据库隔离

如果不同的租户应该在完全不同的数据库上工作，他们必须使用不同的JDBC设置或不同的数据源。

### Schema or Table Isolation

对于基于schema或表的隔离，可以使用单一的数据源，这意味着连接池等资源可以在多个引擎之间共享。

为了实现这一点，你需要：

* 配置选项[databaseTablePrefix]({{<ref "/reference/deployment-descriptors/tags/process-engine.md#configuration-protperties">}})可用于配置数据库访问。
* 考虑打开 `useSharedSqlSessionFactory` 配置项。该设置控制每个流程引擎实例是否应该解析和维护mybatis映射文件的本地副本，或者是否可以使用单一的共享副本。由于映射需要大量的堆（>30MB），建议将其打开。这样就只需要分配一个副本。

{{< note title="考虑使用 useSharedSqlSessionFactory 设置" class="warning" >}}
`useSharedSqlSessionFactory` 设置会导致mybatis sql事务工厂在一个静态字段中的缓存，一旦建立。
当使用这个配置设置时，你需要注意的是：

* 只有当所有使用该设置的流程引擎共享相同的数据源和事务工厂时，才能使用它。
* 字段中的引用，一旦被设置，就不会被清除。这通常不是一个问题，但如果是的话，用户必须通过以下方式手动清除该字段，明确地将其设置为空。

```java
ProcessEngineConfigurationImpl.cachedSqlSessionFactory = null
```
{{< /note >}}

### 多个流程引擎的Job执行器

对于流程和任务的后台执行，流程引擎有一个叫做[Job执行器]({{< ref "/user-guide/process-engine/the-job-executor.md" >}})的组件。Job执行器定期从数据库中获取Job，并将其提交到线程池中执行。对于一台服务器上的所有流程应用，一个线程池用于Job执行。此外，有可能在多个引擎之间共享获取线程。这样，即使使用了大量的流程引擎，资源仍然是可管理的。详情请参见[Job执行器与多个流程引擎]({{< ref "/user-guide/process-engine/the-job-executor.md#the-job-executor-and-multiple-process-engines" >}})一节。

### Schem隔离的示例配置

多租户设置可以应用于配置流程引擎的各种方式。下面是一个[bpm-platform.xml]({{< ref "/user-guide/process-engine/process-engine-bootstrapping.md#configure-process-engine-in-bpm-platformxml" >}})文件的例子，它为两个共享同一数据库但工作在不同模式下的租户指定引擎：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bpm-platform xmlns="http://www.camunda.org/schema/1.0/BpmPlatform"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://www.camunda.org/schema/1.0/BpmPlatform http://www.camunda.org/schema/1.0/BpmPlatform">

  <job-executor>
    <job-acquisition name="default" />
  </job-executor>

  <process-engine name="tenant1">
    <job-acquisition>default</job-acquisition>
    <configuration>org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration</configuration>
    <datasource>java:jdbc/ProcessEngine</datasource>

    <properties>
      <property name="databaseTablePrefix">TENANT_1.</property>

      <property name="history">full</property>
      <property name="databaseSchemaUpdate">true</property>
      <property name="authorizationEnabled">true</property>
      <property name="useSharedSqlSessionFactory">true</property>
    </properties>
  </process-engine>

  <process-engine name="tenant2">
    <job-acquisition>default</job-acquisition>
    <configuration>org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration</configuration>
    <datasource>java:jdbc/ProcessEngine</datasource>

    <properties>
      <property name="databaseTablePrefix">TENANT_2.</property>

      <property name="history">full</property>
      <property name="databaseSchemaUpdate">true</property>
      <property name="authorizationEnabled">true</property>
      <property name="useSharedSqlSessionFactory">true</property>
    </properties>
  </process-engine>
</bpm-platform>
```


## 部署租户的定义

在开发流程应用，即流程定义和补充代码时，一些流程可能会被部署到每个租户的引擎上，而另一些则是针对租户的。processes.xml部署描述符是每个流程应用程序的一部分，它通过 *流程归档* 的概念提供这种灵活性。一个应用程序可以包含任何数量的流程归档部署，每一个都可以部署到具有不同资源的不同流程引擎。详情见[process.xml部署描述符]({{< ref "/user-guide/process-applications/the-processes-xml-deployment-descriptor.md" >}})一节。

下面是一个为两个租户部署不同流程定义的例子。它使用配置属性`resourceRootPath`，指定部署中包含要部署的流程定义的路径。因此，应用程序classpath上`processes/tenant1`下的所有流程被部署到引擎`tenant1`，而`processes/tenant2`下的所有流程被部署到引擎`tenant2`。

```xml
<process-application
  xmlns="http://www.camunda.org/schema/1.0/ProcessApplication"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <process-archive name="tenant1-archive">
    <process-engine>tenant1</process-engine>
    <properties>
      <property name="resourceRootPath">classpath:processes/tenant1/</property>

      <property name="isDeleteUponUndeploy">false</property>
      <property name="isScanForProcessDefinitions">true</property>
    </properties>
  </process-archive>

  <process-archive name="tenant2-archive">
    <process-engine>tenant2</process-engine>
    <properties>
      <property name="resourceRootPath">classpath:processes/tenant2/</property>

      <property name="isDeleteUponUndeploy">false</property>
      <property name="isScanForProcessDefinitions">true</property>
    </properties>
  </process-archive>

</process-application>
```


## 访问租户的流程引擎

要在运行时访问一个特定租户的流程引擎，必须通过其名称来识别。Camunda引擎提供对各种编程模型中的命名引擎的访问。

* **清晰的 Java API**: 通过 [ProcessEngineService]({{< ref "/user-guide/runtime-container-integration/bpm-platform-services.md#processengineservice" >}}) 可以访问任何命名引擎。
* **CDI 注入**: 命名的引擎Bean可以开箱即注入。内置的[CDI Bean生产者]({{< ref "/user-guide/cdi-java-ee-integration/built-in-beans.md" >}})专门用来动态地访问当前租户的引擎。
* **通过JBoss/Wildfly 的 JNDI**: 在JBoss和Wildfly上，每一个容器管理的流程引擎都可以[通过JNDI查询]({{< ref "/user-guide/runtime-container-integration/jboss.md#look-up-a-process-engine-in-jndi" >}})。

Camunda网络应用程序Cockpit、Tasklist和Admin通过[在不同的流程引擎之间切换]({{< ref "/webapps/cockpit/dashboard.md#multi-engine" >}})提供租户特定的视图，开箱即用。

