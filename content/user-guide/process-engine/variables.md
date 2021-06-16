---

title: '流程变量'
weight: 40

menu:
  main:
    identifier: "user-guide-process-engine-variables"
    parent: "user-guide-process-engine"

---

本节描述流程变量的概念。流程变量可以用将数据添加到流程的运行时状态中，或者更具体地说，变量作用域中。改变实体的各种API可以用来更新这些附加的变量。一般来说，一个变量由一个名称和一个值组成。名称用于在整个流程中识别变量。例如，如果一个活动（activity）设置了一个名为 *var* 的变量，那么后续活动中可以通过使用这个名称来访问它。变量的值是一个 Java 对象。


# 变量作用域和变量可访问性

所有可以拥有变量的实体被称为 *变量作用域* 。包括执行（包括流程实例在内）和任务。正如在[概念]({{< ref "/user-guide/process-engine/process-engine-concepts.md#executions" >}})章节所描述的，流程实例的运行状态由一棵执行树表示。参看下面的流程模型，其中红点标志着活动任务。

{{< img src="../img/variables-3.png" title="Variables" >}}

这个流程的运行时结构如下：

{{< img src="../img/variables-4.png" title="Variables" >}}

有一个流程实例有两个子执行，每个子执行都创建了一个任务。所有这五个实体都是变量作用域，箭头标志着父-子关系。在父作用域上定义的变量可以在每个子作用域中被访问，除非子作用域定义了同名的变量。反过来说，子变量不能从父作用域访问。直接附属于有关作用域的变量被称为 *local* 变量。思考一下，下面将变量作用域的分配情况。

{{< img src="../img/variables-5.png" title="Variables" >}}

在这种情况下，当在 *task 1* 工作时，可以访问变量 *worker* 和 *customer* 。请注意，由于作用域的结构，变量 *worker* 可以被定义两次，所以 *task 1* 访问的 *worker* 变量与 *task 2* 访问的不同。然而，两者都共享变量 *customer* ，这意味着如果该变量被其中一个任务更新，这一变化对另一个任务也是可见的。

两个任务都可以访问到这两个变量，但这些变量都不是局部变量。所有三个执行任务都有只有一个局部变量。

现在我们假设，我们在 *task 1* 上设置了一个局部变量 *customer* 。

{{< img src="../img/variables-6.png" title="Variables" >}}

虽然两个名为 *customer* 和 *worker* 的变量仍然可以从 *Task 1* 访问，但 *Execution 1* 上的 *customer* 变量是隐藏的，所以可访问的 *customer* 变量是 *Task 1* 的局部变量。

一般来说，变量可以被访问 有以下几种情况。

* 实例化流程
* 传递信息
* 任务生命周期的转换，如完成或解决
* 从外部设置/获取变量
* 设置/获取[Delegate]({{< ref "/user-guide/process-engine/delegation-code.md" >}})中的变量
* 流程模型中的表达式
* 流程模型中的脚本
* （历史性的）变量查询


# 设置和查询变量 - 概述

为了设置和查询变量，流程引擎提供了一个Java API，允许从Java中设置变量，并以同样的形式查询它们。在流程引擎内部，引擎使用序列化将变量持久化到数据库中。对于大多数应用程序来说，这是一个无关紧要的细节。然而，有时，当使用自定义的Java类时，变量的序列化值是有意义的。想象一下一个管理许多流程应用程序的监控应用程序的情况。它与这些应用程序的类解耦，因此不能访问其Java表示中的自定义变量。对于这些情况，流程引擎提供了一种查询和操作序列化值的方法。可以归结为两个API。

* **Java Object Value API**: 变量被表示为Java对象。这些对象可以直接被设置为值，并以同样的形式被检索。这是更简单的API，在实现代码作为流程应用的一部分时，是推荐的方式。
* **Typed Value API**: 变量值被包裹在所谓的 *typed values* 中，用于设置和查询变量。类型化的值提供了对元数据的访问，如引擎对变量进行序列化的方式，以及根据类型，变量的序列化表示。元数据还包含一个信息，即一个变量是否是瞬时的。

