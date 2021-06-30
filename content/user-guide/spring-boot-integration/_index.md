---

title: "Spring Boot 集成"
weight: 55

menu:
  main:
    identifier: "user-guide-spring-boot-integration"
    parent: "user-guide"

---

通过提供的 Spring Boot starters，Camunda引擎可以在Spring Boot应用程序中使用。
Spring Boot starters 允许通过在classpath中添加依赖项来启用你的Spring-boot应用程序的行为。

这些启动器将预先配置Camunda流程引擎、REST API和Web应用程序，因此它们可以很容易地用于独立的流程应用程序。

如果你不熟悉 [Spring Boot](http://projects.spring.io/spring-boot/), 请阅读 [入门教程](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#getting-started) 或使用 [Camunda Platform 生成器](https://start.camunda.com/).

要启用Camunda平台自动配置，请将以下依赖添加到你的 ```pom.xml```文件中:

```xml
<dependency>
  <groupId>org.camunda.bpm.springboot</groupId>
  <artifactId>camunda-bpm-spring-boot-starter</artifactId>
  <version>{{< minor-version >}}.0</version>
</dependency>
```

这将添加流程引擎 v.{{< minor-version >}}.0 到你的依赖。

可以使用的其他starter是：

* [`camunda-bpm-spring-boot-starter-rest`](rest-api)
* [`camunda-bpm-spring-boot-starter-webapp`](webapps)
* [`camunda-bpm-spring-boot-starter-external-task-client`]({{< ref "/user-guide/ext-client/spring-boot-starter.md" >}})

# 使用企业版本

要在Camunda EE中使用Camunda Spring Boot Starter，你需要定义webapp的EE版本（`camunda-bpm-spring-boot-starter-webapp-ee`），而不是（`camunda-bpm-spring-boot-starter-webapp`），另见[Web Applications](webapps/) 。

```xml
<dependency>
  <groupId>org.camunda.bpm.springboot</groupId>
  <artifactId>camunda-bpm-spring-boot-starter-webapp-ee</artifactId>
  <version>{{< minor-version >}}.0-ee</version>
</dependency>
```

# 支持的部署方案

Camunda支持以下部署方案。

* 带有嵌入式Tomcat和一个嵌入式流程引擎的可执行JAR（必要时加上Webapps）。

还有其他可能的形式，但目前没有经过Camunda的测试。
