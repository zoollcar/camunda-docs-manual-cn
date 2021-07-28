---

title: 'Maven 项目模板 (Archetypes)'
weight: 40

menu:
  main:
    identifier: "user-guide-process-application-archetypes"
    parent: "user-guide-process-applications"

---

我们为 Maven 提供了几个项目模板，也称为原型。
它们支持使用 Camunda 平台开发可用于生产的流程应用程序的快速搭建。
我们将不同应用程序类型的最佳实践纳入模板，以帮助您从坚实的基础开始。

原型可用于生成不同使用部分中详述的项目。
如果你不能自己从 Archetype 生成项目，我们还为每个 Archetype 提供了一个模板 GitHub 存储库。

# 可用的 Maven 原型概述

当前提供了以下原型。 它们通过我们的 Maven 存储库分发: https://app.camunda.com/nexus/service/rest/repository/browse/camunda-bpm/

<table class="table table-bordered">
  <thead>
    <tr><th>原型</th><th>描述</th></tr>
  </thead>
  <tbody>
    <tr>
      <td><a href="https://app.camunda.com/nexus/service/rest/repository/browse/camunda-bpm/org/camunda/bpm/archetype/camunda-archetype-cockpit-plugin/">Camunda Cockpit Plugin</a></td>
      <td>Camunda Cockpit 插件，包含 REST-Backend、MyBatis 数据库查询、HTML 和 JavaScript 前端、用于一键部署的 Ant 构建脚本</td>
    </tr>
    <tr>
      <td><a href="https://app.camunda.com/nexus/service/rest/repository/browse/camunda-bpm/org/camunda/bpm/archetype/camunda-archetype-ejb-war/">Process Application (EJB, WAR)</a></td>
      <td>在 Java EE 容器中使用共享 Camunda 平台引擎的流程应用程序，例如 JBoss Wildfly。
           包含：Camunda EJB 客户端、Camunda CDI 集成、BPMN 流程、Java 委托作为 CDI bean、基于 HTML5 和 JSF 的启动和任务表单，
           JPA (Hibernate) 配置、内存引擎和可视化过程测试覆盖的 JUnit 测试、JBoss AS7 和 Wildfly 的 Arquillian 测试、Maven 插件或 Ant 构建脚本，用于在 Eclipse 中一键部署</td>
    </tr>
    <tr>
      <td><a href="https://app.camunda.com/nexus/service/rest/repository/browse/camunda-bpm/org/camunda/bpm/archetype/camunda-archetype-servlet-war/">Process Application (Servlet, WAR)</a></td>
      <td>在 Servlet 容器中使用共享 Camunda Platform 引擎的流程应用程序，例如 Apache Tomcat。
           包含：Servlet 流程应用程序、BPMN 流程、Java Delegate、基于 HTML5 的启动和任务表单、
           使用内存引擎、Maven 插件或 Ant 构建脚本在 Eclipse 中进行一键部署的 JUnit 测试</td>
    </tr>
    <tr>
      <td><a href="https://app.camunda.com/nexus/service/rest/repository/browse/camunda-bpm/org/camunda/bpm/archetype/camunda-archetype-spring-boot/">Camunda Spring Boot Application</a></td>
      <td>使用 Camunda Spring Boot Starter 的应用程序。
           包含：Spring Boot Process Application、Camunda Webapps、BPMN Process、Java Delegate、基于 HTML5 的启动和任务表单、
           使用内存引擎进行 JUnit 测试，用于打包为可执行应用程序的 Maven 插件。</td>
    </tr>
    <tr>
      <td><a href="https://app.camunda.com/nexus/service/rest/repository/browse/camunda-bpm/org/camunda/bpm/archetype/camunda-archetype-spring-boot-demo/">Camunda Spring Boot Application with Demo Users</a></td>
      <td>与 <i>Spring Boot Application</i> 原型相同，另外还创建了演示用户和组，以便轻松启动 Camunda Webapps（使用 <code>demo/demo</code> 登录）。</td>
    </tr>
    <tr>
      <td><a href="https://app.camunda.com/nexus/service/rest/repository/browse/camunda-bpm/org/camunda/bpm/archetype/camunda-archetype-engine-plugin/">Process Engine Plugin</a></td>
      <td>流程引擎插件的示例。
       包含：流程引擎插件、通过插件注册的 BPMN 解析监听器、添加到每个用户任务的任务监听器、使用内存引擎的 JUnit 测试。</td>
    </tr>
  </tbody>
</table>

# 模板代码仓库