例如，下面的代码对两个整数变量使用了两种API查询和设置方式：

```java
// Java Object API: 查询变量
Integer val1 = (Integer) execution.getVariable("val1");

// Typed Value API: 查询变量
IntegerValue typedVal2 = execution.getVariableTyped("val2");
Integer val2 = typedVal2.getValue();

Integer diff = val1 - val2;

// Java Object API: 设置变量
execution.setVariable("diff", diff);

// Typed Value API: 设置变量
IntegerValue typedDiff = Variables.integerValue(diff);
execution.setVariable("diff", typedDiff);
```

这段代码的具体内容在[Java Object Value API]({{< relref "#java-object-api" >}}) 和 [Typed Value API]({{< relref "#typed-value-api" >}})部分有更详细的描述。

## 将变量设置为特定作用域

有可能从脚本（scripts）、输入/输出映射（input\output mapping）、监听器（listeners）和服务任务（service tasks）中设置变量到特定的作用域。这个功能的实现是使用活动ID来识别目标作用域，如果没有找到可以设置变量的作用域，就会抛出一个异常。一旦找到目标作用域，变量将被设置在它的本地，这意味着即使目标作用域没有给定id的变量，传播到父作用域的流程也不会被执行。

下面是使用脚本executionListener的例子。
```xml
<camunda:executionListener event="end">
        <camunda:script scriptFormat="groovy"><![CDATA[execution.setVariable("aVariable", "aValue","aSubProcess");]]></camunda:script>
</camunda:executionListener>
```

另一个使用例子是使用 "DelegateVariableMapping"实现的输入/输出映射。

```java
public class SetVariableToScopeMappingDelegate implements DelegateVariableMapping {
  @Override
  public void mapInputVariables(DelegateExecution superExecution, VariableMap subVariables) {
  }

  @Override
  public void mapOutputVariables(DelegateExecution superExecution, VariableScope subInstance) {
    superExecution.setVariable("aVariable","aValue","aSubProcess");
  }
}
```
这里的变量将在 "aSubProcess "中本地设置，即使变量没有事先在 "aSubProcess" 中本地设置。而且不会传播到父作用域。

# 支持的变量类型

过程引擎支持以下变量值类型：

{{< img src="../img/variables-1.png" title="Variables" >}}

根据一个变量的实际值不同，会分配一个不同的数据类型。在可用的类型中，有9种 *原始* 的值类型，这意味着它们存储简单的标准JDK类的值，没有额外的元数据：

* `boolean`: 对应 `java.lang.Boolean`
* `bytes`: 对应 `byte[]`
* `short`: 对应 `java.lang.Short`
* `integer`: 对应 `java.lang.Integer`
* `long`: 对应 `java.lang.Long`
* `double`: 对应 `java.lang.Double`
* `date`: 对应 `java.util.Date`
* `string`: 对应 `java.lang.String`
* `null`: 对应 `null`

原始值与其他变量值不同，它们可以在API查询中被使用，如流程实例查询中作为过滤条件。

类型 "file" 可以用来存储文件或输入流的内容及其元数据，如文件名、编码和文件内容对应的MIME类型。

值类型 `object` 代表自定义的Java对象。当这样的变量被持久化时，它的值会根据一个序列化程序被序列化。这些序列化程序是可配置和可替换的。

{{< note title="字符串长度限制" class="warning" >}}
`string` 变量被存储在数据库中的 "(n)varchar" 类型的列中，其长度限制为4000（Oracle为2000）。取决于使用的数据库和这个长度限制可能导致不同数量的真实字符。变量值的长度在Camunda引擎中不被校验的，但是如果超过了长度限制，将产生一个数据库级别的异常。
如果需要校验，可以自行实现，必须在调用Camunda API来设置变量之前进行。
{{< /note >}}

