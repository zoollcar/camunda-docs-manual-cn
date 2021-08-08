---

title: 'Job Execution 与 托管服务'
weight: 60

menu:
  main:
    identifier: "user-guide-runtime-container-integration-job--execution-other-jee"
    parent: "user-guide-runtime-container-integration"

---

对于[支持的环境]({{<relref "../../../introduction/supported-environments.md#container-managed-process-engine-and-camunda-cockpit-tasklist-admin">}}) ，Camunda 平台提供了将Job执行与应用程序服务器的托管线程池集成的服务器模块。如果你使用其中一种环境，建议使用随附的集成。

此页面上的描述适用于 *不提供* 现有资源感知实现的用例。 在这些情况下，建议使用应用程序服务器提供的托管资源而不是使用非托管资源。为了使集成工作，需要一个符合 JEE 7+ 的应用服务器。

# ManagedJobExecutor

没有资源感知实现的应用程序服务器的集成由称为“ManagedJobExecutor”的特定类型的“JobExecutor”提供。 `ManagedJobExecutor` 的目的是通过使用托管资源（主要是：托管线程）确保流程引擎中的作业执行由应用程序服务器正确控制。

引擎必须配置以使用`ManagedJobExecutor`。 例如，当从 Java 代码引导引擎时，你将创建一个“ManagedJobExecutor”的新实例，并通过从应用程序服务器的环境中注入它来提供它所具有的资源依赖关系。 然后可以将“ManagedJobExecutor”设置为流程引擎应该使用的“JobExecutor”。

## 示例用法

以下代码示例了基本配置。

```java

@ApplicationScoped
public class EngineBuilder {

  // 从应用服务器注入 ManagedExecutorService
  @Resource
  private ManagedExecutorService managedExecutorService;
  
  private ProcessEngine processEngine;
  private ManagedJobExecutor managedJobExecutor;

  @PostConstruct
  public void build() {
  	// 创建一个新的 ManagedJobExecutor
  	managedJobExecutor = new ManagedJobExecutor(this.managedExecutorService);

  	// 创建一个流程引擎配置
    ProcessEngineConfigurationImpl engineConfiguration = ...

    // 其他配置

    // 使用 ManagedJobExecutor
    engineConfiguration.setJobExecutor(managedJobExecutor);
    
    // 构建流程引擎
    processEngine = engineConfiguration.buildProcessEngine();
  }
  
  @PreDestroy
  public void stopEngine() {
    // 确保引擎和Job执行器也关闭
    processEngine.close();
    managedJobExecutor.shutdown();
  }
}
```

{{< note title="非托管资源" class="info" >}}
  上面的示例将容器管理的资源“ManagedExecutorService”注入到生命周期不受应用程序服务器**控制的对象（“ManagedJobExecutor”，通过其构造函数实例化）。 这不是通常推荐的做法，因为注入的依赖项可能变得不可用。

   然而，在这个用例中，之所以选择这种方法，是因为`ManagedJobExecutor` 依赖于`ManagedExecutorService` 的存在，并且该接口仅在JEE7 中引入。 早期版本的 JEE 无法满足这种依赖性，并且如果为所有应用程序服务器自动激活该组件，则会出现故障。

   为了避免作业执行器在不可用的资源上运行，我们建议在“ManagedExecutorService”变得不可用时通过其“shutdown()”方法关闭作业执行器。
{{< /note >}}
