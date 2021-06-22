---

title: '委托代码'
weight: 60

menu:
  main:
    identifier: "user-guide-process-engine-delegation-code"
    parent: "user-guide-process-engine"

---


委托代码允许你在流程执行期间发生某些事件时执行外部Java代码、脚本或表达式。

有四种不同类型委托代码：

* **Java 代理类（JAVA Delegates）** 可以附加到 [BPMN 服务任务]({{< ref "/reference/bpmn20/tasks/service-task.md" >}})。
* **变量映射（Delegate Variable Mapping）** 可以附加到 [发起活动]({{< ref "/reference/bpmn20/subprocesses/call-activity.md" >}})。
* **执行监听器（Execution Listeners）** 可以附加到令牌流动的任何实践，例如，启动一个流程实例或进入一个活动。
* **任务监听器（Task Listeners）** 可以附加到用户任务生命周期内的任何事件，例如，用户任务的创建或完成。

你可以通过BPMN 2.0 XML创建通用的委托代码，并使用字段注入来配置。

# Java 代理类

在流程运行中调用的Java代理类，需要实现 `org.camunda.bpm.engine.delegate.JavaDelegate` 接口，并实现 `execute` 方法。当流程执行到达这个特定的步骤时，它将执行该方法中定义的这个方法，并以默认的BPMN 2.0方式完成活动。

举个例子，让我们先创建一个Java类，它可以用来将一个流程变量String改为大写字母。这个类需要实现`org.camunda.bpm.engine.delegate.JavaDelegate`接口，这要求我们实现`execute(DelegateExecution)`方法。这个操作将被引擎调用，并且包含所需要的业务逻辑。（可以通过 {{< javadocref page="?org/camunda/bpm/engine/delegate/DelegateExecution.html" text="DelegateExecution" >}} 接口访问和操作流程实例信息，如流程变量和其他信息）

```java
  public class ToUppercase implements JavaDelegate {

    public void execute(DelegateExecution execution) throws Exception {
      String var = (String) execution.getVariable("input");
      var = var.toUpperCase();
      execution.setVariable("input", var);
    }

}
```

{{< note title="新的实例!" class="info" >}}
每次执行引用活动的委托类时，都会创建这个类的一个新的实例。这意味着，每次活动被执行时，将使用该类的另一个实例来调用 `execute(DelegateExecution)`。
{{< /note >}}

在流程定义中引用的类（即通过使用 `camunda:class` 引用的类）在部署期间 **不会** 被实例化。只有当流程执行到达流程中第一次使用类的地方时，才会创建该类的实例。如果找不到该类，将抛出一个 "ProcessEngineException"。因为，你在部署时的环境（更确切地说，是classpath）往往与实际运行时的环境不同。

# 活动行为

不写Java代理类，也可以提供一个实现`org.camunda.bpm.engine.impl.pvm.delegate.ActivityBehavior`接口的类。然后，实现有更大能力的`ActivityExecution`，例如，还可以影响流程的控制流。然而，请注意，这不是一个很好的做法，应该尽量避免。建议只在高级用例中使用 "ActivityBehavior"接口，你需要清楚地知道自己在做什么。

# 字段（属性）注入

可以向委托类的字段中注入数据。支持以下类型的注入

* 固定字符串值
* 表达式

如果可用，该值将通过委托类上的公共setter方法注入，并遵循Java Bean的命名惯例（例如，字段`firstName`有setter`setFirstName(..)`）。如果该字段没有setter，私有成员的值将在委托中被设置（但不建议使用私有字段，见下面的警告）。

**Regardless of the type of value declared in the process-definition, the type of the
setter/private field on the injection target should always be `org.camunda.bpm.engine.delegate.Expression`**.

{{< note title="" class="warning" >}}
  私有字段并不总是能够修改的! 它对CDI Bean（因为你有代理而不是真实的对象）或一些SecurityManager配置不起作用。请始终为你想注入的字段使用一个public的setter方法。
{{< /note >}}

