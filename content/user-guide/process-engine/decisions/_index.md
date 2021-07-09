---

title: '流程引擎中的决策'
weight: 260

menu:
  main:
    name: "决策"
    identifier: "user-guide-process-engine-decisions"
    parent: "user-guide-process-engine"

---

Camunda 平台集成了 [Camunda DMN 引擎][[Camunda DMN engine]] 以评估业务决策。 本节介绍如何将业务决策（建模为 DMN 决策的）与其他资源一起部署到 Camunda 平台的存储库。 部署的决策可以使用 [Services API][[Services API]] 进行评估，也可以在 BPMN 流程和 CMMN 案例中引用。评估的结果保存在 [历史][History] 中，用于审计和报告。

[Camunda DMN engine]: {{< ref "/user-guide/dmn-engine/_index.md" >}}
[Services API]: {{< ref "/user-guide/process-engine/process-engine-api.md#services-api" >}}
[History]: {{< ref "/user-guide/process-engine/history.md" >}}
