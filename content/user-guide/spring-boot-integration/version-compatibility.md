---

title: "Spring Boot 版本兼容性"
weight: 10

menu:
  main:
    name: "版本兼容性"
    identifier: "user-guide-spring-boot-version-compatibility"
    parent: "user-guide-spring-boot-integration"

---

每个版本的Camunda Spring Boot Starter都与Camunda Platform和Spring Boot的特定版本绑定。
Camunda只推荐（并支持）这些默认组合。
其他组合在生产中使用前必须经过全面测试。

{{< note title="小心" class="info" >}}
  从7.13.0版本开始，Camunda Platform和其兼容的Spring Boot Starter总是共享相同的版本。
  而且，Spring Boot Starter中使用的Camunda Platform版本也不必再被重写了。只需选择与你想使用的Camunda Platform版本相近的Starter版本即可。
{{< /note >}}

<table class="table table-striped">
  <tr>
    <th>Spring Boot Starter 版本</th>
    <th>Camunda Platform 版本</th>
    <th>Spring Boot 版本</th>
  </tr>
  <tr>
    <td>1.0.0&#42;</td>
    <td>7.3.0</td>
    <td>1.2.5.RELEASE</td>
  </tr>
  <tr>
    <td>1.1.0&#42;</td>
    <td>7.4.0</td>
    <td>1.3.1.RELEASE</td>
  </tr>
  <tr>
    <td>1.2.0&#42;</td>
    <td>7.5.0</td>
    <td>1.3.5.RELEASE</td>
  </tr>
  <tr>
    <td>1.2.1&#42;</td>
    <td>7.5.0</td>
    <td>1.3.6.RELEASE</td>
  </tr>
  <tr>
    <td>1.3.0&#42;</td>
    <td>7.5.0</td>
    <td>1.3.7.RELEASE</td>
  </tr>
  <tr>
    <td>2.0.0&#42;&#42;</td>
    <td>7.6.0</td>
    <td>1.4.2.RELEASE</td>
  </tr>
  <tr>
    <td>2.1.x&#42;&#42;</td>
    <td>7.6.0</td>
    <td>1.5.3.RELEASE</td>
  </tr>
  <tr>
    <td>2.2.x&#42;&#42;</td>
    <td>7.7.0</td>
    <td>1.5.6.RELEASE</td>
  </tr>
  <tr>
    <td>2.3.x</td>
    <td>7.8.0</td>
    <td>1.5.8.RELEASE</td>
  </tr>
  <tr>
    <td>3.0.x</td>
    <td>7.9.0</td>
    <td>2.0.x.RELEASE</td>
  </tr>
  <tr>
    <td>3.1.x</td>
    <td>7.10.0</td>
    <td>2.0.x.RELEASE</td>
  </tr>
  <tr>
    <td>3.2.x</td>
    <td>7.10.0</td>
    <td>2.1.x.RELEASE</td>
  </tr>
  <tr>
    <td>3.3.1+</td>
    <td>7.11.0</td>
    <td>2.1.x.RELEASE</td>
  </tr>
  <tr>
    <td>3.4.x</td>
    <td>7.12.0</td>
    <td>2.2.x.RELEASE</td>
  </tr>
  <tr>
    <td>7.13.x<br/>7.13.3+&#42;&#42;&#42;</td>
    <td>7.13.x<br/>7.13.3+</td>
    <td>2.2.x.RELEASE<br/>2.3.x.RELEASE</td>
  </tr>
  <tr>
    <td>7.14.x<br/>7.14.2+&#42;&#42;&#42;</td>
    <td>7.14.x<br/>7.14.2+</td>
    <td>2.3.x.RELEASE<br/>2.4.x</td>
  </tr>
  <tr>
    <td>7.15.x<br/>7.15.3+&#42;&#42;&#42;</td>
    <td>7.15.x<br/>7.15.3+</td>
    <td>2.4.x<br/>2.5.x</td>
  </tr>
  <tr>
    <td>7.16.x</td>
    <td>7.16.x</td>
    <td>2.5.x</td>
  </tr>
</table>

\* 对于这些版本，请使用以下Maven 坐标：
```
<dependency>
  <groupId>org.camunda.bpm.extension</groupId>
  <artifactId>camunda-bpm-spring-boot-starter</artifactId>
  <version>1.x</version> <!-- set correct version here -->
</dependency>
```

\*\* 对于这些版本，请使用以下Maven 坐标：
```
<dependency>
  <groupId>org.camunda.bpm.extension.springboot</groupId>
  <artifactId>camunda-bpm-spring-boot-starter</artifactId>
  <version>2.x</version> <!-- set correct version here -->
</dependency>
```

\*\*\* 对于这些版本，所有列出的Spring Boot版本都被支持，而默认使用最古老的版本。如果你想使用较新的支持版本，请在你的应用程序中配置`dependencyManagement`，例如，在使用Maven时添加以下内容。
```
<dependencyManagement>
  <dependencies>
  ...
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>2.x.y.RELEASE</version> <!-- set correct version here -->
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  ...
  </dependencies>
</dependencyManagement>
```