下面的代码片段显示了如何将一个常量值注入到一个字段。当使用`class`或`delegateExpression`属性时，支持字段注入。请注意，在实际的字段注入声明之前，我们需要声明一个`extensionElements`的XML元素，这是BPMN 2.0 XML Schema的一个要求。

```xml
  <serviceTask id="javaService"
               name="Java service invocation"
               camunda:class="org.camunda.bpm.examples.bpmn.servicetask.ToUpperCaseFieldInjected">
    <extensionElements>
        <camunda:field name="text" stringValue="Hello World" />
    </extensionElements>
  </serviceTask>
```

类`ToUpperCaseFieldInjected`有一个字段`text`，其类型为`org.camunda.bpm.engine.delegate.Expression`。当调用`text.getValue(execution)`时，将返回配置的字符串值`Hello World`。

另外，对于长文本（例如，内嵌的电子邮件），可以使用`camunda:string`子元素。

```xml
  <serviceTask id="javaService"
               name="Java service invocation"
               camunda:class="org.camunda.bpm.examples.bpmn.servicetask.ToUpperCaseFieldInjected">
    <extensionElements>
      <camunda:field name="text">
          <camunda:string>
            Hello World
        </camunda:string>
      </camunda:field>
    </extensionElements>
  </serviceTask>
```

为了注入在运行时动态解析的值，也可以使用表达式。这些表达式可以使用流程变量、CDI或Spring Bean。如前所述，每次执行服务任务时，都会创建一个单独的Java类实例。为了在字段中动态注入值，你可以在`org.camunda.bpm.engine.delegate.Expression`中注入值和方法表达式，该表达式可以使用`execute`方法中传递的`DelegateExecution`进行计算/调用。

```xml
  <serviceTask id="javaService" name="Java service invocation"
               camunda:class="org.camunda.bpm.examples.bpmn.servicetask.ReverseStringsFieldInjected">

    <extensionElements>
      <camunda:field name="text1">
        <camunda:expression>${genderBean.getGenderString(gender)}</camunda:expression>
      </camunda:field>
      <camunda:field name="text2">
         <camunda:expression>Hello ${gender == 'male' ? 'Mr.' : 'Mrs.'} ${name}</camunda:expression>
      </camunda:field>
    </extensionElements>
  </serviceTask>
```

下面的例子中使用注入的表达式，并且使用当前的 "DelegateExecution" 来解决它们：

```java
  public class ReverseStringsFieldInjected implements JavaDelegate {

    private Expression text1;
    private Expression text2;

    public void execute(DelegateExecution execution) {
      String value1 = (String) text1.getValue(execution);
      execution.setVariable("var1", new StringBuffer(value1).reverse().toString());

      String value2 = (String) text2.getValue(execution);
      execution.setVariable("var2", new StringBuffer(value2).reverse().toString());
    }
  }
```

另外，你也可以把表达式设置为一个属性，而不是一个子元素，以使XML不那么冗长。

```xml
  <camunda:field name="text1" expression="${genderBean.getGenderString(gender)}" />
  <camunda:field name="text2" expression="Hello ${gender == 'male' ? 'Mr.' : 'Mrs.'} ${name}" />
```

{{< note title="笔记!" class="info" >}}
  由于该类的一个新的实例将被创建，所以每次服务任务被调用时都会发生注入。当字段被你的代码改变后，这些值将在下次执行该活动时被重新注入覆盖掉。
{{< /note >}}

{{< note title="" class="warning" >}}
  出于上述同样的原因，字段注入不应该（通常）用于Spring Bean，因为Spring Bean默认是单例的。所以，你可能会因为同时修改Bean的字段而遇到不一致的情况。
{{< /note >}}

# 变量映射

要实现一个为输入和输出变量映射的类，这个类需要实现`org.camunda.bpm.engine.delegate.DelegateVariableMapping`接口。该实现必须提供`mapInputVariables(DelegateExecution, VariableMap)`和`mapOutputVariables(DelegateExecution, VariableScope)`方法。请看下面的例子：

