---

title: '连接器'
weight: 100

menu:
  main:
    identifier: "user-guide-process-engine-connectors"
    parent: "user-guide-process-engine"

---


通过依赖项目[camunda-connect](https://github.com/camunda/camunda-connect)，流程引擎支持简单的连接器。以下是目前已实现的连接器：

<table class="table">
  <tr>
    <th>Connector</th>
    <th>ID</th>
  </tr>
  <tr>
    <td>REST HTTP</td>
    <td>http-connector</td>
  </tr>
  <tr>
    <td>SOAP HTTP</td>
    <td>soap-http-connector</td>
  </tr>
</table>

也可以在camunda中实现你自己的自定义连接器。关于扩展连接器的更多信息，请访问[外部连接器]({{< ref "/reference/connect/extending-connect.md" >}})章节。


# 配置 Camunda连接器

由于Camunda连接器只有在使用流程引擎时才部分可用（查看下面的列表）。通过使用预建的发行版，Camunda连接器已经被预先配置好了。

存在以下 "连接" 组件：

* `camunda-connect-core`: 一个只包含核心连接类的jar。该组价已经可以作为流程引擎的依赖项。除了 "camunda-connect-core" 之外，还有 "camunda-connect-http-client" 和 "camunda-connect-soap-http-client" 等单个连接器的实现。当需要重新配置默认连接器或使用自定义连接器实现时，应使用这些依赖关系。
* `camunda-connect-connectors-all`: 没有依赖关系的单一jar，包含HTTP和SOAP连接器。
* `camunda-engine-plugin-connect`: 是一个流程引擎插件，用于向Camunda平台添加连接器。


# Maven方式导入

{{< note title="" class="info" >}}
  请导入[Camunda BOM](/get-started/apache-maven/)，以确保每个Camunda项目的版本正确。
{{< /note >}}


## camunda-connect-core

`camunda-connect-core`包含连接器的核心类。此外，HTTP和SOAP连接器可以通过`camunda-connect-http-client`和`camunda-connect-soap-http-client`的依赖来添加。这些组件将引入他们自身的依赖，如Apache HTTP客户端。为了与流程引擎集成，需要`camunda-engine-plugin-connect`这个工件。Maven的坐标如下：

```xml
<dependency>
  <groupId>org.camunda.connect</groupId>
  <artifactId>camunda-connect-core</artifactId>
</dependency>
```

```xml
<dependency>
  <groupId>org.camunda.connect</groupId>
  <artifactId>camunda-connect-http-client</artifactId>
</dependency>
```

```xml
<dependency>
  <groupId>org.camunda.connect</groupId>
  <artifactId>camunda-connect-soap-http-client</artifactId>
</dependency>
```

```xml
<dependency>
  <groupId>org.camunda.bpm</groupId>
  <artifactId>camunda-engine-plugin-connect</artifactId>
</dependency>
```


## camunda-connect-connectors-all

这个组件包含HTTP和SOAP连接器以及它们的依赖关系。为了避免与这些依赖的其他版本冲突，这些依赖被重新定位到不同的包中。`camunda-connect-connectors-all`的Maven坐标如下。

```xml
<dependency>
  <groupId>org.camunda.connect</groupId>
  <artifactId>camunda-connect-connectors-all</artifactId>
</dependency>
```


## 配置流程引擎插件

`camunda-engine-plugin-connect`包含一个名为`org.camunda.connect.plugin.impl.ConnectProcessEnginePlugin`的类，可以使用[插件机制]({{< ref "/user-guide/process-engine/process-engine-plugins.md" >}})在流程引擎中注册。 例如，如下`bpm-platform.xml`配置了该插件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bpm-platform xmlns="http://www.camunda.org/schema/1.0/BpmPlatform"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.camunda.org/schema/1.0/BpmPlatform http://www.camunda.org/schema/1.0/BpmPlatform ">
  ...
  <process-engine name="default">
    ...
    <plugins>
      <plugin>
        <class>org.camunda.connect.plugin.impl.ConnectProcessEnginePlugin</class>
      </plugin>
    </plugins>
    ...
  </process-engine>
</bpm-platform>
```

{{< note title="" class="info" >}}
  当使用预编译的Camunda平台发行版时，该插件已经预先配置好了。
{{< /note >}}


# 使用连接器

你需要添加Camunda 额外元素[connector]({{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#camunda-connector" >}})来使用连接器。connector使用唯一的[connectorId]({{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#camunda-connectorid" >}})来配置的。它指定了所使用的连接器实现。目前支持的连接器的ID可以在本节的开头找到。此外，可以用[输入/输出映射]({{< ref "/user-guide/process-engine/variables.md#input-output-variable-mapping" >}})配置连接器。所需的输入参数和可用的输出参数取决于连接器的实现。也可以提供额外的输入参数，以便在连接器中使用。

作为一个例子，我们展示了Camunda SOAP连接器实现的一个简略配置。可以在GitHub上的[Camunda示例库](https://github.com/camunda/camunda-bpm-examples)找到完整的[案例](https://github.com/camunda/camunda-bpm-examples/tree/master/servicetask/soap-service)源码。

```xml
<serviceTask id="soapRequest" name="Simple SOAP Request">
  <extensionElements>
    <camunda:connector>
      <camunda:connectorId>soap-http-connector</camunda:connectorId>
      <camunda:inputOutput>
        <camunda:inputParameter name="url">
          http://example.com/webservice
        </camunda:inputParameter>
        <camunda:inputParameter name="payload">
          <![CDATA[
            <soap:Envelope ...>
              ... // the request envelope
            </soap:Envelope>
          ]]>
        </camunda:inputParameter>
        <camunda:outputParameter name="result">
          <![CDATA[
            ... // process response body
          ]]>
        </camunda:outputParameter>
      </camunda:inputOutput>
    </camunda:connector>
  </extensionElements>
</serviceTask>
```

REST连接器的完整[示例](https://github.com/camunda/camunda-bpm-examples/tree/master/servicetask/rest-service)也可以在GitHub的[Camunda示例库](https://github.com/camunda/camunda-bpm-examples)中找到。
