---

title: '扩展（插件）'
weight: 50

menu:
  main:
    identifier: "user-guide-introduction-extensions"
    parent: "user-guide-introduction"

---

Camunda平台是由Camunda与社区合作开发的一个开源项目。“核心项目”（即 "Camunda平台"）是Camunda平台产品的基础，该产品由Camunda以商业形式提供。商业化的[Camunda平台产品](http://camunda.com/bpm/features/)包含额外的（非开源）功能，并通过提供企业支持和[错误修复版本](/enterprise/download)等服务提供给Camunda平台客户。

# 社区扩展

Camunda支持社区在Camunda平台的框架下建立更多的社区扩展。这些社区扩展由社区维护，不属于Camunda平台的商业产品。

{{< note title="Chamunda支持" class="warning" >}}
  Camunda does not support community extensions as part of its commercial services to enterprise subscription customers
{{< /note >}}

The [Camunda Community Hub](https://github.com/camunda-community-hub) is a GitHub Organization that serves as the home of Camunda open source community-contributed extensions. You can migrate an extension you've built to the Hub, search for existing extensions, or get started with open source by helping community extension maintainers with open issue or pull request triage.

## 创建一个社区扩展

Do you have a great idea around open source BPM you want to share with the world? Awesome! Camunda will support you in building your own community extension. Have a look at our [process for creating a new community extension](https://github.com/camunda-community-hub/community#creating-a-new-camunda-community-extension) to find out how to propose a community project, or [contribute code to the core Camunda codebase](https://camunda.com/developers/how-to-contribute/). 

You can also visit the Camunda Community Hub [community repository](https://github.com/camunda-community-hub/community) to learn more about migrating your community extension into the community hub, benefits to joining the Camunda Community Hub Organization, contributing to an extension, and much more.

The Camunda Community Hub enables developers to have a centralized home for their Camunda community extensions, and aids new contributors to open source software in discovering new projects to work on.

# 企业版扩展

{{< enterprise >}}
  Please note that these extensions are only included in the enterprise edition of the Camunda Platform, it is not available in the community edition.
{{< /enterprise >}}

## XSLT 扩展

The XSLT extension depends on the following third-party libraries:

* [SAXON](http://saxon.sourceforge.net/) [(Mozilla Public License 2.0)](https://www.mozilla.org/MPL/2.0/)
