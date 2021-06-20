---

title: '自定义代码的安全性'
weight: 95

menu:
  main:
    identifier: "user-guide-process-engine-securing-custom-code"
    parent: "user-guide-process-engine"

---

流程引擎提供了许多外部扩展点，可以通过使用[Java代码]({{< relref "delegation-code.md" >}})、[表达式语言]({{< relref "expression-language.md" >}})、 [脚本]({{< relref "scripting.md" >}}) 和 [模板]({{< relref "templating.md" >}})来定制流程的行为。虽然这些外部扩展点给了流程实施方面很大的灵活性，但如果落入坏人之手，它们就有可能执行恶意代码。因此，最好限制对允许提交自定义代码的API的访问，只允许受信任的人使用。在提交自定义代码（通过Java或REST API）时，存在以下概念：

* **部署**: 大部分的自定义逻辑是随着流程、案例或决策模型的部署而提交的。例如，在BPMN 2.0的XML中定义了一个执行监听器的调用。
* **查询**: 一些查询可以提交具有动态表达式的参数（目前只有任务查询）。这使用户能够定义可重复使用的查询，这些查询可以重复执行并动态地适应不断变化的情况。例如，任务查询 `taskService.createTaskQuery().dueBeforeExpression(${now()}).list();` 使用了一个表达式，总是返回当前到期的任务。Camunda [Tasklist]({{< ref "/webapps/tasklist/_index.md" >}}) 的 [任务过滤器（task filters）]({{< ref "/webapps/tasklist/filters.md" >}})提供了这样的查询方法。

只有受信任的人才应被授权与这些端点进行互动的能力。如何限制访问，将在接下来的章节中概述。

# 如果Camunda平台自身在受信任的环境

如果Camunda平台部署在一个只有受信任方能访问系统的环境中时（例如由于防火墙策略），任何不受信任方都不能访问API以提交自定义代码，因此无需遵守以下建议。

# 部署

可以通过使用[授权框架]({{< relref "authorization-service.md" >}})来限制对执行部署的访问，并对任何可能不受信任的一方可能访问的端点激活认证检查。进行部署的关键权限是 "部署/创建"。不受信任的用户不应该被授予这个权限。

# 查询

查询访问一般不能用授权来限制。相反，一个查询的结果被简化为用户被授权访问的实体。因此，授权权限不能被用来保护查询中的表达式计算。

流程引擎配置提供了两个标志 *adhoc* 和 *stored* 来切换对查询中表达式的计算。临时查询是直接提交的查询。例如，`taskService.createTaskQuery().list();`会创建并执行一个 adhoc 查询。这样，存储查询会与过滤器一起被持久化，并在过滤器被执行时被执行。通过将配置属性`enableExpressionsInAdhocQueries`设置为`false`，可以禁用临时查询中的表达式。相应地，属性`enableExpressionsInStoredQueries`会禁用存储查询中的表达式。如果在禁用表达式计算的情况下使用表达式，流程引擎会在计算表达式之前引发一个异常，从而防止恶意代码被执行。

存在以下配置组合：

* `enableExpressionsInAdhocQueries`=`true`, `enableExpressionsInStoredQueries`=`true`。表达式评估在任何查询中都被启用。如果所有用户都被信任，请使用此设置。
* `enableExpressionsInAdhocQueries`=`false`, `enableExpressionsInStoredQueries`=`true`: **默认设置**。临时查询不能使用表达式，但是可以定义和执行带有表达式的过滤器。可以通过授予授权权限 `Filter/Create` 来限制创建过滤器的权限。如果所有被授权创建过滤器的用户都是被信任的，请使用此设置。
* `enableExpressionsInAdhocQueries`=`false`, `enableExpressionsInStoredQueries`=`false`: 表达式在所有查询中被禁用。如果上述设置都不能保证安全，请使用此设置。
