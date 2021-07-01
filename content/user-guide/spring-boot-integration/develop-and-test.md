---

title: "部署和测试 Spring Boot 应用"
weight: 70

menu:
  main:
    name: "部署和测试"
    identifier: "user-guide-spring-boot-develop-and-test"
    parent: "user-guide-spring-boot-integration"

---

# 开发

Spring Boot提供[Developer Tools](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.devtools) 提供了在应用程序开发期间自动重启和实时重新加载等功能。

## Spring Developer Tools 和 Classloading

如果你的应用程序处于开发模式，一个额外的流程引擎插件（`ApplicationContextClassloaderSwitchPlugin`）将被加载。

* Spring Developer工具（`spring-boot-devtools`库）在classpath上 * 并且被启用，例如，应用程序在IDE中启动。

该插件确保将应用程序上下文的classloader与用于流程引擎的classloader进行交换，以防止反序列化过程中出现问题。

# 测试

Spring为自动[测试](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testing-introduction)提供强大的支持。
可以通过专用的mock包、测试工具和注解 进行测试
当测试Spring和Spring Boot应用程序时，需要大量的时间来加载`ApplicationContext`。这就是为什么Spring在测试结束后会缓存一个`ApplicationContext`。这使得它可以在以后的测试中以相同的配置重复使用。

## 流程引擎的上下文缓存

要将流程引擎包含在 "ApplicationContext" 缓存中，需要一些额外的配置。
这是因为流程引擎需要一个静态定义的名称（如果没有定义名称，则使用 "default"），这导致Spring试图为相同名称的流程引擎创建多个`ApplicationContext`。这将导致测试的行为不正确，或者在最坏的情况下，完全无法加载`ApplicationContext`。

## 使用不同的 流程引擎/应用 名称

为了使上下文缓存与流程引擎和流程应用正常工作，它们需要对每个不同的测试配置具有唯一的名称。

当定义一个新的测试配置时，确保新的ApplicationContext使用新的流程引擎（和流程应用）的最简单方法是在你的`@SpringBootTest`注解中启用以下属性。

```java
@SpringBootTest(
  // ...其他参数...
  properties = {
    "camunda.bpm.generate-unique-process-engine-name=true",
    // 这只在 SpringBootProcessApplication 中需要
    // 用于测试
    "camunda.bpm.generate-unique-process-application-name=true",
    "spring.datasource.generate-unique-name=true",
    // 额外的属性...
  }
)
```

* `camunda.bpm.generate-unique-process-engine-name=true` 参数将生成流程引擎的唯一名称 (例如 'processEngine2Sc4bg2s1g').
* `camunda.bpm.generate-unique-process-application-name=true` 参数将生成流程应用的唯一名称 (例如 'processApplication2Sc4bg2s1g')。如果你想用多种配置多次部署和测试一个流程应用程序，这会很有用。
* `spring.datasource.generate-unique-name=true` 属性将为每个新的`ApplicationContext`生成一个新的数据源。重复使用（缓存）的`ApplicationContext` 将使用相同的数据源。

{{< note title="" class="warning" >}} 
请注意，`generate-unique-process-engine-name`和`process-engine-name`属性是互斥的。同时设置这两个属性将导致异常。
{{< /note >}}

如果在一个给定的测试中需要使用一个静态访问器（例如processEngines.getProcessEngine(name)），那么可以使用以下属性：

```java
@SpringBootTest(
  // 其他参数
  properties = {
    "camunda.bpm.process-engine-name=foo",
    // 这只在 SpringBootProcessApplication 中需要
    // 用于测试
    "camunda.bpm.generate-unique-process-application-name=true",
    "spring.datasource.generate-unique-name=true",
    // 额外的属性
  }
)
```
这里的 `camunda.bpm.process-engine-name=foo` 将设置（一个唯一的名称）"foo" 作为流程引擎的名称。

## 禁用 遥测报告（Telemetry）

遥测报告是由Camunda Platform 7.14.0引入的。为了防止发送测试期间产生的数据，我们鼓励你禁用[telemetry reporter][engine-config-telemetryReporterActivate]。请在[Telemetry][telemetry-initial-report]的专门页面上阅读更多关于该主题的内容。

在Spring Boot设置中禁用遥测报告的例子。

```
camunda.bpm:
  generic-properties.properties:
    telemetry-reporter-activate: false
```

[engine-config-telemetryReporterActivate]: {{< ref "/reference/deployment-descriptors/tags/process-engine.md#telemetryReporterActivate" >}}
[telemetry-initial-report]: {{< ref "/introduction/telemetry.md#initial-data-report" >}}

## Camunda 断言

[Camunda Platform Assertions]({{< ref "/user-guide/testing/_index.md#camunda-assertions" >}})库与Camunda Spring Boot Starter集成，可以使Spring Boot应用程序的测试过程更加简单。

### 使用带有上下文缓存的断言

默认情况下，Camunda平台断言库会尝试使用默认引擎或可用的（单一）引擎。由于在使用上下文缓存时，在不同的上下文中使用多个引擎，为了使缓存和断言正常工作，需要将正确的流程引擎绑定到Camunda断言库中。这可以通过测试类中的以下初始化代码完成：

```java
  @Autowired
  ProcessEngine processEngine;  

  @Before
  public void setUp() {
    init(processEngine);
  }
```

这需要在[上面一节](#使用不同的-流程引擎-应用-名称)中描述的 _不同的的流程引擎/应用程序名称_ 要求之上进行。
