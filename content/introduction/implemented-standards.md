---

title: '遵循的标准'
weight: 20

menu:
  main:
    identifier: "user-guide-introduction-standards"
    parent: "user-guide-introduction"

---

Camunda在业务流程方面实现了三种不同的标准：BPMN 2.0、CMMN 1.1和DMN 1.3。
这三个标准是由[Object Management Group][OMG]在Camunda的积极合作下定义的。

Camunda平台提供执行和[建模工具][modelers]的开源实现。

# BPMN

Business Process Model and Notation (BPMN) 是一个工作流和流程自动化标准。

Camunda 支持 BPMN 的 2.0 版本

* 快速入门实现BPMN流程: [Quick Start (Java / JS)]
* 快速了解BPMN建模语言: [BPMN Modeling Tutorial]
* BPMN建模相关信息: [BPMN Modeling Reference]
* BPMN建模工具: [BPMN Modeler][modelers]
* 实现BPMN 流程: [BPMN Implementation Reference]
* 执行BPMN: [Process Engine]

# CMMN

Case Management Model and Notation (CMMN) 是一个案例管理标准

Camunda 支持 CMMN 的 1.1 版本。

* 快速入门实现CMMN案件: [CMMN Getting Started]
* 实现CMMN案例: [CMMN Implementation Reference]
* 执行CMMN: [Process Engine]

# DMN

Decision Model and Notation (DMN) 是一个业务决策标准。

Camunda 支持 DMN 的 1.1 版本。

* 快速入门实现 DMN 决策表: [DMN Getting Started]
* 了解 DMN: [DMN Modeling Tutorial]
* 学习用于编辑 DMN 的工具: [DMN Editor][modelers]
* 实现DMN 决策: [DMN Implementation Reference]
* 执行DMN: [DMN Engine]


[OMG]: http://www.omg.org/
[modelers]: {{< ref "/modeler/_index.md" >}}
[BPMN Modeling Tutorial]: https://camunda.org/bpmn/tutorial/
[BPMN Modeling Reference]: https://camunda.org/bpmn/reference/
[Quick Start (Java / JS)]: /get-started/quick-start/
[BPMN Implementation Reference]: {{< ref "/reference/bpmn20/_index.md" >}}
[CMMN Implementation Reference]: {{< ref "/reference/cmmn11/_index.md" >}}
[DMN Getting Started]: /get-started/dmn11/
[DMN Implementation Reference]: {{< ref "/reference/dmn/_index.md" >}}
[DMN Modeling Tutorial]: https://camunda.org/dmn/tutorial/
[Process Engine]: {{< ref "/user-guide/process-engine/_index.md" >}}
[DMN Engine]: {{< ref "/user-guide/dmn-engine/_index.md" >}}
