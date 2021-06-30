---

title: "流程引擎配置"
weight: 20

menu:
  main:
    name: "配置"
    identifier: "user-guide-spring-boot-configuration"
    parent: "user-guide-spring-boot-integration"

---

starter 使用`org.camunda.bpm.engine.impl.cfg.ProcessEnginePlugin` 机制来配置引擎。

配置分为 _sections_ 。 这些 _sections_ 被如下接口定义：

* `org.camunda.bpm.spring.boot.starter.configuration.CamundaProcessEngineConfiguration`
* `org.camunda.bpm.spring.boot.starter.configuration.CamundaDatasourceConfiguration`
* `org.camunda.bpm.spring.boot.starter.configuration.CamundaHistoryConfiguration`
* `org.camunda.bpm.spring.boot.starter.configuration.CamundaHistoryLevelAutoHandlingConfiguration`
* `org.camunda.bpm.spring.boot.starter.configuration.CamundaJobConfiguration`
* `org.camunda.bpm.spring.boot.starter.configuration.CamundaDeploymentConfiguration`
* `org.camunda.bpm.spring.boot.starter.configuration.CamundaJpaConfiguration`
* `org.camunda.bpm.spring.boot.starter.configuration.CamundaAuthorizationConfiguration`
* `org.camunda.bpm.spring.boot.starter.configuration.CamundaFailedJobConfiguration`
* `org.camunda.bpm.spring.boot.starter.configuration.CamundaMetricsConfiguration`

## 默认配置

以下是启动器提供的默认配置和最佳实践配置，可以自定义或重写。

### `DefaultProcessEngineConfiguration`

设置流程引擎的名称，并自动添加所有 `ProcessEnginePlugin` beans 到配置中。

### `DefaultDatasourceConfiguration`

配置Camunda数据源并启用[事务集成]({{< ref "/user-guide/spring-framework-integration/transactions.md" >}})。 默认情况下，主 `DataSource` 和 `PlatformTransactionManager` beans 通过流程引擎配置连接。

如果要[配置多个数据源](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto-two-datasources) 
而且不想将 `@Primary` 用于使用流程引擎，那么你可以使用名称创建单独的数据源 `camundaBpmDataSource` 这将由Camunda自动连接。

```java
@Bean
@Primary
@ConfigurationProperties(prefix="datasource.primary")
public DataSource primaryDataSource() {
  return DataSourceBuilder.create().build();
}

@Bean(name="camundaBpmDataSource")
@ConfigurationProperties(prefix="datasource.secondary")
public DataSource secondaryDataSource() {
  return DataSourceBuilder.create().build();
}
```

如果你不想使用`@Primary`事务管理器，可以创建一个单独的事务管理器，名称为`camundaBpmTransactionManager`，它将与用于Camunda的数据源（`@Primary`或`camundaBpmDataSource`）连接。

```java
@Bean
@Primary
public PlatformTransactionManager primaryTransactionManager() {
  return new JpaTransactionManager();
}

@Bean(name="camundaBpmTransactionManager")
public PlatformTransactionManager camundaTransactionManager(@Qualifier("camundaBpmDataSource") DataSource dataSource) {
  return new DataSourceTransactionManager(dataSource);
}
```

{{< note title="" class="warning" >}}
  数据源和事务管理器的beans必须匹配，即确保事务管理器实际管理着Camunda数据源。如果不是这样，流程引擎将对数据源连接使用自动提交模式，可能会导致数据库中的不一致。
{{< /note >}}

### `DefaultHistoryConfiguration`

配置应用于流程引擎的历史配置。如果没有配置，则使用历史级别[FULL]({{< ref "/user-guide/process-engine/history.md#choose-a-history-level" >}})。
如果你想使用一个自定义的`HistoryEventHandler`，你只需要提供一个实现接口的bean。

```java
@Bean
public HistoryEventHandler customHistoryEventHandler() {
  return new CustomHistoryEventHanlder();
}
```

### `DefaultHistoryLevelAutoHandlingConfiguration`
由于camunda版本>=7.4支持`自动历史级别`，这个配置增加了对<=7.3版本的支持。

为了对处理有更多的控制，你可以提供你自己的

