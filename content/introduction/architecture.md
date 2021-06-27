---

title: '软件架构总览'
weight: 30

menu:
  main:
    identifier: "user-guide-introduction-architecture"
    parent: "user-guide-introduction"

---


Camunda平台是一个基于Java的框架。主要组件是用Java编写的，我们总体上专注于为Java开发者提供他们在JVM上设计、实施和运行业务流程和工作流所需的工具。然而，我们也想让非Java开发者也能使用流程引擎技术。这就是为什么Camunda平台还提供了REST API，允许你与远程流程引擎应用建立连接。

Camunda平台既可以作为一个独立的流程引擎服务器使用，也可以嵌入到定制的Java应用程序中。可嵌入性要求是Camunda平台许多架构决定的核心。例如，我们努力使流程引擎组件成为一个轻量级的组件，尽量减少对[第三方库]({{< ref "/introduction/third-party-libraries/_index.md" >}})的依赖性。此外，可嵌入性促使流程引擎支持使用Spring Managed或JTA[事务和线程模型]({{< ref "/user-guide/process-engine/transactions-in-processes.md" >}})的能力。


# 流程引擎架构

{{< img src="../img/process-engine-architecture.png" title="Process Engine Architecture" >}}

* [流程引擎公共API]({{< ref "/user-guide/process-engine/process-engine-api.md" >}})： 面向服务的API允许Java应用程序与流程引擎交互。流程引擎的不同部分（即流程存储库、运行时的流程交互、任务管理等）被分离到各个服务中。公共API具有[命令式访问模式](http://en.wikipedia.org/wiki/Command_pattern)的特点：进入流程引擎的流程要经过一个命令拦截器，该拦截器用于设置流程上下文。
* **BPMN 2.0 核心引擎**: 这是流程引擎的核心。它有一个轻量级的图结构执行引擎（PVM - Process Virtual Machine），一个将BPMN 2.0 XML文件转化为Java对象的BPMN 2.0解析器，以及一套BPMN行为实现（为BPMN 2.0结构提供实现，如网关或服务任务）。
* [Job 执行器]({{< ref "/user-guide/process-engine/the-job-executor.md" >}}): Job执行者负责处理异步的后台工作，如流程中的定时器或异步延续。
* **持久层**: 流程引擎有一个持久层，负责将流程实例状态持久化到关系数据库中。我们使用MyBatis映射引擎来实现对象的关系映射。


## 依赖的第三方库

见[第三方库]({{< ref "/introduction/third-party-libraries/_index.md" >}})一节。


# Camunda平台 架构

Camunda平台是一个灵活的框架，可以部署在不同的场景中。本节对最常见的部署场景进行了概述。


## 嵌入式流程引擎

{{< img src="../img/embedded-process-engine.png" title="Embedded Process Engine" >}}

在这种情况下，流程引擎被作为一个库添加到一个自定义的应用程序中。这样，流程引擎可以很容易地随着应用程序的生命周期启动和停止。也可以在一个共享数据库之上运行多个嵌入式流程引擎。


## 分布式的、由容器管理的流程引擎

{{< img src="../img/shared-process-engine.png" title="Shared Process Engine" >}}

在这种情况下，流程引擎在运行时容器（Servlet容器、应用服务器...）内启动。流程引擎作为容器服务提供的，可以被部署在容器内的所有应用程序共享。这个概念就像JMS消息队列，它由运行时提供，可以被所有应用程序使用。流程部署和应用程序之间有一对一的映射：流程引擎跟踪应用程序部署的流程定义，并将执行委托给相对应的应用程序。


## 独立运行的（远程）流程引擎服务

{{< img src="../img/standalone-process-engine.png" title="Standalone Process Engine" >}}

在这种情况下，流程引擎被作为一个网络服务提供。在网络上运行的不同应用程序可以通过一个远程通信与流程引擎进行交互。使流程引擎可以远程访问的最简单方法是使用内置的REST API。不同的通信渠道，如SOAP Webservices或JMS也是可以的，但需要由用户自行实现。


# 集群

为了提供扩展或故障转移能力，流程引擎可以发布到集群中的不同节点。然后每个流程引擎实例必须连接到同一个共享数据库。

{{< img src="../img/clustered-process-engine.png" title="Process Engine Cluster" >}}

各个流程引擎实例不会在事务中维护会话状态。每当流程引擎运行一个事务时，完整的状态被储存到共享数据库。这使得将在同一力促恒实例中进行工作的后续请求路由到不同的集群节点成为可能。这个处理方式非常简单，容易理解，在部署集群安装时，它的限制也很有限。就流程引擎而言，用于扩展的设置和用于故障转移的设置之间没有区别（因为流程引擎在事务之中不保留会话状态）。

## 集群环境中的会话状态

Camunda平台不提供开箱即用的负载均衡功能和会话复制功能。负载均衡功能需要由第三方系统提供，而会话复制则需要由主机应用服务器提供。

在集群设置中，如果用户要登录到网络应用程序，需要采取额外的步骤来确保用户不会被要求多次登录。有两个方法：

1. 可以在你的负载平衡解决方案中配置和启用 "粘性会话"。这将确保在一个可配置的时间段内，所有来自特定用户的请求都被引导到同一个实例。
2. 会话共享可以在你的应用服务器中启用，这样应用服务器实例就可以共享会话状态。这将允许一个用户连接到集群中的多个实例，而不会被要求多次登录。

如果以上两种方法都没有在集群设置中实现，那么连接到多个节点，无论是故意的或通过负载平衡解决方案的，都将导致多次登录请求。

## 集群环境中的 Job 执行器

流程引擎 [job 执行器]({{< ref "/user-guide/process-engine/the-job-executor.md" >}}) 也是集群的，在每个节点上都存在。这样，就流程引擎而言，就没有单点故障。job执行器可以同时在[同质和异质集群]({{< ref "/user-guide/process-engine/the-job-executor.md#cluster-setups" >}})中运行。

{{< note title="时区" class="info" >}}
camunda对[集群中的时区选择]({{< ref "/user-guide/process-engine/time-zones.md#cluster-setup" >}})有一些限制。
{{< /note >}}


# 多租户

为了用一个Camunda的服务于多个独立的应用，流程引擎支持
多租户。支持以下多租户模式：

* 通过使用不同的数据库模式或数据库进行表级数据分离
* 通过使用租户标记进行行级数据分离

用户应该选择适合其数据分离需求的模式。Camunda的API提供了对每个流程和相关数据的访问。
流程和每个租户的相关数据。
学习更多细节可以查看[多租户章节]({{< ref "/user-guide/process-engine/multi-tenancy.md" >}})。


# 网络应用程序架构

Camunda平台的网络应用程序是基于RESTful架构的。

使用的框架有：

* [JAX-RS](https://jax-rs-spec.java.net) 基于REAT API.
* [AngularJS](http://angularjs.org)
* [RequireJS](http://requirejs.org)
* [jQuery](http://jquery.com)
* [Twitter Bootstrap](http://getbootstrap.com)

由Camunda工程师开发以及其他自定义框架。

* [camunda-bpmn.js](https://github.com/camunda/camunda-bpmn.js): Camunda BPMN 2.0 JavaScript 库
* [ngDefine](https://github.com/Nikku/requirejs-angular-define): integration of AngularJS into RequireJS powered applications
* [angular-data-depend](https://github.com/Nikku/angular-data-depend): toolkit for implementing complex, data heavy AngularJS applications
