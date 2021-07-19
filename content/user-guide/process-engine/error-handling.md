---

title: '错误处理'
weight: 280

menu:
  main:
    identifier: "examples-tutorials-error-handling"
    parent: "user-guide-process-engine"

---

# 错误处理策略

有几种基本策略可以处理流程中的错误和异常。决定使用哪种策略取决于：

 *  技术还是业务错误：该错误是否具有某种商业意义并导致替代流程（例如“商品不在库存”）还是技术故障（例如“网络当前关闭”）？
 *  显式错误处理或通用处理：在某些情况下，你希望明确建模发生错误时应该怎么做（通常是业务错误）。 在很多情况下，你不想这样做，但有一些适用于错误的通用机制，简化了你的流程模型（典型的技术错误，想象一下你必须对可能发生的每项任务的网络中断进行建模？ 你将无法再识别你的业务流程）。

在流程引擎的上下文中，错误通常作为必须处理的 Java 异常引发。 让我们来看看如何处理它们。

## 事务回滚

标准的处理策略是向客户端抛出异常，即回滚当前事务。 这意味着进程状态回滚到最后的等待状态。 此行为在 [用户指南]({{< ref "/user-guide/_index.md" >}})的[流程中的事务]({{< ref "/user-guide/process-engine/transactions-in-processes.md" >}})章节中有详细描述。错误处理由引擎委托给客户端。

让我们用一个具体的例子来说明这一点：用户在前端收到一个错误对话框，指出由于网络错误，当前无法访问库存管理软件。 要执行重试，用户可能必须再次单击同一按钮。 即使这通常不是所希望的，它仍然是一种适用于许多情况的简单策略。

## 异步并失败的Job

如果你不希望向用户显示异常，一种选择是流程服务调用，这可能会导致错误、异步（如 [流程中的事务]({{< ref "/user-guide/process-engine/transactions-in-processes.md" >}})) 中所述）。在这种情况下，异常存储在流程引擎数据库和 [Job]({{< ref "/user-guide/process-engine/the-job-executor.md" >}})中，后台被标记为失败（更准确地说，异常被存储并且一些重试计数器递减）。

在上面的例子中，这意味着用户不会看到错误，而是“一切都成功”的对话框。 异常存储在Job中。 现在，要么聪明的重试策略将在稍后（当网络再次可用时）自动重新触发Job，要么操作员需要查看错误并触发额外的重试。 这将在后面更详细地显示。

这种策略非常强大，经常在实际项目中应用，但是，它仍然隐藏了 BPMN 图中的错误，因此对于你希望在流程图中可见的业务错误，最好使用 [错误事件]({{< relref "#bpmn-2-0-错误事件" >}}).

## 捕获异常并使用基于数据的异或网关

如果调用可以抛出异常的 Java 代码，则可以在 Java 委托类、CDI Bean 或任何其他内容中捕获异常。 也许记录一些信息并继续就已经足够了，这意味着你可以忽略错误。更常见的是，你将结果写入流程变量，并在稍后的流程操作中被异或网关使用，以便在发生错误时采用不同的路径。

在这种情况下，你可以在流程模型中明确地对错误处理进行建模，但你使其看起来像正常结果而不是错误。 从商业角度来看，这不是错误而是结果，因此不应轻率做出决定。 一条经验法则是，可以通过这种方式处理结果，而不应以这种方式处理异常错误。 然而，BPMN 的观点并不总是必须与技术实现相匹配。

例子：

{{< img src="../img/error-result-xor.png" title="Error Result XOR" >}}

我们触发了“检查数据完整性”任务。 Java 服务可能会抛出“DataIncompleteException”。 但是，如果我们检查完整性，不完整的数据不是异常，而是预期的结果，因此我们更喜欢在评估流程变量的流程中使用 异或网关，例如，“#{dataComplete==false}” 。

## BPMN 2.0 错误事件

BPMN 2.0 错误事件使你可以明确地对错误进行建模，从而解决业务错误的用例。 最突出的例子是“中间捕获错误事件”，它可以附加到一个活动的边界上。 定义边界错误事件对嵌入式子流程、调用活动或服务任务最有意义。 一个错误将导致替代流程被触发：

