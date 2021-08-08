---

title: 'Apache Tomcat 整合'
weight: 30

menu:
  main:
    name: "Apache Tomcat"
    identifier: "user-guide-runtime-container-integration-tomcat"
    parent: "user-guide-runtime-container-integration"

---


# JNDI 绑定

要在 Apache Tomcat 上使用 Camunda 平台服务的 JNDI 绑定，你必须将文件“META-INF/context.xml”添加到流程应用程序并添加以下 [ResourceLinks](http://tomcat.apache.org/tomcat-9.0-doc/config/context.html#Resource_Links):

```xml
<Context>
  <ResourceLink name="ProcessEngineService"
    global="global/camunda-bpm-platform/process-engine/ProcessEngineService!org.camunda.bpm.ProcessEngineService"
    type="org.camunda.bpm.ProcessEngineService" />

  <ResourceLink name="ProcessApplicationService"
    global="global/camunda-bpm-platform/process-engine/ProcessApplicationService!org.camunda.bpm.ProcessApplicationService"
    type="org.camunda.bpm.ProcessApplicationService" />
</Context>
```

这些元素用于创建到在`$TOMCAT_HOME/conf/server.xml` 中定义的全局 JNDI 资源的链接。

此外，在`WEB-INF/web.xml` 部署描述符中声明对JNDI 绑定的依赖。

```xml
<web-app>
  <resource-ref>
    <description>Process Engine Service</description>
    <res-ref-name>ProcessEngineService</res-ref-name>
    <res-type>org.camunda.bpm.ProcessEngineService</res-type>
    <res-auth>Container</res-auth>
  </resource-ref>
  <resource-ref>
    <description>Process Application Service</description>
    <res-ref-name>ProcessApplicationService</res-ref-name>
    <res-type>org.camunda.bpm.ProcessApplicationService</res-type>
    <res-auth>Container</res-auth>
  </resource-ref>
  ...
</web-app>
```

**注意**: 你可以为流程引擎服务和流程应用服务选择不同的资源链接名。资源链接名必须与“WEB-INF/web.xml”中相应“<resource-ref>”元素内的“<res-ref-name>”元素的值相匹配。我们建议流程引擎服务的名为“ProcessEngineService”，流程应用程序服务的名为“ProcessApplicationService”。

要查找 Camunda 平台服务，你必须使用资源链接名称来获取链接的全局资源。 例如：

* Process Engine Service: `java:comp/env/ProcessEngineService`
* Process Application Service: `java:comp/env/ProcessApplicationService`

如果你声明了我们建议的其他资源链接名称，则必须使用`java:comp/env/$YOUR_RESOURCE_LINK_NAME` 进行查找以获取相应的 Camunda 平台服务。


# Job执行器配置

## Tomcat 默认 Job执行器

Apache Tomcat 9.x 上的 Camunda 平台使用默认Job执行程序。 默认的 [job执行器]({{< ref "/user-guide/process-engine/the-job-executor.md" >}}) 使用 ThreadPoolExecutor 来管理线程池和Job队列。

核心池大小、队列大小、最大池大小和保持活动时间可以在 `bpm-platform.xml` 中配置。
配置job获取后，可以在`<properties>`标签下设置值。语法可以参考[文档]({{< ref "/reference/deployment-descriptors/tags/job-executor.md" >}}) 。

除了队列大小之外，前面提到的所有属性都可以在运行时通过 JMX 客户端进行修改。


## 核心池大小

ThreadPoolExecutor 会自动调整线程池的大小。 线程池中的线程数将趋向于与设置为核心池大小的线程数达到平衡。
如果一个新的Job提交给Job执行器并且池中的线程总数小于核心，那么将创建一个新线程。 因此，在初次使用时，线程池中的线程数将增加到核心线程数。

* 默认核心池大小为 3 。


## 队列大小

ThreadPoolExecutor 包括一个用于缓冲Job的Job队列。 一旦达到（并正在使用）线程的核心数量，提交给Job执行器的新Job将导致该Job被添加到 ThreadPoolExecutor Job队列。

* Job队列的默认最大长度为 3 。


## 最大池大小

如果队列的长度超过最大队列大小，并且线程池中的线程数小于最大池大小，则将额外的线程添加到线程池中。 这将一直持续到池中的线程数等于最大池大小：

* 默认的最大池大小为 10 。


## keepalive

如果线程在线程池中空闲的时间长于 keepalive 时间，并且线程数超过核心池大小，则该线程将被终止。 因此，池倾向于围绕核心线程数来解决。

* 默认 keepalive 时间为 0。


## 集群部署

在集群部署中，多个Job执行程序将相互协作（注意：请参阅 [Job
在异构集群中执行]({{< ref "/user-guide/process-engine/the-job-executor.md#job-execution-in-heterogeneous-clusters" >}})）.
在启动时，每个Job执行器分配一个 UUID，用于标识Job表中锁定的Job所有权。 因此，在一个双节点集群中，Job执行程序总共可能有多达 20 个并发执行线程。
