---

title: "Process Applications"
weight: 30
layout: "single"

menu:
  main:
    identifier: "user-guide-process-applications"
    parent: "user-guide"

---

 Process Applications 是一个普通的 Java 应用程序，它使用 Camunda 流程引擎来实现 BPM 和工作流功能。 大多数此类应用程序将启动自己的流程引擎（或使用运行时容器提供的流程引擎），部署一些 BPMN 2.0 流程定义并与从这些流程定义派生的流程实例进行交互。 由于大多数 Process Applications 运行非常相似的引导、部署和运行时任务，因此我们将此功能概括为一个名为 - *Surprise!* - `ProcessApplication` 的 Java 类。 这个概念类似于 JAX-RS 中的 `javax.ws.rs.core.Application` 类：添加 Process Applications 类允许你引导和配置提供的服务。

将“ProcessApplication”类添加到你的 Java 应用程序可为你的应用程序提供以下服务：

* **引导**嵌入式流程引擎或查找容器管理的流程引擎。你可以在添加到应用程序的“processes.xml”文件中定义多个流程引擎。 `ProcessApplication` 类确保在部署/取消部署应用程序时选择该文件并启动和停止定义的流程引擎。
* **自动部署**类路径下的 BPMN 2.0 资源。你可以在“processes.xml”文件中定义多个部署（流程档案）。 `ProcessApplication` 类确保在部署应用程序时执行部署。也支持扫描你的应用程序以查找流程定义资源文件（以 *.bpmn20.xml 或 *.bpmn 结尾）。
* **在多应用程序部署的情况下解决应用程序本地 Java 委托实现**和 Bean。 `ProcessApplication` 类允许你的 Java 应用程序将你的本地 Java 委托实现或 Spring/CDI bean 公开给一个共享的、容器管理的流程引擎。通过这种方式，你可以启动单个流程引擎，该引擎分派到可以独立（重新）部署的多个 Process Applications 。

将现有的 Java 应用程序转换为 Process Applications 很容易且非侵入性。 你只需添加：

* 一个 `ProcessApplication` 类：`ProcessApplication` 类构成了你的应用程序和流程引擎之间的接口。 你可以扩展不同的基类以反映不同的环境（例如，Servlet 与 EJB 容器）。
* 创建 META-INF 下的 `processes.xml` 文件：部署描述符文件允许你提供此 Process Applications 对流程引擎进行的部署的声明性配置。 它可以是空的（参见 [empty processes.xml 部分]({{< ref "/user-guide/process-applications/the-processes-xml-deployment-descriptor.md#empty-processes-xml" >}} )) 并作为简单的标记文件。 如果它不存在，则引擎将启动，但不会执行自动部署。

{{< note title="教程" class="info" >}}
 你可能想先查看 [入门教程](http://docs.camunda.org/get-started)，因为它逐步解释了 Process Applications 的创建或 [Maven 的项目模板]({ {< ref "/user-guide/process-applications/maven-archetypes.md" >}})，它为你提供了一个完整的开箱即用的 Process Applications 。
{{< /note >}}