过程变量可以用[Camunda Spin插件]({{< ref "/user-guide/data-formats/_index.md" >}}) 提供的JSON和XML等格式存储。Spin为 `object` 类型的变量提供了序列化器，这样Java变量就可以以这JSON或XML格式持久化到数据库中了。此外，通过`xml`和`json`的值类型，可以直接将JSON和XML文档存储为Spin对象。相对于普通的`string`变量，Spin对象提供了一个流畅的API来对这类文档进行普通的操作，如读写变量对象的属性。


## Object值 序列化

当一个 `Object` 的值被传递给流程引擎时，可以指定一个 *序列化格式* 来告诉进程引擎以特定的格式来存储这个值。根据这个格式，引擎会查找一个 *序列化器* 。序列化器能够将一个Java对象序列化为指定的格式，也能从该格式的结果中反序列化它。这意味着，不同的格式可能有不同的序列化器，而且有可能实现自定义的序列化器，以便以特定的格式存储自定义对象。

进程引擎为 `application/x-java-serialized-object` 格式提供了一个内置的对象序列化器。它能够序列化实现了 `java.io.Serializable` 接口的Java对象，并应用标准的Java对象序列化。

所需的序列化格式可以在使用类型化值API设置变量时指定。

```java
CustomerData customerData = new CustomerData();

ObjectValue customerDataValue = Variables.objectValue(customerData)
  .serializationDataFormat(Variables.SerializationDataFormats.JAVA)
  .create();

execution.setVariable("someVariable", customerDataValue);
```

除此之外，流程引擎配置有一个选项 `defaultSerializationFormat` ，在没有要求特定格式时使用。这个选项默认为 `application/x-java-serialized-object'。

{{< note title="使用任务表单中的自定义对象" class="info" >}}
  请注意，内置的序列化器将对象转换为字节流，只能解析简单的Java类。当实现基于复杂对象的表单时，应该使用基于文本的序列化格式，因为 Tasklist 不能解释这些字节流。关于如何整合XML和JSON等序列化格式的细节，请参见 *将对象序列化为XML和JSON* 这一框。
{{< /note >}}

{{< note title="将对象序列化为XML和JSON" class="info" >}}
  [Camunda Spin 插件]({{< ref "/user-guide/data-formats/_index.md" >}}) 提供了能够将对象序列化为XML和JSON的序列化器。当希望序列化的值可以被人类解释时，或者当序列化的值应该是有意义的而没有相应的Java类时，就可以使用它们。当使用预构建的Camunda发行版时，Camunda Spin 已经预先配置好了，你可以无需进一步配置的尝试使用这些格式。
{{< /note >}}


# Java Object API

从Java处理流程变量的最方便的方法是使用它们的Java Object表示。只要流程引擎提供变量访问，就可以用这种表示方法访问流程变量，因为对于自定义对象来说，引擎知道所涉及的类。例如，下面的代码为一个给定的流程实例设置和查询一个变量。

```java
com.example.Order order = new com.example.Order();
runtimeService.setVariable(execution.getId(), "order", order);

com.example.Order retrievedOrder = (com.example.Order) runtimeService.getVariable(execution.getId(), "order");
```

请注意，这段代码在变量作用域的层次结构中尽可能高的位置设置一个变量。这意味着，如果该变量已经存在（无论是在这个执行中还是在它的任何父作用域中），它将被更新。如果变量还不存在，它将在最高的作用域，即进程实例中被创建。如果一个变量应该在所提供的执行中被精确地设置，可以使用 *local* 方法。例如：

```java
com.example.Order order = new com.example.Order();
runtimeService.setVariableLocal(execution.getId(), "order", order);

