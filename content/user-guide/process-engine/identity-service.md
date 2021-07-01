---

title: '身份服务 Identity Service'
weight: 230

menu:
  main:
    identifier: "user-guide-process-engine-identity-service"
    parent: "user-guide-process-engine"

---


身份服务是对各种用户/组库的API抽象。其基本实体是：

* User: 使用不同ID区分的不同用户
* Group: 使用不同ID区分的不同组
* Membership: 组与用户之间的关系
* Tenant: 使用不同ID区分的不同租户
* Tenant Membership: 租户与 用户/组 之间的关系

实例：

```java
User demoUser = processEngine.getIdentityService()
  .createUserQuery()
  .userId("demo")
  .singleResult();
```

Camunda平台区分了只读和可写的用户资源库。只读的用户资源库提供对基础用户/组数据库的只读访问。可写用户资源库允许对用户数据库进行写访问，包括创建、更新和删除用户和组。

也提供一个自定义的身份提供者实现，可以实现如下接口：

* {{< javadocref page="?org/camunda/bpm/engine/impl/identity/ReadOnlyIdentityProvider.html" text="org.camunda.bpm.engine.impl.identity.ReadOnlyIdentityProvider" >}}
* {{< javadocref page="?org/camunda/bpm/engine/impl/identity/WritableIdentityProvider.html" text="org.camunda.bpm.engine.impl.identity.WritableIdentityProvider" >}}

# 用户、组和租户ID的自定义白名单

用户、组和租户ID可以与白名单相匹配，以确定所提供的ID是否可以接受。默认的（全局）正则表达式匹配模式是 **"[a-zA-Z0-9]+|camunda-admin "** ，即任何字母数字值的组合或 _'camunda-admin'_ 。

如果你的组织允许使用额外的字符（例如：特殊字符），应该在流程引擎的配置文件ProcessEngineConfiguration属性`generalResourceWhitelistPattern`中设置适当的模式。可以使用标准的[Java 正则表达式](https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html)语法。例如，要接受所有字符，可以使用以下属性值。

```xml
<property name="generalResourceWhitelistPattern" value=".+"/>
```

通过使用适当的配置方式，可以为用户、组和租户ID定义不同的匹配模式。

```xml
<property name="userResourceWhitelistPattern" value="[a-zA-Z0-9-]+" />
<property name="groupResourceWhitelistPattern" value="[a-zA-Z]+" />
<property name="tenantResourceWhitelistPattern" value=".+" />
```

请注意，如果没有定义某种模式（例如租户白名单模式），将使用一般模式，要么是默认模式（`"[a-zA-Z0-9]+|camunda-admin"`），要么是配置文件中定义的。   

# 基于数据库的身份服务

数据库身份服务使用流程引擎数据库来管理用户和组。如果没有提供其他身份服务实现，这是默认的身份服务实现。

数据库身份服务同时实现了 "只读身份提供者" 和 "可写身份提供者"，在用户、组和会员中提供完整的CRUD功能。


# 基于 LDAP 的身份服务

LDAP身份服务提供对基于LDAP的用户/组资源库的只读访问。该身份服务提供方式作为[流程引擎插件]({{< ref "/user-guide/process-engine/process-engine-plugins.md" >}})实现，可以被添加到流程引擎配置中。添加后，它会取代默认的数据库身份服务。

使用LDAP身份服务，需要将`camunda-identity-ldap.jar`库添加到流程引擎的classloader中。

{{< note title="" class="info" >}}
  请导入[Camunda BOM](/get-started/apache-maven/)，以确保每个Camunda项目的版本正确。
{{< /note >}}

```xml
<dependency>
  <groupId>org.camunda.bpm.identity</groupId>
  <artifactId>camunda-identity-ldap</artifactId>
</dependency>
```

## 激活LDAP插件

以下是如何使用 Spring XML 配置 LDAP Identity Provider Plugin 的示例：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans   http://www.springframework.org/schema/beans/spring-beans.xsd">
  <bean id="processEngineConfiguration" class="org.camunda.bpm.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration">
    ...
    <property name="processEnginePlugins">
      <list>
        <ref bean="ldapIdentityProviderPlugin" />
      </list>
    </property>
  </bean>
  <bean id="ldapIdentityProviderPlugin" class="org.camunda.bpm.identity.impl.ldap.plugin.LdapIdentityProviderPlugin">
    <property name="serverUrl" value="ldap://localhost:3433/" />
    <property name="managerDn" value="uid=daniel,ou=office-berlin,o=camunda,c=org" />
    <property name="managerPassword" value="daniel" />
    <property name="baseDn" value="o=camunda,c=org" />

    <property name="userSearchBase" value="" />
    <property name="userSearchFilter" value="(objectclass=person)" />
    <property name="userIdAttribute" value="uid" />
    <property name="userFirstnameAttribute" value="cn" />
    <property name="userLastnameAttribute" value="sn" />
    <property name="userEmailAttribute" value="mail" />
    <property name="userPasswordAttribute" value="userpassword" />

    <property name="groupSearchBase" value="" />
    <property name="groupSearchFilter" value="(objectclass=groupOfNames)" />
    <property name="groupIdAttribute" value="ou" />
    <property name="groupNameAttribute" value="cn" />
    <property name="groupMemberAttribute" value="member" />

    <property name="authorizationCheckEnabled" value="false" />
  </bean>