```java
public class DelegatedVarMapping implements DelegateVariableMapping {

  @Override
  public void mapInputVariables(DelegateExecution execution, VariableMap variables) {
    variables.putValue("inputVar", "inValue");
  }

  @Override
  public void mapOutputVariables(DelegateExecution execution, VariableScope subInstance) {
    execution.setVariable("outputVar", "outValue");
  }
}
```

`mapInputVariables` 方法在调用活动执行之前被调用，以映射输入变量。
输入变量应该被放入给定的变量map中。
`mapOutputVariables` 方法在调用活动被执行后被调用，以映射输出变量。
输出变量应该直接设置到调用者的执行中。
类加载的方法类似于[Java 代理类]({{< ref "/user-guide/process-engine/delegation-code.md#java-delegate" >}})。


# 执行监听器

执行监听器允许你在流程执行过程中发生某些事件时执行外部Java代码或计算一个表达式。可以捕获的事件有：

* 启动或结束一个流程实例。
* 进行一个过渡（Transition）。
* 启动或结束一个活动。
* 启动或结束一个网关。
* 启动或结束一个中间事件。
* 结束一个 start event 或启动一个 end event.

下面的流程定义中包含3个执行监听器：

```xml
  <process id="executionListenersProcess">
    <extensionElements>
      <camunda:executionListener
          event="start"
          class="org.camunda.bpm.examples.bpmn.executionlistener.ExampleExecutionListenerOne" />
    </extensionElements>

    <startEvent id="theStart" />

    <sequenceFlow sourceRef="theStart" targetRef="firstTask" />

    <userTask id="firstTask" />

    <sequenceFlow sourceRef="firstTask" targetRef="secondTask">
      <extensionElements>
        <camunda:executionListener>
          <camunda:script scriptFormat="groovy">
            println execution.eventName
          </camunda:script>
        </camunda:executionListener>
      </extensionElements>
    </sequenceFlow>

    <userTask id="secondTask">
      <extensionElements>
        <camunda:executionListener expression="${myPojo.myMethod(execution.eventName)}" event="end" />
      </extensionElements>
    </userTask>

    <sequenceFlow sourceRef="secondTask" targetRef="thirdTask" />

    <userTask id="thirdTask" />

    <sequenceFlow sourceRef="thirdTask" targetRef="theEnd" />

    <endEvent id="theEnd" />
  </process>
```

第一个执行监听器在流程开始时被通知。该监听器是一个外部Java类（如ExampleExecutionListenerOne），并且应该实现了`org.camunda.bpm.engine.delegate.ExecutionListener`接口。当事件发生时（本例为结束事件），`notify(DelegateExecution execution)`方法被调用。

```java
  public class ExampleExecutionListenerOne implements ExecutionListener {

    public void notify(DelegateExecution execution) throws Exception {
      execution.setVariable("variableSetInExecutionListener", "firstValue");
      execution.setVariable("eventReceived", execution.getEventName());
    }
  }
```

也可以使用实现了`org.camunda.bpm.engine.delegate.JavaDelegate`接口的委托类。然后，这些委托类可以在其他结构中重复使用，比如服务任务。

第二个执行监听器在执行过渡时被调用。请注意，监听器元素并没有定义事件，因为只有事件是在执行过渡时被触发。当监听器被定义在一个过渡上时，事件属性中的值会被忽略。它还包含一个 [camunda:script][camunda-script] 子元素，它定义了一个将被执行监听器执行的脚本。另外，也可以将脚本源代码指定为外部资源（见关于脚本任务的 [script sources][script-sources] 的文档）。

最后一个执行监听器在活动secondTask结束时被调用。与其在监听器声明中使用类，不如定义一个表达式，当事件被触发时被计算或调用

```xml
  <camunda:executionListener expression="${myPojo.myMethod(execution.eventName)}" event="end" />
```