com.example.Order retrievedOrder = (com.example.Order) runtimeService.getVariable(execution.getId(), "order");
com.example.Order retrievedOrder = (com.example.Order) runtimeService.getVariableLocal(execution.getId(), "order");
// 两种方法都会返回变量
```

每当一个变量在其Java代码中被设置时，流程引擎会自动确定一个合适的值序列化器，或者在所提供的值不能被序列化时引发一个异常。


# 类型值（Typed Value） API

在访问一个变量的序列化方式的情况下，或者在必须提示引擎以某种格式序列化一个值的情况下，可以使用基于类型化值的API。与基于Java-Object的API相比，它将一个变量值赋给一个所谓的 *类型值（Typed Value）* 中。这样一个类型化的值允许更丰富的变量值的表示。

为了方便构建类型值，Camunda平台提供了 `org.camunda.bpm.engine.variables` 类。该类包含静态方法，允许创建单一类型的值，以及用流畅的方式创建类型值的映射。


## 原始值

下面的代码通过指定一个类型的值来设置一个单一的 `String` 变量。

```java
StringValue typedStringValue = Variables.stringValue("a string value");
runtimeService.setVariable(execution.getId(), "stringVariable", typedStringValue);

StringValue retrievedTypedStringValue = runtimeService.getVariableTyped(execution.getId(), "stringVariable");
String stringValue = retrievedTypedStringValue.getValue(); // 结果还是 "a string value"
```

请注意，在这个API中，围绕着变量值还有一个抽象层次包。因此，为了访问真正的值，有必要 *拆包* 。


## 文件（File）值

当然，对于普通的 "String" 值，基于Java-Object的API更加简洁。因此，让我们考虑更丰富的数据结构的值。

文件可以作为BLOB保存在数据库中。`file` 值类型允许存储额外的元数据，如文件名和mime类型。下面的示例代码从一个文本文件中创建一个文件值。

```java
FileValue typedFileValue = Variables
  .fileValue("addresses.txt")
  .file(new File("path/to/the/file.txt"))
  .mimeType("text/plain")
  .encoding("UTF-8")
  .create();
runtimeService.setVariable(execution.getId(), "fileVariable", typedFileValue);

FileValue retrievedTypedFileValue = runtimeService.getVariableTyped(execution.getId(), "fileVariable");
InputStream fileContent = retrievedTypedFileValue.getValue(); // a byte stream of the file contents
String fileName = retrievedTypedFileValue.getFilename(); // equals "addresses.txt"
String mimeType = retrievedTypedFileValue.getMimeType(); // equals "text/plain"
String encoding = retrievedTypedFileValue.getEncoding(); // equals "UTF-8"
```

### 更改文件值

要改变或更新一个 `文件值` ，你必须创建一个具有相同名称和新内容的新`文件值`来替换旧的，因为所有类型的值都是不可改变的。

```java
InputStream newContent = new FileInputStream("path/to/the/new/file.txt");
FileValue fileVariable = execution.getVariableTyped("addresses.txt");  
Variables.fileValue(fileVariable.getName()).file(newContent).encoding(fileVariable.getEncoding()).mimeType(fileVariable.getMimeType()).create();
```

## 对象值

自定义Java对象可以用值类型 `object` 进行序列化。使用`object`值类型API的例子。

```java
com.example.Order order = new com.example.Order();
ObjectValue typedObjectValue = Variables.objectValue(order).create();
runtimeService.setVariableLocal(execution.getId(), "order", typedObjectValue);

ObjectValue retrievedTypedObjectValue = runtimeService.getVariableTyped(execution.getId(), "order");
com.example.Order retrievedOrder = (com.example.Order) retrievedTypedObjectValue.getValue();
```

这和基于Java-Object的API类似。然而，现在可以告诉引擎在持久化值时使用哪种序列化格式。比如说：

```java
ObjectValue typedObjectValue = Variables
  .objectValue(order)
  .serializationDataFormat(Variables.SerializationDataFormats.JAVA)
  .create();
 ```

创建一个值，由引擎内置的Java对象序列化器进行序列化。这样，查询到的`ObjectValue`实例提供了额外的变量细节。

```java
// 返回 true
boolean isDeserialized = retrievedTypedObjectValue.isDeserialized();

