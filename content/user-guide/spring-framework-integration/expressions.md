---

title: '流程引擎中的 Spring Beans'
weight: 40

menu:
  main:
    name: "Spring Bean 解释"
    identifier: "user-guide-spring-framework-integration-expressions"
    parent: "user-guide-spring-framework-integration"
    pre: "Use Spring Beans in Processes"

---

# Limit the Exposing Spring Beans in Expressions

使用 ProcessEngineFactoryBean 时，默认情况下，BPMN 流程中的所有表达式也将“看到”所有 Spring bean。 可以使用可配置的映射来限制要在表达式中公开的 bean，甚至根本不公开 bean。 下面的示例公开了一个 bean（printer），可在键 `printer` 下使用。 要完全不暴露 bean，只需在 `SpringProcessEngineConfiguration` 上传递一个空列表作为 `beans` 属性。 当没有设置 `beans` 属性时，上下文中的所有 Spring bean 都将可用。 

```xml
<bean id="processEngineConfiguration"
      class="org.camunda.bpm.engine.spring.SpringProcessEngineConfiguration">
  ...
  <property name="beans">
    <map>
      <entry key="printer" value-ref="printer" />
    </map>
  </property>
</bean>

<bean id="printer"
      class="org.camunda.bpm.engine.spring.test.transaction.Printer" />
```

# 在表达式中使用 Spring Bean

暴露的 bean 可以在表达式中使用。 例如，`SpringTransactionIntegrationTest``testBasicActivitiSpringIntegration.bpmn20.xml` 显示了如何使用 UEL 方法表达式调用 Spring bean 上的方法：

```xml
<definitions id="definitions" ...>

  <process id="helloProcess" isExecutable="true">
  
    <startEvent id="start" />
    <sequenceFlow id="flow1" sourceRef="start" targetRef="print" />
    
    <serviceTask id="print" 
                 camunda:expression="#{printer.printMessage(execution)}" />
    <sequenceFlow id="flow2" sourceRef="print" targetRef="userTask" />
    
    <userTask id="userTask" />
    <sequenceFlow id="flow3" sourceRef="userTask" targetRef="end" />
    
    <endEvent id="end" />
    
  </process>

</definitions>
```

Printer 看起来像这样：

```java
public class Printer {

  public void printMessage(ActivityExecution execution) {
    execution.setVariable("myVar", "Hello from Printer!");
  }
}
```

Spring bean 配置如下所示：

```xml
<beans ...>
  ...

  <bean id="printer"
        class="org.camunda.bpm.engine.spring.test.transaction.Printer" />
</beans>
```

# 使用共享流程引擎进行表达式解析

在共享流程引擎部署场景中，你的一个流程引擎可以分派到多个应用程序。在这种情况下，没有单一的 Spring 应用程序上下文，每个应用程序都可以维护自己的应用程序上下文。流程引擎不能将单个表达式解析器用于单个应用程序上下文，但必须委托给适当的流程应用程序，具体取决于当前正在执行的流程。

此功能由 `org.camunda.bpm.engine.spring.application.SpringProcessApplicationElResolver` 提供。此类是委托给本地应用程序上下文的“ProcessApplicationElResolver”实现。表达式解析然后按以下方式工作：

1. 共享流程引擎检查哪个流程应用程序对应于它当前正在执行的流程。
2. 然后它委托该流程应用程序来解析表达式。
3. 流程应用程序委托给`SpringProcessApplicationElResolver`，它使用本地Spring 应用程序上下文来解析bean。 

{{< note title="" class="info" >}}
  The `SpringProcessApplicationElResolver` class is automatically detected if the `camunda-engine-spring` module is included as a library of the process application, not as a global library.
{{< /note >}}

# 在脚本中使用 Spring Bean

使用 ProcessEngineFactoryBean 时，所有 Spring bean 都可以在 Groovy、JavaScript 和 Jython 中访问。 例如，`ScriptTaskTest-applicationContext.xml` 公开了 bean 'testbean'：

```xml
<beans ...>
  ...

  <bean id="testbean" 
        class="org.camunda.bpm.engine.spring.test.scripttask.Testbean" />
</beans>
```
Testbean 如下所示：

```java
@Component
public class Testbean {
  private String name = "name property of testbean";

  public String getName() {
    return name;
  }
}
```

`Testbean` 被引用，然后 JavaScript：

```javascript
  execution.setVariable('foo', testbean.name);
```
