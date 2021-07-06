---

title: 'Authorization Service'
weight: 240

menu:
  main:
    identifier: "user-guide-process-engine-authorization-service"
    parent: "user-guide-process-engine"

---

Camunda 允许用户授权访问其管理的数据。 这使得配置哪些用户可以访问哪些流程实例、任务等成为可能...

授权具有一定性能成本并引入了一些复杂性。 它应该只在需要时使用。

# 什么时候需要授权？

并非每个 Camunda 设置都需要启用授权。 在许多场景中，Camunda 被嵌入到应用程序中，应用程序本身确保用户只能访问他们有权访问的数据。 一般来说，只有不受信任方直接与流程引擎API交互时才需要授权。 如果将流程引擎嵌入到 Java 应用程序中，通常是不需要启用授权的。 应用程序可以自行控制 API 的访问方式。

需要授权的情况：

* Camunda Rest API 可供不应具有完全访问权限的用户访问，即使通过身份验证也是如此。
* Camunda Webapplication 可供不应具有完全访问权限的用户访问，即使通过身份验证也是如此。
* 其他不受信任的用户可以直接构造在流程引擎上执行的查询和命令的情况。

*不需要* 授权的情况：

* 应用程序完全控制在流程引擎上调用的 API 方法。
* Camunda Web 应用程序可供在身份验证后具有完全访问权限的用户访问。

**例子**:

假设你有以下授权要求：*作为普通用户，我只能看到分配给我的任务。*

如果引擎嵌入到 Java 应用程序中，应用程序可以通过限制对“assignee”属性的任务查询来轻松做到这一点。应用程序可以保证限制，因为 Camunda API 不会直接暴露给用户。

相比之下，如果 Camunda Rest API 通过网络直接暴露给 Javascript 应用程序，那么恶意用户一旦通过身份验证，就可以向服务器发送请求，查询所有任务，甚至是未分配给该用户的任务。在这种情况下，需要开启授权以确保不管查询参数如何，用户只看到他被授权看到的任务。

# 基本原则

## 授权

授权给某身份一组权限，使其可以与限定的资源进行交互。

**例子**

* 用户 'jonny' 被授权创建新用户。
* 组'marketing'无权删除组'sales'。
* 组'marketing'不允许使用tasklist应用程序。

## 身份

Camunda 平台区分两种类型的身份：用户和组。授权范围可以是所有用户（userId = ANY）、单个用户或一组用户。


## 权限

例如，没有流程实例资源的“更新”权限和批资源的“创建”权限的用户可以通过创建批处理来异步修改多个流程实例，尽管他无法同步执行此操作。

权限定义了允许身份与特定资源交互的方式。

引擎中可用的基本权限是：

* None 无
* All 全部
* Read 读
* Update 更新
* Create 创造
* Delete 删除
* Access 使用权

请注意，权限“None”并不意味着不授予任何权限，它代表“无操作”。
此外，如果单个权限被撤销，“All”权限将从用户中去除。

有关可用权限的详细列表，请查看 [按资源授权]({{< relref "#按资源授权" >}}) 部分。

单个授权对象可以为单个用户和资源分配多个权限：

```java
authorization.addPermission(Permissions.READ);
authorization.addPermission(Permissions.UPDATE);
authorization.addPermission(Permissions.DELETE);
```

## 资源

资源是用户交互的实体。

有以下资源可用：

