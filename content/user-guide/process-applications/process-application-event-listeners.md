---

title: 'Process Application 事件监听'
weight: 30

menu:
  main:
    identifier: "user-guide-process-application-event-listeners"
    parent: "user-guide-process-applications"

---


流程引擎支持定义两种类型的事件侦听器：[任务事件侦听器和执行事件侦听器]({{< ref "/user-guide/process-engine/delegation-code.md#execution-listener" >}})。
任务事件侦听器允许对任务事件做出反应（任务已创建、已分配、已完成）。 执行侦听器允许对在执行过程中通过流程设计触发的事件做出反应：活动开始、结束和转换正在进行。

使用流程应用程序 API 时，流程引擎确保将事件委托给正确的流程应用程序。 例如，假设有一个流程应用程序部署为“invoice.war”，它部署了一个名为“invoice”的流程定义。 发票流程有一个名为“归档发票”的任务。 应用程序“invoice.war”进一步提供了一个实现 [执行侦听器]({{< ref "/user-guide/process-engine/delegation-code.md#execution-listener" >}}) 接口的 Java Class 并进行了配置 每当在“归档发票”活动上触发结束事件时调用。 流程引擎确保将事件委托给位于流程应用程序内部的侦听器类：

{{< img src="../img/process-application-events.png" title="Process Application Events" >}}

在 [BPMN 2.0 XML 中明确配置]({{< ref "/user-guide/process-engine/delegation-code.md#execution-listener" >}}) 的执行和任务侦听器之上， 流程应用程序 API 支持定义一个全局 ExecutionListener 和一个全局 TaskListener ，它们会收到有关流程应用程序部署的流程中发生的*所有事件*的通知：

    @ProcessApplication
    public class InvoiceProcessApplication extends ServletProcessApplication {

      public TaskListener getTaskListener() {
        return new TaskListener() {
          public void notify(DelegateTask delegateTask) {
            // handle all Task Events from Invoice Process
          }
        };
      }

      public ExecutionListener getExecutionListener() {
        return new ExecutionListener() {
          public void notify(DelegateExecution execution) throws Exception {
            // handle all Execution Events from Invoice Process
          }
        };
      }
    }

要使用全局流程应用事件监听器，你需要激活相应的[流程引擎插件]({{< ref "/user-guide/process-engine/process-engine-plugins.md" >}})：

    <process-engine name="default">
      ...
      <plugins>
        <plugin>
          <class>org.camunda.bpm.application.impl.event.ProcessApplicationEventListenerPlugin</class>
        </plugin>
      </plugins>
    </process-engine>

请注意，该插件在预打包的 Camunda 平台发行版中默认激活。

如果你想[结合共享流程引擎使用CDI Events]({{< ref "/user-guide/cdi-java-ee-integration/the-cdi-event-bridge.md" >}})流程应用，可以在Event Listener接口添加CdiEventListener桥接器。
