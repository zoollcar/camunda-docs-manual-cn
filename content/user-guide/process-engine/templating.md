---

title: '模板'
weight: 90

menu:
  main:
    identifier: "user-guide-process-engine-templating"
    parent: "user-guide-process-engine"

---


Camunda平台支持模板引擎，兼容JSR-223的脚本引擎。因此，模板用于任何可以使用脚本的地方。

在Camunda平台的社区版本中，以下模板引擎是开箱即用的：

* [FreeMarker][freemarker]

以下模板引擎是作为插件添加：

* [Apache Velocity][velocity]

脚本引擎包装器的实现可以在[camunda-template-engines][camunda-template-engines]资源库中找到。

此外，以下模板引擎可以在企业版中使用：

* [XSLT](/enterprise/download/#additional-information)


# 安装一个模板引擎

## 为嵌入式流程引擎安装模板引擎

模板引擎和脚本引擎是用相同方式安装的。这意味着模板引擎必须被添加到进程引擎的classpath中。

使用嵌入式流程引擎时，模板引擎库必须添加到应用程序部署中。在maven`war`项目中使用流程引擎时，模板引擎依赖必须作为依赖项添加到maven`pom.xml`文件中。

{{< note title="" class="info" >}}
   请导入[Camunda BOM](/get-started/apache-maven/)，以确保Camunda项目的版本是正确的。
{{< /note >}}

```xml
<dependencies>

  <!-- freemarker -->
  <dependency>
    <groupId>org.camunda.template-engines</groupId>
    <artifactId>camunda-template-engines-freemarker</artifactId>
  </dependency>

  <!-- apache velocity -->
  <dependency>
    <groupId>org.camunda.template-engines</groupId>
    <artifactId>camunda-template-engines-velocity</artifactId>
  </dependency>

</dependencies>
```


## 为分布式流程引擎安装一个模板引擎

当使用一个分布式流程引擎时，模板需要添加到分布式流程引擎的classpath中。classpath的位置取决于应用服务器。在Apache Tomcat中，库必须被添加到共享的`lib/`文件夹中。

{{< note title="" class="info" >}}
  [FreeMarker](http://freemarker.org/) 是封装在Camunda预编译发行版中的。
{{< /note >}}


# 使用模板引擎

如果模板引擎库在classpath中，你可以在BPMN流程中任何可以[使用脚本][use-scripts]的地方使用模板，例如作为一个脚本任务或输入输出映射。
FreeMarker模板引擎是Camunda平台发布的一部分。

在模板内部，BPMN元素范围的所有流程变量都是可用的。模板也可以从外部资源加载，如[脚本源部分][script-source]所述。

下面的例子显示了一个FreeMarker模板，它的结果被保存在流程变量`text`中：

```xml
<scriptTask id="templateScript" scriptFormat="freemarker" camunda:resultVariable="text">
  <script>
    Dear ${customer},

    thank you for working with Camunda Platform ${version}.

    Greetings,
    Camunda Developers
  </script>
</scriptTask>
```

在输入输出映射中，使用外部模板来生成 "camunda:connector" 的有效载荷是非常有用的。

```xml
<bpmn2:serviceTask id="soapTask" name="Send SOAP request">
  <bpmn2:extensionElements>
    <camunda:connector>
      <camunda:connectorId>soap-http-connector</camunda:connectorId>
      <camunda:inputOutput>

        <camunda:inputParameter name="soapEnvelope">
          <camunda:script scriptFormat="freemarker" resource="soapEnvelope.ftl" />
        </camunda:inputParameter>

        <!-- ... remaining connector config omitted -->

      </camunda:inputOutput>
    </camunda:connector>
  </bpmn2:extensionElements>
</bpmn2:serviceTask>
```


# 使用模板引擎 XSLT

{{< enterprise >}}
  请注意，该功能只包含在Camunda平台的企业版中，社区版中没有。
{{< /enterprise >}}


## 安装 XSLT 模板引擎

The XSLT Template Engine can be downloaded from the [Enterprise Edition Download page](/enterprise/download/#additional-information).

Instructions on how to install the template engine can be found inside the downloaded distribution.


## 在嵌入式流程引擎中使用XSLT模板引擎

When using an embedded process engine, the XSLT template engine library must be added to the
application deployment. When using the process engine in a maven `war` project, the template engine
dependency must be added as dependencies to the maven `pom.xml` file:

{{< note title="" class="info" >}}
  Please import the [Camunda BOM](/get-started/apache-maven/) to ensure correct versions for every Camunda project.
{{< /note >}}

```xml
<dependencies>

  <!-- XSLT -->
  <dependency>
    <groupId>org.camunda.bpm.extension.xslt</groupId>
    <artifactId>camunda-bpm-xslt</artifactId>
  </dependency>

</dependencies>
```

## 使用 XSLT 模板

The following is an example of a BPMN ScriptTask used to execute an XSLT Template:

```xml
<bpmn2:scriptTask id="ScriptTask_1" name="convert input"
                  scriptFormat="xslt"
                  camunda:resource="org/camunda/bpm/example/xsltexample/example.xsl"
                  camunda:resultVariable="xmlOutput">

  <bpmn2:extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="camunda_source">${customers}</camunda:inputParameter>
    </camunda:inputOutput>
  </bpmn2:extensionElements>

</bpmn2:scriptTask>
```

As shown in the example above, the XSL source file can be referenced using the `camunda:resource`
attribute. It may be loaded from the classpath or the deployment (database) in the same way as
described for [script tasks][script-source].

The result of the transformation can be mapped to a variable using the `camunda:resultVariable`
attribute.

Finally, the input of the transformation must be mapped using the special variable `camunda_source`
using a `<camunda:inputParameter ... />` mapping.

A [full example of the XSLT Template Engine][xslt-example] in Camunda Platform can be found in the
examples repository..


[freemarker]: http://freemarker.org/
[velocity]: http://velocity.apache.org/
[camunda-template-engines]: https://github.com/camunda/camunda-template-engines-jsr223
[use-scripts]: {{< ref "/user-guide/process-engine/scripting.md" >}}
[script-source]: {{< ref "/user-guide/process-engine/scripting.md#script-source" >}}
[xslt-example]: https://github.com/camunda/camunda-bpm-examples/tree/master/scripttask/xslt-scripttask
