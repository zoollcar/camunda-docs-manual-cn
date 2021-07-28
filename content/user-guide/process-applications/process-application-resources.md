---

title: 'Process Application 资源访问'
weight: 31

menu:
  main:
    identifier: "user-guide-process-application-resource-access"
    parent: "user-guide-process-applications"

---

流程应用程序提供特定于它们所包含的流程的资源并对其进行逻辑分组。 有些资源是应用程序本身的一部分，例如类加载器及其类和资源，以及在运行时由流程引擎管理的资源，例如一组 [脚本引擎]({{< ref "/user-guide/process-engine/scripting.md" >}}) 或 [Spin 数据格式]({{< ref "/user-guide/data-formats/_index.md" >}})。 本节描述了流程引擎在哪些条件下在流程应用程序级别查询资源以及如何执行该查询。

{{< img src="../img/process-application-context.png" title="Process Application Context" >}}


# 上下文切换

在执行流程实例时，流程引擎必须知道哪个流程应用程序提供了相应的资源。然后它在内部执行*上下文切换*。这有以下影响：

* 流程上下文类加载器设置为流程应用类加载器。这允许从流程应用程序加载类，例如 Java 委托实现。
* 流程引擎可以访问它为该特定流程应用程序管理的资源。这允许调用特定于流程应用程序的脚本引擎或 Spin 数据格式。

例如，在调用 Java Delegate 之前，流程引擎执行上下文切换到相应的流程应用程序。因此，可以将流程上下文类加载器设置为流程应用程序类加载器。如果不执行上下文切换，则只有那些可在流程引擎级别访问的资源可用。这通常是不同的类加载器和一组不同的托管资源。

{{< note title="上下文切换背后的机制" >}}
请注意，上下文切换背后的实际机制取决于平台。 例如：在Apache Tomcat这样的servlet容器中，只需要将Thread当前的Context Classloader设置为web应用的Classloader即可。 上下文特定的操作，例如应用程序本地 JNDI 名称的解析，都建立在此之上。 在 EJB 容器中，这更复杂。 这就是 ProcessApplication 类在该环境中本身就是 EJB 的原因（请参阅：[Ejb Process Application]({{< ref "/user-guide/process-applications/the-process-application-class.md#invocation-semantics- of-the-ejbprocessapplication" >}}))。 然后，流程引擎可以将对该 EJB 的业务方法的调用添加到调用堆栈中，并让应用服务器在后台执行其特定的逻辑。
{{</ note >}}

在以下情况下可以保证上下文切换：

* **委托代码调用**：每当流程引擎调用 Java 委托、执行/任务侦听器（Java 代码或脚本）等委托代码时
* **显式流程应用程序上下文声明**：对于每个引擎 API 调用，当使用实用程序类 `org.camunda.bpm.application.ProcessApplicationContext` 声明流程应用程序时

# 声明流程应用程序上下文

每当自定义代码使用不属于委托代码一部分的引擎 API 时，以及需要上下文切换以实现正确功能时，都必须声明流程应用程序上下文。

## 案例

为了阐明用例，我们假设流程应用程序使用 [功能以 JSON 格式序列化对象类型变量]({{< ref "/user-guide/data-formats/json.md#serializing-process-variables">}})。 但是，对于该应用程序，应自定义 JSON 序列化（考虑将日期序列化为 JSON 字符串的多种方法）。 因此，流程应用程序包含一个 Camunda Spin 数据格式配置器实现，它以所需的方式配置 Spin JSON 数据格式。 反过来，流程引擎为该特定流程应用程序管理 Spin 数据格式以序列化对象值。 现在，我们假设一个 Java servlet 调用流程引擎 API 来提交一个 Java 对象并将其序列化为 JSON 格式。 代码可能如下所示：

```java
public class ObjectValueServlet extends HttpServlet {

  protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
    JsonSerializable object = ...; // a custom Java object
    ObjectValue jsonValue = Variables
      .objectValue(jsonSerializable)
      .serializationDataFormat("application/json")
      .create();

    RuntimeService runtimeService = ...; // obtain runtime service
    runtimeService.setVariable("processInstanceId", "variableName", jsonValue);
  }
}
```

请注意，引擎 API 不是从委托代码中调用的，而是从 servlet 调用的。 因此，流程引擎不知道流程应用程序上下文，并且无法执行上下文切换以使用正确的 JSON 数据格式进行变量序列化。 因此，特定于流程应用程序的 JSON 配置不适用。

在这种情况下，可以使用类“org.camunda.bpm.application.ProcessApplicationContext”提供的静态方法来声明流程应用程序上下文。 特别是，`#setCurrentProcessApplication` 方法声明了流程应用程序要切换到以下引擎 API 调用。 方法`#clear` 重置这个声明。 在示例中，我们相应地包装了 `#setVariable` 调用：

```java
try {
  ProcessApplicationContext.setCurrentProcessApplication("nameOfTheProcessApplication");
  runtimeService.setVariable("processInstanceId", "variableName", jsonValue);
} finally {
  ProcessApplicationContext.clear();
}
```

现在，流程引擎知道在哪个上下文中执行 `#setVariable` 调用。因此它可以访问正确的 JSON 数据格式并正确地序列化变量。

## Java API

方法 ProcessApplicationContext#setCurrentProcessApplication 为所有后续 API 调用声明流程应用程序上下文，直到调用 ProcessApplicationContext#clear 。 因此，建议使用 try-finally 块以确保即使在引发异常时也能清除。 此外，方法 ProcessApplicationContext#withProcessApplicationContext 执行 Callable 并在 Callable 执行期间声明上下文。

## 编程模型集成

每当调用引擎 API 时声明流程应用程序上下文会导致高度重复的代码。 根据你的编程模型，你可以考虑在适用于所有所需业务逻辑的地方以横切方式声明上下文。 例如，在 CDI 中，可以定义基于注解的存在触发的方法调用拦截器。 这样的拦截器可以根据注解识别流程应用并透明地声明上下文。
