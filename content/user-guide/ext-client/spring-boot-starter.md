---

title: '外部任务客户端 Spring Boot Starter'
weight: 200

menu:
  main:
    name: "Spring Boot Starter"
    identifier: "external-task-client-spring-boot-starter"
    parent: "external-task-client"

---

Camunda 为外部任务客户端提供了一个 Spring Boot Starter。 这允许你可以通过将以下 Maven 依赖项添加到你的 `pom.xml` 文件，轻松地将外部任务客户端添加到你的 Spring Boot 应用程序：
```xml
<dependency>
  <groupId>org.camunda.bpm.springboot</groupId>
  <artifactId>camunda-bpm-spring-boot-starter-external-task-client</artifactId>
  <version>{{< minor-version >}}.0</version>
</dependency>
```

请查看我们的 [外部任务客户端 Spring Boot Starter 示例](https://github.com/camunda/camunda-bpm-examples/tree/{{<minor-version>}}#external-task-client-spring-boot).

客户端可以订阅在你的 BPMN 流程模型中定义的一个或多个主题名。
当执行在外部任务中等待时，客户端会执行你的自定义业务逻辑。
例如，检查客户的信用评分，如果成功，则外部任务可以标记为已完成并继续执行。

## 主题订阅

允许实现自定义业务逻辑并与引擎交互的接口称为“ExternalTaskHandler”。 订阅由主题名称标识，并使用对“ExternalTaskHandler” bean 的引用进行配置。

你可以通过定义一个返回类型为 `ExternalTaskHandler` 的 bean 并使用以下注释对该 bean 进行注释，从而为客户端订阅主题名称 `creditScoreChecker`：

```java
@ExternalTaskSubscription("creditScoreChecker")
```

注释至少需要一个主题名称。 但是，你可以通过引用“application.yml”文件中的主题名称来使用更多配置选项：

```yaml
camunda.bpm.client:
  base-url: http://localhost:8080/engine-rest
  subscriptions:
    creditScoreChecker:
        process-definition-key: loan_process
        include-extension-properties: true
        variable-names: defaultScore
```

或者，通过在注释中定义配置属性：

```java
@ExternalTaskSubscription(
  topicName = "creditScoreChecker",
  processDefinitionKey = "loan_process",
  includeExtensionProperties = true,
  variableNames = "defaultScore"
)
```

请在 {{< javadocref page="?org/camunda/bpm/client/spring/annotation/ExternalTaskSubscription.html" text="Javadocs" >}} 中找到完整的属性列表。

**请注意：** `application.yml` 文件中定义的属性始终覆盖通过注释以编程方式定义的相应属性。

### 处理程序配置示例

请考虑以下完整的处理程序 bean 示例：

```java
@Configuraton
@ExternalTaskSubscription("creditScoreChecker")
public class CreditScoreCheckerHandler implements ExternalTaskHandler {

  @Override
  public void execute(ExternalTask externalTask, 
                      ExternalTaskService externalTaskService) {
    // add your business logic here
  }

}
```

如果要在一个配置类中定义多个处理程序 bean，可以按如下方式进行：

```java
@Configuraton
public class HandlerConfiguration {

  @Bean
  @ExternalTaskSubscription("creditScoreChecker")
  public ExternalTaskHandler creditScoreCheckerHandler() {
    return (externalTask, externalTaskService) -> {
      // 在此添加你的业务逻辑
      externalTaskService.complete(externalTask);
    };
  }

  @Bean
  @ExternalTaskSubscription("loanGranter")
  public ExternalTaskHandler loanGranterHandler() {
    return (externalTask, externalTaskService) -> {
      // 在此添加你的业务逻辑
      externalTaskService.complete(externalTask);
    };
  }

}
```

### 打开/关闭主题订阅

如果没有进一步配置，Spring Boot 应用程序启动时会自动打开一个主题订阅，这意味着客户端立即启动以获取与主题名称相关的外部任务。

可能存在应用程序启动时不应立即打开主题订阅的情况。 你可以通过 [`auto-open`](#auto-open) 标志来控制它。

SpringTopicSubscription 接口允许你在订阅初始化后立即以编程方式打开或关闭主题。 应用程序一启动就会触发初始化过程。

当订阅被初始化时，会发出一个`SubscriptionInitializedEvent`，并且可以打开或关闭主题订阅：

```java
@Configuration
public class SubscriptionInitializedListener 
    implements ApplicationListener<SubscriptionInitializedEvent> {

  @Override
  public void onApplicationEvent(SubscriptionInitializedEvent event) {

    SpringTopicSubscription topicSubscription = event.getSource();

    String topicName = topicSubscription.getTopicName();
    boolean isOpen = topicSubscription.isOpen();
    if ("creditScoreChecker".equals(topicName)) {

      if(!isOpen) {
        // 开始获取外部任务
        topicSubscription.open();

      } else {
        // 停止获取外部任务
        topicSubscription.close();

      }
    }
  }

}
```

## 配置

### `application.yml` 文件

The central configuration point is the `application.yml` file.

#### 客户自动启动

请确保将属性与`camunda.bpm.client`前缀一起配置：

示例配置：

```yaml
camunda.bpm.client:
  base-url: http://localhost:8080/engine-rest
  worker-id: spring-boot-worker
```

可用的属性包括：

<table class="table desc-table">
  <thead>
    <tr>
      <th style="width: 20%;">属性名</th>
      <th style="width: 60%;">描述</th>
      <th style="width: 20%;">默认值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>base-url</code></td>
      <td>
        <strong>强制的:</strong> Camunda 平台运行时 REST API 的 Base URL。
      </td>
      <td></td>
    </tr>
    <tr>
      <td><code>worker-id</code></td>
      <td>
        流程引擎可用的自定义 worker ID。 <br>
        <strong>注意:</strong>确保 worker ID 唯一。
      </td>
      <td>hostname + 128 bit UUID</td>
    </tr>
    <tr>
      <td><code>max-tasks</code></td>
      <td>
        指定在一个请求中可以获取的最大任务数。
      </td>
      <td><code>10</code></td>
    </tr>
    <tr>
      <td><code>use-priority</code></td>
      <td>
        指定是根据任务的优先级还是任意地获取任务。
      </td>
      <td><code>true</code></td>
    </tr>
    <tr>
      <td><code>async-response-timeout</code></td>
      <td>
        如果给出超时，则启用异步响应（长轮询）。
        指定获取和锁定的外部任务响应的最长等待时间。 如果在发出请求时外部任务可用，则立即执行响应。
      </td>
      <td>
        <code>null</code>
      </td>
    </tr>
    <tr>
      <td><code>disable-auto-fetching</code></td>
      <td>
        在引导客户端后禁用对外部任务的立即获取。 要开始获取 <code>External Task Client#start()</code> 必须被调用。
      </td>
      <td><code>false</code></td>
    </tr>
    <tr>
      <td><code>disable-backoff-strategy</code></td>
      <td>
        禁用客户端退避策略。 当设置为 <code>true</code> 时，<code>BackoffStrategy</code> bean 将被忽略。<br /><br /> <strong>注意事项：</strong> 请记住禁用 客户端退避可能会导致引擎端出现重负载情况。 为避免这种情况，请指定适当的<code>async-response-timeout</code>.
      </td>
      <td><code>false</code></td>
    </tr>
    <tr>
      <td><code>lock-duration</code></td>
      <td>
        指定外部任务被锁定的毫秒数。 必须大于零。 它被主题订阅上配置的锁定持续时间覆盖
      </td>
      <td><code>20,000</code></td>
    </tr>
    <tr>
      <td><code>date-format</code></td>
      <td>指定用于 序列化/反序列化 日期变量的日期格式。</td>
      <td><code>yyyy-MM-dd'T'HH:mm:ss.SSSZ</code></td>
    </tr>
    <tr>
      <td><code>default-serialization-format</code></td>
      <td>
        指定在未请求特定格式时用于序列化对象的序列化格式。
      </td>
      <td><code>application/json</code></td>
    </tr>
  </tbody>
</table>

#### 主题订阅

主题订阅的属性位于：`camunda.bpm.client.subscriptions`

可以为每个主题名称应用配置属性，如下所示：

```yaml
camunda.bpm.client:
  # 在此添加客户端配置
  subscriptions:
    creditScoreChecker:
        process-definition-key: loan_process
        include-extension-properties: true
        variable-names: defaultScore
    loanGranter:
        process-definition-key: loan_process
```

可用的属性：

<table class="table desc-table">
  <thead>
    <tr>
      <th style="width: 20%;">属性名</th>
      <th style="width: 60%;">描述</th>
      <th style="width: 20%;">默认值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>${TOPIC_NAME}</code></td>
      <td>
        客户订阅的 BPMN 流程模型中服务任务的主题名称。
      </td>
      <td></td>
    </tr>
    <tr id="auto-open">
      <td><code>auto-open</code></td>
      <td>
        当<code>false</code>时，应用开始调用<code>SpringTopicSubscription#open()</code>后，即可开启主题订阅。 否则，客户端立即开始获取外部任务。
      </td>
      <td><code>true</code></td>
    </tr>
    <tr>
      <td><code>lock-duration</code></td>
      <td>
        指定外部任务被锁定的毫秒数。 必须大于零。 覆盖在引导客户端时配置的锁定持续时间。
      </td>
      <td><code>20,000</code></td>
    </tr>
    <tr>
      <td><code>variable-names</code></td>
      <td>
        应该查询的变量的变量名称。
        默认情况下查询所有变量。
      </td>
      <td><code>null</code></td>
    </tr>
    <tr>
      <td><code>local-variables</code></td>
      <td>
        是否应该从比外部任务更大的范围内获取变量。 当 <code>false</code> 时，将获取范围内可见的所有变量。 当 <code>true</code> 时，只会获取外部任务范围内的局部变量。
      </td>
      <td><code>false</code></td>
    </tr>
    <tr>
      <td><code>include-extension-properties</code></td>
      <td>
        是否为获取的外部任务包含自定义扩展属性。 当<code>true</code> 时，将提供外部服务任务中定义的所有<code>extensionProperties</code>。 当<code>false</code> 时，外部服务任务中定义的<code>extensionProperties</code> 将被忽略。
      </td>
      <td><code>false</code></td>
    </tr>
    <tr>
      <td><code>business-key</code></td>
      <td>
        仅获取与指定business key相关的外部任务。
      </td>
      <td></td>
    </tr>
    <tr>
      <td><code>process-definition-id</code></td>
      <td>
        仅获取与指定流程定义 ID 相关的外部任务。
      </td>
      <td></td>
    </tr>
    <tr>
      <td><code>process-definition-id-in</code></td>
      <td>
        仅获取与流程定义 ID 的指定列表相关的外部任务。 id 列表具有逻辑 OR 语义。
      </td>
      <td></td>
    </tr>
    <tr>
      <td><code>process-definition-key</code></td>
      <td>
        仅获取与指定流程定义Key相关的外部任务。
      </td>
      <td></td>
    </tr>
    <tr>
      <td><code>process-definition-key-in</code></td>
      <td>
        仅获取与流程定义Key的指定列表相关的外部任务。 Key列表具有逻辑 OR 语义。
      </td>
      <td></td>
    </tr>
    <tr>
      <td><code>process-definition-version-tag</code></td>
      <td>
        仅获取与指定流程定义版本标记相关的外部任务。
      </td>
      <td></td>
    </tr>
    <tr>
      <td><code>process-variables</code></td>
      <td>
        仅获取与流程变量（key：变量名，value：变量值）的指定映射相关的外部任务。 变量映射具有逻辑 OR 语义。
      </td>
      <td></td>
    </tr>
    <tr>
      <td><code>without-tenant-id</code></td>
      <td>
        仅获取没有 租户ID 的外部任务。
      </td>
      <td></td>
    </tr>
    <tr>
      <td><code>tenant-id-in</code></td>
      <td>
        仅获取与指定 租户ID 列表相关的外部任务。 id 列表具有逻辑 OR 语义。
      </td>
      <td></td>
    </tr>
  </tbody>
</table>

#### 日志

要记录客户端的内部工作，你可以将记录器 `org.camunda.bpm.client.spring` 的级别设置为 `DEBUG`。

你可以在“application.yml”文件中设置日志级别，如下所示：

```yaml
logging.level.org.camunda.bpm.client.spring: DEBUG
```

如果要调试，提高记录器 `org.camunda.bpm.client` 的级别也可能会有所帮助。

### 请求拦截器

每当客户端执行 HTTP 请求时，都会调用请求拦截器。 例如，你可以使用此扩展点来实现 OAuth 2.0 等自定义身份验证策略。

你可以通过定义`ClientRequestInterceptor`类型的bean来注册一个或多个请求拦截器：

```java
@Configuration
public class RequestInterceptorConfiguration implements ClientRequestInterceptor {
  // ...
}
```

### 退避策略

默认情况下，客户端使用指数退避策略。你可以通过定义一个 `BackoffStrategy` 类型的 bean 来用自定义策略替换它：

```java
@Configuration
public class BackoffStrategyConfiguration implements BackoffStrategy {
  // ...
}
```

### 解析属性

通过定义一个`PropertySourcesPlaceholderConfigurer`类型的bean，可以从自定义属性文件中解析基于字符串的客户端配置属性：

```java
@Configuration
public class PropertyPlaceholderConfiguration 
    extends PropertySourcesPlaceholderConfigurer {

  public PropertyPlaceholderConfiguration() {
    // 指定包含属性占位符的 *.properties 文件名
    Resource location = new ClassPathResource("client.properties");
    setLocation(location);
  }

}
```

使用上面显示的示例时，客户端尝试从 `client.properties` 文件解析基于字符串的属性，如下所示：

```properties
client.baseUrl=http://localhost:8080/engine-rest
client.workerId=spring-boot-worker
client.dateFormat=yyyy-MM-dd'T'HH:mm:ss.SSSZ
client.serializationFormat=application/json
```

确保在你的“application.yml”文件中引用上面定义的相应占位符：

```yaml
camunda.bpm.client:
  base-url: ${client.baseUrl}
  worker-id: ${client.workerId}
  date-format: ${client.dateFormat}
  default-serialization-format: ${client.serializationFormat}
```

### 自定义客户端

你可以以编程方式引导客户端，这会跳过客户端的内部创建：

```java
@Configuration
public class CustomClientConfiguration { 

  @Bean
  public ExternalTaskClient customClient() {
    return ExternalTaskClient.create()
        .baseUrl("http://localhost:8080/engine-rest")
        .build();
  }

}
```

## Beans

你可以定义处理程序 bean，但更多的 bean 是在内部定义的，它们超出你的控制。
但是，可以通过自动连接访问这些 bean。

### 客户端 Bean

如果用户尚未定义（请参阅 [Custom Client](#custom-client)），则会构造一个名为“externalTaskClient”、类型为“ExternalTaskClient”的 bean。

### 订阅 Bean

基于用`@ExternalTaskSubscription` 注释的处理程序bean，构造了一个类型为`SpringTopicSubscription` 的订阅bean。 bean的名称由以下部分组成：

```
handler bean name + "Subscription"
```

例如，以下处理程序 bean 定义：

```java
@Bean
@ExternalTaskSubscription("creditScoreChecker")
public ExternalTaskHandler creditScoreCheckerHandler() {
  // ...
}
```

将导致订阅 bean 名称：

```
creditScoreCheckerHandlerSubscription
```

## 仅Spring可用的模块

如果你想使用 Spring 而不是 Spring Boot，你可以将以下 Maven 依赖添加到你的 `pom.xml` 文件中：

```xml
<dependency>
  <groupId>org.camunda.bpm</groupId>
  <artifactId>camunda-external-task-client-spring</artifactId>
  <version>{{< minor-version >}}.0</version>
</dependency>
```

要引导客户端，请使用类注释 `@EnableExternalTaskClient`。 你可以在 {{< javadocref page="?org/camunda/bpm/client/spring/annotation/EnableExternalTaskClient.html" text="Javadocs" >}} 中找到所有配置属性。