与其他表达式一样，流程变量可以被使用。因为execution有一个暴露事件名称的属性，所以可以使用execution.eventName将事件名称传递给你的方法。

执行监听器也支持使用delegateExpression，类似于服务任务。

```xml
  <camunda:executionListener event="start" delegateExpression="${myExecutionListenerBean}" />
```


# 任务监听器

任务监听器被用来在某个任务相关事件发生时执行自定义的Java逻辑或表达式。它只能作为用户任务的一个子元素添加到流程定义中。请注意，这也必须作为BPMN 2.0 extensionElements的一个子元素，并在Camunda命名空间中定义，因为任务监听器是Camunda引擎专用的。

```xml
  <userTask id="myTask" name="My Task" >
    <extensionElements>
      <camunda:taskListener event="create" class="org.camunda.bpm.MyTaskCreateListener" />
    </extensionElements>
  </userTask>
```

## 任务监听器事件生命周期

任务监听器的执行取决于以下任务相关事件的启动顺序。

当任务被创建并且所有的任务属性被设置时， **create** 事件被触发。在 **create** 事件之前，没有其他与任务相关的事件会被触发。该事件允许我们在监听器中收到创建任务时检查其所有属性。

当一个已经创建的任务上的任务属性（如受让人、所有者、优先级等）被改变时，**update** 事件就会发生。不但包括这包括任务的属性（如受让人、所有者、优先级等）发生改变，还包括附属实体（如附件、评论、任务本地变量）的变化。请注意，任务的初始化并不触发更新事件（任务正在被创建）。这也意味着 *update* 事件总是在 *create* 事件已经发生之后发生。
 
**assignment** 事件专门跟踪任务的 `受让人（assignee）` 属性的变化。该事件可能在两种情况下被触发。

1. 当一个在流程定义中明确定义了 "受让人" 的任务被创建。在这种情况下， *assignment* 事件将在 *create* 事件之后被触发。
1. 当一个已经创建的任务被分配，即任务的 "受让人" 属性被改变。因为改变 `assignee` 属性会导致一个 *update* 的事件，在这种情况下，*assignment* 事件将在 *update* 事件之后发生。

当受让人被设置， *assignment* 事件可用于更精细的检查。

当与该任务监听器相关的定时器到期后，**timeout** 事件就会发生。注意，这需要定义一个定时器。"timeout" 事件可能发生在一个任务被 "create" 之后和 "complete" 之前。

当任务成功完成时，在任务从运行时数据库中删除之前，会发生 **complete** 事件。一个任务的 **complete** 任务监听器的成功执行后任务事件生命周期才会结束。
 
在任务从运行时数据库中被删除之前，只有在如下情况下， **delete** 事件会发生：

1. 一个中断的边界事件。
1. 一个中断的事件子流程。
1. 一个流程实例的删除。
1. 任务监听器内抛出的BPMN错误。
 
在 *delete* 事件之后，不会有其他事件被触发，因为它将导致任务事件生命周期的结束。这意味着， *delete* 事件与 *complete* 事件互斥的。

### 任务事件链

上面的描述阐述了任务事件被触发的一般顺序。然而，在以下情况下，不会遵循一般顺序：

1. 在一个任务监听器内调用 `Task#complete()` 时，**complete** 事件将被立即触发。**complete** 监听器将被调用，之后再处理其余任务监听器。
1. 通过在任务监听器内使用 `TaskService` 方法，这可能会引起额外的任务事件的触发。与上面提到的 **complete** 事件一样，这些任务事件将立即调用其相关的监听器，之后再处理剩余的任务监听器。然而，应该注意的是，通过调用 `TaskService` 方法，在任务监听器内部触发的事件链，将按照之前描述的顺序执行。
1. 通过在任务监听器内部抛出一个BPMN错误事件（例如，一个 **complete** 事件的任务监听器）。这将取消任务，并导致一个 **delete** 事件被触发。

在上述条件下，需要注意不要创建任务事件循环。

