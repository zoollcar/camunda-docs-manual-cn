---

title: '自动资源部署'
weight: 30

menu:
  main:
    name: "部署"
    identifier: "user-guide-spring-framework-integration-deployment"
    parent: "user-guide-spring-framework-integration"
    pre: "Deploy Resources on Startup"

---

Spring 集成还有一个用于部署资源的特殊功能。 在流程引擎配置中，你可以指定一组资源。 创建流程引擎时，将扫描和部署所有这些资源。 有适当的过滤可以防止重复部署。 只有在资源实际发生变化的情况下，才会将新部署部署到引擎数据库。 这在很多时候中是有意义的，其中 Spring 容器经常重新启动（例如，测试环境）。

这是一个案例：

```xml
<bean id="processEngineConfiguration"
      class="org.camunda.bpm.engine.spring.SpringProcessEngineConfiguration">
  ...
  <property name="deploymentResources"
            value="classpath*:/mytest/autodeploy.*.bpmn20" />
  <property name="deploymentResources">
    <list>
      <value>classpath*:/mytest/autodeploy.*.bpmn20</value>
      <value>classpath*:/mytest/autodeploy.*.png</value>
    </list>
  </property>
</bean>

<bean id="processEngine"
      class="org.camunda.bpm.engine.spring.ProcessEngineFactoryBean">
  <property name="processEngineConfiguration" ref="processEngineConfiguration" />
</bean>
```
