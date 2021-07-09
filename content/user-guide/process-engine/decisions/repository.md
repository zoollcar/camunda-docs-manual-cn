---

title: '流程引擎存储库中的决策'
weight: 20

menu:
  main:
    name: "储存库"
    identifier: "user-guide-process-engine-decisions-repository"
    parent: "user-guide-process-engine-decisions"
    pre: "Deploy and Manage Decisions"

---

要进行 Camunda 中的 DMN 决策，它必须包含在 [部署][Deployment] 中。部署决策后，可以通过其key和版本来引用它。 该平台支持 [DMN 1.3][DMN 1.3] XML 文件。

# 部署决策

要部署 DMN 决策，你可以使用存储库服务或将其添加到流程应用程序。 平台会将所有类似`.dmn` 或`.dmn11.xml`的文件识别为 DMN 资源。

## 使用存储库服务（Repository Service）部署决策

可以使用存储库服务创建新部署并向其添加 DMN 资源。 例如，以下代码将在类路径中为 DMN 文件创建一个新部署。

```java
String resourceName = "MyDecision.dmn11.xml";
Deploymnet deployment = processEngine
  .getRepositoryService()
  .createDeployment()
  .addClasspathResource(resourceName)
  .deploy();
```

## 使用流程应用程序部署决策

如果你部署 [流程应用程序][Process Application]，你可以将 DMN 文件作为额外资源添加到你的档案（例如，BPMN 流程）中。 DMN 文件必须具有“.dmn”或“.dmn11.xml”文件扩展名才能被识别为 DMN 资源。

如果你的 [流程档案][Process Archive] 设置为自动扫描，它也会自动部署 DMN 定义。 这是默认设置。

```xml
<process-archive name="loan-approval">
  <properties>
    <property name="isScanForProcessDefinitions">true</property>
  </properties>
</process-archive>
```

否则，你必须在 [流程档案][Process Archive] 中明确指定 DMN 资源

```xml
<process-archive name="loan-approval">
  <resource>bpmn/invoice.bpmn</resource>
  <resource>dmn/assign-approver.dmn</resource>
  <properties>
    <property name="isScanForProcessDefinitions">false</property>
  </properties>
</process-archive>
```

# 决策的版本

当 DMN 资源部署到平台时，每个受支持的 DMN 决策都会转换为 *决策定义* 。 决策定义代表平台中的单个 DMN 决策。 除其他外，它具有以下属性：

- `id`: 平台生成的决策定义的唯一标识符。
- `key`: XML 文件中的 DMN 决策 `id` 属性。
- `name`: XML 文件中的 DMN 决策 `name` 属性。
- `version`: 平台生成的 DMN 决策定义版本。

## 决策定义Key

决策定义Key 相当于 DMN XML 中 DMN 决策的“id”属性。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="https://www.omg.org/spec/DMN/20191111/MODEL/" id="definitions" name="definitions" namespace="http://camunda.org/schema/1.0/dmn">
  <decision id="my-decision" name="My Decision">
    <decisionTable>
      <output id="output1"/>
    </decisionTable>
  </decision>
</definitions>
```

部署上述 DMN XML 文件时，将创建具有以下属性的决策定义：

- `id`: GENERATED
- `key`: `my-decision`
- `name`: `My Decision`
- `version`: 1

## 决策定义版本

部署决策时，会检查是否已部署具有相同Key的定义。
如果不是，则为该键分配决策定义版本“1”。 如果已存在具有相同键的决策定义，则新部署的决策定义将成为现有决策定义的新版本，并将其版本增加1。

这种决策定义的版本控制允许用户更新决策，但如果需要，仍然能够使用以前的决策版本。

## 决策定义 ID

决策定义的id **不等同于** DMN XML 决策的 `id` 属性。 它由平台作为唯一标识符生成。 这意味着决策定义 id 直接对应于决策定义Key和版本组合。

## 参考决策定义

要在平台上下文中引用已部署的决策定义，可以使用决策定义ID 或决策定义Key和版本。 如果使用决策定义Key但未指定版本，则默认使用决策定义的最新版本。

# 决策需求图的版本控制

除了决策定义之外，已部署 DMN 的 [决策需求图]({{< ref "/reference/dmn/drg/_index.md" >}})（即 XML 中的“definitions”元素） 资源转换为 *决策需求定义* 。 除其他外，它具有以下属性：

- `id`: 平台生成的决策需求定义的唯一标识符。
- `key`: XML 文件中的定义`id` 属性。
- `name`: XML 文件中的定义`name` 属性。
- `version`: 平台生成的决策需求定义的版本。

请注意，仅当 DMN 资源包含多个决策时才创建决策需求定义。

## 决策需求定义Key

决策需求定义Key等效于 DMN XML 中“definitions”元素的“id”属性。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="https://www.omg.org/spec/DMN/20191111/MODEL/" 
             id="my-drg" 
             name="My DRG" 
             namespace="http://camunda.org/schema/1.0/dmn">
  <!-- ... -->
</definitions>
```

