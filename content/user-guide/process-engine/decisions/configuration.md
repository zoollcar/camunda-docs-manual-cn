---

title: '配置 DMN 引擎'
weight: 10

menu:
  main:
    name: "配置"
    identifier: "user-guide-process-engine-decisions-configuration"
    parent: "user-guide-process-engine-decisions"
    pre: "Configure the DMN engine"

---

DMN引擎的配置是流程引擎配置的一部分。 这取决于你使用的是应用程序管理的还是共享的、容器管理的流程引擎。 有关详细信息，请参阅 [Process Engine Bootstrapping][Process Engine Bootstrapping]。

本节展示如何配置 DMN 引擎：

* [通过 Java API 编程]({{< relref "#configure-the-dmn-engine-using-java-api" >}})
* [通过 XML配置 声明]({{< relref "#configure-the-dmn-engine-using-spring-xml" >}})

在示例中，输入表达式的默认表达式语言设置为 `groovy`。 在 [DMN 引擎配置][DMN Engine Configuration] 部分可以找到所有可能配置的列表。

# 使用 Java API 配置 DMN 引擎

首先，你需要创建一个 [ProcessEngineConfiguration]({{< ref "/user-guide/process-engine/process-engine-bootstrapping.md#使用java-api启动流程引擎" >}}) 用于流程引擎的对象和 DMN 引擎的“DmnEngineConfiguration”对象。 现在你可以使用“DmnEngineConfiguration”对象配置 DMN 引擎。 完成后，在“ProcessEngineConfiguration”上设置对象并调用“buildProcessEngine()”来创建流程引擎。

```java
// 创建流程引擎配置
ProcessEngineConfigurationImpl processEngineConfiguration = // ...
    
// 创建 DMN 引擎配置 
DefaultDmnEngineConfiguration dmnEngineConfiguration = (DefaultDmnEngineConfiguration) 
  DmnEngineConfiguration.createDefaultDmnEngineConfiguration();

// 配置DMN引擎...
// 例如 将输入表达式的默认表达式语言设置为 `groovy`
dmnEngineConfiguration.setDefaultInputExpressionExpressionLanguage("groovy");

// 在流程引擎配置上设置 DMN 引擎配置
processEngineConfiguration.setDmnEngineConfiguration(dmnEngineConfiguration);

// 构建包含 DMN 引擎的流程引擎
processEngineConfiguration.buildProcessEngine();
```

# 使用 Spring XML 配置 DMN 引擎

按照 [教程]({{< ref "/user-guide/process-engine/process-engine-bootstrapping.md#使用camunda-cfg-xml配置流程引擎" >}}) 创建基础 `camunda .cfg.xml` 流程引擎的 XML 配置。

添加类 `org.camunda.bpm.dmn.engine.impl.DefaultDmnEngineConfiguration` 的新配置 bean。 使用 bean 配置 DMN 引擎并将其设置为 `processEngineConfiguration` bean 上的 `dmnEngineConfiguration` 属性。

```xml
<beans xmlns="http://www.springframework.org/schema/beans" 
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="processEngineConfiguration" 
        class="org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration">
  
    <property name="dmnEngineConfiguration">
      <bean class="org.camunda.bpm.dmn.engine.impl.DefaultDmnEngineConfiguration">
        
        <!-- 配置DMN引擎... --> 
        <!-- 例如 将输入表达式的默认表达式语言设置为 `groovy` -->
        <property name="defaultInputExpressionExpressionLanguage" value="groovy" />
        
      </bean>
    </property>
    
  </bean>
</beans>
```

[Process Engine Bootstrapping]: {{< ref "/user-guide/process-engine/process-engine-bootstrapping.md" >}}
[DMN Engine Configuration]: {{< ref "/user-guide/dmn-engine/embed.md#configuration-of-the-dmn-engine" >}}

