---

title: '下载'
weight: 10

menu:
  main:
    identifier: "user-guide-introduction-downloading-camunda"
    parent: "user-guide-introduction"

---


# 先决条件

在下载 Camunda 之前，首先你需要安装 JRE (Java Runtime Environment) 或者 JDK
(Java Development Kit)（JDK是更好的选择）。 检查你的Java版本是否是 [支持的 Java 版本]({{< ref "/introduction/supported-environments.md#java" >}}).

[JDK 下载地址][get-jdk]


# 下载运行时

Camunda是一个灵活的框架，可以用在不同的环境中运行。详情参见[架构概述]
({{< ref "/introduction/architecture.md" >}})。 根据你想使用Camunda的方式，你可以选择不同的运行方式。


## 社区版 vs. 企业版

Camunda为社区用户和企业订阅客户提供不同的运行时下载：

* [社区版下载地址][community-download-page]
* [企业版下载地址][enterprise-download-page]

也可以通过[Spring Boot][run-with-spring-boot] 和 [Docker][run-with-docker]的方式运行Camunda平台。


## 整合包

如果你想使用 [分布式流程引擎][shared-engine] 或者想快速了解Camunda，不想做任何额外的设置或安装步骤，可以下载整合包。

整合包捆绑包括了：

* 配置好的 [分布式流程引擎][shared-engine],
* 网络应用程序 (Tasklist, Cockpit, Admin),
* Rest Api 接口,
* 容器/应用服务器本身.

{{< note title="服务器/容器" class="info" >}}
  如果你下载了开源应用服务器/容器的整合包，那么容器本身也会包括在内。例如，如果你下载tomcat发行版，tomcat本身就包括在内，Camunda二进制文件（流程引擎和网络应用程序）也会预先安装在容器中。而Oracle WebLogic和IBM WebSphere的下载则不会这样的。这些包是不包括应用服务器本身。
{{< /note >}}

{{< note title="Wildfly应用程序服务器" class="info" >}}
  Wildfly应用服务器是作为档案的一部分提供的，作为一种方便。 关于源代码的副本、版权声明以及其他相关信息，请参见https://github.com/wildfly/wildfly。如果你[联系我们的开源合规团队](https://docs.camunda.org/manual/latest/introduction/licenses/#contact)，我们也会向你提供一份源代码的副本。 在您下载存档后三年内的任何时候（我们可能会对此收取象征性的费用）。Wildfly应用服务器的版权属于 copyright © JBoss, Home of Professional Open Source, 2010, Red Hat Middleware LLC [..and contributors].
{{< /note >}}

详情参见 [安装教程]][installation-guide-full] 。


## 独立的网络应用程序发布

如果你想独立使用 Cockpit, Tasklist, Admin ，可以在[嵌入式流程引擎][embedded-engine]页面下载**可执行 WAR 包**运行网络应用程序

独立的网络应用程序包，包括下列内容：

* [嵌入式流程引擎][embedded-engine]配置,
* 网络应用程序 (Tasklist, Cockpit, Admin),
* Rest Api 接口,

独立的网络应用程序可以被部署到任何支持的应用服务器上。

流程引擎的配置是基于Spring框架的。如果你想改变数据库配置，请编辑WAR文件中的`WEB_INF/applicationContext.xml`文件。

有关其他细节，请参见[安装指南][installation-guide-standalone]。


# 下载 Camunda Modeler

Camunda Modeler是一款用于BPMN 2.0和DMN 1.3的建模工具。Camunda Modeler可以从
从[社区下载页面][community-download-page]下载.



[get-jdk]: https://www.oracle.com/technetwork/java/javase/downloads/index.html
[community-download-page]: https://downloads.camunda.cloud/release/camunda-bpm/
[enterprise-download-page]: /enterprise/download
[shared-engine]: {{< ref "/introduction/architecture.md#shared-container-managed-process-engine" >}}
[embedded-engine]: {{< ref "/introduction/architecture.md#embedded-process-engine" >}}
[installation-guide-standalone]: {{< ref "/installation/standalone-webapplication.md" >}}
[installation-guide-full]: {{< ref "/installation/_index.md" >}}
[run-with-spring-boot]: {{< ref "/user-guide/spring-boot-integration/_index.md" >}}
[run-with-docker]: {{< ref "/installation/docker.md" >}}
