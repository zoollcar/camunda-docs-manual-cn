---

title: 'Camunda Platform 服务'
weight: 10

menu:
  main:
    identifier: "user-guide-runtime-container-integration-services"
    parent: "user-guide-runtime-container-integration"

---

`org.camunda.bpm.BpmPlatform` 类提供对 `ProcessEngineService` 和 `ProcessApplicationService` 的访问，可以用来检查配置的流程引擎和部署的流程应用程序的当前状态。


# ProcessEngineService

{{< javadocref page="?org/camunda/bpm/ProcessEngineService.html" text="ProcessEngineService" >}} 可以通过调用 `BpmPlatform.getProcessEngineService()` 来访问。它提供对默认流程引擎以及流程引擎配置中指定的名称的任何流程引擎的访问。它返回“ProcessEngine”对象，可以从中访问特定引擎的任何服务。


# ProcessApplicationService

{{< javadocref page="?org/camunda/bpm/ProcessApplicationService.html" text="ProcessApplicationService" >}} 可以通过 `BpmPlatform.getProcessApplicationService()` 访问。它提供了在其运行的应用程序服务器上进行的流程应用程序部署的详细信息。这意味着它不提供跨集群中所有节点的全局视图。

通过流程应用程序名称，可以检索包含有关此流程应用程序进行的部署的详细信息的“ProcessApplicationInfo”对象。这些对应于 [processes.xml]({{< ref "/user-guide/process-applications/the-processes-xml-deployment-descriptor.md" >}}) 中声明的流程档案。

此外，可以检索特定于应用程序的属性，例如 servlet 流程应用程序中的 servlet 上下文路径。