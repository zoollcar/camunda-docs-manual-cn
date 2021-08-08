---

title: "Spring 框架集成"
weight: 50

menu:
  main:
    identifier: "user-guide-spring-framework-integration"
    parent: "user-guide"

---

camunda-engine Spring 框架集成位于 `camunda-engine-spring` maven 模块内，可以通过以下依赖项添加到基于 maven 的项目中：

{{< note title="" class="info" >}}
  请导入 [Camunda BOM](/get-started/apache-maven/) 以确保每个 Camunda 项目的版本正确。
  
{{< /note >}}

```xml
<dependency>
  <groupId>org.camunda.bpm</groupId>
  <artifactId>camunda-engine-spring</artifactId>
</dependency>
```

`camunda-engine-spring` 依赖应该添加到流程应用中。
它可以与 Spring 4 或 5 一起使用。必须在所需版本中添加以下最小的 Spring 依赖项集：

```xml
<properties>
  <spring.version>5.1.6.RELEASE</spring.version>
</properties>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-framework-bom</artifactId>
      <version>${spring.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-tx</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-orm</artifactId>
  </dependency>
</dependencies>
```