我们为每个 Camunda Archetype 提供了一个模板存储库。
每个存储库都包含一个从一个特定模板生成的项目。
您可以在 [GitHub](https://github.com/camunda?q=%22camunda-bpm-archetype-%22) 上找到整个列表。

随着 Archetypes 的每个新版本，我们也将使用新版本更新这些存储库。
这允许调查从一个 Camunda 版本到另一个版本的可能更新路径，还使您能够通过提取最新更改来简单地更新现有项目。

如果您的项目需要更多的灵活性和自定义，您可以使用下一节中详述的方法之一自行生成项目。

# 使用 Eclipse IDE

## 概括

1. Add archetype catalog (**Preferences -> Maven -> Archetypes -> Add Remote Catalog**):
    **https://app.camunda.com/nexus/content/repositories/camunda-bpm/**
2. Create Maven project from archetype (**File -> New -> Project... -> Maven -> Maven Project**)


## 详细说明

1. Go to **Preferences -> Maven -> Archetypes -> Add Remote Catalog**
{{< img src="../img/eclipse-00-preferences-maven-archetypes.png" title="Eclipse Preferences: Maven Archetypes" >}}
2. Enter the following URL and description, click on **Verify...** to test the connection and if that worked click on **OK** to save the catalog.

    Catalog File: **https://app.camunda.com/nexus/content/repositories/camunda-bpm/**

    Description: **Camunda Platform**
{{< img src="../img/eclipse-01-add-remote-archetype-catalog.png" title="Eclipse Preferences: Add Maven Archetype Catalog" >}}

Now you should be able to use the archetypes when creating a new Maven project in Eclipse:

1. Go to **File -> New -> Project...** and select **Maven -> Maven Project**
{{< img src="../img/eclipse-02-create-maven-project.png" title="Create new Maven project" >}}
2. Select a location for the project or just keep the default setting.
{{< img src="../img/eclipse-03-select-maven-project-location.png" title="Eclipse: Select Maven project location" >}}
3. Select the archetype from the catalog that you created before.
{{< img src="../img/eclipse-04-select-archetype-from-catalog.png" title="Eclipse: Select Maven archetype from catalog" >}}
4. Specify Maven coordinates and Camunda version and finish the project creation.
{{< img src="../img/eclipse-05-specify-maven-coordinates-and-camunda-version.png" title="Eclipse: Specify Maven coordinates and Camunda version" >}}

生成的项目应如下所示：

{{< img src="../img/eclipse-06-generated-maven-project.png" title="Generated Maven Project in Eclipse" >}}


## 故障排除

有时在 Eclipse 中创建第一个 Maven 项目会失败。 如果您遇到这种情况，请再试一次。 大多数情况下，第二次尝试有效。 如果问题仍然存在，[联系我们](https://forum.camunda.org/).

# 使用 IntelliJ IDEA

1. On the "Welcome to IntelliJ IDEA" screen, click on "Configure" and select "Plugins" in the dropdown.
2. In the plugins dialog, click on "Browse repositories...".
3. Search for the plugin "Maven Archetype Catalogs" and click on "Install".
4. Restart IntelliJ IDEA.
5. On the "Welcome to IntelliJ IDEA" screen, click on "Configure" and select "Preferences" in the dropdown.
6. In the preferences window, navigate to: "Build, Execution, Deployment > Build Tools > Maven Archetype Catalogs".
7. In the Maven Archetype Catalogs window, click on the "+" button, and in the opened "Add Archetype Catalog URL"
   modal window add the  URL of the catalog file: https://app.camunda.com/nexus/content/repositories/camunda-bpm/archetype-catalog.xml.
8. To create a Maven project from an archetype, click on the "Welcome to IntelliJ IDEA" screen on "Create New Project".
9. In the new project dialog, click on the left side on "Maven", check "Create from archetype" 
   and select any `org.camunda.bpm.archetype` entry.

# 使用 Command Line

## 交互式生成

Run the following command in a terminal to generate a project. Maven will allow you to select an archetype and ask you for all parameters needed to configure it:

<pre class="console">
mvn archetype:generate -Dfilter=org.camunda.bpm.archetype:
</pre>


## 自动化生成

The following command completely automates the project generation and can be used in shell scripts or Ant builds:
<pre class="console">
mvn archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=org.camunda.bpm.archetype \
  -DarchetypeArtifactId=camunda-archetype-ejb-war \
  -DarchetypeVersion=7.10.0 \
  -DgroupId=org.example.camunda.bpm \
  -DartifactId=camunda-bpm-ejb-project \
  -Dversion=0.0.1-SNAPSHOT \
  -Dpackage=org.example.camunda.bpm.ejb
</pre>


# 源代码和定制

You can also customize the project templates for your own technology stack. Just [fork them on GitHub](https://github.com/camunda/camunda-archetypes)!
