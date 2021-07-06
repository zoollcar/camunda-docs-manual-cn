---

title: '支持的运行环境'
weight: 40

menu:
  main:
    identifier: "user-guide-introduction-supported-environments"
    parent: "user-guide-introduction"

---


你可以在任何可运行Java的环境中运行Camunda平台。但以下环境中运行可以得到了我们的质量保证。你可以在我们的[企业支持](http://camunda.com/bpm/enterprise/)页面中获得更多的信息。

{{< note title="支持的运行环境" class="info" >}}
  请注意，本节中列出的环境与Camunda平台的版本有关。请选择本文档的相应版本，以查看适合你的Camunda平台版本的环境。例如，[7.3版本的支持环境](http://docs.camunda.org/7.3/guides/user-guide/#introduction-supported-environments)。
{{< /note >}}


# 容器/应用 服务器

## 嵌入应用程序的流程引擎

* 所有的 Java 应用服务器
* Camunda Spring Boot Starter: 启用 Tomcat （参见 [支持的版本]({{< ref "/user-guide/spring-boot-integration/version-compatibility.md" >}}) 和 [部署场景]({{< ref "/user-guide/spring-boot-integration/_index.md#supported-deployment-scenarios" >}})）

## 容器管理的流程引擎 和 Camunda Cockpit, Tasklist, Admin

* Apache Tomcat 9.0
* JBoss EAP 7.0 / 7.1 / 7.2 / 7.3
* Wildfly Application Server 13.0 / 14.0 / 15.0 / 16.0 / 17.0 / 18.0 / 19.0 / 20.0 / 21.0 / 22.0
* IBM WebSphere Application Server 8.5 / 9.0 ([Enterprise Edition only](http://camunda.com/enterprise/))
* Oracle WebLogic Server 12c (12R2) ([Enterprise Edition only](http://camunda.com/enterprise/))


# 数据库

## 支持的数据库产品

* MySQL 5.7 / 8.0
* MariaDB 10.2 / 10.3
* Oracle 12c / 19c
* IBM DB2 11.1 (excluding IBM z/OS for all versions)
* PostgreSQL 9.6 / 10 / 11 / 12 / 13
* Amazon Aurora PostgreSQL compatible with PostgreSQL 9.6 / 10.4 / 10.7 / 10.13 / 12.4
* Microsoft SQL Server 2014/2016/2017/2019 (see [Configuration Note]({{< ref "/user-guide/process-engine/database/mssql-configuration.md" >}}))
* H2 1.4 (not recommended for [Cluster Mode]({{< ref "/introduction/architecture.md#clustering-model" >}}) - see [Deployment Note]({{< ref "/user-guide/process-engine/deployments.md" >}}))
* CockroachDB v20.1.3 (see [Configuration guide]({{< ref "/user-guide/process-engine/database/cockroachdb-configuration.md" >}}) for more details)

## 数据库集群 & 复制

使用支持集群或复制的数据库，需要满足以下条件下。Camunda平台和数据库集群之间的通信必须与相应的非集群/非复制（non-clustered / non-replicated）的配置。尤为重要的是，数据库集群的配置必须保证READ-COMMITTED或者更高的隔离级别。

* MariaDB Galera Cluster。支持MariaDB的Galera Cluster，有特定的配置设置和一些已知的限制。见[详情]({{< ref "/user-guide/process-engine/database/mariadb-galera-configuration.md" >}})。

# Web 浏览器

* Google Chrome 最新版
* Mozilla Firefox 最新版
* Microsoft Edge 最新版


# Java

* Java 8 / 9 / 10 / 11 / 12 / 13 / 14 (前提是你的应用服务器/容器支持)


# Java 运行时

* Oracle JDK 8 / 9 / 10 / 11 / 12 / 13 / 14
* IBM JDK 8 (with J9 JVM)
* OpenJDK 8 / 9 / 10 / 11 / 12 / 13 / 14, 包括以下的发行版：
  * Oracle OpenJDK
  * AdoptOpenJDK (with HotSpot JVM)
  * Amazon Corretto
  * Azul Zulu

# Camunda模型编辑器（Camunda Modeler）

支持以下平台：

* Windows 7 / 10
* Mac OS X 10.11
* Ubuntu LTS (latest)

正在努力支持：

* Ubuntu 12.04 and newer
* Fedora 21
* Debian 8

# 维护政策

查看我们的[企业公告页面](/enterprise/announcement/) ，了解对我们支持的环境的变化情况。

## 添加支持环境

每当以下环境的新版本发布时，我们会在Camunda平台的下一个小版本中对该新版本进行支持。

* Java 语言
* Wildfly 应用服务器
* Oracle 数据库

我们支持一个新环境的确切版本取决于环境的发布日期和所需的开发工作量等因素。

对其他环境的版本支持是根据具体情况决定的，主要是基于我们用户群的需求。

## 移除支持环境

每当以下环境的一个新版本被支持时，我们通常会在同一版本中停止对最旧版本的支持。

* Wildfly 应用服务器

请注意，我们可能会根据具体情况确定是否真的执行这一决定。