- `org.camunda.bpm.spring.boot.starter.jdbc.HistoryLevelDeterminator` 名为`historyLevelDeterminator`。

重要提示：默认配置会在所有其他默认配置之后使用排序机制进行应用。

### `DefaultJobConfiguration`

将Job执行属性应用于流程引擎。

为了对执行本身有更多的控制，你可以提供你自己的

- `org.camunda.bpm.engine.impl.jobexecutor.JobExecutor`
- `org.springframework.core.task.TaskExecutor` 名为 `camundaTaskExecutor`

重要提示：Job执行器在配置中没有被启用。
这是在spring context成功加载后启动的 (see `org.camunda.bpm.spring.boot.starter.runlistener`).

### `DefaultDeploymentConfiguration`

如果启用了自动部署（默认情况下是这样的），在classpath中发现的所有进程都会被部署。
资源模式可以通过属性来改变（见 [properties](#camunda-engine-properties)).

### `DefaultJpaConfiguration`

如果JPA被启用并且配置了一个`entityManagerFactory`bean，那么流程引擎就会被启用以使用JPA（见[properties](#camunda-engine-properties)）。

### `DefaultAuthorizationConfiguration`

将授权配置应用于流程引擎。如果没有配置，则使用 "camunda" 的默认值（见[properties](#camunda-engine-properties)）。

## 覆盖默认的配置

提供一个实现标记接口之一的Bean。例如，自定义数据源的配置。

```java
@Configuration
public class MyCamundaConfiguration {

	@Bean
	public static CamundaDatasourceConfiguration camundaDatasourceConfiguration() {
		return new MyCamundaDatasourceConfiguration();
	}

}
```

## 添加额外的配置

你只需要提供一个或多个实现`org.camunda.bpm.engine.impl.cfg.ProcessEnginePlugin`接口的bean（或者`org.camunda.bpm.spring.boot.starter.configuration.impl.AbstractCamundaConfiguration`扩展）。
配置的应用是通过spring的排序机制（`@Order`注解和`Ordered`接口）进行排序。
因此，如果你希望你的配置在默认配置之前被应用，请在你的类中添加`@Order(Ordering.DEFAULT_ORDER - 1)`注解。
如果你想让你的配置在默认配置之后应用，请在你的类中添加一个`@Order(Order.DEFAULT_ORDER + 1)`注解。

```java
@Configuration
public class MyCamundaConfiguration {

	@Bean
	@Order(Ordering.DEFAULT_ORDER + 1)
	public static ProcessEnginePlugin myCustomConfiguration() {
		return new MyCustomConfiguration();
	}

}
```

Or, if you have component scan enabled:

```java
@Component
@Order(Ordering.DEFAULT_ORDER + 1)
public class MyCustomConfiguration implements ProcessEnginePlugin {

	@Override
	public void preInit(ProcessEngineConfigurationImpl processEngineConfiguration) {
		//...
	}

	...

}
```

or

```java

@Component
@Order(Ordering.DEFAULT_ORDER + 1)
public class MyCustomConfiguration extends AbstractCamundaConfiguration {

	@Override
	public void preInit(SpringProcessEngineConfiguration springProcessEngineConfiguration) {
		//...
	}

	...

}
```

## Camunda 引擎特性
除了基于Bean的覆盖流程引擎配置属性的方式之外，也可以通过<code>application.yaml</code>配置文件来设置这些属性。关于如何使用它的说明可以在 <a href="https://docs.camunda.org/get-started/spring-boot/configuration/">Spring Boot Starter 向导</a>中找到。

可用的特性如下：

<table class="table desc-table">
<tr>
<th>Prefix</th>
 <th>Property name</th>
 <th>Description</th>
  <th>Default value</th>
  </tr>
<tr><td colspan="4"><b>General</b></td></tr>

<tr><td rowspan="15"><code>camunda.bpm</code></td>
<td><code>.enabled</code></td>
<td>Switch to disable the Camunda auto-configuration. Use to exclude Camunda in integration tests.</td>
<td><code>true</code></td>
</tr>

<tr>
<td><code>.process-engine-name</code></td>
<td>Name of the process engine</td>
<td>Camunda default value</td>
</tr>

<tr>
<td><code>.generate-unique-process-engine-name</code></td>
<td>Generate a unique name for the process engine (format: 'processEngine' + 10 random alphanumeric characters)</td>
<td>false</td>
</tr>

<tr>
<td><code>.generate-unique-process-application-name</code></td>
<td>Generate a unique Process Application name for every Process Application deployment (format: 'processApplication' + 10 random alphanumeric characters)</td>
<td>false</td>
</tr>

<tr>
<td><code>.default-serialization-format</code></td>
<td>Default serialization format</td>
<td>Camunda default value</td>
</tr>

<tr>
<td><code>.history-level</code></td>
<td>Camunda history level</td>
<td>FULL</td>
</tr>

<tr>
<td><code>.history-level-default</code></td>
<td>Camunda history level to use when <code>history-level</code> is <code>auto</code>, but the level can not determined automatically</td>
<td>FULL</td>
</tr>

<tr>
<td><code>.auto-deployment-enabled</code></td>
<td>If processes should be auto deployed. This is disabled when using the SpringBootProcessApplication</td>
<td><code>true</code></td>
</tr>

<tr>
<td><code>.default-number-of-retries</code></td>
<td>Specifies how many times a job will be executed before an incident is raised</td>
<td><code>3</code></td>
</tr>

<tr>
<td><code>.job-executor-acquire-by-priority</code></td>
<td>If set to true, the job executor will acquire the jobs with the highest priorities</td>
<td><code>false</code></td>
</tr>

<tr>
<td><a name="license-file"></a><code>.license-file</code></td>
<td>Provides a URL to your Camunda license file and is automatically inserted into the DB when the application starts (but only if no valid license key is found in the DB).</br></br>
<b>Note:</b> This property is only available when using the <a href="{{<ref "/user-guide/spring-boot-integration/webapps.md#enterprise-webapps" >}}">camunda-bpm-spring-boot-starter-webapp-ee</a>
</td>
<td>By default, the license key will be loaded:
 <ol>
  <li>from the URL provided via the this property (if present)</li>
  <li>from the file with the name <code>camunda-license.txt</code> from the classpath (if present)</li>
  <li>from path <i>${user.home}/.camunda/license.txt</i> (if present)</li>
 </ol>
 The license must be exactly in the format as we sent it to you including the header and footer line. Bear in mind 
 that for some licenses there is a minimum <a href="{{<ref "/user-guide/license-use.md#license-compatibility" >}}">version requirement</a>.
</td>
</tr>

<tr>
<td><code>.id-generator</code></td>
<td>Configure idGenerator. Allowed values: <code>simple</code>, <code>strong</code>, <code>prefixed</code>. <code>prefixed</code> id generator is like <code>strong</code>, but uses a Spring application name (<code>${spring.application.name}</code>) as the prefix for each id.</td>
<td><code>strong</code></td>
</tr>

<tr>
<td><code>.version</code></td>
<td>Version of the process engine</td>
<td>Read only value, e.g., 7.4.0</td>
</tr>

<tr>
<td><code>.formatted-version</code></td>
<td>Formatted version of the process engine</td>
<td>Read only value, e.g., (v7.4.0)</td>
</tr>

<tr>
<td><code>.deployment-resource-pattern</code></td>
<td>Location for auto deployment</td>
<td><code>classpath*:**/*.bpmn, classpath*:**/*.bpmn20.xml, classpath*:**/*.dmn, classpath*:**/*.dmn11.xml, classpath*:**/*.cmmn, classpath*:**/*.cmmn10.xml, classpath*:**/*.cmmn11.xml</code></td>
</tr>

<tr><td colspan="4"><b>Job Execution</b></td></tr>

<tr>
<td rowspan="14"><code>camunda.bpm.job-execution</code></td>
<td><code>.enabled</code></td>
<td>If set to <code>false</code>, no JobExecutor bean is created at all. Maybe used for testing.</td>
<td><code>true</code></td>
</tr>

<tr>
<td><code>.deployment-aware</code></td>
<td>If job executor is deployment aware</td>
<td><code>false</code></td>
</tr>
<tr>
<td><code>.core-pool-size</code></td>
<td>Set to value > 1 to activate parallel job execution.</td>
<td><code>3</code></td>
</tr>
<tr>
<td><code>.keep-alive-seconds</code></td>
<td>Specifies the time, in milliseconds, for which threads are kept alive when there are no more tasks present. When the time expires, threads are terminated so that the core pool size is reached.</td>
<td><code>0</code></td>
</tr>
<tr>
<td><code>.lock-time-in-millis</code></td>
<td>Specifies the time in milliseconds an acquired job is locked for execution. During that time, no other job executor can acquire the job.</td>
<td><code>300000</code></td>
</tr>
<tr>
<td><code>.max-jobs-per-acquisition</code></td>
<td>Sets the maximal number of jobs to be acquired at once.</td>
<td><code>3</code></td>
</tr>
<tr>
<td><code>.max-pool-size</code></td>
<td>Maximum number of parallel threads executing jobs.</td>
<td><code>10</code></td>
</tr>
<tr>
<td><code>.queue-capacity</code></td>
<td>Sets the size of the queue which is used for holding tasks to be executed.</td>
<td><code>3</code></td>
</tr>
<tr>
<td><code>.wait-time-in-millis</code></td>
<td>Specifies the wait time of the job acquisition thread in milliseconds in case there are less jobs available for execution than requested during acquisition. If this is repeatedly the case, the wait time is increased exponentially by the factor <code>waitIncreaseFactor</code>. The wait time is capped by <code>maxWait</code>.</td>
<td><code>5000</code></td>
</tr>
<tr>
<td><code>.max-wait</code></td>
<td>Specifies the maximum wait time of the job acquisition thread in milliseconds in case there are less jobs available for execution than requested during acquisition.</td>
<td><code>60000</code></td>
</tr>
<tr>
<td><code>.backoff-time-in-millis</code></td>
<td>Specifies the wait time of the job acquisition thread in milliseconds in case jobs were acquired but could not be locked. This condition indicates that there are other job acquisition threads acquiring jobs in parallel. If this is repeatedly the case, the backoff time is increased exponentially by the factor <code>waitIncreaseFactor</code>. The time is capped by <code>maxBackoff</code>. With every increase in backoff time, the number of jobs acquired increases by <code>waitIncreaseFactor</code> as well.</td>
<td><code>0</code></td>
</tr>
<tr>
<td><code>.max-backoff</code></td>
<td>Specifies the maximum wait time of the job acquisition thread in milliseconds in case jobs were acquired but could not be locked.</td>
<td><code>0</code></td>
</tr>
<tr>
<td><code>.backoff-decrease-threshold</code></td>
<td>Specifies the number of successful job acquisition cycles without a job locking failure before the backoff time is decreased again. In that case, the backoff time is reduced by <code>waitIncreaseFactor</code>.</td>
<td><code>100</code></td>
</tr>
<tr>
<td><code>.wait-increase-factor</code></td>
<td>Specifies the factor by which wait and backoff time are increased in case their activation conditions are repeatedly met.</td>
<td><code>2</code></td>
</tr>

<tr><td colspan="4"><b>Datasource</b></td></tr>

<tr>
<td rowspan="5"><code>camunda.bpm.database</code></td>
<td><code>.schema-update</code></td>
<td>If automatic schema update should be applied, use one of [true, false, create, create-drop, drop-create]</td>
<td><code>true</code></td>
</tr>

<tr>
<td><code>.type</code></td>
<td>Type of the underlying database. Possible values: <code>h2</code>, mysql, mariadb, oracle, postgres, mssql, db2.</td>
<td>Will be automatically determined from datasource</td>
</tr>

<tr>
<td><code>.table-prefix</code></td>
<td>Prefix of the camunda database tables. Attention: The table prefix will <b>not</b> be applied if you  are using <code>schema-update</code>!</td>
<td><i>Camunda default value</i></td>
</tr>

<tr>
<td><code>.schema-name</code></td>
<td>The dataBase schema name</td>
<td><i>Camunda default value</i></td>
</tr>

<tr>
<td><code>.jdbc-batch-processing</code></td>
<td>Controls if the engine executes the jdbc statements as Batch or not.
It has to be disabled for some databases.
See the <a href="{{<ref "/user-guide/process-engine/database/database-configuration.md#jdbc-batch-processing" >}}">user guide</a> for further details.</td>
<td><i>Camunda default value: true</i></td>
</tr>

<tr><td colspan="4"><b>Eventing</b></td></tr>
<tr>
<td rowspan="3"><code>camunda.bpm.eventing</code></td>
<td><code>.execution</code></td>
<td>Enables eventing of delegate execution events.
See the <a href="{{<ref "/user-guide/spring-boot-integration/the-spring-event-bridge.md" >}}">user guide</a> for further details.</td>
<td><code>true</code></td>
</tr>

<tr>
<td><code>.history</code></td>
<td>Enables eventing of history events.
See the <a href="{{<ref "/user-guide/spring-boot-integration/the-spring-event-bridge.md" >}}">user guide</a> for further details.</td>
<td><code>true</code></td>
</tr>

<tr>
<td><code>.task</code></td>
<td>Enables eventing of task events.
See the <a href="{{<ref "/user-guide/spring-boot-integration/the-spring-event-bridge.md" >}}">user guide</a> for further details.</td>
<td><code>true</code></td>
</tr>



<tr><td colspan="4"><b>JPA</b></td></tr>
<tr>
<td rowspan="4"><code>camunda.bpm.jpa</code></td>
<td><code>.enabled</code></td>
<td>Enables jpa configuration</td>
<td><code>true</code>. Depends on <code>entityManagerFactory</code> bean.</td>
</tr>

<tr>
<td><code>.persistence-unit-name</code></td>
<td>JPA persistence unit name</td>
<td>-</td>
</tr>

<tr>
<td><code>.close-entity-manager</code></td>
<td>Close JPA entity manager</td>
<td><code>true</code></td>
</tr>

<tr>
<td><code>.handle-transaction</code></td>
<td>JPA handle transaction</td>
<td><code>true</code></td>
</tr>

<tr><td colspan="4"><b>Management</b></td></tr>
<tr>
<td><code>camunda.bpm.management</code></td>
<td><code>.health.camunda.enabled</code></td>
<td>Enables default camunda health indicators</td>
<td><code>true</code></td>
</tr>

<tr><td colspan="4"><b>Metrics</b></td></tr>
<tr>
<td rowspan="2"><code>camunda.bpm.metrics</code></td>
<td><code>.enabled</code></td>
<td>Enables metrics reporting</td>
<td><i>Camunda default value</i></td>
</tr>

<tr>
<td><code>.db-reporter-activate</code></td>
<td>Enables db metrics reporting</td>
<td><i>Camunda default value</i></td>
</tr>

<tr><td colspan="4"><b>Webapp</b></td></tr>
<tr>
<td rowspan="3"><code>camunda.bpm.webapp</code></td>
<td><code>.enabled</code></td>
<td>Switch to disable the Camunda Webapp auto-configuration.</td>
<td><code>true</code></td>
</tr>

<tr>
<td><code>.index-redirect-enabled</code></td>
<td>Registers a redirect from <code>/</code> to camunda's bundled <code>index.html</code>.
<br/>
If this property is set to <code>false</code>, the
<a href="https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-web-applications.html#boot-features-spring-mvc-welcome-page">default</a>
Spring Boot behaviour is taken into account.</td>
<td><code>true</code></td>
</tr>

<tr>
<td><code>.application-path</code></td>
<td>Changes the application path of the webapp.
<br/>
When setting to <code>/</code>, the legacy behavior of Camunda Spring Boot Starter <= 3.4.x is restored.
</td>
<td><code>/camunda</code></td>
</tr>

<tr id="csrf">
  <td rowspan="10"><code>camunda.bpm.webapp.csrf</code></td>
</tr>
<tr>
<td><code>.target-origin</code></td>
<td>Sets the application expected deployment domain. See the <a href="{{<ref "/webapps/shared-options/csrf-prevention.md" >}}">user guide</a> for details.</td>
<td><i>Not set</i></td>
</tr>
<tr>
<td><code>.deny-status</code></td>
<td>Sets the HTTP response status code used for a denied request. See the <a href="{{<ref "/webapps/shared-options/csrf-prevention.md" >}}">user guide</a> for details.</td>
<td><code>403</code></td>
</tr>
<tr>
<td><code>.random-class</code></td>
<td>Sets the name of the class used to generate tokens. See the <a href="{{<ref "/webapps/shared-options/csrf-prevention.md" >}}">user guide</a> for details.</td>
<td><code>java.security.SecureRandom</code></td>
</tr>
<tr>
<td><code>.entry-points</code></td>
<td>Sets additional URLs that will not be tested for the presence of a valid token. See the <a href="{{<ref "/webapps/shared-options/csrf-prevention.md" >}}">user guide</a> for details.</td>
<td><i>Not set</i></td>
</tr>
<tr>
  <td><code>.enable-secure-cookie</code></td>
  <td>
    If set to <code>true</code>, the cookie flag <a href="{{< ref "/webapps/shared-options/cookie-security.md#secure" >}}">Secure</a> is enabled.
  </td>
  <td><code>false</code></td>
</tr>
<tr>
  <td><code>.enable-same-site-cookie</code></td>
  <td>
    If set to <code>false</code>, the cookie flag <a href="{{< ref "/webapps/shared-options/cookie-security.md#samesite" >}}">SameSite</a> is disabled. The default value of the <code>SameSite</code> cookie is <code>LAX</code> and it can be changed via <code>same-site-cookie-option</code> configuration property.
  </td>
  <td><code>true</code></td>
</tr>
<tr>
  <td><code>.same-site-cookie-option</code></td>
  <td>
    Can be configured either to <code>STRICT</code> or <code>LAX</code>.<br><br>
    <strong>Note:</strong>
    <ul>
      <li>Is ignored when <code>enable-same-site-cookie</code> is set to <code>false</code></li>
      <li>Cannot be set in conjunction with <code>same-site-cookie-value</code></li>
    </ul>
  </td>
  <td><i>Not set</i></td>
</tr>
<tr>
  <td><code>.same-site-cookie-value</code></td>
  <td>
    A custom value for the cookie property.<br><br>
    <strong>Note:</strong>
    <ul>
      <li>Is ignored when <code>enable-same-site-cookie</code> is set to <code>false</code></li>
      <li>Cannot be set in conjunction with <code>same-site-cookie-option</code></li>
    </ul>
  </td>
  <td><i>Not set</i></td>
</tr>
<tr>
  <td><code>.cookie-name</code></td>
  <td>
      A custom value to change the cookie name.<br>
      <strong>Note:</strong> Please make sure to additionally change the cookie name for each webapp 
      (e. g. <a href="{{< ref "/webapps/cockpit/extend/configuration.md#change-csrf-cookie-name" >}}">Cockpit
      </a>) separately.
  </td>
  <td><code>XSRF-TOKEN</code></td>
</tr>

<tr id="header-security">
  <td rowspan="12"><code>camunda.bpm.webapp.header-security</code></td>
</tr>
<tr>
  <td><code>.xss-protection-disabled</code></td>
  <td>
    The header can be entirely disabled if set to <code>true</code>. <br>
    Allowed set of values is <code>true</code> and <code>false</code>.
  </td>
  <td><code>false</code></td>
</tr>
<tr>
  <td><code>.xss-protection-option</code></td>
  <td>
    The allowed set of values:
    <ul>
      <li><code>BLOCK</code>: If the browser detects a cross-site scripting attack, the page is blocked completely</li>
      <li><code>SANITIZE</code>: If the browser detects a cross-site scripting attack, the page is sanitized from suspicious parts (value <code>0</code>)</li>
    </ul>
    <strong>Note:</strong>
    <ul>
      <li>Is ignored when <code>.xss-protection-disabled</code> is set to <code>true</code></li>
      <li>Cannot be set in conjunction with <code>.xss-protection-value</code></li>
    </ul>
  </td>
  <td><code>BLOCK</code></td>
</tr>
<tr>
  <td><code>.xss-protection-value</code></td>
  <td>
    A custom value for the header can be specified.<br><br>
    <strong>Note:</strong>
    <ul>
      <li>Is ignored when <code>.xss-protection-disabled</code> is set to <code>true</code></li>
      <li>Cannot be set in conjunction with <code>.xss-protection-option</code></li>
    </ul>
  </td>
  <td><code>1; mode=block</code></td>
</tr>
<tr>
  <td><code>.content-security-policy-disabled</code></td>
  <td>
    The header can be entirely disabled if set to <code>true</code>. <br>
    Allowed set of values is <code>true</code> and <code>false</code>.
  </td>
  <td><code>false</code></td>
</tr>
<tr>
  <td><code>.content-security-policy-value</code></td>
  <td>
    A custom value for the header can be specified.<br><br>
    <strong>Note:</strong> Property is ignored when <code>.content-security-policy-disabled</code> is set to <code>true</code>
  </td>
  <td><code>base-uri 'self'</code></td>
</tr>
<tr>
  <td><code>.content-type-options-disabled</code></td>
  <td>
    The header can be entirely disabled if set to <code>true</code>. <br>
    Allowed set of values is <code>true</code> and <code>false</code>.
  </td>
  <td><code>false</code></td>
</tr>
<tr>
  <td><code>.content-type-options-value</code></td>
  <td>
    A custom value for the header can be specified.<br><br>
    <strong>Note:</strong> Property is ignored when <code>.content-security-policy-disabled</code> is set to <code>true</code>
  </td>
  <td><code>nosniff</code></td>
</tr>
<tr>
  <td><code>.hsts-disabled</code></td>
  <td>
      Set to <code>false</code> to enable the header. The header is disabled by default.<br>
      Allowed set of values is <code>true</code> and <code>false</code>. 
  </td>
  <td><code>true</code></td>
</tr>
<tr>
  <td><code>.hsts-max-age</code></td>
  <td>
      Amount of seconds, the browser should remember to access the webapp via HTTPS.<br><br>
      <strong>Note:</strong>
      <ul>
        <li>Corresponds by default to one year</li>
        <li>Is ignored when <code>hstsDisabled</code> is <code>true</code></li>
        <li>Cannot be set in conjunction with <code>hstsValue</code></li>
        <li>Allows a maximum value of 2<sup>31</sup>-1</li>
      </ul>
  </td>
  <td><code>31536000</code></td>
</tr>
<tr>
  <td><code>.hsts-include-subdomains-disabled</code></td>
  <td>
      HSTS is additionally to the domain of the webapp enabled for all its subdomains.<br><br>
      <strong>Note:</strong>
      <ul>
        <li>Is ignored when <code>hstsDisabled</code> is <code>true</code></li>
        <li>Cannot be set in conjunction with <code>hstsValue</code></li>
      </ul>
  </td>
  <td><code>true</code></td>
</tr>
<tr>
  <td><code>.hsts-value</code></td>
  <td>
      A custom value for the header can be specified.<br><br>
      <strong>Note:</strong>
      <ul>
        <li>Is ignored when <code>hstsDisabled</code> is <code>true</code></li>
        <li>Cannot be set in conjunction with <code>hstsMaxAge</code> or 
        <code>hstsIncludeSubdomainsDisabled</code></li>
      </ul>
  </td>
  <td><code>max-age=31536000</code></td>
</tr>

<tr><td colspan="4"><b>Authorization</b></td></tr>
<tr>
<td rowspan="4"><code>camunda.bpm.authorization</code></td>
<td><code>.enabled</code></td>
<td>Enables authorization</td>
<td><i>Camunda default value</i></td>
</tr>

<tr>
<td><code>.enabled-for-custom-code</code></td>
<td>Enables authorization for custom code</td>
<td><i>Camunda default value</i></td>
</tr>

<tr>
<td><code>.authorization-check-revokes</code></td>
<td>Configures authorization check revokes</td>
<td><i>Camunda default value</i></td>
</tr>

<tr>
<td><code>.tenant-check-enabled</code></td>
<td>Performs tenant checks to ensure that an authenticated user can only access data that belongs to one of his tenants.</td>
<td><code>true</code></td>
</tr>

<tr><td colspan="4"><b>Admin User</b></td></tr>
<tr>
<td rowspan="3"><code>camunda.bpm.admin-user</code></td>
<td><code>.id</code></td>
<td>The username (e.g., 'admin')</td>
<td>-</td>
</tr>

<tr>
<td><code>.password</code></td>
<td>The initial password</td>
<td>=<code>id</code></td>
</tr>

<tr>
<td><code>.firstName</code>, <code>.lastName</code>, <code>.email</code></td>
<td>Additional (optional) user attributes</td>
<td>Defaults to value of 'id'</td>
</tr>

<tr><td colspan="4"><b>Filter</b></td></tr>
<tr>
<td><code>camunda.bpm.filter</code></td>
<td><code>.create</code></td>
<td>Name of a "show all" filter. If set, a new filter is created on start that displays all tasks. Useful for testing on h2 db.</td>
<td>-</td>
</tr>

</table>


### 通用属性

上面描述的配置方法并没有涵盖所有可用的流程引擎属性。要覆盖任何未公开的流程引擎配置属性（即上面列出的），你可以使用generic-perties。

```yaml
camunda:
  bpm:
    generic-properties:
      properties:
        ...
```

{{< note title="笔记:" class="info" >}}
  使用<code>generic-properties</code>关键字重写一个已经公开的属性不会影响流程引擎的配置。所有公开的属性只能用其公开的标识符来重写。
{{< /note >}}

### 案例
使用公开的属性重写配置：

```yaml
camunda.bpm:
  admin-user:
    id: kermit
    password: superSecret
    firstName: Kermit
  filter:
    create: All tasks
```
使用通用属性覆盖配置。

```yaml
camunda:
  bpm:
    generic-properties:
      properties:
        enable-password-policy: true
```

## Session Cookie

你可以通过`application.yaml`配置文件为Spring Boot应用程序配置 *Session Cookie*。

Camunda Spring Boot Starter版本：

<= 2.3 (Spring Boot version 1.x)

```yaml
server:
  session:
    cookie:
      secure: true
      http-only: true # 在 1.5.14 版本之前不可用
```

\>= 3.0 (Spring Boot version 2.x)

```yaml
server:
  servlet:
    session:
      cookie:
        secure: true
        http-only: true # 在 2.0.3 版本之前不可用
```

## 配置 Spin 数据格式

当在classpath上检测到`camunda-spin-dataformat-jon-jackson`依赖时，Camunda Spring Boot Starter会自动配置Spin Jackson Json DataFormat。
`camunda-spin-dataformat-jon-jackson`的依赖关系时，就会自动配置Spin Jackson Json数据格式。要包含一个
`DataFormatConfigurator`的所需Jackson Java 8模块，适当的依赖也需要包含在classpath中。请注意，`camunda-engine-plugin-spin`也需要作为一个依赖项被包含，以便自动配置器能够工作。

自动配置目前支持以下[Jackson Java 8模块](https://github.com/FasterXML/jackson-modules-java8):

1. Parameter names (`jackson-module-parameter-names`)
2. Java 8 Date/time (`jackson-datatype-jdk8`)
3. Java 8 Datatypes (`jackson-datatype-jsr310`)

{{< note title="小心!" class="warning" >}}
当使用`camunda-spin-dataformat-all`作为依赖时，Spin Jackson Json DataFormat的自动配置被禁用。`camunda-spin-dataformat-all`工件会覆盖Jackson库，这破坏了与普通Jackson模块的兼容性。如果有必要使用`camunda-spin-dataformat-all`，请使用[Spin Custom DataFormat configuration]({{< ref "/reference/spin/extending-spin.md#custom-dataformats" >}})的标准方法。
{{< /note >}}

例如，为了在Spin中提供对Java 8日期/时间类型的支持，需要在Spring Boot应用程序的`pom.xml`文件中添加以下依赖，以及相应的版本标记。
 
 ```xml
<dependencies>
    <dependency>
      <groupId>org.camunda.bpm</groupId>
      <artifactId>camunda-engine-plugin-spin</artifactId>
    </dependency>
    <dependency>
      <groupId>org.camunda.spin</groupId>
      <artifactId>camunda-spin-dataformat-json-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.datatype</groupId>
      <artifactId>jackson-datatype-jdk8</artifactId>
    </dependency>
</dependencies>
```

Spring Boot还提供了一些不错的配置属性，以进一步配置Jackson `ObjectMapper`。它们可以在[这里](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#howto-customize-the-jackson-objectmapper)找到。

为了提供额外的配置，需要进行以下操作。

1. 提供一个`org.camunda.spin.spi.DataFormatConfigurator'的自定义实现。
1. 在`META-INF/spring.plants`文件中添加接口和实现的全限定类名的适当键值对。
1. 确保包含配置器的组件可以从Spin的classloader中到达。
 