## 定义一个任务监听器

任务监听器支持以下属性：

* **event (必要)**: 任务监听器将监听哪种任务事件：
    可能的事件为: **create**, **assignment**, **update**, **complete**, **delete** 和
     **timeout**;

    注意，**timeout** 事件需要任务监听器中的[timerEventDefinition][timerEventDefinition]子元素，并且只有在[Job Executor][job-executor]被启用时才会触发。

* **class**: 被调用的委托类。这个类必须实现`org.camunda.bpm.engine.impl.pvm.delegate.TaskListener`接口。

    ```java
    public class MyTaskCreateListener implements TaskListener {

      public void notify(DelegateTask delegateTask) {
        // Custom logic goes here
      }

    }
    ```

    也可以使用字段注入来传递流程变量或执行到委托类。请注意，每次执行引用活动的委托类时，将创建这个类的新的实例。

* **expression**（不能与class属性一起使用）：指定事件发生时将被执行的表达式。可以将DelegateTask对象和事件的名称（使用 task.eventName ）作为参数传递给被调用对象.

    ```xml
    <camunda:taskListener event="create" expression="${myObject.callMethod(task, task.eventName)}" />
    ```

* **delegateExpression**: 允许指定一个表达式，该表达式可解析为实现TaskListener接口的对象，类似于服务任务。

    ```xml
    <camunda:taskListener event="create" delegateExpression="${myTaskListenerBean}" />
    ```

* **id**: 用户任务范围内监听器的唯一标识符，只有当`event`被设置为`timeout`时才需要。


除了 `class`, `expression` 和 `delegateExpression` 属性, [camunda:script][camunda-script] 子元素可以用来指定一个脚本作为任务监听器。外部脚本资源也可以用`camunda:script` 元素的资源属性来声明（参见关于脚本任务的[script sources][script-sources]的文档）。

```xml
  <userTask id="task">
    <extensionElements>
      <camunda:taskListener event="create">
        <camunda:script scriptFormat="groovy">
          println task.eventName
        </camunda:script>
      </camunda:taskListener>
    </extensionElements>
  </userTask>
```

此外, [timerEventDefinition][timerEventDefinition] 子元素可以和`event`类型`timeout`一起使用，以便定义相关的计时器。当定时器到期时，指定的委托将被[Job 执行器][job-executor]调用。用户任务的执行将 **不会** 被打断。

```xml
  <userTask id="task">
    <extensionElements>
      <camunda:taskListener event="timeout" delegateExpression="${myTaskListenerBean}" id="friendly-reminder" >
        <timerEventDefinition>
          <timeDuration xsi:type="tFormalExpression">PT1H</timeDuration>
        </timerEventDefinition>
      </camunda:taskListener>
    </extensionElements>
  </userTask>
```

# 监听器的字段注入

当使用类属性配置的监听器时，可以使用字段注入。这与 Java代理类 描述的机制完全相同，包含字段注入的所有可能性。

下面的片段显示了一个简单的例子流程，其中有一个执行监听器，并注入了字段：

```xml
  <process id="executionListenersProcess">
    <extensionElements>
      <camunda:executionListener class="org.camunda.bpm.examples.bpmn.executionListener.ExampleFieldInjectedExecutionListener" event="start">
        <camunda:field name="fixedValue" stringValue="Yes, I am " />
        <camunda:field name="dynamicValue" expression="${myVar}" />
      </camunda:executionListener>
    </extensionElements>

    <startEvent id="theStart" />
    <sequenceFlow sourceRef="theStart" targetRef="firstTask" />

    <userTask id="firstTask" />
    <sequenceFlow sourceRef="firstTask" targetRef="theEnd" />

    <endEvent id="theEnd" />
  </process>
```

监听器实现可能如下所示：