<table class="table matrix-table table-condensed table-hover table-bordered">
  <tr>
    <th>资源名</th>
    <th>整数表示</th>
    <th>资源 Id</th>
  </tr>
  <tr>
    <td>Application (Cockpit, Tasklist, ...)</td>
    <td>0</td>
    <td>admin/cockpit/tasklist/*</td>
  </tr>
  <tr>
    <td>Authorization</td>
    <td>4</td>
    <td>Authorization Id</td>
  </tr>
  <tr>
    <td>Batch</td>
    <td>13</td>
    <td>Batch Id</td>
  </tr>
  <tr>
    <td>Decision Definition</td>
    <td>10</td>
    <td>Decision Definition Key</td>
  </tr>
  <tr>
    <td>Decision Requirements Definition</td>
    <td>14</td>
    <td>Decision Requirements Definition Key</td>
  </tr>
  <tr>
    <td>Deployment</td>
    <td>9</td>
    <td>Deployment Id</td>
  </tr>
  <tr>
    <td>Filter</td>
    <td>5</td>
    <td>Filter Id</td>
  </tr>
  <tr>
    <td>Group</td>
    <td>2</td>
    <td>Group Id</td>
  </tr>
  <tr>
    <td>Group Membership</td>
    <td>3</td>
    <td>Group Id</td>
  </tr>
  <tr>
    <td>Process Definition</td>
    <td>6</td>
    <td>Process Definition Key</td>
  </tr>
  <tr>
    <td>Process Instance</td>
    <td>8</td>
    <td>Process Instance Id</td>
  </tr>
  <tr>
    <td>Task</td>
    <td>7</td>
    <td>Task Id</td>
  </tr>
  <tr>
    <td>Historic Task</td>
    <td>19</td>
    <td>Historic Task Id</td>
  </tr>
  <tr>
    <td>Historic Process Instance</td>
    <td>20</td>
    <td>Historic Process Instance Id</td>
  </tr>
  <tr>
    <td>Tenant</td>
    <td>11</td>
    <td>Tenant Id</td>
  </tr>
  <tr>
    <td>Tenant Membership</td>
    <td>12</td>
    <td>Tenant Id</td>
  </tr>
  <tr>
    <td>User</td>
    <td>1</td>
    <td>User Id</td>
  </tr>
  <tr>
    <td>Report</td>
    <td>15</td>
    <td>Report Id</td>
  </tr>
  <tr>
    <td>Dashboard</td>
    <td>16</td>
    <td>Dashboard Id</td>
  </tr>
  <tr>
    <td>User Operation Log Category</td>
    <td>17</td>
    <td>User Operation Log Entry Category</td>
  </tr>
</table>

**注意：** 当你仅使用 CREATE 权限创建新授权时，资源 ID 应为“*”。


## 授权类型

授权分为三种：

<table class="table matrix-table table-condensed table-hover table-bordered">
  <tr>
    <th>授权类型</th>
    <th>描述</th>
    <th>整数表示</th>
  </tr>
  <tr>
    <td>Global Authorization (<code>AUTH_TYPE_GLOBAL</code>)</td>
    <td>范围涵盖所有用户和组 (<code>userId = ANY</code>)，通常用于固定资源的“基本”权限。</td>
    <td>0</td>
  </tr>
  <tr>
    <td>Grant Authorization (<code>AUTH_TYPE_GRANT</code>)</td>
    <td>横跨用户和组授予一组权限。 通常用于向全局授权撤销的用户或组添加权限。</td>
    <td>1</td>
  </tr>
  <tr>
    <td>Revoke Authorization (<code>AUTH_TYPE_REVOKE</code>)</td>
    <td>横跨用户和组撤销一组权限。 通常用于撤销全局授权授予的用户或组的权限。</td>
    <td>2</td>
  </tr>  
</table>

{{< note class="warning" title="撤销授权的执行" >}}
参阅本页上的 [性能注意事项]({{< relref "#性能注意事项" >}}) 部分。
{{< /note >}}

## 授权优先级

授权可能涵盖所有用户、单个用户或一组用户，或者它们可能适用于单个资源实例或相同类型的所有实例 (resourceId = ANY)。 优先级如下：

* 适用于单个资源实例的授权优先于适用于同一资源类型的所有实例的授权。
* 对个人用户的授权优先于对组的授权。
* 组授权优先于全体授权。
* 组授予授权优先于组回收权限。
* 用户授予授权优先于用户回收权限。

## 何时检查授权？

以下情况，会检查授权：

* 配置项 `authorizationEnabled` 设置为 `true`（默认值为 `false`）。
* 当前有一个通过身份验证的用户。

最后一项意味着即使启用了授权，也只有在用户当前通过身份验证时才执行授权检查。
如果用户没有通过身份验证，则引擎不会执行任何检查。

使用 Camunda Webapps 时，需要始终确保用户在访问任何受限资源之前已通过身份验证。
将流程引擎嵌入到自定义应用程序中时，如果需要执行授权检查，应用程序需要处理身份验证。

{{< note class="info" title="身份验证与授权" >}}
身份验证和授权是两个不同的概念，详情见[此处](https://en.wikipedia.org/wiki/Authentication#Authorization)叙述。
{{< /note >}}

# 按资源授权

本节叙述哪些权限可用于哪些资源。

## 读取、更新、创建、删除

大多数资源都具有读取、更新、创建和删除权限。
下表列出了可用的资源：

<table class="table matrix-table table-condensed table-hover table-bordered">
<thead>
  <tr>
    <th></th>
    <th>读取</th>
    <th>更新</th>
    <th>创建</th>
    <th>删除</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>授权 Authorization</th>
      <td>X</td>
      <td>X</td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>批处理 Batch</th>
      <td>X</td>
      <td>X</td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>决策定义 Decision Definition</th>
      <td>X</td>
      <td>X</td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <th>决策需求定义 Decision Requirements Definition</th>
      <td>X</td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <th>部署 Deployment</th>
      <td>X</td>
      <td></td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>过滤器 Filter </th>
      <td>X</td>
      <td>X</td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>组 Group </th>
      <td>X</td>
      <td>X</td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>组关系 Group Membership </th>
      <td></td>
      <td></td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>流程定义 Process Definition </th>
      <td>X</td>
      <td>X</td>
      <td></td>
      <td>X</td>
    </tr>
    <tr>
      <th>流程实例 Process Instance </th>
      <td>X</td>
      <td>X</td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>任务 Task </th>
      <td>X</td>
      <td>X</td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>历史任务 Historic Task </th>
      <td>X</td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <th>历史流程实例 Historic Process Instance </th>
      <td>X</td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <th>租户 Tenant </th>
      <td>X</td>
      <td>X</td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>租户关系 Tenant Membership </th>
      <td></td>
      <td></td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>用户 User </th>
      <td>X</td>
      <td>X</td>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>用户操作日志类别 User Operation Log Category </th>
      <td>X</td>
      <td>X</td>
      <td></td>
      <td>X</td>
    </tr>
  </tbody>
</table>

要[异步]({{< ref "/user-guide/process-engine/batch.md">}}) 执行操作，只需要批处理资源的“创建”权限。 但是，在同步执行相同操作时，会检查特定权限（例如对流程实例资源的“删除”）。

例如，没有流程实例资源的“更新”权限但具有批资源的“创建”权限的用户可以通过创建批处理来异步修改多个流程实例，尽管他无法同步执行此操作。

## 任务资源特有权限

除了更新、读取和删除之外，任务资源还具有以下权限：

* 任务分配
* 任务工作
* 更新变量

用户可以对任务执行不同的操作，例如分配任务、声明任务或完成任务。
如果用户对任务具有“更新”权限（或对相应流程定义具有“更新任务”权限），则该用户有权执行 _所有_ 这些任务操作。
如果需要更细粒度的授权，可以使用 “任务项（Task Work）” 和 “任务指派（Task Assign）" 权限。
“Task Work”背后的直觉是它只授权用户_work_完成一项任务（即声明并完成它），而不是将其分配给另一个用户或以另一种方式“分配工作”给同事。

如果用户对任务具有“更新变量”权限（或对相应流程定义具有“更新任务变量”权限），则该用户有权执行设置/删除任务变量操作。

下表详细概述了哪些权限授权用户执行哪些任务操作：

<table class="table matrix-table table-condensed table-hover table-bordered">
<thead>
  <tr>
    <th></th>
    <th>任务项（Task Work）</th>
    <th>任务指派（Task Assign）</th>
    <th>更新变量</th>
    <th>更新</th>
    </tr>
  </thead>
  <tbody>
     <tr>
      <th>声明 Claim</th>
      <td>X</td>
      <td></td>
      <td></td>
      <td>X</td>
    </tr>
    <tr>
      <th>完成 Complete</th>
      <td>X</td>
      <td></td>
      <td></td>
      <td>X</td>
    </tr>
    <tr>
      <th>增加候选用户 Add Candidate User</th>
      <td></td>
      <td>X</td>
      <td></td>
      <td>X</td>
    </tr>
    <tr>
      <th>删除候选用户 Delete Candidate User</th>
      <td></td>
      <td>X</td>
      <td></td>
      <td>X</td>
    </tr>
    <tr>
      <th>设置受让人 Set Assignee</th>
      <td></td>
      <td>X</td>
      <td></td>
      <td>X</td>
    </tr>
    <tr>
      <th>设置拥有者 Set Owner</th>
      <td></td>
      <td>X</td>
      <td></td>
      <td>X</td>
    </tr>
    <tr>
      <th>增加候选组 Add Candidate Group</th>
      <td></td>
      <td>X</td>
      <td></td>
      <td>X</td>
    </tr>
    <tr>
      <th>删除候选组 Delete Candidate Group</th>
      <td></td>
      <td>X</td>
      <td></td>
      <td>X</td>
    </tr>
    <tr>
      <th>保存任务 Save Task</th>
      <td></td>
      <td>X</td>
      <td></td>
      <td>X</td>
    </tr>
    <tr>
      <th>设置任务优先级 Set Task Priority</th>
      <td></td>
      <td>X</td>
      <td></td>
      <td>X</td>
    </tr>
	<tr>
      <th>设置任务变量 Set Task Variable</th>
      <td></td>
      <td></td>
      <td>X</td>
      <td>X</td>
    </tr>
	<tr>
      <th>删除任务变量 Remove Task Variable</th>
      <td></td>
      <td></td>
      <td>X</td>
      <td>X</td>
    </tr>
  </tbody>
</table>

任务项、任务指派和更新变量权限的授予和回收权限的优先级优先于更新和更新任务。

### 默认任务权限

当用户是受让人、所有者或属于候选用户、候选组时，用户将根据配置获得的默认权限为“任务项（Task Work）”或“更新” “defaultUserPermissionNameForTask”。

如果未设置“defaultUserPermissionNameForTask”，则默认授予“更新”权限.

## 流程定义特有权限

除了更新、读取和删除之外，流程定义资源还提供以下权限：

* 读取任务 Read Task
* 更新任务 Update Task
* 任务项 Task Work
* 任务分配 Task Assign
* 创建实例 Create Instance
* 读取实例 Read Instance
* 更新实例 Update Instance
* 重试Job Retry Job
* 暂停 Suspend
* 暂停实例 Suspend Instance
* 更新实例变量 Update Instance Variable
* 更新任务变量 Update Task Variable
* 迁移实例 Migrate Instance
* 删除实例 Delete Instance
* 读取历史 Read History
* 删除历史 Delete History
* 更新历史 Update History

启动新流程实例需要“创建实例”权限。

{{< note title="启动一个新实例" class="info" >}}
  要执行该操作，用户还需要对流程实例资源具有“创建”权限。

{{< /note >}}

重试Job、暂停、暂停实例、更新实例变量和更新任务变量权限的授予和回收授权优先于更新。
请记住，被允许执行变量更新的用户可以通过更新变量来触发流程中的其他更改。 例如，成功评估与变量相关的条件事件。 

## 流程实例特有权限

除了创建、读取、更新和删除之外，流程实例资源还具有以下权限：

* 重试Job Retry Job
* 暂停 Suspend
* 更新变量 Update Variable

重试Job、暂停和更新变量权限的授予和回收授权优先于更新。
请记住，被允许执行变量更新的用户可以通过更新变量来触发流程中的其他更改。 例如，成功评估与此变量相关的条件事件。

## 决策定义权限

除了更新、读取和删除之外，决策定义资源还提供以下权限：

* 创建实例 Create Instance
* 阅读历史 Read History
* 删除历史 Delete History

使用决策服务评估决策需要“创建实例”权限。

## 批处理特有权限

除了创建、更新、读取和删除之外，批处理资源还具有以下权限：

* 阅读历史 Read History
* 删除历史 Delete History
* 创建批量迁移流程实例 Create Batch Migrate Process Instances
* 创建批量修改流程实例 Create Batch Modify Process Instances
* 创建批量重启流程实例 Create Batch Restart Process Instances
* 创建批量删除正在运行的流程实例 Create Batch Delete Running Process Instances
* 创建批量删除完成的流程实例 Create Batch Delete Finished Process Instances
* 创建批量删除决策实例 Create Batch Delete Decision Instances
* 创建批处理Job重试 Create Batch Set Job Retries
* 创建批处理外部任务重试 Create Batch Set External Task Retries
* 创建批量更新流程实例暂停 Create Batch Update Process Instances Suspend
* 创建批次集移除时间 Create Batch Set Removal Time
* 创建批处理集变量 Create Batch Set Variables

特定的“创建某某某”权限比一般的“创建”权限具有更高的优先级。


## 默认读取变量权限
当在流程引擎配置中启用`enforceSpecificVariablePermission`时，为了读取变量，用户需要被授予以下权限：

如果是 任务（Tasks）

* 读取变量（用于进程和独立任务）

如果是 历史任务

* 读取变量（仅在启用[历史实例权限](#historic-instance-permissions)时强制执行）

如果是 流程定义

* 读取实例变量（用于运行时流程实例变量）
* 读取历史变量（用于历史变量）
* 读取任务变量（用于运行时任务变量）

## 应用程序(Application)权限

“应用程序(Application)”资源只支持“访问”权限。
访问权限确定用户是否可以访问 Camunda webapplication。 开箱即用，可以授予以下应用程序（资源 ID）：

* `admin`
* `cockpit`
* `tasklist`
* `optimize`
* `*` (Any / All)

## 用户操作日志权限

“用户操作日志类型（User Operation Log Category）”资源确定用户能否访问指定类型的用户操作日志条目。
默认情况下，可以授予以下类别（资源 ID（resource ids））：

* `TaskWorker`
* `Admin`
* `Operator`
* `*` (Any / All)

## 历史实例权限

确定用户能否读取与特定实例相关的历史记录。

与运行时权限相比，历史权限不会在相关实例完成后立即删除。 而是会根据[基于Removal-Time的历史清理策略][Removal-Time-based History Cleanup Strategy] 删除历史权限。

你可以使用[流程引擎配置标志][hist-inst-perm-config-flag]启用权限：

```xml
<property name="enableHistoricInstancePermissions">true</property>
```

由于两个原因，默认情况下禁用该功能：

1. 启用后，SQL 查询会更加复杂，因为会执行额外的授权检查。
    更复杂的查询可能会降低性能。
2. 当启用并将身份链接添加到任务时，相应的用户或组被授权读取关联的历史记录（例如，对于任务、变量或身份链接历史记录）。
    在 Camunda 版本 <= 7.12 时，在这种情况下无法读取历史记录。

### 历史任务（Tasks）权限

授予历史任务权限后，你可以使用以下查询来检索与历史任务相关的实体：

* 历史任务实例查询 Historic Task Instance Query
* 历史变量实例查询 Historic Variable Instance Query
* 历史详情查询 Historic Detail Query
* 身份链接日志查询 Identity Link Log Query
* 用户操作日志查询 User Operation Log Query

### 历史流程实例权限

当对历史流程实例授予授权时，你可以使用以下查询来检索与历史流程实例相关的实体：

* 历史流程实例查询 Historic Process Instance Query
* 历史活动实例查询 Historic Activity Instance Query
* 历史任务实例查询 Historic Task Instance Query
* 历史变量实例查询 Historic Variable Instance Query
* 历史详情查询 Historic Detail Query
* 身份链接日志查询 Identity Link Log Query
* 历史事件查询 Historic Incident Query
* 作业日志查询 Job Log Query
* 外部任务日志查询 External Task Log Query
* 用户操作日志查询 User Operation Log Query

# 管理员

Camunda 平台没有明确的“管理员”概念，它可以使用授予了对所有资源的所有授权的用户模拟。

## "camunda-admin" 组

下载 Camunda平台发行版时，invoice示例应用程序会创建一个 ID 为“camunda-admin”的组，并将所有资源的所有授权授予该组。

在没有演示应用程序的情况下，此任务由 [Camunda Admin Web Application]({{< ref "/webapps/admin/user-management.md#initial-user-setup" >}}) 执行。 如果第一次启动 Camunda webapplication 并且数据库中不存在用户，它会要求你执行“初始设置”。 在这个过程中，`camunda-admin` 组将被创建并被授予对所有资源的所有权限。

{{< note title="LDAP" class="info" >}}
使用 LDAP 时不会创建组“camunda-admin”（因为 LDAP 只能以只读方式访问）。 另请参阅以下有关管理员授权插件的部分。
{{< /note >}}

## 管理员授权插件

管理员授权插件是一个流程引擎插件，具有以下功能：当流程引擎启动时，它会向配置的组或用户授予管理访问权限。 这意味着它将所有资源的所有权限授予已配置的组或用户。

这通常用于引导 LDAP 安装：向初始用户授予管理访问权限，然后该用户可以登录到 Admin 并使用 UI 配置其他授权。

下面案例说明如何在 bpm-platform.xml / processes.xml 中配置管理员授权插件：

```xml
<process-engine name="default">
  ...
  <plugins>
    <plugin>
      <class>org.camunda.bpm.engine.impl.plugin.AdministratorAuthorizationPlugin</class>
      <properties>
        <property name="administratorUserName">admin</property>
      </properties>
    </plugin>
  </plugins>
</process-engine>
```

该插件将确保在流程引擎启动时对所有资源授予管理员授权（所有权限）。

{{< note title="" class="info" >}}
  无需配置所有应具有管理员权限的 LDAP 用户和组。 通常配置单个用户并使用该用户登录 Web 应用程序并使用用户界面创建其他授权就足够了。
{{< /note >}}

配置属性的完整列表：

<table class="table table-striped">
  <tr>
    <th>属性</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>administratorUserName</code></td>
    <td>管理员用户的名称。 如果此名称设置为非空和非空值，插件将在所有内置资源上创建用户级管理员授权。</td>
  </tr>
  <tr>
    <td><code>administratorGroupName</code></td>
    <td>管理员组的名称。 如果此名称设置为非空和非空值，插件将在所有内置资源上创建组级管理员授权。</td>
  </tr>
</table>

# 配置选项

本节解释了与授权相关的可用流程引擎配置选项。

## 启用授权检查

可以使用配置选项“authorizationEnabled”全局启用或禁用授权检查。 此配置选项的默认设置为 `false`。

## 启用用户代码的授权检查

配置选项“authorizationEnabledForCustomCode”控制是否对委托代码（即 Java委托）执行的命令执行授权检查。 此配置选项的默认设置为 `false`。

## 检查撤销授权

配置选项“authorizationCheckRevokes”控制授权检查是否考虑“Revoke”类型的授权。

可用值为：

* `always`：始终启用对撤销授权的检查。这种模式等于 &lt; 7.5 行为。 *注意：*对于具有高潜在基数的资源（如任务或流程实例），检查撤销授权非常昂贵，并且可以使对流程引擎的授权访问在大多数数据库上有效地无法使用。因此，强烈建议你不要使用此模式。

* `never`：从不检查撤销授权。此模式具有最佳性能并有效禁用撤销授权的使用。 *注意*：强烈建议使用此模式。

* `auto`（**默认值**）：如果当前用户或用户所属的组中至少存在一个撤销授权，则此模式仅检查撤销授权。为了实现这一点，每个命令检查一次是否存在潜在适用的撤销授权。根据结果​​，授权检查然后使用撤销与否。 *注意：*对于具有高潜在基数的资源（如任务或流程实例），检查撤销授权非常昂贵，并且可以使对流程引擎的授权访问在大多数数据库上有效地无法使用。

另请参阅此页面上的 [性能注意事项]({{< relref "#performance- Considerations" >}}) 部分。

# Java API 案例

在用户/组和资源之间创建授权。 它描述了用户/组访问该资源的权限。 一个授权可以表达不同的权限，例如读取、更新、删除资源的权限。 （有关详细信息，请参阅授权）。

为了授予访问某个资源的权限，需要创建一个授权对象。 例如，要访问某个过滤器：

```java
Authorization auth = authorizationService.createNewAuthorization(AUTH_TYPE_GRANT);

// The authorization object can be configured either for a user or a group:
auth.setUserId("john");
//  -OR-
auth.setGroupId("management");

//and a resource:
auth.setResource("filter");
auth.setResourceId("2313");
// a resource can also be a process definition
auth.setResource(Resources.PROCESS_INSTANCE);
// the process defintion key is the resource id
auth.setResourceId("invoice");

// finally the permissions to access that resource can be assigned:
auth.addPermission(Permissions.READ);
// more than one permission can be granted
auth.addPermission(Permissions.CREATE);

// and the authorization object is saved:
authorizationService.saveAuthorization(auth);
```

因此，给定的用户或组将有权读取引用的过滤器。

另一个可能的示例是限制允许启动特定流程的人员组：

```java
//我们需要授权，一个访问流程定义，另一个创建流程实例
Authorization authProcessDefinition = authorizationService.createNewAuthorization(AUTH_TYPE_GRANT);
Authorization authProcessInstance = authorizationService.createNewAuthorization(AUTH_TYPE_GRANT);

authProcessDefinition.setUserId("johnny");
authProcessInstance.setUserId("johnny");

authProcessDefinition.setResource(Resources.PROCESS_DEFINITION);
authProcessInstance.setResource(Resources.PROCESS_INSTANCE);
//流程定义的资源 ID 是流程定义key
authProcessDefinition.setResourceId("invoice");
//允许使用 *
authProcessInstance.setResourceId("*")
// allow the user to create instances of this process definition
authProcessDefinition.addPermission(Permissions.CREATE_INSTANCE);
// and to create processes
authProcessInstance.addPermission(Permissions.CREATE);

authorizationService.saveAuthorization(authProcessDefinition);
authorizationService.saveAuthorization(authProcessInstance);
```
# Camunda Admin Webapp

Camunda Admin Web 应用程序提供了一个开箱即用的 [用于配置授权的 UI]({{< ref "/webapps/admin/authorization-management.md" >}}).

# 性能注意事项

使用数据库进行授权是最有效率的。示例：在执行任务查询时，数据库查询仅返回用户具有“读”权限的任务。

## 查询授予授权的性能

当只使用授予授权时，由于授权表可以与资源表（任务表、流程实例表等...）连接，所以查询非常有效。

## 查询撤销授权的性能

撤销授权的检查成本很高。 检查需要考虑授权的优先级。 示例：用户级别的授予强于组级别的撤销。 一系列嵌套的 SQL `CASE` 语句和一个子选择用于说明优先级。 这有两个缺点：

* 检查与资源表的基数成线性比例（任务数加倍使查询速度减慢一倍）
* 基于`CASE`语句的特定结构在以下数据库上表现极差：PostgreSQL、DB2

在这些数据库上，撤销授权实际上是不可用的。

其他参见本章的 [配置选项](#检查撤销授权) 。

[hist-inst-perm-config-flag]: {{< ref "/reference/deployment-descriptors/tags/process-engine.md#enable-historic-instance-permissions" >}}
[Removal-Time-based History Cleanup Strategy]: {{< ref "/user-guide/process-engine/history.md#removal-time-based-strategy" >}}
