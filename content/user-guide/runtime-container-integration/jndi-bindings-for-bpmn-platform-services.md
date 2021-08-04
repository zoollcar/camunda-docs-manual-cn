---

title: 'Camunda Platform 服务 的 JNDI 绑定'
weight: 20

menu:
  main:
    name: "JNDI 绑定"
    identifier: "user-guide-runtime-container-integration-jndi"
    parent: "user-guide-runtime-container-integration"

---

Camunda 平台服务（即流程引擎服务和流程应用程序服务）通过具有以下 JNDI 名称的 JNDI 绑定提供：

* Process Engine Service: `java:global/camunda-bpm-platform/process-engine/ProcessEngineService!org.camunda.bpm.ProcessEngineService`
* Process Application Service: `java:global/camunda-bpm-platform/process-engine/ProcessApplicationService!org.camunda.bpm.ProcessApplicationService`

在 JBoss EAP 和 WildFly 上，你可以通过 JNDI 查找获得任何这些 Camunda 平台服务。
但是，在 Apache Tomcat 上，你必须做更多的事情才能进行查找以获取这些 Camunda 平台服务之一。
