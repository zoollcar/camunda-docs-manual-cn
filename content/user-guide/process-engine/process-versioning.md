---

title: '流程版本控制'
weight: 110

menu:
  main:
    identifier: "user-guide-process-engine-versioning"
    parent: "user-guide-process-engine"

---


# 流程定义的版本管理

业务流程本质上是需要长期运行的。流程实例可能会持续数周，或数月。在这期间，流程实例的状态被存储到数据库中。但总有那么一天，你需要在仍有运行中的实例的情况下改变流程定义。

流程引擎支持这一点：

* 如果你重新部署一个改变了的流程定义，你会在数据库中得到一个新的版本。
* 正在运行的流程实例将继续在它所在的旧版本执行。
* 新的流程实例将在新的版本中运行 - 除非明确指定执行版本。
* 在一定范围内支持将流程实例迁移到新版本。

你可以在流程定义表中看到不同的版本，以及流程实例与版本的关联。

{{< img src="../img/versioning.png" title="Versioning" >}}

{{< note title="多租户" class="info" >}}
如果你正在使用[以tenantID区分的多租户]({{< ref "/user-guide/process-engine/multi-tenancy.md#single-process-engine with-tenant-identifiers" >}})，那么每个租户都有自己的流程定义，其版本与其他租户无关。参见[多租户部分]({{< ref "/user-guide/process-engine/multi-tenancy.md#versioning-of-tenant-specific-definitions" >}})。
{{< /note >}}


# 流程实例会使用哪个版本

当你：

* 通过 **key** 启动一个流程实例时。它启动一个 **最新部署版本** 的流程定义的实例。
 *通过 **id** 启动一个流程实例时。它以数据库ID启动已部署的流程定义的一个实例。通过使用它，你可以启动一个 **特定版本** 的流程。

默认和推荐的用法是使用 `startProcessInstanceByKey` ，它总是使用最新的版本。

```java
processEngine.getRuntimeService().startProcessInstanceByKey("invoice");
// will use the latest version (2 in our example)
```

如果你想专门启动一个旧版本的流程实例，可以在流程定义中查询来找到正确的流程定义id，然后使用`startProcessInstanceById`。

```java
ProcessDefinition pd = processEngine.getRepositoryService().createProcessDefinitionQuery()
    .processDefinitionKey("invoice")
    .processDefinitionVersion(1).singleResult();
processEngine.getRuntimeService().startProcessInstanceById(pd.getId());
```

当你使用 [BPMN CallActivities]({{< ref "/reference/bpmn20/subprocesses/call-activity.md" >}}) 时，你可以配置使用哪个版本:

```xml
<callActivity id="callSubProcess" calledElement="checkCreditProcess"
  camunda:calledElementBinding="latest|deployment|version"
  camunda:calledElementVersion="17">
</callActivity>
```
or
```xml
<callActivity id="callSubProcess" calledElement="checkCreditProcess"
  camunda:calledElementBinding="versionTag"
  camunda:calledElementVersionTag="ver-tag-1.0.1">
</callActivity>
```

这些选项的意义是：

* latest: 使用最新版本的流程定义 (类似于 `startProcessInstanceByKey`).
* deployment: 使用与调用流程的版本相匹配版本中的流程定义。如果它们被一起部署，也是可以的 -- 因为它们随后总是会一起被版本化（参见[流程应用部署]({{< ref "/user-guide/process-applications/the-processes-xml-deployment-descriptor.md#deployment-descriptor-process-application-deployment" >}})以了解更多细节）。
* version: 在XML中指定硬编码的版本。
* versionTag: 在XML中指定硬编码的版本标记。


# 流程定义的Key vs Id

你可能已经发现，在流程定义表中存在两个不同的属性，具有不同的含义：

* key：key是XML中流程定义的唯一标识符，所以它的值是从XML中的id属性读取的。

    ```xml
    <bpmn2:process id="invoice" ...
    ```

* Id：是数据库的主键和通常是由key、版本和生成的id组合而成的（注意，ID可能被缩短以适应数据库列，所以不能保证所有id都是这样建立的）。

# 版本标签

可以用版本标签属性来标记流程定义。方法是，添加[camunda:versionTag]({{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#versiontag" >}})扩展属性到流程中：

```xml
<bpmn2:process camunda:versionTag="1.5-patch2" ..
```

在 `ProcessDefinition` 中你可以获取的versionTag字段：

```java
ProcessDefinition pd = processEngine.getRepositoryService().createProcessDefinitionQuery()
    .processDefinitionKey("invoice")
    .processDefinitionVersion(1).singleResult();

pd.getVersionTag();
```

或获取包含指定版本的所有已部署流程定义的列表：

```java
List<ProcessDefinition> pdList = processEngine.getRepositoryService().createProcessDefinitionQuery()
    .versionTag("1.5-patch2")
    .list();

```

你也可以使用 `versionTagLike` 来查询一系列的版本：

```java
List<ProcessDefinition> pdList = processEngine.getRepositoryService().createProcessDefinitionQuery()
    .versionTagLike("1.5-%")
    .list();
```

下面的例子显示了如何启动一个版本标记的最新流程定义的流程实例：

```java
ProcessDefinition pd = processEngine.getRepositoryService().createProcessDefinitionQuery()
    .processDefinitionKey("invoice")
    .versionTag("1.5-patch2")
    .orderByVersion().
    .desc()
    .listPage(0,1);

processEngine.getRuntimeService().startProcessInstanceById(pd.getId());
```

{{< note title="版本标签" class="info" >}}
版本标签仅用于标记，既不会影响 "startProcessInstanceByKey"也不会影响 "startProcessInstanceById"行为。
{{< /note >}}

{{< note title="最新版本" class="info" >}}
流程定义`version`和`versionTag`是独立的属性。当使用`ProcessDefinitionQuery#latestVersion()`进行查询时，将为给定的键找到具有最大`version`数字的流程定义。如果最新的流程定义不包含所查询的版本标签，向该查询添加一个版本标签过滤器可能会提供一个空结果。
{{< /note >}}

# 流程实例迁移

默认情况下，当部署新的流程版本时，运行在以前版本上的流程实例不受影响。如果想将以前版本的流程迁移到新版本，可以参考[流程实例迁移]({{< ref "/user-guide/process-engine/process-instance-migration.md" >}})。
