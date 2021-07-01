---

title: 'Spring 事件总线'
weight: 60

menu:
  main:
    identifier: "user-guide-spring-event-bridge"
    parent: "user-guide-spring-boot-integration"

---


流程引擎可以连接到Spring事件总线。我们称其为 "Spring事件桥(Eventing Bridge)"。这使得我们可以使用标准的Spring事件机制来通知流程事件。默认情况下，Spring事件由一个引擎插件启用。该事件由三个`camunda.bpm.eventing`属性控制。它们是：

```
camunda.bpm.eventing.execution=true
camunda.bpm.eventing.history=true
camunda.bpm.eventing.task=true
```
这些属性分别控制执行、历史事件和任务的三个事件流。
监听器可以订阅可变或不可变的事件对象的流。
不可变监听器在使用异步监听器时特别有用--例如使用`TransactionalEventListener`时。
可变事件流对象在创建和接收监听器异步订阅的事件之间可以被多次修改。不可变的事件对象反映了事件创建时的数据，不管它们最终被监听器接收的时间。

在执行事件流中，可以接收 `DelegateExecution`（可变）和 `ExecutionEvent`（不可变）。
任务事件流提供`DelegateTask`（可变）和`TaskEvent`（不可变）。
在历史事件流只提供 `HistoryEvent`（可变）。

下面的例子概述了如何在Spring Bean中接收流程事件。这样，你可以通过为Spring Bean提供注释方法来实现任务和委托监听器，而不是实现`TaskListener`和`ExecutionListener`接口。

```java
import org.camunda.bpm.engine.delegate.DelegateExecution;
import org.camunda.bpm.engine.delegate.DelegateTask;
import org.camunda.bpm.engine.impl.history.event.HistoryEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;


@Component
class MyListener {

  @EventListener
  public void onTaskEvent(DelegateTask taskDelegate) {
    // 处理可变的任务事件
  }

  @EventListener
  public void onTaskEvent(TaskEvent taskEvent) {
    // 处理不可变的任务事件
  }

  @EventListener
  public void onExecutionEvent(DelegateExecution executionDelegate) {
    // 处理可变的执行事件
  }

  @EventListener
  public void onExecutionEvent(ExecutionEvent executionEvent) {
    // 处理不可改的执行事件
  }
  
  @EventListener
  public void onHistoryEvent(HistoryEvent historyEvent) {
    // 处理历史事件
  }

}
```

{{< note title="" class="info" >}}
  如果用`EventListener`注释的方法返回一个非`void`的结果，Spring会将把它作为一个新的事件扔到Spring事件总线上。这允许建立事件处理程序链进行处理。关于事件处理的更多信息，请参考Spring手册。
{{< /note >}}

# 指定事件类型

Spring允许通过在 `@EventListener` 注解中提供SpEL条件来指定传递给监听器的事件。例如，你可以为一个任务事件注册一个监听器，该事件是通过创建具有特定任务定义key的用户任务而触发的。以下是代码示例。

```java
@Component
class MyTaskListener {

  @EventListener(condition="#taskDelegate.eventName=='create' && #taskDelegate.taskDefinitionKey=='task_confirm'")
  public void onTaskEvent(DelegateTask taskDelegate) {
  // 处理被task_confirm任务触发的任务事件
  }
}
```

# 订阅事件侦听器

同一事件类型的事件监听器可以按指定顺序执行。要做到这一点，请为事件监听器提供一个`Order`注解。

```java
@Component
class MyTaskListener {

  @Order(1)
  @EventListener
  public void firstListener(DelegateTask taskDelegate) {
  // 处理任务事件
  }

  @Order(2)
  @EventListener
  public void secondListener(DelegateTask taskDelegate) {
  // 处理任务事件
  }

}
```
