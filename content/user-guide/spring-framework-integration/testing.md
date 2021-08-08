---

title: 'Spring-Based 测试'
weight: 50

menu:
  main:
    name: "测试"
    identifier: "user-guide-spring-framework-integration-testing"
    parent: "user-guide-spring-framework-integration"
    pre: "Write Spring-Based Unit Tests"

---

与 Spring 集成时，可以使用标准的 Camunda 测试工具非常轻松地测试业务流程（在范围 2 中，请参阅 [测试范围][Testing Scopes]）。 以下示例显示了如何在典型的基于 Spring 的单元测试中测试业务流程：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:org/camunda/bpm/engine/spring/test/junit4/springTypicalUsageTest-context.xml")
public class MyBusinessProcessTest {

  @Autowired
  private RuntimeService runtimeService;

  @Autowired
  private TaskService taskService;

  @Autowired
  @Rule
  public ProcessEngineRule processEngineRule;

  @Test
  @Deployment
  public void simpleProcessTest() {
    runtimeService.startProcessInstanceByKey("simpleProcess");
    Task task = taskService.createTaskQuery().singleResult();
    assertEquals("My Task", task.getName());

    taskService.complete(task.getId());
    assertEquals(0, runtimeService.createProcessInstanceQuery().count());

  }
}
```

请注意，要使其正常工作，你需要在 Spring 配置中定义一个 {{< javadocref page="?org/camunda/bpm/engine/test/ProcessEngineRule.html" text="ProcessEngineRule" >}} bean（即 在上面的例子中通过自动布线注入）。

```xml
<bean id="processEngineRule" class="org.camunda.bpm.engine.test.ProcessEngineRule">
  <property name="processEngine" ref="processEngine" />
</bean>
```

[Testing Scopes]: {{< ref "/user-guide/testing/_index.md#scoping-tests" >}}