</beans>
```

以下是如何在 bpm-platform.xml/processes.xml 中配置 LDAP Identity Provider Plugin 的示例：

```xml
<process-engine name="default">
  <job-acquisition>default</job-acquisition>
  <configuration>org.camunda.bpm.engine.impl.cfg.StandaloneProcessEngineConfiguration</configuration>
  <datasource>java:jdbc/ProcessEngine</datasource>

  <properties>...</properties>

  <plugins>
    <plugin>
      <class>org.camunda.bpm.identity.impl.ldap.plugin.LdapIdentityProviderPlugin</class>
      <properties>

        <property name="serverUrl">ldap://localhost:4334/</property>
        <property name="managerDn">uid=jonny,ou=office-berlin,o=camunda,c=org</property>
        <property name="managerPassword">s3cr3t</property>

        <property name="baseDn">o=camunda,c=org</property>

        <property name="userSearchBase"></property>
        <property name="userSearchFilter">(objectclass=person)</property>

        <property name="userIdAttribute">uid</property>
        <property name="userFirstnameAttribute">cn</property>
        <property name="userLastnameAttribute">sn</property>
        <property name="userEmailAttribute">mail</property>
        <property name="userPasswordAttribute">userpassword</property>

        <property name="groupSearchBase"></property>
        <property name="groupSearchFilter">(objectclass=groupOfNames)</property>
        <property name="groupIdAttribute">ou</property>
        <property name="groupNameAttribute">cn</property>

        <property name="groupMemberAttribute">member</property>

        <property name="authorizationCheckEnabled">false</property>

      </properties>
    </plugin>
  </plugins>