// 返回引擎使用的格式值序列化为数据库
String serializationDataFormat = retrievedTypedObjectValue.getSerializationDateFormat();

// 返回变量的序列化表示;实际值取决于使用的序列化格式
String serializedValue = retrievedTypedObjectValue.getValueSerialized();

// 返回Class com.example.Order
Class<com.example.Order> valueClass = retrievedTypedObjectValue.getObjectType();

// 返回字符串 "com.example.Order"
String valueClassName = retrievedTypedObjectValue.getObjectTypeName();
```

当调用的应用程序不拥有实际变量值的类时（即`com.example.Order`不知道）的情况下，`runtimeService.getVariableTyped(execution.getId(), "order")`将引发一个异常，因为它立即试图对变量值进行反序列化。在这种情况下，可以使用调用`runtimeService.getVariableTyped(execution.getId(), "order", false)`。额外的布尔参数告诉进程引擎不要尝试反序列化。在这种情况下，调用`isDeserialized()`将返回`false`，而诸如`getValue()`和`getObjectType()`的调用将引发异常。尽管如此，调用`getValueSerialized()`和`getObjectTypeName()`也是一种访问变量的方式。

同样地，也可以通过序列化的表示法来设置一个变量。

```java
String serializedOrder = "...";
ObjectValue serializedValue =
  Variables
    .serializedObjectValue(serializedOrder)
    .serializationDataFormat(Variables.SerializationDataFormats.JAVA)
    .objectTypeName("com.example.Order")
    .create();

runtimeService.setVariableLocal(execution.getId(), "order", serializedValue);

ObjectValue retrievedTypedObjectValue = runtimeService.getVariableTyped(execution.getId(), "order");
com.example.Order retrievedOrder = (com.example.Order) retrievedTypedObjectValue.getValue();
```

{{< note title="不一致的变量状态" class="warning" >}}
  当设置一个序列化的变量值时，不会检查序列化值的结构是否与变量值的类型是否兼容。当设置上述例子中的变量时，提供的序列化值并不会根据`com.example.Order`的结构进行验证。因此，只有在调用`runtimeService#getVariableTyped'时才会发现无效的变量值。
{{< /note >}}

{{< note title="Java serialization format" class="warning" >}}
  请注意，当使用变量的序列化表示时，Java序列化格式默认是被禁止的。你应该使用任意一种格式（JSON或XML）或明确启用Java序列化，
  可以查看 [`javaSerializationFormatEnabled`]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#javaSerializationFormatEnabled" >}}) 配置。
  但是，在启用前请务必阅读[安全]({{< ref "/user-guide/security.md#variable-values-from-untrusted-sources" >}})相关内容。
{{< /note >}}

## JSON和XML值

Camunda Spin插件为JSON和XML文档提供了一个抽象，以方便它们的处理和操作。这通常比将此类文档存储为普通的 "字符串" 变量更方便。参见Camunda SPIN的文档[存储为JSON]({{< ref "/user-guide/data-formats/json.md#native-json-variable-value" >}})和[存储为XML]({{< ref "/user-guide/data-formats/xml.md#native-xml-variable-value" >}})

## 瞬时变量

瞬时变量的声明只能通过基于类型值的API来实现。它们不会被保存到数据库中，只会在当前事务中存在。在进程实例的执行过程中，每一个等待状态都会导致所有瞬时变量的丢失。这种情况通常发生在诸如外部服务当前不可用、用户任务已达到或进程执行正在等待一个消息、信号或条件的时候。请谨慎使用这一功能。

[任何类型]({{<relref "#supported-variable-values">}}) 都可以通过使用`Variables`类将参数`isTransient`设置为true来声明为瞬时的。

