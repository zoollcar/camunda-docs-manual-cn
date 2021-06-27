---

title: '部署缓存'
weight: 150

menu:
  main:
    identifier: "user-guide-process-engine-pd-cache"
    parent: "user-guide-process-engine"

---
---

所有的流程定义都是缓存的（在它们被解析后），以避免每次需要流程定义时都查询数据库，因为流程定义数据不会改变。这减少了引用流程定义的延迟，从而提高了系统的性能。

# 自定义缓存的最大容量

如果有很多流程定义，缓存可能会占用大量的内存，缓存容量可能达到极限。因此，在达到最大容量后，最近使用最少的流程定义条目会从高速缓存中被驱逐，以满足容量条件。然而，如果仍然遇到内存不足的问题，可能需要降低缓存的最大容量。

改变最大容量，会影响到以下所有的缓冲区组件：

 * 流程定义
 * 案例定义
 * 决策定义
 * 决策需求定义
   
在流程引擎的配置中，可以指定缓存的最大容量。默认值是 *1000* 。当流程引擎被创建时，这个属性将被设置，所有的资源将被相应地扫描和部署。作为一个例子，最大容量可以被设置为*120*，如下所示：

```xml
<bean id="processEngineConfiguration" class="org.camunda.bpm.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration">
	<!-- Your property definitions! -->
					....
					
	<property name="cacheCapacity" value="120" />  
</bean>
```

__注意:__ 上述的所有组件都将使用相同的容量，不能为每个组件单独设置容量大小。此外，在默认的缓存实现中，将容量大小与缓存中使用的最大元素数相对应。这意味着，你所使用的物理存储的绝对数量（如百万字节）取决于各自流程定义所需的大小。


# 提供自定义缓存实现

缓存的默认实现是在超过最大容量后立即释放最久未使用的条目。如果有必要通过不同的标准来选择被释放的缓存条目，可以提供自己的缓存实现。

我们可以通过实现 *org.camunda.util.commons* 包中的Cache接口来做到这一点。例如，我们假设已经实现了下面这个类：

```java
public class MyCacheImplementation<K, V> implements Cache<K, V> {
	
	// implement interface methods and your own cache logic here
}
```

接下来，我们需要将 *MyCacheImplementation* 插入到一个自定义的 *CacheFactory* 中：

```java
public class MyCacheFactory extends CacheFactory {

  @Override
  public <T> Cache<String, T> createCache(int maxNumberOfElementsInCache) {
    return new MyCacheImplementation<String, T>(maxNumberOfElementsInCache);
  }
}
```
    
这个工厂类被用来为不同的缓存组件提供缓存实现，如流程定义或案例定义。一旦完成这些，就可以使用流程引擎配置，在这里可以指定一组资源。当流程引擎被创建时，所有这些资源将被扫描和部署。在给定的例子中，自定义缓存工厂现在可以按以下方式部署：

```xml
<bean id="processEngineConfiguration" class="org.camunda.bpm.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration">
	<!-- Your property definitions! -->
					....

	<property name="cacheFactory">
			<bean class="org.camunda.bpm.engine.test.api.cfg.MyCacheFactory" />
	</property>
</bean>
```




