---

title: '密码格式要求'
weight: 155

menu:
  main:
    identifier: "user-guide-process-engine-password-policy"
    parent: "user-guide-process-engine"

---
本章是关于为引擎管理的用户账户配置和使用密码策略。一个密码策略可以确保只允许符合某些标准的密码。一个策略可以由任何数量的规则组成。违反策略中的一条规则会导致错误，用户不会被保存。

从7.11.0版本开始，引擎自带一个标准的密码策略，默认情况下是禁用的，必须配置才能使用。

**注意:** 这只适用于在Camunda引擎内管理的用户。如果你使用LDAP进行用户管理，那么密码策略对这些用户没有影响。

# 内置密码策略

内置的密码策略要求所有的密码必须符合以下条件：

* 不能包含用户资料（即用户ID、名字、姓氏、电子邮件）
* 最小长度为10个字符
* 至少1个大写字符
* 至少1个小写字母
* 至少1个数字
* 至少1个特殊字符

# 自定义密码策略

你可以使用流程引擎配置来启用/禁用密码策略或插入一个自定义策略。请参阅[Process Engine Bootstrapping](../process-engine-bootstrapping)，了解如何为你的Camunda环境设置属性。

要启用或禁用密码策略检查，你需要设置 `enablePasswordPolicy` 属性。

如果你想使用自定义的密码策略，需要实现`org.camunda.bpm.engine.identity`包中的`PasswordPolicy`和`PasswordPolicyRule`接口，并设置`passwordPolicy`属性向流程引擎配置提供你的实现类。

```java
public class MyPasswordPolicy implements PasswordPolicy {

  @Override
  public List<PasswordPolicyRule> getRules() {
    // create rules
  }
}
```
```java
public class MyPasswordPolicyRule implements PasswordPolicyRule {

  @Override
  public String getPlaceholder() {
    // 此占位符可用于显示国际化的错误消息。
    return "PASSWORD_POLICY_RULE_PLACEHOLDER";
  }

  @Override
  public Map<String, String> getParameters() {
    // 这些参数可以注入错误消息。
  }

  @Override
  public boolean execute(String candidatePassword, User user) {
    // 验证候选密码
    // 如果有效则返回true，否则返回false
  }
}
```
通过`getPlaceholder`和`getParameters`提供规则占位符和参数，自定义前端可以根据规则和它们的配置显示错误信息。例如，"密码必须至少有X个字符的长度"，X是可配置的，并在参数图中传递）。

一个规则的`执行'方法检查输入的密码是否符合这个规则。当试图保存一个用户时，它被执行。