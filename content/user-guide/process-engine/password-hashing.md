---

title: '密码哈希加密'
weight: 150

menu:
  main:
    identifier: "user-guide-process-engine-password-hashing"
    parent: "user-guide-process-engine"

---

本章是关于Camunda平台中如何进行密码哈希的。特别是，正在使用的哈希算法和盐的生成。如果你不熟悉这些主题，我们建议阅读有关[加密哈希函数百科](https://en.wikipedia.org/wiki/Cryptographic_hash_function)、 [盐](https://en.wikipedia.org/wiki/Salt_(cryptography)) 和[安全密码哈希](https://crackstation.net/hashing-security.htm) 的相关文章。

Camunda 7.6版和更早的版本使用加密哈希函数 [SHA-1](https://en.wikipedia.org/wiki/SHA-1)。 从Camunda 7.7版本开始，开始使用哈希函数 [SHA-512](https://en.wikipedia.org/wiki/SHA-2)。 如果需要另一个自定义哈希函数，可以在Camunda中插入一个[自定义密码哈希算法](#自定义哈希算法) 。

在盐的生成过程中，每个用户都会生成16字节的随机值，它是用[SecureRandom](http://docs.oracle.com/javase/6/docs/api/java/security/SecureRandom.html)生成的。 如果需要，也可以[自定义盐的生成](#自定义盐的生成)。

# 自定义哈希算法

如果有必要使用更安全的哈希算法，你可以提供你自己的实现。

你可以通过实现`org.camunda.bpm.engine.impl.digest`包中的`PasswordEncryptor`接口来做到这一点。该接口确保所有密码哈希的必要功能都能实现。你可以看看`org.camunda.bpm.engine.impl.digest`包中的`Base64EncodedHashDigest`和`ShaHashDigest`类，看看Camunda中是如何实现的。你自己实现的模板可以是这样的：


```java
public class MyPasswordEncryptor implements PasswordEncryptor {

  @Override
  public String encrypt(String password) {
    // do something
  }

  @Override
  public boolean check(String password, String encrypted) {
    // do something
  }
  
  @Override
  public String hashAlgorithmName() {
	// This name is used to resolve the algorithm used for the encryption of a password.
	return "NAME_OF_THE_ALGORITHM";
  }
}
```

一旦完成，你可以使用流程引擎配置，通过设置`passwordEncryptor`属性到你的自定义实现，例如`MyPasswordEncryptor`，来插入自定义实现。请参阅[Process Engine Bootstrapping](.../process-engine-bootstrapping)，了解你必须为你的Camunda环境设置该属性的地方。

请注意，即使你已经创建了用其他算法加密的用户，例如，旧的自定义算法或Camunda默认的哈希算法`SHA-512`，他们仍然可以被引擎自动解决，尽管你后来添加了你的自定义算法。属性`customPasswordChecker`是一个用于检查（旧）密码的哈希算法的列表。Camunda默认的哈希算法是自动添加的，所以请只在该列表中添加你之前自定义的`passwordEncryptor`实现。

{{< note title="注意!" class="info" >}}

请不要使用你自己的哈希函数的实现，而应使用经过同行评审的标准!

{{< /note >}}


# 自定义盐的生成

与哈希算法类似，盐的生成也可以进行调整。首先，实现`org.camunda.bpm.engine.impl.digest`中的`SaltGenerator`接口。这可以确保所有必要的功能都被实现。你可以看看`org.camunda.bpm.engine.impl.digest`包中的`Base64EncodedSaltGenerator`和`Default16ByteSaltGenerator`类，看看Camunda中是如何实现的。你自己实现的模板可以是这样的：

```java
public class MyCustomSaltGenerator implements SaltGenerator {

  @Override
  public String generateSalt() {
    // do something
  }
}
```

一旦完成，你可以使用流程引擎配置，通过设置 "saltGenerator "属性来插入自定义的实现，例如 "MyCustomSaltGenerator"。请参阅[Process Engine Bootstrapping](.../process-engine-bootstrapping)，了解你必须为你的Camunda环境设置该属性的地方。