```java
// 原始值
TypedValue typedTransientStringValue = Variables.stringValue("foobar", true);

// 对象值
com.example.Order order = new com.example.Order();
TypedValue typedTransientObjectValue = Variables.objectValue(order, true).create();

// 文件值
TypedValue typedTransientFileValue = Variables.fileValue("file.txt", true)
  .file(new File("path/to/the/file.txt"))
  .mimeType("text/plain")
  .encoding("UTF-8")
  .create();
``` 

瞬时变量可以通过REST API使用，可以查看[启动一个新的流程实例]({{< ref "/reference/rest/process-definition/post-start-process-instance.md">}}).

## 设置多个类型值（Typed Values）

与基于Java-Object的API类似，也可以在一次API调用中设置多个类型的值。Variables类提供了一个链式API来构建一个类型值的映射。

```java
com.example.Order order = new com.example.Order();

VariableMap variables =
  Variables.createVariables()
    .putValueTyped("order", Variables.objectValue(order).create())
    .putValueTyped("string", Variables.stringValue("a string value"))
    .putValueTyped("stringTransient", Variables.stringValue("foobar", true));
runtimeService.setVariablesLocal(execution.getId(), "order", variables);
```


# API之间的相互替换

这两种API提供了对相同实体的不同处理方式，因此可以根据需要进行组合。例如，使用基于Java-Object的API设置的变量可以作为一个类型值被检索，反之亦然。由于`VariableMap`类实现了`Map`接口，所以也可以把普通的Java对象以及类型化的值放入这个Map中。

你应该使用哪个API？最适合你的目的的那个。当你确定你总是可以访问所涉及的值类时，比如在像 "JavaDelegate"这样的流程应用中实现代码时，那么基于Java-Object的API就更容易使用。当你需要访问特定的值元数据时，如序列化格式或将变量定义为瞬时变量，那么基于类型值的API是最合适的方式。

# 输入/输出变量映射

为了提高源代码和业务逻辑的可重用性，Camunda平台提供输入/输出流程变量的映射。
这可用于任务、事件和子流程中。

为了使用变量映射，Camunda扩展元素[inputOutput][]必须被添加到
到该元素中。它可以包含多个[inputParameter][]和[outputParameter][]元素。
指定哪些变量应该被映射。[inputParameter][] 的`name`属性表示
活动中的变量名称（要创建的局部变量），而 [outputParameter][] 的 `name`属性表示活动外的变量名称。

inputParameter/outputParameter 的内容指定了被映射到相应的
变量。它可以是一个简单的常量字符串或一个表达式。
如果什么都不填则变量为 `null`。

```xml
<camunda:inputOutput>
  <camunda:inputParameter name="x">foo</camunda:inputParameter>
  <camunda:inputParameter name="willBeNull"/>
  <camunda:outputParameter name="y">${x}</camunda:outputParameter>
  <camunda:outputParameter name="z">${willBeNull == null}</camunda:outputParameter>
</camunda:inputOutput>
```

甚至可以使用[lists][list]和[maps][map]这样的复杂结构。这两种结构也可以嵌套使用。

```xml
<camunda:inputOutput>
  <camunda:inputParameter name="x">
    <camunda:list>
      <camunda:value>a</camunda:value>
      <camunda:value>${1 + 1}</camunda:value>
      <camunda:list>
        <camunda:value>1</camunda:value>
        <camunda:value>2</camunda:value>
        <camunda:value>3</camunda:value>
      </camunda:list>
    </camunda:list>
  </camunda:inputParameter>
  <camunda:outputParameter name="y">
    <camunda:map>
      <camunda:entry key="foo">bar</camunda:entry>
      <camunda:entry key="map">
        <camunda:map>
          <camunda:entry key="hello">world</camunda:entry>
          <camunda:entry key="camunda">bpm</camunda:entry>
        </camunda:map>
      </camunda:entry>
    </camunda:map>
  </camunda:outputParameter>
</camunda:inputOutput>
```

