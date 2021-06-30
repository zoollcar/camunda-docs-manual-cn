---

title: "REST API"
weight: 30

menu:
  main:
    name: "REST API"
    identifier: "user-guide-spring-boot-rest-api"
    parent: "user-guide-spring-boot-integration"

---

你可以使用如下 `pom.xml` 启用 [REST API]({{< ref "/reference/rest/_index.md">}}):

```xml
<dependency>
  <groupId>org.camunda.bpm.springboot</groupId>
  <artifactId>camunda-bpm-spring-boot-starter-rest</artifactId>
  <version>{project-version}</version>
</dependency>
```

默认情况下，应用程序的路径是 "engine-rest"，所以不需要任何进一步的配置，你可以通过地址 `http://localhost:8080/engine-rest` 访问api。

因为我们使用Jersey，所以可以使用spring boot的[通用应用程序属性](http://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html). 
例如，要更改应用程序路径，请使用：
```properties
spring.jersey.application-path=myapplicationpath
```

为了修改配置或注册额外的资源，可以提供一个集成自 `org.camunda.bpm.spring.boot.starter.rest.CamundaJerseyResourceConfig` 的配置类:

```java
@Component
@ApplicationPath("/engine-rest")
public class JerseyConfig extends CamundaJerseyResourceConfig {

  @Override
  protected void registerAdditionalResources() {
    register(...);
  }

}
```