```java
  public class ExampleFieldInjectedExecutionListener implements ExecutionListener {

    private Expression fixedValue;

    private Expression dynamicValue;

    public void notify(DelegateExecution execution) throws Exception {
      String value =
        fixedValue.getValue(execution).toString() +
        dynamicValue.getValue(execution).toString();

      execution.setVariable("var", value);
    }
  }
```

类`ExampleFieldInjectedExecutionListener`将注入2个字段（一个是固定的，另一个是动态的），并将其存储在流程变量_var_中。

```java
  @Deployment(resources = {
    "org/camunda/bpm/examples/bpmn/executionListener/ExecutionListenersFieldInjectionProcess.bpmn20.xml"
  })
  public void testExecutionListenerFieldInjection() {
    Map<String, Object> variables = new HashMap<String, Object>();
    variables.put("myVar", "listening!");

    ProcessInstance processInstance = runtimeService.startProcessInstanceByKey("executionListenersProcess", variables);

    Object varSetByListener = runtimeService.getVariable(processInstance.getId(), "var");
    assertNotNull(varSetByListener);
    assertTrue(varSetByListener instanceof String);

    // Result is a concatenation of fixed injected field and injected expression
    assertEquals("Yes, I am listening!", varSetByListener);
  }
```


# 访问流程引擎服务

可以从委托代码中访问公共API Service（`RuntimeService`、`TaskService`、`RepositoryService` ...）。下面是一个例子，显示 如何从一个 "JavaDelegate" 实现中访问 "TaskService"：

```java
  public class DelegateExample implements JavaDelegate {

    public void execute(DelegateExecution execution) throws Exception {
      TaskService taskService = execution.getProcessEngineServices().taskService();
      taskService.createTaskQuery()...;
    }

  }
```


# 从委托代码中抛出BPMN Errors

可以从委托代码（Java委托、执行和任务监听器）抛出`BpmnError`：

```java
public class BookOutGoodsDelegate implements JavaDelegate {

  public void execute(DelegateExecution execution) throws Exception {
    try {
        ...
    } catch (NotOnStockException ex) {
        throw new BpmnError(NOT_ON_STOCK_ERROR);
    }
  }

}
```


## 从监听器中抛出BPMN错误

当实现一个错误捕获事件时，请记住，在以下监听器的正常流程中被抛出时，`BpmnError`将被捕获。

* 活动、网关和中间事件的开始和结束执行监听器
* 在转换过程中采取执行监听器
* 创建、分配和完成任务监听器

`BpmnError` 将不会被以下监听器捕获。

* 开始或结束流程监听器
* 删除任务监听器
* 在正常流程之外调用的监听器。
    * 执行流程修改，触发子流程范围初始化，其一些监听器抛出了一个错误
    * 一个流程实例的删除调用了一个结束监听器，引发了一个错误
    * 由于中断边界事件的执行而触发了一个监听器，例如子流程上的消息关联调用了终端监听器，从而引发了一个错误。


{{< note title="笔记!" class="info" >}}

在委托代码中抛出一个`BpmnError`，结果与错误结束事件类似。关于行为的细节，特别是错误边界事件，请参见[参考指南]({{< ref "/reference/bpmn20/events/error-events.md#error-boundary-event" >}})。如果在作用域上没有发现错误边界事件，执行就会结束。

{{< /note >}}


# 从委托代码中设置Business Key

为已经运行的流程实例设置Business Key，参考下面的例子：

```java
public class BookOutGoodsDelegate implements JavaDelegate {

  public void execute(DelegateExecution execution) throws Exception {
    ...
    String recalculatedKey = (String) execution.getVariable("recalculatedKeyVariable");
    execution.setProcessBusinessKey(recalculatedKey);
    ...
  }

}
```

[script-sources]: {{< ref "/user-guide/process-engine/scripting.md#script-source" >}}
[camunda-script]: {{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#camunda-script" >}}
[timerEventDefinition]: {{< ref "/reference/bpmn20/events/timer-events.md#defining-a-timer" >}}
[job-executor]: {{< ref "/user-guide/process-engine/the-job-executor.md" >}}