也可以用一个脚本来提供变量值。请参阅相应的
[脚本章节][script-io] 查看如何指定一个脚本

输入/输出映射的好处的一个简单例子是复杂的计算。
一个复杂的计算如果被多个流程定义所使用的。则这个计算可以被开发成独立的委托代码或脚本，并在每个流程中重复使用，即使这些流程使用不同的变量集。
输入映射被用来将不同的流程变量映射到复杂计算活动所需输入参数。
输出映射允许在后面的流程执行中继续使用计算结果。

更详细的一个例子：让我们假设这种计算是由一个Java委托类`org.camunda.bpm.example.ComplexCalculation`实现的。
这个委托需要一个`userId`和一个`costSum`变量作为输入参数。
然后它计算三个值：`pessimisticForecast`, `realisticForecast` and `optimisticForecast`。
这是对客户所需要的未来成本的不同预测。在第一个流程中，两个输入变量都可以作为流程变量，但名称不同（`id`, `sum`）。对于三个输出结果，该流程只使用了`realisticForecast`，它在后续活动中以`forecast`的名称继续使用。则相应的输入/输出映射看起来如下：

```xml
<serviceTask camunda:class="org.camunda.bpm.example.ComplexCalculation">
  <extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="userId">${id}</camunda:inputParameter>
      <camunda:inputParameter name="costSum">${sum}</camunda:inputParameter>
      <camunda:outputParameter name="forecast">${realisticForecast}</camunda:outputParameter>
    </camunda:inputOutput>
  </extensionElements>
</serviceTask>
```

在第二个流程中，让我们假设 "costSum" 变量必须从三个不同的map中获取。同时，这个流程
后续需要一个变量 `avgForecast` 作为三个预测的平均值。在这种情况下，映射看起来如下。

```xml
<serviceTask camunda:class="org.camunda.bpm.example.ComplexCalculation">
  <extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="userId">${id}</camunda:inputParameter>
      <camunda:inputParameter name="costSum">
        ${mapA[costs] + mapB[costs] + mapC[costs]}
      </camunda:inputParameter>
      <camunda:outputParameter name="avgForecast">
        ${(pessimisticForecast + realisticForecast + optimisticForecast) / 3}
      </camunda:outputParameter>
    </camunda:inputOutput>
  </extensionElements>
</serviceTask>
```


## 多实例IO映射

输入映射也可以用于多实例结构，在这种情况下，映射被应用于创建的每个实例。例如，对于一个有五个实例的多实例子流程，映射被执行五次，涉及的变量在五个子流程的每个作用域中被创建，以便它们可以被独立访问。

{{< note title="多实例构造不支持输出映射" class="info" >}}
引擎不支持多实例结构的输出映射。输出映射的每一个实例都会覆盖之前的实例所设置的变量，最终的变量状态将变得难以预测。
{{< /note >}}

## 取消的活动的IO映射

如果一个活动被取消（例如由于抛出一个 BPMN 错误），IO 映射仍然被执行。如果输出映射引用了当时不存在于活动范围内的变量，则这可能导致异常。

以上是默认行为，引擎会仍然试图在取消的活动上执行输出映射，如果没有找到变量，就会出现异常。可以通过启用 [跳过取消活动的输出映射]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#skipOutputMappingOnCanceledActivities" >}}) 配置项 (设置其为 `true`) 引擎将不对任何被取消的活动进行输出映射。

[inputOutput]: {{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#camunda-inputoutput" >}}
[inputParameter]: {{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#inputparameter" >}}
[outputParameter]: {{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#outputparameter" >}}
[list]: {{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#camunda-list" >}}
[map]: {{< ref "/reference/bpmn20/custom-extensions/extension-elements.md#camunda-map" >}}
[script-io]: {{< ref "/user-guide/process-engine/scripting.md#use-scripts-as-inputoutput-parameters" >}}
