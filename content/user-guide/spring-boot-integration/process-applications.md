---

title: "Process applications"
weight: 50

menu:
  main:
    name: "Process Applications"
    identifier: "user-guide-spring-boot-process-applications"
    parent: "user-guide-spring-boot-integration"

---

默认情况下，camunda-spring-boot-starter 使用`SpringProcessEngineConfiguration` 配置自动部署功能。
从1.2.0开始，你也可以通过 `SpringBootProcessApplication` 配置。这将禁用`SpringProcessEngineConfiguration` 的自动部署功能。
自动部署功能，使用所需的 `META-INF/processes.xml` 作为资源扫描的目录。
允许使用的所有 `processes.xml` 配置项在 [这里]({{<ref "/user-guide/process-applications/the-processes-xml-deployment-descriptor.md">}}) 列出。

你这只需要添加 `@EnableProcessApplication` 注解到Spring Boot application类：

```java
@SpringBootApplication
@EnableProcessApplication("myProcessApplicationName")
public class MyApplication {

...

}
```

一些配置可以通过Spring Boot配置参数完成。详情见[可用参数的列表]({{<ref "/user-guide/spring-boot-integration/configuration.md#camunda-bpm-application">}}). 

## 使用部署回调函数

由于使用`@EnableProcessApplication`时，我们没有扩展`ProcessApplication`类，所以我们不能使用`@PostDeploy`和`@PreUndeploy`方法注释。相反，这些回调是通过Spring事件发布机制提供的。所以你可以使用以下事件监听器。

```java
@EventListener
public void onPostDeploy(PostDeployEvent event) {
  ...
}

@EventListener
public void onPreUndeploy(PreUndeployEvent event) {
  ...
}
```