部署上述 DMN XML 文件时，将创建具有以下属性的决策需求定义：

- `id`: GENERATED
- `key`: `my-drg`
- `name`: `My DRG`
- `version`: 1

## 决策需求定义版本

当部署决策需求图时，会检查是否已经部署了具有相同Key的定义。
如果不是，则为该密钥分配决策需求定义版本“1”。 如果已存在具有相同键的决策需求定义，则新部署的定义将成为现有定义的新版本，并将其版本增加1。

请注意，如果所包含的决策定义的版本也部署在其他 DMN 资源内，作为 DMN 资源内的单个决策，或稍后添加，则包含的决策定义的版本可能与决策需求定义不同。

# 查询决策库

所有部署的决策定义和决策需求定义都可以通过存储库服务 API 进行查询。

## 查询决策定义

获取 ID 为“decisionDefinitionId”的决策定义：

```java
DecisionDefinition decisionDefinition = processEngine
  .getRepositoryService()
  .getDecisionDefinition("decisionDefinitionId");
```

使用Key“decisionDefinitionKey”和版本 1 查询决策定义：

```java
DecisionDefinition decisionDefinition = processEngine
  .getRepositoryService()
  .createDecisionDefinitionQuery()
  .decisionDefinitionKey("decisionDefinitionKey")
  .decisionDefinitionVersion(1)
  .singleResult();
```

使用关键字“decisionDefinitionKey”查询最新版本的决策定义：

```java
DecisionDefinition decisionDefinition = processEngine
  .getRepositoryService()
  .createDecisionDefinitionQuery()
  .decisionDefinitionKey("decisionDefinitionKey")
  .latestVersion()
  .singleResult();
```

使用关键字“decisionDefinitionKey”查询所有版本的决策定义：

```java
List<DecisionDefinition> decisionDefinitions = processEngine
  .getRepositoryService()
  .createDecisionDefinitionQuery()
  .decisionDefinitionKey("decisionDefinitionKey")
  .list();
```

此外，存储库服务可用于获取 DMN XML 文件、DMN 模型实例或部署的图解图像.

```java
RepositoryService repositoryService = processEngine.getRepositoryService();

DmnModelInstance dmnModelInstance = repositoryService
  .getDmnModelInstance("decisionDefinitionId");

InputStream modelInputStream = repositoryService
  .getDecisionModel("decisionDefinitionId");

InputStream diagramInputStream = repositoryService
  .getDecisionDiagram("decisionDefinitionId");
```

## 查询决策需求定义

可以以与决策定义类似的方式查询决策需求定义。

```java
// 按关键字查询最新版本的决策需求定义
DecisionRequirementsDefinition decisionRequirementsDefinition = processEngine
  .getRepositoryService()
  .createDecisionRequirementsDefinitionQuery()
  .decisionRequirementsDefinitionKey(key)
  .latestVersion()
  .singleResult();
  
// 按名称查询所有版本的决策需求定义
List<DecisionRequirementsDefinition> decisionRequirementsDefinitions = processEngine
  .getRepositoryService()
  .createDecisionRequirementsDefinitionQuery()
  .decisionRequirementsDefinitionName(name)
  .list();  
``` 

# 查询决策库的授权

用户需要对资源“DECISION_DEFINITION”的“READ”权限才能查询决策定义。 从存储库中检索决策定义、决策模型和决策图也需要此权限。 授权的资源 id 是决策定义Key。

要查询决策需求定义，用户需要资源“DECISION_REQUIREMENTS_DEFINITION”的“READ”权限。 授权的资源 id 是决策需求定义的Key。

有关授权的更多信息，请参阅[授权服务]({{< ref "/user-guide/process-engine/authorization-service.md" >}}) 部分。

# Cockpit

可以在 [Cockpit][Cockpit] 网络应用程序中查看已部署的决策定义。

[Deployment]: {{< ref "/user-guide/process-engine/deployments.md" >}}
[DMN 1.3]: {{< ref "/reference/dmn/_index.md" >}}
[Process Application]: {{< ref "/user-guide/process-applications/_index.md" >}}
[Process Archive]: {{< ref "/reference/deployment-descriptors/tags/process-archive.md" >}}
[Cockpit]: {{< ref "/webapps/cockpit/dmn/_index.md" >}}

