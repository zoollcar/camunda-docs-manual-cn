---

title: '表达式语言'
weight: 70

menu:
  main:
    identifier: "user-guide-process-engine-expression-language"
    parent: "user-guide-process-engine"

---

Camunda平台支持统一表达式语言（EL），它是JSP 2.1标准的一部分（[JSR-245][]）。使用了开源的[JUEL][]实现。要获得更多关于表达式语言使用的一般信息，请阅读[官方文档][official documentation]。

特别是提供了很好的[实例][examples]，可以方便的学习表达式语法。

在Camunda平台内，EL可以在许多情况下用于评估小型的脚本式表达。下面的例子是一个使用了EL的BPMN元素：

<table class="table desc-table">
  <tr>
    <th>BPMN element</th>
    <th>EL support</th>
  </tr>
  <tr>
    <td>
      <a href="#delegation-code">
        Service Task, Business Rule Task, Send Task,
        Message Intermediate Throwing Event, Message End Event, Execution Listener and
        Task Listener
      </a>
    </td>
    <td>Expression language as delegation code</td>
  </tr>
  <tr>
    <td>
      <a href="#conditions">
        Sequence Flows, Conditional Events
      </a>
    </td>
    <td>Expression language as condition expression</td>
  </tr>
  <tr>
    <td>
        <a href="#inputoutput-parameters">
          All Tasks, All Events, Transaction, Subprocess and Connector
        </a>
    </td>
    <td>Expression language inside an inputOutput parameter mapping</td>
  </tr>
  <tr>
    <td>
        <a href="#value">
          Different Elements
        </a>
    </td>
    <td>Expression language as the value of an attribute or element</td>
  </tr>
  <tr>
    <td>
      <a href="{{< ref "/user-guide/process-engine/the-job-executor.md#specifying-priorities-in-bpmn-xml" >}}">
        All Flow Nodes, Process Definition
      </a>
    </td>
    <td>Expression language to determine the priority of a job</td>
  </tr>
</table>


# 表达式语言的用法

## 委托代码

除了Java代码，Camunda平台还支持将表达式作为委托代码进行计算。关于委托代码的信息，请参见相应的[章节]({{< ref "/user-guide/process-engine/delegation-code.md" >}}).

目前表达式支持两种类型：`camunda:expression`和`camunda:delegateExpression`。

通过`camunda:expression`，可以计算一个值表达式或调用一个方法表达式。你可以使用在表达式或Spring和CDI Bean中可用的特殊变量。关于[变量][variables]、[Spring][Spring]，以及[CDI][]bean的更多信息，请参见相应章节。

```xml
  <process id="process">
    <extensionElements>
      <!-- execution listener which uses an expression to set a process variable -->
      <camunda:executionListener event="start" expression="${execution.setVariable('test', 'foo')}" />
    </extensionElements>

    <!-- ... -->

    <userTask id="userTask">
      <extensionElements>
        <!-- task listener which calls a method of a bean with current task as parameter -->
        <camunda:taskListener event="complete" expression="${myBean.taskDone(task)}" />
      </extensionElements>
    </userTask>

    <!-- ... -->

    <!-- service task which evaluates an expression and saves it in a result variable -->
    <serviceTask id="serviceTask"
        camunda:expression="${myBean.ready}" camunda:resultVariable="myVar" />

    <!-- ... -->

  </process>
```

属性`camunda:delegateExpression`用于表达式，该表达式可以调用一个委托对象。这个委托对象必须实现了`JavaDelegate`或`ActivityBehavior`接口。

```xml
  <!-- 服务任务调用一个实现JavaDelegate接口的Bean -->
  <serviceTask id="task1" camunda:delegateExpression="${myBean}" />

  <!-- 服务任务调用一个返回委托对象的方法 -->
  <serviceTask id="task2" camunda:delegateExpression="${myBean.createDelegate()}" />
```


## 作为条件

为了使用条件序列流（译者注：就是带条件的分支线，一般用在网关后面）或条件事件，通常使用表达式语言。对于条件序列流，必须使用序列流的 `conditionExpression` 元素。对于条件性事件，必须使用条件性事件的 `condition` 元素。两者都是 `tFormalExpression` 的类型。该元素的文本内容是要被计算的表达式。

在表达式中，一些特殊的变量是可用的，这些变量可以访问当前的上下文。要找到关于可用变量的更多信息，请参见[变量章节][variables]。

关于表达式语言作为序列流条件的用法，请看下面的例子：

