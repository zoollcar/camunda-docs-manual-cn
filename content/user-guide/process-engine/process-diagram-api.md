---

title: '流程设计可视化'
weight: 250

menu:
  main:
    identifier: "user-guide-process-engine-pd-api"
    parent: "user-guide-process-engine"

---


BPMN 流程图一个强大的地方，是可以将你的流程周围的信息可视化。 我们建议使用 JavaScript 库来显示流程图并用附加信息丰富它们。

在我们的网络应用程序 [Cockpit]({{< ref "/webapps/cockpit/_index.md" >}}) 和 [Tasklist]({{< ref "/webapps/tasklist/_index.md" >}}) 中， 我们使用 [bpmn.io](http://bpmn.io/)，这是一个直接在浏览器中呈现 BPMN 2.0 流程模型的工具包。 它允许向图表添加附加信息，并包括用户交互的方式。 虽然 bpmn.io 还在开发中，但它的 API 已经相当稳定了。

之前的 JavaScript BPMN 渲染器仍然可以在 [camunda-bpmn.js](https://github.com/camunda/camunda-bpmn.js) 找到，但它不再积极开发了。

{{< img src="../img/process-diagram-bpmn-js.png" title="Process Diagram Rendering" >}}


# bpmn.io 图表渲染器

要呈现流程图，你需要通过 {{< javadocref page="?org/camunda/bpm/engine/RepositoryService.html" text="Java-" >}} 或 [REST]({ {< ref "/reference/rest/process-definition/get-xml.md" >}}) API。 以下示例显示如何使用 bpmn.io 呈现 流程XML。 有关图表注释和用户交互的更多文档，请参阅 [bpmn.io](https://github.com/bpmn-io/bpmn-js) 页面。

```javascript
var BpmnViewer = require('bpmn-js');

var xml = getBpmnXml(); // 通过 REST 获取 流程xml
var viewer = new BpmnViewer({ container: 'body' });

viewer.importXML(xml, function(err) {

  if (err) {
    console.log('error rendering', err);
  } else {
    console.log('rendered');
  }
});
```

除此之外，你可以使用 Camunda commons UI 中的 [bpmn-viewer 小部件](https://github.com/camunda/camunda-bpm-platform/blob/master/webapps/camunda-commons-ui/lib/widgets/bpmn-viewer/cam-widget-bpmn-viewer.html)。
