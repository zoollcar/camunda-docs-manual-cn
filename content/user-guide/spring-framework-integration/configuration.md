---

title: 'Process Engine 配置'
weight: 10

menu:
  main:
    name: "引导"
    identifier: "user-guide-spring-framework-integration-configuration"
    parent: "user-guide-spring-framework-integration"
    pre: "Bootstrap the Process Engine via Spring context XML or JavaConfig"

---

你可以使用 Spring 应用上下文 XML 文件来引导流程引擎。 可以通过 Spring 引导应用管理和容器管理的流程引擎。

请注意，你还可以使用 [Spring JavaConfig]({{< relref "#using-spring-javaconfig" >}}) 代替 XML 进行引导。

# 配置应用程序管理的流程引擎

ProcessEngine 可以配置为常规的 Spring bean。 集成的起点是类 `org.camunda.bpm.engine.spring.ProcessEngineFactoryBean`。 该 bean 采用流程引擎配置并创建流程引擎。 这意味着 Spring 属性的创建和配置与配置部分中记录的相同。 对于 Spring 集成，配置和引擎 bean 将如下所示：

```xml
<bean id="processEngineConfiguration"
      class="org.camunda.bpm.engine.spring.SpringProcessEngineConfiguration">
    ...
</bean>

<bean id="processEngine"
      class="org.camunda.bpm.engine.spring.ProcessEngineFactoryBean">
  <property name="processEngineConfiguration" ref="processEngineConfiguration" />
</bean>
```

请注意， processEngineConfiguration bean 使用 {{< javadocref page="?org/camunda/bpm/engine/spring/SpringProcessEngineConfiguration.html" text="SpringProcessEngineConfiguration" >}} 类。


# 将容器管理的流程引擎配置为 Spring Bean

如果你希望流程引擎注册到 Camunda Platform ProcessEngineService，你必须使用 `org.camunda.bpm.engine.spring.container.ManagedProcessEngineFactoryBean` 而不是上面示例中显示的 ProcessEngineFactoryBean。 你还需要确保：

1. 你的所有 web 应用都不要在它们自己的 lib 文件夹中包含 camunda-webapp\*.jar，这应该处于共享级别。
2. server.xml 包含“ProcessEngineService”和“ProcessApplicationService”的 JNDI 条目，如下所示：

```xml
<!-- Global JNDI resources
     Documentation at /docs/jndi-resources-howto.html
-->
  <GlobalNamingResources>

    <Resource name="java:global/camunda-bpm-platform/process-engine/ProcessEngineService!org.camunda.bpm.ProcessEngineService"
              auth="Container"
              type="org.camunda.bpm.ProcessEngineService"
              description="Camunda Platform Process Engine Service"
              factory="org.camunda.bpm.container.impl.jndi.ProcessEngineServiceObjectFactory" />

    <Resource name="java:global/camunda-bpm-platform/process-engine/ProcessApplicationService!org.camunda.bpm.ProcessApplicationService"
              auth="Container"
              type="org.camunda.bpm.ProcessApplicationService"
              description="Camunda Platform Process Application Service"
              factory="org.camunda.bpm.container.impl.jndi.ProcessApplicationServiceObjectFactory" />
       ...
  </GlobalNamingResources>
```

在这种情况下，构建的流程引擎对象会注册到 Camunda 平台，并且可以被引用以创建流程应用程序部署并通过运行时容器集成公开。


# 配置流程引擎插件

在 Spring 中，你可以通过将列表值设置为
`processEngineConfiguration` bean 的 `processEnginePlugins` 属性：

```xml
<bean id="processEngineConfiguration" class="org.camunda.bpm.engine.spring.SpringProcessEngineConfiguration">
  ...
  <property name="processEnginePlugins">
    <list>
      <bean id="spinPlugin"
            class="org.camunda.spin.plugin.impl.SpinProcessEnginePlugin" />
    </list>
  </property>
</bean>
```

# 使用 Spring JavaConfig

除了 Spring 应用程序上下文 XML 文件之外，你还可以使用 Spring JavaConfig 引导流程引擎。 配置类可以是这样的：

```java
@Configuration
public class ExampleProcessEngineConfiguration {

  @Bean
  public DataSource dataSource() {
     // 使用 JNDI 数据源或从 env 或属性文件中读取属性。
     // 注意：以下仅演示了使用 内存数据库H2 作为一个简单数据源。

    SimpleDriverDataSource dataSource = new SimpleDriverDataSource();
    dataSource.setDriverClass(org.h2.Driver.class);
    dataSource.setUrl("jdbc:h2:mem:camunda;DB_CLOSE_DELAY=-1");
    dataSource.setUsername("sa");
    dataSource.setPassword("");
    return dataSource;
  }

  @Bean
  public PlatformTransactionManager transactionManager() {
    return new DataSourceTransactionManager(dataSource());
  }

  @Bean
  public SpringProcessEngineConfiguration processEngineConfiguration() {
    SpringProcessEngineConfiguration config = new SpringProcessEngineConfiguration();

    config.setDataSource(dataSource());
    config.setTransactionManager(transactionManager());

    config.setDatabaseSchemaUpdate("true");
    config.setHistory("audit");
    config.setJobExecutorActivate(true);

    return config;
  }

  @Bean
  public ProcessEngineFactoryBean processEngine() {
    ProcessEngineFactoryBean factoryBean = new ProcessEngineFactoryBean();
    factoryBean.setProcessEngineConfiguration(processEngineConfiguration());
    return factoryBean;
  }

  @Bean
  public RepositoryService repositoryService(ProcessEngine processEngine) {
    return processEngine.getRepositoryService();
  }

  @Bean
  public RuntimeService runtimeService(ProcessEngine processEngine) {
    return processEngine.getRuntimeService();
  }

  @Bean
  public TaskService taskService(ProcessEngine processEngine) {
    return processEngine.getTaskService();
  }

  // 更多的引擎服务和额外的 bean ...

}
```

请注意，你可以在配置类中定义自定义 bean，结合附加的 XML 文件或使用组件扫描。 下面的示例在配置类中添加了一个组件扫描，以检测并实例化包“com.example”中的所有bean。

```java
@Configuration
@ComponentScan("com.example")
public class ExampleProcessEngineConfiguration {

  // ...

}
```


