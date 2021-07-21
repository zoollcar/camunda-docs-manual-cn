---

title: '脚本'
weight: 80

menu:
  main:
    identifier: "user-guide-process-engine-scripting"
    parent: "user-guide-process-engine"

---


Camunda平台支持与JSR-223兼容的脚本语言。目前，我们对Groovy、JavaScript、JRuby和Jython进行了测试。为了使用一个脚本语言，有必要在classpath中添加相应的jar。

{{< note title="" class="info" >}}
  我们在预编译的 Camunda发行版 中包含了 **GraalVM JavaScript**。
  有关更多信息，请参阅 [JavaScript 考虑事项](#javascript-考虑事项)。

  **Groovy** 也包含在预编译的Camunda发行版中。
{{< /note >}}

下表展示了支持执行脚本的BPMN元素：

<table class="table desc-table">
  <tr>
    <th>BPMN element</th>
    <th>Script support</th>
  </tr>
  <tr>
    <td><a href="#use-script-tasks">Script Task</a></td>
    <td>嵌入在脚本任务中的脚本</td>
  </tr>
  <tr>
    <td>
      <a href="#use-scripts-as-execution-listeners">
        Processes, Activities, Sequence Flows, Gateways and Events
      </a>
    </td>
    <td>作为执行监听器的脚本</td>
  </tr>
  <tr>
    <td>
      <a href="#use-scripts-as-task-listeners">
        User Tasks
      </a>
    </td>
    <td>作为任务监听器的脚本</td>
  </tr>
  <tr>
    <td>
      <a href="#use-scripts-as-conditions">
        Sequence Flows
      </a>
    </td>
    <td>作为顺序流条件表达式的脚本</td>
  </tr>
  <tr>
    <td>
        <a href="#use-scripts-as-inputoutput-parameters">
          All Tasks, All Events, Transactions, Subprocesses and Connectors
        </a>
    </td>
    <td>输入输出参数映射中的脚本</td>
  </tr>
</table>


# 使用脚本任务

通过BPMN 2.0脚本任务，你可以向你的BPM流程添加一个脚本（更多信息请参见[BPMN 2.0参考]({{< ref "/reference/bpmn20/tasks/script-task.md" >}})）。

下面的流程是一个简单的例子，有一个Groovy脚本任务，对一个数组的元素进行求和：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
                   targetNamespace="http://camunda.org/example">
  <process id="process" isExecutable="true">
    <startEvent id="start"/>
    <sequenceFlow id="sequenceFlow1" sourceRef="start" targetRef="task"/>
    <scriptTask id="task" name="Groovy Script" scriptFormat="groovy">
      <script>
        <![CDATA[
        sum = 0

        for ( i in inputArray ) {
          sum += i
        }

        println "Sum: " + sum
        ]]>
      </script>
    </scriptTask>
    <sequenceFlow id="sequenceFlow2" sourceRef="task" targetRef="end"/>
    <endEvent id="end"/>
  </process>
</definitions>
```

为了启动这个过程，需要一个变量`inputArray`：

```java
Map<String, Object> variables = new HashMap<String, Object>();
variables.put("inputArray", new Integer[]{5, 23, 42});
runtimeService.startProcessInstanceByKey("process", variables);
```


# 在执行监听器中使用脚本

除了Java代码和表达式语言，Camunda平台还支持执行监听器脚本的执行。关于执行监听器的一般信息，请参见相应的[章节]({{< ref "/user-guide/process-engine/delegation-code.md#execution-listener" >}}).

要在执行监听器中使用脚本，必须在`camunda:executionListener`元素下添加一个`camunda:script`子元素。在脚本执行期间，变量`execution`是可用的，它对应于`DelegateExecution`接口。

下面的例子展示了如何在执行监听器中使用脚本。

```xml
<process id="process" isExecutable="true">
  <extensionElements>
    <camunda:executionListener event="start">
      <camunda:script scriptFormat="groovy">
        println "Process " + execution.eventName + "ed"
      </camunda:script>
    </camunda:executionListener>
  </extensionElements>

  <startEvent id="start">
    <extensionElements>
      <camunda:executionListener event="end">
        <camunda:script scriptFormat="groovy">
          println execution.activityId + " " + execution.eventName + "ed"
        </camunda:script>
      </camunda:executionListener>
    </extensionElements>
  </startEvent>
  <sequenceFlow id="flow1" startRef="start" targetRef="task">
    <extensionElements>
      <camunda:executionListener>
        <camunda:script scriptFormat="groovy" resource="org/camunda/bpm/transition.groovy" />
      </camunda:executionListener>
    </extensionElements>
  </sequenceFlow>

  <!--
    ... remaining process omitted
  -->
</process>
```


# 在任务监听器中使用脚本

与执行监听器类似，任务监听器也可以使用脚本实现。关于任务监听器的更多信息，请参见相应的[章节]({{< ref "/user-guide/process-engine/delegation-code.md#task-listener" >}}).

要在任务监听器中使用脚本，必须在`camunda:taskListener`元素中添加`camunda:script`子元素。在脚本中，变量`task`是可用的，它对应于`DelegateTask`接口。

下面的例子显示了脚本作为任务监听器的用法：

```xml
<userTask id="userTask">
  <extensionElements>
    <camunda:taskListener event="create">
      <camunda:script scriptFormat="groovy">println task.eventName</camunda:script>
    </camunda:taskListener>
    <camunda:taskListener event="assignment">
      <camunda:script scriptFormat="groovy" resource="org/camunda/bpm/assignemnt.groovy" />
    </camunda:taskListener>
  </extensionElements>
</userTask>
```

# 使用脚本作为条件表达

Camunda平台允许你使用 **脚本** 代替 **表达式语言** 作为条件序列流程的 "条件表达"。要做到这一点，"conditionExpression" 元素的 "language" 属性必须被设置为所需的脚本语言。脚本源代码是该元素的文本内容，与表达式语言一样。另一种指定脚本源代码的方法是定义一个外部源，参见[脚本源部分]({{< relref "#script-source" >}}).

下面的例子显示了脚本作为条件的用法。Groovy变量`status`是一个流程变量，可以在脚本中使用：

```xml
<sequenceFlow>
  <conditionExpression xsi:type="tFormalExpression" language="groovy">
    status == 'closed'
  </conditionExpression>
</sequenceFlow>

<sequenceFlow>
  <conditionExpression xsi:type="tFormalExpression" language="groovy"
      camunda:resource="org/camunda/bpm/condition.groovy" />
</sequenceFlow>
```

# 使用脚本作为输入输出参数

通过Camunda的 `inputOutput` 扩展元素，你可以用脚本映射 `inputParameter` 或`outputParameter`。下面的例子流程使用了前面例子中的Groovy脚本，将Groovy变量`sum`分配给一个Java委托的流程变量`x`。

{{< note title="脚本返回值" class="info" >}}
  请注意，脚本的最后一条语句的结果会被返回。这适用于Groovy、JavaScript和JRuby，但不适用于Jython。如果你想使用Jython，你的脚本必须是一个单一的表达式，如`a + b`或`a > b`，其中`a`和`b`已经是流程变量。否则，Jython脚本引擎将不会返回值。
{{< /note >}}

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
                   xmlns:camunda="http://activiti.org/bpmn"
                   targetNamespace="http://camunda.org/example">
  <process id="process" isExecutable="true">
    <startEvent id="start"/>
    <sequenceFlow id="sequenceFlow1" sourceRef="start" targetRef="task"/>
    <serviceTask id="task" camunda:class="org.camunda.bpm.example.SumDelegate">
      <extensionElements>
        <camunda:inputOutput>
          <camunda:inputParameter name="x">
             <camunda:script scriptFormat="groovy">
              <![CDATA[

              sum = 0

              for ( i in inputArray ) {
                sum += i
              }

              sum
              ]]>
            </camunda:script>
          </camunda:inputParameter>
        </camunda:inputOutput>
      </extensionElements>
    </serviceTask>
    <sequenceFlow id="sequenceFlow2" sourceRef="task" targetRef="end"/>
    <endEvent id="end"/>
  </process>
</definitions>
```

在脚本为`sum`变量赋值后，`x`可以在Java委托代码中使用：

```java
public class SumDelegate implements JavaDelegate {

  public void execute(DelegateExecution execution) throws Exception {
    Integer x = (Integer) execution.getVariable("x");

    // do something
  }

}
```

脚本的源代码也可以从外部资源加载，方法与[脚本任务]({{< relref "#script-source" >}})一致。

```xml
<camunda:inputOutput>
  <camunda:inputParameter name="x">
     <camunda:script scriptFormat="groovy" resource="org/camunda/bpm/example/sum.groovy"/>
  </camunda:inputParameter>
</camunda:inputOutput>
```
# 脚本引擎缓存

每当流程引擎到达必须执行脚本的时候，流程引擎就会按语言名称寻找一个脚本引擎。默认的行为是，如果是第一次请求，就会创建一个新的脚本引擎。如果脚本引擎声明是线程安全的，它也会被缓存起来。通过缓存可以防止流程引擎为同一脚本语言的每个请求创建一个新的脚本引擎。

默认情况下，脚本引擎的缓存发生在流程应用程序层面。每个流程应用程序都持有一个特定语言的脚本引擎的实例。这种行为可以通过将名为 "enableFetchScriptEngineFromProcessApplication" 的流程引擎配置设置为false来禁用。因此，脚本引擎在流程引擎层面被全局缓存，它们在每个流程应用程序之间被共享。关于流程引擎配置标志 "enableFetchScriptEngineFromProcessApplication" 的更多信息，请阅读关于[引用流程应用程序类]({{< ref "/user-guide/process-engine/scripting.md#reference-process-application-provided-classes" >}})的部分。

如果大多数情况下不希望对脚本引擎进行缓存，可以通过将流程引擎配置 `enableScriptEngineCaching` 设置为false来禁用它。


# 脚本编译

大多数脚本引擎在执行脚本之前，都会将脚本源代码编译成Java类或其他的中间格式。实现Java `Compilable`接口的脚本引擎允许程序查询和缓存脚本编译。流程引擎的默认会检查一个脚本引擎是否支持编译。如果支持，则会启用脚本引擎的缓存功能，也就是脚本引擎会编译脚本，然后缓存编译的结果。这可以防止流程引擎在每次执行同一个脚本任务时都要编译脚本源代码。

默认情况下，脚本的编译是启用的。如果你想禁用脚本编译，你可以将名为 "enableScriptCompilation" 的流程引擎配置设置为false。

# 加载脚本引擎

如果名为 "enableFetchScriptEngineFromProcessApplication" 的流程引擎配置配置被设置为 "true"，也可以从流程应用程序的classpath中加载脚本引擎。为此，脚本引擎可以被打包成流程应用程序中的一个库。也可以全局安装脚本引擎。

如果脚本引擎模块全局安装，并且使用JBoss，就有必要给脚本引擎添加一个模块依赖关系。这可以通过在流程应用程序中添加`jboss-deployment-structure.xml`来实现，例如：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jboss-deployment-structure>
  <deployment>
    <dependencies>
      <module name="org.codehaus.groovy.groovy-all"
              services="import" />
    </dependencies>
  </deployment>
</jboss-deployment-structure>
```

# Configure Script Engine

Most script engines offer configuration options to adjust their script execution semantics.
We provide the following default configurations for supported script engines before executing code on them:

<table class="table desc-table">
  <tr>
    <th>Script Engine</th>
    <th>Default configuration</th>
  </tr>
  <tr>
    <td>GraalVM JavaScript</td>
    <td>
      Configured to allow host acces and host class lookup by setting <code>polyglot.js.allowHostAccess</code> and 
      <code>polyglot.js.allowHostClassLookup</code> to <code>true</code>.
    </td>
  </tr>
  <tr>
    <td>Groovy</td>
    <td>Configured to only hold weak references to Java methods by setting <code>#jsr223.groovy.engine.keep.globals</code> to <code>weak</code>.</td>
  </tr>
</table>

Besides those default options, you can configure the script engines by any of the following:

* Set script engine-specific configuration flags in process engine configuration.
* Provide script engine-specific system properties.
* Provide a custom implementation of the `ScriptEngineResolver` interface.

Note that for JavaScript execution you might be able to choose the script engine to use depending on your setup. Consult
[JavaScript Considerations](#javascript-considerations) for further information.

## Process engine flags

You can use the following process engine configuration flags to influence the configuration of specific script engines:

* [configureScriptEngineHostAccess]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#configureScriptEngineHostAccess" >}}) - 
  Specifies whether host language resources like classes and their methods are accessible or not.
* [enableScriptEngineLoadExternalResources]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#enableScriptEngineLoadExternalResources" >}}) - 
  Specifies whether external resources can be loaded from file system or not.
* [enableScriptEngineNashornCompatibility]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#enableScriptEngineNashornCompatibility" >}}) - 
  Specifies whether Nashorn compatibility mode is enabled or not.

## System properties

Depending on the script engine, specific system properties can be used to influence the setup of the script engine.
Consult the development guides of the script engine you want to configure for further information on available parameters.
Note that the supported options can differ between versions of the script engine.

You can set system properties either programmatically through `System.setProperty(parameter, value)` or as JVM arguments, 
for example upon application start on command line via `-Dparameter=value`. Most application servers like JBoss AS/Wildfly, 
Tomcat, Websphere, and Weblogic support providing JVM arguments via environment variables `JAVA_OPTS` or `JAVA_OPTIONS`. 
Consult your application server's documentation to learn how to pass on JVM arguments. Camunda Platform Run supports setting 
JVM arguments via the `JAVA_OPTS` environment variable as well.

## Custom ScriptEngineResolver

You can provide a custom `ScriptEngineResolver` implementation to configure script engines. Depending on the specifc script engine to configure, 
you can gain more configuration options with this approach. You can add your custom script engine resolver to the engine configuration 
with the `#setScriptEngineResolver(ScriptEngineResolver)` method.

You can inherit from the `org.camunda.bpm.engine.impl.scripting.engine.DefaultScriptEngineResolver` for starters in case configuring an existing 
script engine instance is sufficient for you. By overriding the `#configureScriptEngines(String, ScriptEngine)` method of the `DefaultScriptEngineResolver`, 
you can change settings on the script engine instance provided to that method as shown in the following example:

```java
public class CustomScriptEngineResolver extends DefaultScriptEngineResolver {

  public CustomScriptEngineResolver(ScriptEngineManager scriptEngineManager) {
    super(scriptEngineManager);
  }

  protected void configureScriptEngines(String language, ScriptEngine scriptEngine) {
    super.configureScriptEngines(language, scriptEngine);
    if (ScriptingEngines.GROOVY_SCRIPTING_LANGUAGE.equals(language)) {
      // make sure Groovy compiled scripts only hold weak references to java methods
      scriptEngine.getContext().setAttribute("#jsr223.groovy.engine.keep.globals", "soft", ScriptContext.ENGINE_SCOPE);
    }
  }
}
```

If you need more flexibility in configuring a script engine, you can override a method further up the chain in the script engine creation
or provide your own plain implementation of the interface. Have a look at the following example that provides a custom **GraalVM JavaScript** 
instance with Nashorn Compatibility Mode enabled:

```java
public class CustomScriptEngineResolver extends DefaultScriptEngineResolver {

  public CustomScriptEngineResolver(ScriptEngineManager scriptEngineManager) {
    super(scriptEngineManager);
  }

  @Override
  protected void configureGraalJsScriptEngine(ScriptEngine scriptEngine) {
    // do nothing
  }

  @Override
  protected ScriptEngine getJavaScriptScriptEngine(String language) {
    return com.oracle.truffle.js.scriptengine.GraalJSScriptEngine.create(null,
        org.graalvm.polyglot.Context.newBuilder("js")
        // make sure GraalVM JS can provide access to the host and can lookup classes
        .allowHostClassLookup(s -> true)
        // enable Nashorn Compatibility Mode
        .allowExperimentalOptions(true)
        .option("js.nashorn-compat", "true"));
  }
}
```

# 使用应用程序提供的类

脚本可以像下面的groovy脚本例子那样通过来导入应用程序提供的类：

```java
import my.process.application.CustomClass

sum = new CustomClass().calculate()
execution.setVariable('sum', sum)
```

为了避免在脚本执行过程中可能出现的类加载问题，建议将流程引擎配置`enableFetchScriptEngineFromProcessApplication` 设置为true。

请注意，流程引擎配置项`enableFetchScriptEngineFromProcessApplication`只在使用分布式流程引擎的情况下才有意义。

# 脚本执行中可用的变量

在脚本的执行过程中，所有在当前范围内可见的流程变量都是可用的。可以通过变量的名称直接访问（例如，`sum`）。但JRuby不同，你必须把变量当做ruby全局变量来访问（在前面加一个美元符号，即`$sum`）。

还有一些特殊的变量：

1. `execution`, 如果脚本是在一个执行范围内执行的（例如，脚本任务），`execution`是可用的({{< javadocref page="?org/camunda/bpm/engine/delegate/DelegateExecution.html" text="DelegateExecution" >}})。
1. `task`, 如果脚本是在一个任务范围内执行的（例如，任务监听器），`task`是可用的({{< javadocref page="?org/camunda/bpm/engine/delegate/DelegateTask.html" text="DelegateTask" >}})。
1. `connector`，如果脚本在连接器的变量范围内执行（例如，camunda:connector的outputParameter），`connector`是可用的。 ({{< javadocref page="?org/camunda/connect/plugin/impl/ConnectorVariableScope.html" text="ConnectorVariableScope" >}})。

这些变量对应着 "DelegateExecution"、"DelegateTask" 或 "ConnectorVariableScope" 接口，这意味着它可以用来获取和设置变量以及访问流程引擎服务。

```java
// 获取流程变量
sum = execution.getVariable('x')

// 设置流程变量
execution.setVariable('y', x + 15)

// 获取 task service 查询 task
task = execution.getProcessEngineServices().getTaskService()
  .createTaskQuery()
  .taskDefinitionKey("task")
  .singleResult()
```

# 在脚本中访问流程引擎服务

Camunda 的 Java API 提供的访问流程引擎服务可以用脚本来访问：

{{< javadocref page="?org/camunda/bpm/engine/ProcessEngineServices.html" text="Process Engine Services" >}} \

{{< javadocref page="?org/camunda/bpm/engine/package-summary.html" text="Public Java API of Camunda Platform Engine" >}}

下面的案例，创建了一个 message key 为 "work" 的BPMN消息：

```javascript
execution.getProcessEngineServices().getRuntimeService().createMessageCorrelation("work").correlateWithResult();
```


# 使用脚本打印到控制台

在执行脚本的过程中，由于记录和调试的需要，可能需要打印信息到控制台。

下面展示了如何在各种脚本语言中做到：

* Goovy:

```groovy
println 'This prints to the console'
```

* Java:

```java
var system = java.lang.System;
system.out.println('This prints to the console');
```


# 脚本来源

在BPMN XML模型中指定脚本源代码的标准方式是直接将其添加到XML文件中。尽管如此，Camunda平台提供了额外的方式来指定脚本源。

如果你使用的是表达式语言以外的另一种脚本语言，你也可以将脚本源指定为一个表达式，该表达式返回要执行的源代码。例如，源代码可以设置在一个流程变量中。

在下面的例子中，流程引擎将在每次执行元素时在当前上下文中评估表达式`${sourceCode}`：

```xml
<!-- 在脚本任务中 -->
<scriptTask scriptFormat="groovy">
  <script>${sourceCode}</script>
</scriptTask>

<!-- 在执行监听器中 -->
<camunda:executionListener>
  <camunda:script scriptFormat="groovy">${sourceCode}</camunda:script>
</camunda:executionListener>

<!-- 作为一个条件表达 -->
<sequenceFlow id="flow" sourceRef="theStart" targetRef="theTask">
  <conditionExpression xsi:type="tFormalExpression" language="groovy">
    ${sourceCode}
  </conditionExpression>
</sequenceFlow>

<!-- 作为一个输入输出参数 -->
<camunda:inputOutput>
  <camunda:inputParameter name="x">
    <camunda:script scriptFormat="groovy">${sourceCode}</camunda:script>
  </camunda:inputParameter>
</camunda:inputOutput>
```

你也可以在`scriptTask`和`conditionExpression`元素上指定`camunda:resource`属性，分别对应各自`camunda:script`元素上的`resource`属性。这个扩展属性指定了一个外部资源的位置，用于指向脚本源代码。
可以在资源路径前加上类似于URL的前缀，用来说明资源是否包含在deployment或classpath中。默认的行为是，deployment是classpath的一部分。这意味着以下例子中的前两个脚本指向是一样的：

```xml
<!-- 在脚本任务上 -->
<scriptTask scriptFormat="groovy" camunda:resource="org/camunda/bpm/task.groovy"/>
<scriptTask scriptFormat="groovy" camunda:resource="classpath://org/camunda/bpm/task.groovy"/>
<scriptTask scriptFormat="groovy" camunda:resource="deployment://org/camunda/bpm/task.groovy"/>

<!-- 在执行监听器中 -->
<camunda:executionListener>
  <camunda:script scriptFormat="groovy" resource="deployment://org/camunda/bpm/listener.groovy"/>
</camunda:executionListener>

<!-- 作为条件表达式 -->
<conditionExpression xsi:type="tFormalExpression" language="groovy"
    camunda:resource="org/camunda/bpm/condition.groovy" />

<!-- 作为输入参数 -->
<camunda:inputParameter name="x">
  <camunda:script scriptFormat="groovy" resource="org/camunda/bpm/mapX.groovy" />
</camunda:inputParameter>
```

资源路径也可以被指定为一个表达式，在调用脚本任务时被计算：

```xml
<scriptTask scriptFormat="groovy" camunda:resource="${scriptPath}"/>
```

想要了解更多信息， 参见 [Custom Extensions]({{< ref "/reference/bpmn20/custom-extensions/_index.md" >}}) 中的 [camunda:resource]({{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#resource" >}})章节。

# JavaScript 考虑事项

JavaScript code execution is part of the Java Runtime (JRE) with the **Nashorn** script engine until Java 14 and thus only there available out of the box.
We include **GraalVM JavaScript** in the pre-packaged Camunda distributions as a replacement regardless of the JRE version.
JavaScript code executes on GraalVM JavaScript with preference in the process engine context if this script engine is available.
If this script engine cannot be found, the process engine defaults to let the JVM select an appropriate script engine.

You can set the default JavaScript engine to use for languages `javascript` and `ecmascript` with the process engine configuration property named `scriptEngineNameJavaScript`.
Set this value to `nashorn` to configure the process engine to use the Nashorn script engine by default.
Note that if no script engine related to that value can be found, the process engine does not look for a fallback and throws an exception.

Consult the [official GraalVM JavaScript Guide](https://www.graalvm.org/reference-manual/js/ScriptEngine/) for questions around that script engine. 
It also contains a guide on [Migration from Nashorn](https://www.graalvm.org/reference-manual/js/NashornMigrationGuide/).
