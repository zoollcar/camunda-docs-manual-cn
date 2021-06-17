---

title: '公共API'
weight: 80

menu:
  main:
    identifier: "user-guide-introduction-public-api"
    parent: "user-guide-introduction"

---


Camunda平台提供了一个公共API。本节主要包括公共API的定义和版本更新的向后兼容性问题。


# 公共 API 的定义

Camunda平台的公共API包含两部分：

Java API: 

以下模块的所有非实现的Java包（包名不包含`impl`，也就是接口类）：

* `camunda-engine`
* `camunda-engine-spring`
* `camunda-engine-cdi`
* `camunda-engine-dmn`
* `camunda-bpmn-model`
* `camunda-cmmn-model`
* `camunda-dmn-model`
* `camunda-spin-core`
* `camunda-connect-core`
* `camunda-commons-typed-values`

HTTP API (REST API):

* `camunda-engine-rest`: HTTP接口（REST API接受的一组HTTP请求，详情见[REST API参考]({{< ref "/reference/rest/_index.md" >}})。Java类不是公共API的一部分.


# 公共API的向后兼容

Camunda的版本管理方案遵循[语义化版本](http://semver.org/)提出的 大版本号.小版本号.补丁 模式。Camunda将保持小版本号版本更新是向后兼容性的。例如：从版本 `7.1.x` 到 `7.2.x` 公共API是向后兼容的。
