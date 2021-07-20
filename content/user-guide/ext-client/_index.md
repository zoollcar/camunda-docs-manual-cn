---

title: '外部任务客户端'
weight: 270
layout: "single"

menu:
  main:
    identifier: "external-task-client"
    parent: "user-guide"

---

**Camunda 外部任务客户端** 允许为你的工作流设置远程服务任务。 支持 [Java](https://github.com/camunda/camunda-bpm-platform/clients/java)
和 [JavaScript](https://github.com/camunda/camunda-external-task-client-js) 实例。

## 特性
* 完整的外部任务
* 扩展外部任务的锁定持续时间
* 解锁外部任务
* 报告BPMN错误和故障
* 与流程引擎共享变量

## 启动客户端


{{< img src="img/externalTaskCient.png" title="External Task Cient Architecture" >}}


客户端可以处理“外部（external）”类型的服务任务。 为了方便配置和实例化客户端，所有支持的实现都提供了一个方便的接口。
客户端和 Camunda 流程引擎之间的通信是 HTTP。 因此，REST API 的相应 URL 是必填信息。

### 请求拦截器
要向执行的 REST API 请求添加额外的 HTTP 标头，可以使用请求拦截器方法。 这在例如的上下文中变得必要。 验证。

#### 基本的身份验证
在某些情况下，有必要通过基本身份验证来保护 Camunda 流程引擎的 REST API。 对于这种情况，客户端提供了基本身份验证实现。在配置用户凭据后，基本身份验证标头将添加到每个 REST API 请求中。

#### 自定义拦截器
可以在引导客户端时添加自定义拦截器。 有关实施的更多详细信息，请查看与对应客户端相关文档。


### 主题订阅

如果将“外部（external）”类型的服务任务放置在工作流中，则必须指定主题名称。 相应的 BPMN 2.0 XML 可能如下所示：

```xml
...
<serviceTask id="checkCreditScoreTask"
  name="Check credit score"
  camunda:type="external"
  camunda:topic="creditScoreChecker" />
...
```

一旦流程引擎到达 BPMN 流程中的外部任务，就会创建一个相应的活动实例，等待客户端获取和锁定。

客户端订阅主题并不断获取工作流引擎提供的新外部任务。 每个获取的外部任务都标有临时锁。如此一来，没有其他客户端可以同时处理这个外部任务。 锁定在指定的时间段内有效并且可以延长。

设置新主题订阅时，必须指定主题名称和处理程序函数。
订阅主题后，客户端可以通过轮询流程引擎的 API 开始接收工作项。

### 处理程序
处理程序可用于实现自定义方法，只要外部任务被成功获取和锁定，就会调用这些方法。
对于每个主题订阅，都提供了一个外部任务处理程序接口。

处理程序为每个获取并锁定的外部任务顺序调用。

### 完成任务（Task）
一旦处理程序中指定的自定义方法完成后，外部任务就可以完成了。 对于流程引擎来说，这意味着执行将继续进行。 为此，所有支持的实现都有一个可以在处理程序函数中调用的“complete”方法。 但是，外部任务只有在当前被客户端锁定的情况下才能完成。

### 延长任务的锁定时间
有时自定义方法的完成需要比预期更长的时间。 在这种情况下，需要延长锁定持续时间。
可以通过调用 `extendLock` 方法并传递新锁定持续时间来执行此操作。
如果外部任务当前被客户端锁定，则只能延长锁定持续时间。

### 解锁任务（Task）
如果需要解锁外部任务以便允许其他客户端再次获取和锁定此任务，则可以调用“unlock”方法。 只有在任务被当前客户端锁定时，才能解锁外部任务。

### 报告失败
如果客户端遇到无法成功完成外部任务的问题，可以将此问题报告给工作流引擎。 如果外部任务当前被客户端锁定，则只能报告失败。
你可以在 Camunda 平台 [用户指南]({{<ref "/user-guide/process-engine/external-tasks.md#reporting-task-failure">}}) 中找到有关此操作的详细文档。

### 报告BPMN错误
[错误边界事件]({{<ref "/reference/bpmn20/events/error-events.md#error-boundary-event">}}) 由 BPMN 错误触发。 如果外部任务当前被客户端锁定，则只能报告 BPMN 错误。
你可以在 Camunda 平台 [用户指南]({{<ref "/user-guide/process-engine/external-tasks.md#reporting-bpmn-error">}}) 中找到有关此操作的详细文档。

### 变量
两个外部任务客户端都与 Camunda引擎 [支持]({{<ref "/user-guide/process-engine/variables.md#supported-variable-values">}}) 的所有数据类型兼容。
可以使用类型化的或无类型的 API 访问/更改变量。


#### 流程与局部变量
变量可以分为流程或局部变量。
前者设置在变量作用域的最高可能层次结构上，并在整个流程中对其子作用域可用。
相反，如果应该在提供的执行范围内准确设置变量，则可以使用局部类型。

**笔记：** 设置变量并不能确保变量被持久化。 在客户端本地设置的变量仅在运行时可用，如果未通过成功完成当前锁的外部任务与工作流引擎共享，则会丢失。

#### 无类型变量
无类型变量通过使用其值的相应类型来存储。 可以存储/查询单个变量/多个变量。


#### 类型化变量
设置类型变量需要明确指定类型。 也可以检索类型化变量，接收到的对象提供除类型和值之外的各种信息。 当然也可以设置和获取多个类型变量。

##### 示例：使用键入的JSON，XML和对象变量

```java
// 通过订阅获得
ExternalTaskService externalTaskService = ..;
ExternalTask externalTask = ..;

VariableMap variables = Variables.createVariables();

JsonValue jsonCustomer = externalTask.getVariableTyped("customer");
// 反序列化 jsonCustomer.getValue() 为 customer object
// 并修改 ...
variables.put("customer", ClientValues.jsonValue(customerJsonString));

XmlValue xmlContract = externalTask.getvariableTyped("contract");
// 反序列化 xmlContract.getValue() 为 contract object
// 并修改 ...
variables.put("contract", ClientValues.xmlValue(contractXmlString));

TypedValue typedInvoice = externalTask.getVariableTyped("invoice");
Invoice invoice = (Invoice) typedInvoice.getValue();
// 修改 invoice object
variables.put("invoice", ClientValue.objectValue(invoice)
    .serializationDataFormat("application/xml").create();

externalTaskService.complete(externalTask, variables);
```

### 报告

客户端实现支持记录客户端生命周期中的各种事件。
因此，可以报告以下情况：

* 无法成功获取和锁定外部任务
* 出现异常在...
   * 调用处理程序时
   * 反序列化变量
   * 调用请求拦截器
   * ...

有关更多详细信息，请查看与感兴趣的客户相关的文档。

## 实例

可以在 GitHub 上找到有关如何设置不同外部任务客户端的完整示例 ([Java](https://github.com/camunda/camunda-bpm-examples/tree/{{< minor-version >}}/clients/java),
[JavaScript](https://github.com/camunda/camunda-external-task-client-js/tree/master/examples)).

## 外部任务吞吐量

对于外部任务的高吞吐量，你应该在外部任务实例的数量、客户端的数量和处理工作的持续时间之间取得平衡。

长时间运行任务（可能超过 30 秒）的经验法则是，一个一个地获取并锁定任务（maskTasks = 1）并根据需要调整长轮询间隔（可能为 60 秒，asyncResponseTime = 60000）。
Java 客户端支持指数退避，默认为 500 毫秒，因子为 2，限制为 60000 毫秒。 这也可以满足你的需求。

由于外部任务客户端在内部不使用任何线程，你应该根据需要启动尽可能多的客户端并平衡你的操作系统的负载。