</process-engine>
```

{{< note title="管理员授权插件" class="info" >}}
LDAP Identity Provider Plugin 通常与[Administrator Authorization Plugin]({{< ref "/user-guide/process-engine/authorization-service.md#the-administrator-authorization-plugin" >}}) 结合使用 它允许你为特定的 LDAP用户/组 授予管理员权限。
{{< /note >}}

{{< note title="多租户" class="info" >}}
目前，LDPA 身份服务不支持 [多租户]({{< ref "/user-guide/process-engine/multi-tenancy.md#single-process-engine-with-tenant-identifiers" >}})。 这意味着无法从 LDAP 获取租户，并且默认情况下透明的多租户访问限制不起作用。
{{< /note >}}

## LDAP 插件的可配置属性

LDAP身份提供程序提供以下可配置属性：

<table class="table table-striped">
  <tr>
    <th>属性</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>serverUrl</code></td>
    <td>要连接的 LDAP 服务器的 URL。</td>
  </tr>
  <tr>
    <td><code>managerDn</code></td>
    <td>LDAP目录管理员用户的managerDn。</td>
  </tr>
  <tr>
    <td><code>managerPassword</code></td>
    <td>LDAP目录管理员用户密码</td>
  </tr>
  <tr>
    <td><code>baseDn</code></td>
    <td>
      <p>base DN：标识 LDAP 目录的根。 附加到为搜索用户或组而组成的所有 DN 名称。</p>
      <p><em>例如：</em> <code>o=camunda,c=org</code></p>
    </td>
  </tr>
  <tr>
    <td><code>userSearchBase</code></td>
    <td>
      <p>标识 LDAP树 中插件应在其下搜索用户的节点。必须对应
      <code>baseDn</code>。</p>
      <p><em>例如：</em> <code>ou=employees</code></p>
    </td>
  </tr>
  <tr>
    <td><code>userSearchFilter</code></td>
    <td>
      <p>搜索用户时使用的 LDAP 查询字符串。 <em>例如:</em> <code>(objectclass=person)</code></p>
    </td>
  </tr>
  <tr>
    <td><code>userIdAttribute</code></td>
    <td>
      <p>用户ID属性的名称。 <em>例如:</em> <code>uid</code></p>
    </td>
  </tr>
  <tr>
    <td><code>userFirstnameAttribute</code></td>
    <td>
      <p>firstname 属性的名称。<em>例如:</em> <code>cn</code></p>
    </td>
  </tr>
  <tr>
    <td><code>userLastnameAttribute</code></td>
    <td>
      <p>lastname 属性的名称。<em>例如:</em> <code>sn</code></p>
    </td>
  </tr>
  <tr>
    <td><code>userEmailAttribute</code></td>
    <td>
      <p>email 属性的名称 <em>例如:</em> <code>mail</code></p>
    </td>
  </tr>
  <tr>
    <td><code>userPasswordAttribute</code></td>
    <td>
      <p>password 属性的名称 <em>例如:</em> <code>userpassword</code></p>
    </td>
  </tr>
  <tr>
    <td><code>groupSearchBase</code></td>
    <td>
      <p>标识 LDAP 树中插件应在其下搜索组的节点。 必须对应
      <code>baseDn</code>.</p>
      <p><em>例如：</em> <code>ou=roles</code></p>
    </td>
  </tr>
  <tr>
    <td><code>groupSearchFilter</code></td>
    <td>
      <p>搜索组时使用的 LDAP 查询字符串。 <em>例如：</em> <code>(objectclass=groupOfNames)</code></p>
    </td>
  </tr>
  <tr>
    <td><code>groupIdAttribute</code></td>
    <td>
      <p>组 Id 属性的名称。 <em>例如：</em> <code>ou</code></p>
    </td>
  </tr>
  <tr>
    <td><code>groupNameAttribute</code></td>
    <td>
      <p>组 Name 属性的名称。 <em>例如：</em> <code>cn</code></p>
    </td>
  </tr>
  <tr>
    <td><code>groupTypeAttribute</code></td>
    <td><p>组 Type 属性的名称。 <em>例如：</em> <code>cn</code></p></td>
  </tr>
  <tr>
    <td><code>groupMemberAttribute</code></td>
    <td>
      <p>成员属性的名称。 <em>例如：</em> <code>member</code></p>
    </td>
  </tr>
  <tr>
    <td><code>acceptUntrustedCertificates</code></td>
    <td>
      <p>如果 LDAP 服务器使用 SSL，则接受不被信任的证书。 <strong>警告：</strong> 我们强烈建议不要使用此属性。 最好将不被信任的证书安装到 JDK 密钥库。</p>
    </td>
  </tr>
  <tr>
    <td><code>useSsl</code></td>
    <td>
      <p>如果 LDAP 连接使用 SSL，则设置为 true。 <em>默认：</em> <code>false</code></p>
    </td>
  </tr>
  <tr>
    <td><code>initialContextFactory</code></td>
    <td>
      <p> <code>java.naming.factory.initial</code> 属性的值。 <em>默认:</em> <code>com.sun.jndi.ldap.LdapCtxFactory</code></p>
    </td>
  </tr>
  <tr>
    <td><code>securityAuthentication</code></td>
    <td>
      <p><code>java.naming.security.authentication</code> 属性的值. <em>默认:</em> <code>simple</code></p>
    </td>
  </tr>
  <tr>
    <td><code>usePosixGroups</code></td>
    <td>
      <p>表示是否使用 posix 组。 如果为 true，当按组成员而不是完整 DN 查询组时，连接器将使用一个简单的（非限定）用户 ID。
         <em>默认:</em> <code>false</code>
      </p>
    </td>
  </tr>
  <tr>
    <td><code>allowAnonymousLogin</code></td>
    <td>
      <p>
        允许匿名登录，无需密码。 <em>默认:</em> <code>false</code>
      </p>
      <p>
        <strong>警告:</strong> 我们强烈建议不要使用此属性。 你应该配置你的 LDAP 使用无需匿名登录的简单身份验证。
      </p>
    </td>
  </tr>
  <tr>
    <td><code>authorizationCheckEnabled</code></td>
    <td>
      <p>
        如果这个属性设置为 <code>true</code>，那么在查询用户或组时就执行授权检查。 否则在查询用户或组时不执行授权检查。 <em>默认:</em> <code>true</code>
      </p>
      <p>
        <strong>注意:</strong> 如果你有大量 LDAP 用户或组，我们建议将此属性设置为 <code>false</code> 以提高用户和组查询的性能。
      </p>
    </td>
  </tr>
  <tr>
   <td><code>sortControlSupported</code></td>
   <td>
      <p>
        如果此属性设置为 <code>true</code>，则启用搜索结果排序。 否则，搜索查询中的 orderBy 子句将被简单地忽略。
         <em>默认：</em> <code>false</code>
      </p>
      <p>
       <strong>注意：</strong> 搜索结果排序的支持并不是每个 LDAP 服务器都实现的。
         确保你当前使用的 LDAP 服务器实现了 <a href="https://tools.ietf.org/html/rfc2891">RFC 2891</a>。
      </p>
    </td>
  </tr>
</table>

# 限制登录尝试

存在一种机制来防止后续不成功的登录尝试。其本质是用户在登录尝试不成功后的特定时间内无法登录。
每次尝试后都会计算时间量，但受最大延迟时间限制。
在预定义的失败尝试次数后，用户将被锁定，只有管理员有权[解锁]({{< ref "/webapps/admin/user-management.md" >}}) 他们。

该机制可以使用以下属性和它们的默认值进行配置。

* `loginMaxAttempts=10`
* `loginDelayFactor=2`
* `loginDelayMaxTime=60`
* `loginDelayBase=3`

有关更多信息，请查看流程引擎的 [登录属性]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#login-parameters" >}}) 部分。

延迟的计算通过以下公式完成：<code>baseTime * factor^(attempt-1)</code>。
默认配置的行为将是：
第一次尝试失败后延迟 3 秒，第二次尝试后延迟 6 秒，12 秒、24 秒、48 秒、60 秒、60 秒等。 第 10 次尝试后，如果用户再次登录失败，用户将被锁定。

## LDAP 细节

如果你的引擎上有 LDAP 设置，则需要处理 LDAP 端的限制。 你系统中的登录机制不会受到上述属性的影响。