```xml
  <sequenceFlow>
    <conditionExpression xsi:type="tFormalExpression">
      ${test == 'foo'}
    </conditionExpression>
  </sequenceFlow>
```

关于表达式语言在条件事件上的用法，请看下面的例子：

```xml
<conditionalEventDefinition>
  <condition type="tFormalExpression">${var1 == 1}</condition>
</conditionalEventDefinition>
```


## 输入输出参数

通过Camunda `inputOutput` 扩展，你可以用表达式语言映射 `inputParameter` 或 `outputParameter` 。

在表达式中，一些特殊的变量是可用的，这些变量可以访问当前的上下文。要找到关于可用变量的更多信息，请参见[变量章节][variables]。

下面的例子显示了一个输入参数 "inputParameter"，它使用表达式语言来调用一个bean的方法。

```xml
  <serviceTask id="task" camunda:class="org.camunda.bpm.example.SumDelegate">
    <extensionElements>
      <camunda:inputOutput>
        <camunda:inputParameter name="x">
          ${myBean.calculateX()}
        </camunda:inputParameter>
      </camunda:inputOutput>
    </extensionElements>
  </serviceTask>
```

## 外部任务错误处理

外部任务可以定义 [camunda:errorEventDefinition]({{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#erroreventdefinition" >}})元素，可以用JUEL表达式提供。该表达式在`ExternalTaskService#complete`和`ExternalTaskService#handleFailure`时被计算。如果表达式结果为 "true"，就会抛出一个BPMN错误，这个错误可以被[错误边界事件]({{< ref "/reference/bpmn20/events/error-events.md#error-boundary-event" >}})捕获。

在外部任务的范围内，表达式可以通过key `externalTask`对象访问{{< javadocref page="?org/camunda/bpm/engine/externaltask/ExternalTask.html" text="ExternalTaskEntity" >}}，它为 "errorMessage"、"errorDetails"、"workerId"、"retries" 等提供getter方法。

**案例:**

下面的例子展示了如何访问外部任务对象：

```xml
<bpmn:serviceTask id="myExternalTaskId" name="myExternalTask" camunda:type="external" camunda:topic="myTopic">
  <bpmn:extensionElements>
    <camunda:errorEventDefinition id="myErrorEventDefinition" errorRef="myError" expression="${externalTask.getWorkerId() == 'myWorkerId'}" />
  </bpmn:extensionElements>
</bpmn:serviceTask>
```

如何匹配一个错误信息：

```xml
<bpmn:serviceTask id="myExternalTaskId" name="myExternalTask" camunda:type="external" camunda:topic="myTopic">
  <bpmn:extensionElements>
    <camunda:errorEventDefinition id="myErrorEventDefinition" errorRef="myError" expression="${externalTask.getErrorDetails().contains('myErrorMessage')}" />
  </bpmn:extensionElements>
</bpmn:serviceTask>
```

关于外部任务背景下错误事件定义功能的进一步细节，请查阅[外部任务指南]({{< ref "/user-guide/process-engine/external-tasks.md#error-event-definitions" >}}).

## Value

很多的BPMN和CMMN元素允许通过表达式指定其内容或属性值。请参阅参考文献中[BPMN][]和[CMMN][]的相应章节，以了解更详细的例子。

# 表达式语言内变量与函数

## 流程变量

当前范围内的所有过程变量在表达式中都可以直接使用。所以条件序列流可以直接检查一个变量的值：

```xml
  <sequenceFlow>
    <conditionExpression xsi:type="tFormalExpression">
      ${test == 'start'}
    </conditionExpression>
  </sequenceFlow>
```

## 内置环境变量

根据当前的执行环境，在计算表达式时，可以使用特殊的内置环境变量：

<table class="table">
  <thead>
    <tr>
      <th>Variable</th>
      <th>Java Type</th>
      <th>Context</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>execution</code></td>
      <td><code>{{< javadocref page="?org/camunda/bpm/engine/delegate/DelegateExecution.html" text="DelegateExecution" >}}</code></td>
      <td>
        Available in a BPMN execution context like a service task, execution listener or sequence
        flow.
      </td>
    </tr>
    <tr>
      <td><code>task</code></td>
      <td><code>{{< javadocref page="?org/camunda/bpm/engine/delegate/DelegateTask.html" text="DelegateTask" >}}</code></td>
      <td>Available in a task context like a task listener.</td>
    </tr>
    <tr>
      <td><code>externalTask</code></td>
      <td><code>{{< javadocref page="?org/camunda/bpm/engine/externaltask/ExternalTask.html" text="ExternalTask" >}}</code></td>
      <td>Available during an external task context activity (e.g. in <a href="{{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#erroreventdefinition" >}}">camunda:errorEventDefinition</a> expressions).</td>
    </tr>
    <tr>
      <td><code>caseExecution</code></td>
      <td><code>{{< javadocref page="?org/camunda/bpm/engine/delegate/DelegateCaseExecution.html" text="DelegateCaseExecution" >}}</code></td>
      <td>Available in a CMMN execution context.</td>
    </tr>
    <tr>
      <td><code>authenticatedUserId</code></td>
      <td><code>String</code></td>
      <td>
        The id of the currently authenticated user. Only returns a value if the id of the currently
        authenticated user has been set through the corresponding methods of the
        <code>IdentityService</code>. Otherwise it returns <code>null</code>.
      </td>
    </tr>
  </tbody>
</table>

以下示例展示了，将变量`test`设置为执行侦听器的当前事件名称的表达式；

```xml
  <camunda:executionListener event="start"
    expression="${execution.setVariable('test', execution.eventName)}" />
```


## 从 Spring 和 CDI 的外部上下文中获取外部变量

如果流程引擎与 Spring 或 CDI 集成，则可以在表达式中访问 Spring 和 CDI Bean。请参阅 [Spring][] 和 [CDI][] 的相应章节以了解更多信息。下面的例子显示了一个实现了 "JavaDelegate "接口的Bean作为委托表达式的用法。

```xml
  <serviceTask id="task1" camunda:delegateExpression="${myBean}" />
```

With the expression attribute any method of a bean can be called.

```xml
  <serviceTask id="task2" camunda:expression="${myBean.myMethod(execution)}" />
```


## 内置上下文

在对表达式进行评估时，可以使用以下内置上下文函数：

<table class="table">
  <thead>
    <tr>
      <th>Function</th>
      <th>Return Type</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>currentUser()</code></td>
      <td><code>String</code></td>
      <td>
        Returns the user id of the currently authenticated user or <code>null</code> no user is authenticated at the moment.
      </td>
    </tr>
    <tr>
      <td><code>currentUserGroups()</code></td>
      <td><code>List of Strings</code></td>
      <td>
        Returns a list of the group ids of the currently authenticated user or <code>null</code> if no user is authorized at the moment.
      </td>
    </tr>
    <tr>
      <td><code>now()</code></td>
      <td><code>Date</code></td>
      <td>Returns the current date as a Java Date object.</td>
    </tr>
    <tr>
      <td><code>dateTime()</code></td>
      <td><code>DateTime</code></td>
      <td>
        Returns a Joda-Time DateTime object of the current date. Please see the
        <a href="http://joda-time.sourceforge.net/api-release/org/joda/time/DateTime.html">Joda-Time</a>
        documentation for all available functions.
      </td>
    </tr>
  </tbody>
</table>

下面的例子将一个用户任务的到期日设置为任务创建3天后：

```xml
<userTask id="theTask" name="Important task" camunda:dueDate="${dateTime().plusDays(3).toDate()}"/>
```


## Camunda Spin 内置的方法

如果激活了Camunda Spin流程引擎插件，Spin函数`S`、`XML`和`JSON`在表达式中也可用。参见[数据格式部分][spin-section]了解详情。

```xml
  <serviceTask id="task" camunda:expression="${XML(xml).attr('test').value()}" resultVariable="test" />
```


[JSR-245]: https://jcp.org/aboutJava/communityprocess/final/jsr245/index.html
[JUEL]: http://juel.sourceforge.net/
[official documentation]: http://docs.oracle.com/javaee/5/tutorial/doc/bnahq.html
[examples]: http://docs.oracle.com/javaee/5/tutorial/doc/bnahq.html#bnain
[variables]: {{< relref "#availability-of-variables-and-functions-inside-expression-language" >}}
[Spring]: {{< ref "/user-guide/spring-framework-integration/_index.md#expression-resolving" >}}
[CDI]: {{< ref "/user-guide/cdi-java-ee-integration/expression-resolving.md" >}}
[BPMN]: {{< ref "/reference/bpmn20/_index.md" >}}
[CMMN]: {{< ref "/reference/cmmn11/_index.md" >}}
[spin-section]: {{< ref "/user-guide/data-formats/_index.md" >}}
