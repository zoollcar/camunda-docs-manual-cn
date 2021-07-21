---

title: '版本兼容性'
weight: 300

menu:
  main:
    name: "版本兼容性"
    identifier: "external-task-client-compatibility-matrix"
    parent: "external-task-client"

---

Camunda 平台的每个版本都绑定到 **外部任务客户端** 的特定版本。

{{< note title="注意" class="info" >}}
  从 7.15.0 版本开始，Camunda 平台及其兼容的 **Java** 外部任务客户端始终使用相同的版本号。
{{< /note >}}

<table class="table table-striped">
  <tr>
    <th>Camunda平台版本</th>
    <th>NodeJs</th>
    <th>Java</th>
  </tr>
  <tr>
    <td>7.9.x</td>
    <td>1.0.x</td>
    <td>1.0.x</td>
  </tr>
  <tr>
    <td>7.10.x</td>
    <td>1.1.x</td>
    <td>1.1.x</td>
  </tr>
  <tr>
    <td>7.11.x</td>
    <td>1.1.x, 1.2.x</td>
    <td>1.2.x</td>
  </tr>
  <tr>
    <td>7.12.x</td>
    <td>1.3.x</td>
    <td>1.3.x</td>
  </tr>
  <tr>
    <td>7.13.x</td>
    <td>2.0.x</td>
    <td>1.3.x</td>
  </tr>
  <tr>
    <td>7.14.x</td>
    <td>2.0.x</td>
    <td>1.4.x</td>
  </tr>
  <tr>
    <td>7.15.x</td>
    <td>2.1.x</td>
    <td>7.15.x</td>
  </tr>
</table>

Camunda 仅推荐（并支持）这些默认组合。 尽管如此，外部任务客户端的每个版本都可以与 Camunda 平台工作流引擎的更新补丁版本结合使用。
