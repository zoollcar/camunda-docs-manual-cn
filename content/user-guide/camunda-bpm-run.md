---

title: 'Camunda Platform Run'
weight: 50

menu:
  main:
    identifier: "camunda-bpm-run-guide"
    parent: "user-guide"

---

本指南介绍了 Camunda Platform Run，这是 Camunda Platform 的一个预打包的轻量级发行版。 Camunda Platform Run 易于配置，不需要 Java 知识。

# 前提条件和受众

要使用本指南，您至少应该知道 Camunda 平台是什么以及它的作用。 如果您之前从未使用过Camunda平台，请查看 [入门指南](https://docs.camunda.org/get-started/quick-start/)。如果您是 Camunda 平台的新手， [安装指南]({{< ref "/installation/camunda-bpm-run.md" >}})也值得一看。

本指南将教您有关 Camunda Platform Run 以及如何配置它。 它可以作为配置和操作选项的参考页面。 它不会为您提供有关如何安装 Camunda Platform Run 的分步指南。 有关如何安装和启动 Camunda Platform Run 的详细信息，请转到 [安装指南]({{< ref "/installation/camunda-bpm-run.md" >}})。

# 什么是 Camunda Platform Run ？

Camunda Platform Run 是 Camunda Platform 的完整发行版。 这包括：

* Camunda webapps
  * Cockpit
  * Tasklist
  * Admin
* [REST API]({{< ref "/reference/rest/overview/_index.md" >}})
* [Swagger UI](https://github.com/swagger-api/swagger-ui) (用于查看 REST API 的 Web 应用)

# Camunda Platform Run 启动

要开始使用 Camunda Platform Run，请下载 [发行版](https://downloads.camunda.cloud/release/camunda-bpm/run/) ([企业发行版](https://downloads.camunda.cloud/enterprise-release/camunda-bpm/run/)) 并解压它。 您会看到如下结构：

```
camunda-bpm-run
├── configuration/
│   ├── keystore/
│   │   └── put your SSL key store here if you want to use HTTPS
│   ├── resources/
│   │   └── put your BPMN files, forms and scripts here
│   ├── sql/
│   │   └── necessary SQL scripts to prepare your database system
│   ├── userlib/
│   │   └── put your database driver jar here
│   ├── default.yml
│   └── production.yml
├── internal/
├── start.bat
└── start.sh
```
执行两个启动脚本之一（Windows 的`start.bat`，Linux/Mac 的`start.sh`）。几秒钟后，您将能查看 Camunda webapps 于 http://localhost:8080/camunda/app/ ， REST API 于 http://localhost:8080/engine-rest/ 和 Swagger UI 于 http://localhost:8080/swaggerui/。


## 使用 Docker 启动 Camunda Platform Run

Camunda Platform Run 也可用作 Docker 映像运行。更多详细信息，请参阅 Camunda Docker 文档 [此处]({{< ref "/installation/docker.md#start-camunda-bpm-run-using-docker" >}}) 的 Camunda Platform Run 部分。


## 禁用组件

默认情况下，Camunda Platform Run 与 webapps、REST API 和 Swagger UI 模块一起启动。 如果您只想启用其中的一个子集，请通过命令行界面执行启动脚本，并使用任何 `--webapps`、`--rest` 或 `--swaggerui` 参数来启用特定模块。


## 在默认配置和生产配置之间进行选择

Camunda Platform Run 附带了两个不同的配置文件，它们都位于“configuration”文件夹中。

* `default.yml` 配置文件仅包含必要的配置，例如 H2 数据库、演示用户和用于来自客户端应用程序 REST 调用的 [CORS](#cross-origin-resource-sharing)。
* `production.yml` 配置文件用于根据 [安全说明]({{< ref "/user-guide/security.md" >}}) 提供的推荐属性。
  在生产环境中使用 Camunda Platform Run 时，请确保您的自定义配置基于此配置并仔细阅读安全说明。

默认情况下，Run 使用 `default.yml` 配置启动。 要启用 `production.yml` 配置，请通过命令行界面使用 `--production` 属性执行启动脚本。
使用 `--production` 将禁用 Swagger UI。 它可以通过显式传递 `--swaggerui` 来启用，但是，不建议在生产中使用 Swagger UI。


## 连接数据库

Camunda Platform Run 已预先配置为使用基于文件的 H2 数据库进行测试。 引擎第一次启动时，会自动创建数据库架构和所有必需的表。 如果要使用自定义独立数据库，请执行以下步骤：

1. 确保您的数据库系统在 [支持的数据库系统]({{< ref "/introduction/supported-environments.md#supported-database-products" >}}) 中。
1. 为 Camunda 平台创建一个数据库。
1. Install the database schema to create all required tables and default indices using our [database schema installation guide]({{< ref "/installation/database-schema.md" >}}).
1. 在 `configuration/userlib` 文件夹中为您的数据库系统放置一个 JDBC 驱动程序 jar 文件。
1. 将 JDBC URL 和登录凭据添加到配置文件中，[如下所述](#数据库).
1. 重启 Camunda Platform Run


## 部署 BPMN 模型

在发行版解压后，您会看到一个 `resources` 文件夹。 所有文件（包括 BPMN、DMN、CMMN、表单和脚本文件）将在您启动 Camunda Platform Run 时部署。

You can reference forms and scripts in the BPMN diagram with `embedded:deployment:/my-form.html`, `camunda-forms:deployment:/myform.form`, or `deployment:/my-script.js`. The deployment requires adding an extra `/` as a prefix to the filename.

也可以通过 [REST API]({{< ref "/reference/rest/deployment/post-deployment.md" >}}) 部署。


## 自动领取许可证密钥

如果您下载了 Camunda Platform Run 的企业版，则需要许可证密钥才能启用企业功能。 请参阅文档的 [专用许可证部分]({{< ref "/user-guide/license-use.md#with-the-camunda-spring-boot-starter-camunda-run" >}})部分以了解更多。


# 配置 Camunda Platform Run

就像所有其他发行版一样，您可以根据自己的需要定制 Camunda Platform Run。 为此，您只需选择配置文件夹中找到的一个 [配置文件](#在默认配置和生产配置之间进行选择) 编辑即可。

{{< note title="注意:" class="info" >}}
Camunda Platform Run 基于 [Camunda Spring Boot Starter](https://github.com/camunda/camunda-bpm-spring-boot-starter).
所有的 [配置项]({{< ref "/user-guide/spring-boot-integration/configuration.md#camunda-engine-properties" >}}) 来自 camunda-spring-boot-starter 可用于自定义 Camunda Platform Run。
{{< /note >}}


## 数据库

该发行版带有一个基于文件的 h2 数据库用于测试。 建议生产环境连接到独立的数据库系统。

<table class="table desc-table">
  <tr>
      <th>前缀</th>
      <th>参数名</th>
      <th>描述</th>
      <th>默认值</th>
  </tr>
  <tr>
      <td rowspan="15"><code>spring.datasource</code></td>
      <td><code>.url</code></td>
      <td>数据库的JDBC URL。</td>
      <td><code>-</code></td>
  </tr>
  <tr>
      <td><code>.driver-class-name</code></td>
      <td>数据库系统的 JDBC 驱动程序的类名。 记得把你的数据库系统的驱动jar放在<code>configuration/userlib</code>。</td>
      <td>-</td>
  </tr>
  <tr>
      <td><code>.username</code></td>
      <td>数据库连接的用户名。</td>
      <td>-</td>
  </tr>
  <tr>
      <td><code>.password</code></td>
      <td>数据库连接的密码。</td>
      <td>-</td>
  </tr>
</table>


## 认证

要向针对 [REST API]({{< ref "/reference/rest/overview/_index.md" >}}) 的请求添加身份验证，您可以启用基本身份验证。

<table class="table desc-table">
  <tr>
      <th>前缀</th>
      <th>参数名</th>
      <th>描述</th>
      <th>默认值</th>
  </tr>
  <tr>
      <td rowspan="15"><code>camunda.bpm.run.auth</code></td>
      <td><code>.enabled</code></td>
      <td>是否启用对 REST API 请求的基本身份验证。</td>
      <td><code>false</code></td>
  </tr>
  <tr>
      <td><code>.authentication</code></td>
      <td>认证方式，目前只支持basic。</td>
      <td>basic</td>
  </tr>
</table>


## 跨域资源共享

如果你想允许对[REST API]({{< ref "/reference/rest/overview/_index.md" >}}) 的跨域请求，你需要启用CORS。
<table class="table desc-table">
  <tr>
      <th>前缀</th>
      <th>参数名</th>
      <th>描述</th>
      <th>默认值</th>
  </tr>
  <tr>
      <td rowspan="15"><code>camunda.bpm.run.cors</code></td>
      <td><code>.enabled</code></td>
      <td>是否启用 CORS。</td>
      <td><code>false</code></td>
  </tr>
  <tr>
      <td><code>.allowed-origins</code></td>
      <td>允许发出 CORS 请求的源。 多个来源可以用逗号分隔。 同时支持 HTTP 身份验证和 CORS， <a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/Errors/CORSNotSupportingCredentials"><code>allowed-origins</code> 不能设为</a> <code>\*</code>。 To allow Camunda Modeler to deploy with authentication, including <code>file://</code> in the allowed origins.</td>
    <td><code>\*</code> (all origins, including <code>file://</code>)</td>
  </tr>
</table>


## LDAP 身份服务

Camunda 平台可以自行管理用户和授权，但如果您想使用现有的 LDAP 身份验证数据库，您可以启用[LDAP 身份服务插件]({{< ref "/user-guide/process-engine/identity-service.md#the-ldap-identity-service" >}})
它提供对 LDAP 存储库的只读访问。

在 [LDAP 插件指南]({{< ref "/user-guide/process-engine/identity-service.md#configuration-properties-of-the-ldap-plugin" >}}) 中查找所有可用的配置属性

<table class="table desc-table">
  <tr>
      <th>前缀</th>
      <th>参数名</th>
      <th>描述</th>
      <th>默认值</th>
  </tr>
  <tr>
      <td rowspan="15"><code>camunda.bpm.run.ldap</code></td>
      <td><code>.enabled</code></td>
      <td>是否启用 LDAP 身份服务插件。</td>
      <td><code>false</code></td>
  </tr>
</table>


## HTTPS

Camunda Platform Run 支持基于 SSL 的 HTTPS。 要启用它，您需要一个由受信任的提供商签署并存储在密钥库文件（.jks 或 .p12）中的有效 SSL 证书。
为了测试，我们包含了一个自签名证书。 你不应该在生产中使用它。 要启用它，请将以下属性添加到您的配置文件中。

```yaml
server:
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: camunda
    key-store-type: pkcs12
    key-alias: camunda
    key-password: camunda
  port: 8443
```
启动 Camunda Platform Run 后，你可以通过 https://localhost:8443/camunda/app/ 和 REST API https://localhost:8443/engine-rest/ 访问应用。

<table class="table desc-table">
  <tr>
      <th>前缀</th>
      <th>参数名</th>
      <th>描述</th>
      <th>默认值</th>
  </tr>
  <tr>
      <td rowspan="15"><code>server.ssl</code></td>
      <td><code>.key-store</code></td>
      <td>保存 SSL 证书的密钥库文件的名称。 此文件必须放在 <code>configuration/keystore</code> 文件夹中，并且必须是 .jks 或 .p12 文件。</td>
      <td><code>-</code></td>
  </tr>
  <tr>
      <td><code>.key-store-password</code></td>
      <td>访问密钥库的密码。</td>
      <td><code>-</code></td>
  </tr>
  <tr>
      <td><code>.key-store-type</code></td>
      <td>密钥库的类型。 可以是 <code>jks</code> 或 <code>p12</code></td>
      <td><code>-</code></td>
  </tr>
  <tr>
      <td><code>.key-alias</code></td>
      <td>在密钥库中标识 SSL 证书的名称。</td>
      <td><code>-</code></td>
  </tr>
  <tr>
      <td><code>.key-password</code></td>
      <td>访问密钥库中 SSL 证书的密码。</td>
      <td><code>-</code></td>
  </tr>
</table>


## Logging

Camunda 平台提供细粒度和可定制的日志记录。 可以在 [日志用户指南]({{< ref "/user-guide/logging.md#process-engine" >}}) 中找到可用日志类别的概述。
要在 Camunda Platform Run 中配置日志记录行为，请使用以下属性自定义配置文件。

有关日志配置的更多信息，请访问 [Spring Boot 日志指南](https://docs.spring.io/spring-boot/docs/2.4.0/reference/html/spring-boot-features.html#boot-features-logging).

<table class="table desc-table">
  <tr>
      <th>前缀</th>
      <th>参数名</th>
      <th>描述</th>
      <th>默认值</th>
  </tr>
  <tr>
      <td rowspan="15"><code>logging</code></td>
      <td><code>.level.root</code></td>
      <td>Set a logging level for all available logging categories. Value can be one of the following: <code>OFF</code>. <code>ERROR</code>. <code>WARN</code>. <code>INFO</code>. <code>DEBUG</code>. <code>ALL</code></td>
      <td><code>-</code></td>
  </tr>
  <tr>
      <td><code>.level.{logger-name}</code></td>
      <td>Set a logging level for a specific logging category. Find an overview over the available categories in the <a href="{{<ref "/user-guide/logging.md#process-engine" >}}">Logging User Guide</a>.
      Value can be one of the following: <code>OFF</code>. <code>ERROR</code>. <code>WARN</code>. <code>INFO</code>. <code>DEBUG</code>. <code>ALL</code></td>
      <td><code>-</code></td>
  </tr>
  <tr>
      <td><code>.file.name</code></td>
      <td>Specify a log file location. (e.g. <code>logs/camunda-bpm-run-log.txt</code>)</td>
      <td><code>-</code></td>
  </tr>
</table>