{{< img src="../img/bpmn.boundary.error.event.png" title="Error Boundary Event" >}}


参考 [BPMN 2.0 实施参考]({{< ref "/reference/bpmn20/_index.md" >}})中的[错误事件]({{< ref "/reference/bpmn20/events/error-events.md" >}}) 章节 和 [用户指南]({{< ref "/user-guide/_index.md" >}})中的[从委托代码中抛出BPMN Errors]({{< ref "/user-guide/process-engine/delegation-code.md#从委托代码中抛出bpmn-errors" >}}) 章节了解更多信息。

## BPMN 2.0 Compensation and Business Transactions

BPMN 2.0 transactions and compensations allow you to model business transaction boundaries (however, not in a technical ACID manner) and make sure already executed actions are compensated during a rollback. Compensation means to make the effect of the action invisible, e.g. book in goods if you have previously booked out the goods. See the [BPMN Compensation event]({{< ref "/reference/bpmn20/events/cancel-and-compensation-events.md" >}}) and the [BPMN Transaction Subprocess]({{< ref "/reference/bpmn20/subprocesses/transaction-subprocess.md" >}}) sections of the [BPMN 2.0 Implementation Reference]({{< ref "/reference/bpmn20/_index.md" >}}) for details.


# 监控和恢复策略

如果发生错误，可以应用不同的恢复策略。

## 让用户重试

如上所述，最简单的错误处理策略是向客户端抛出异常，这意味着用户必须自己重试该操作。 他如何做到这一点取决于用户，通常是重新加载页面或再次单击。

## 重试失败的Job

如果你使用 Jobs (`async`)，你可以利用 Cockpit 作为监控工具来处理失败的Job，在这种情况下，最终用户不会看到异常。 然后，当重试耗尽时，你通常会在驾驶舱中看到失败（请参阅 [失败的Job]({{< ref "/user-guide/process-engine/the-job-executor.md#failed-jobs" >}}) [Web 应用程序]({{< ref "/webapps/cockpit/_index.md" >}}) 的部分以获取更多信息）。

参考 [Web 应用]({{< ref "/webapps/cockpit/_index.md" >}}) 中的 [Cockpit中失败的Job]({{< ref "/webapps/cockpit/bpmn/failed-jobs.md" >}}) 章节获取更多细节。

如果不想使用 Cockpit，也可以自己通过 API 查找失败的Job：

```java
List<Job> failedJobs = processEngine.getManagementService().createJobQuery().withException().list();
for (Job failedJob : failedJobs) {
  processEngine.getManagementService().setJobRetries(failedJob.getId(), 1);
}
```

## 显式建模

当然，你始终可以按照 [BPMN 2.0 中的重试在哪里](http://www.bpm-guide.de/2012/06/15/where-is-the-retry-in-bpmn-2-0/) 中指出的那样对重试机制进行显式建模：

{{< img src="../img/retry.png" title="Retry Mechanism" >}}

我们建议将其限制在你有充分理由希望在流程图中看到它的情况。 我们更喜欢异步延续，因为它不会使你的流程图膨胀，并且基本上可以用更少的运行时开销来做同样的事情，因为“遍历”建模循环涉及额外的操作，例如，编写审计日志。

## 使用用户任务

我们经常在项目中看到这样的事情：

{{< img src="../img/error-handling-user-task.png" title="User Task Error Handling" >}}

实际上，这是一种有效的方法，你可以将错误分配给操作员作为用户任务，并为他解决问题的选项建模。 然而，这是一种奇怪的混合：我们想要处理一个技术错误，但将其添加到我们的业务流程模型中。 我们在哪里停下来？ 我们现在是否必须在每个服务任务上对其进行建模？

对于这种情况，使用失败的Job列表而不是使用“正常”任务列表感觉是一种更自然的方法，这就是为什么我们通常推荐另一种可能性并且不认为这是最佳实践的原因。
