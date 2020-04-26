# Spring-IOC-从单例缓存中获取单例 Bean


## 1 `getSingleton`

```java
//DefaultSingletonBeanRegistry.java

public Object getSingleton(String beanName) {
    	// allowEarlyReference 参数，allowEarlyReference 表示是否允许其他 bean 引用
    	// 参数true设置标识允许早期依赖
		return getSingleton(beanName, true);
	}

@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// <1> 从单例缓存中加载 bean
		Object singletonObject = this.singletonObjects.get(beanName);
		// 缓存中的 bean 为空，且当前 bean 正在创建
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				// <2> 从 earlySingletonObjects 获取
				singletonObject = this.earlySingletonObjects.get(beanName);
				// earlySingletonObjects 中没有，且允许提前创建
				if (singletonObject == null && allowEarlyReference) {
					// <3> 从 singletonFactories 中获取对应的 ObjectFactory
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						// 添加 bean 到 earlySingletonObjects 中
						this.earlySingletonObjects.put(beanName, singletonObject);
						// 从 singletonFactories 中移除对应的 ObjectFactory
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

上面代码的分析:

1. <1> 处，从 `singletonObjects` 中，获取 Bean 对象。
2. <2> 处，若为空且当前 bean 正在创建中，则从 `earlySingletonObjects` 中获取 Bean 对象。
3. <3> 处，若为空且允许提前创建，则从 `singletonFactories` 中获取相应的 `ObjectFactory` 对象。若`ObjectFactory` 不为空，则调用其 `ObjectFactory#getObject(String name)` 方法，创建 Bean 对象，然后将其加入到 `earlySingletonObjects` ，然后从 `singletonFactories` 删除。

上面代码涉及到了好几个缓存集合，补充一下这几个缓存集合的作用。

*   缓存集合的定义：

```java
//DefaultSingletonBeanRegistry.java

/** 存放的是单例 bean 的映射。对应关系为 bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** 
存放的是早期的 bean，对应关系也是 bean name --> bean instance  
与singletonObjects的不同之处在于，当一个单例bean被放到这里面后，那么当bean还在创建过程中，就可以通过getBean方法获取到了，其目的是用来检测循环引用。
*/
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

/** 存放的是 ObjectFactory。对应关系是 bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

* 缓存集合的用途：

| 缓存                    | 用途                                                         |
| :---------------------- | :----------------------------------------------------------- |
| `singletonObjects`      | 用于存放完全初始化好的 bean，从该缓存中取出的 bean 可以直接使用 |
| `earlySingletonObjects` | 用于存放还在初始化中的 bean，用于解决循环依赖                |
| `singletonFactories`    | 用于存放 bean 工厂。bean 工厂所产生的 bean 是还未完成初始化的 bean。如代码所示，bean 工厂所生成的对象最终会被缓存到` earlySingletonObjects `中 |

## 2 `getObjectForBeanInstance`

代码如下：

```java
// AbstractBeanFactory.java

protected Object getObjectForBeanInstance(
        Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

    // <1> 若为工厂类引用（name 以 & 开头）
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }
        // 如果 beanInstance 不是 FactoryBean 类型，则抛出异常
        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
        }
        if (mbd != null) {
            mbd.isFactoryBean = true;
        }
        return beanInstance;
    }

    // <2> beanInstance 不为 FactoryBean 类型，则直接返回
    if (!(beanInstance instanceof FactoryBean)) {
        return beanInstance;
    }

    Object object = null;
    // <3> 若 BeanDefinition 为 null，则从缓存中加载 Bean 对象
    if (mbd != null) {
        mbd.isFactoryBean = true;
    }
    else {
        object = getCachedObjectForFactoryBean(beanName);
    }
    // 若 object 依然为空，则beanInstance 一定是 FactoryBean 
    if (object == null) {
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        // containsBeanDefinition在所有已经加载的类中检测是否定义 beanName
        if (mbd == null && containsBeanDefinition(beanName)) {
            // 将存储 XML 配置文件的 GenericBeanDefinition 转换为 RootBeanDefinition，
            // 如果指定 BeanName 是子 Bean 的话同时会合并父类的相关属性
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        // synthetic 字面意思是"合成的"。通过全局查找，我发现在 AOP 相关的类中会将该属性设为 true。
        // 所以我觉得该字段可能表示某个 bean 是不是被 AOP 增强过，也就是 AOP 基于原始类合成了一个新的代理类。
        // 不过目前只是猜测，没有深究。如果有朋友知道这个字段的具体意义，还望不吝赐教
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        // 核心处理方法，使用 FactoryBean 获得 Bean 对象
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
```

上面代码主要是做检测工作，核心在于委托给`getObjectFromFactoryBean`获得 Bean 对象，主要如下：

1. ` <1>` 处，若 `name` 为工厂相关的（以 & 开头），且 `beanInstance` 为 `NullBean` 类型则直接返回，如果 `beanInstance` 不为 `FactoryBean` 类型则抛出 `BeanIsNotAFactoryException` 异常。这里主要是**校验** `beanInstance` 的**正确性**。
2. `<2>` 处，如果 `beanInstance` 不为 `FactoryBean` 类型或者 `name` 也不是与工厂相关的，则直接返回 `beanInstance` 这个 Bean 对象。**这里主要是对非 `FactoryBean` 类型处理**。
3. `<3>` 处，如果 `BeanDefinition` 为空，则从 `factoryBeanObjectCache` 中加载 Bean 对象(对应代码36行)。如果还是空，则 `beanInstance` 一定是` FactoryBean` 类型，则委托 `#getObjectFromFactoryBean(FactoryBean factory, String beanName, boolean shouldPostProcess)` 方法，进行处理，**使用 `FactoryBean `获得 Bean 对象**。

### 2.1 `getObjectFromFactoryBean`

`getObjectFromFactoryBean`方法代码如下：

```java
// FactoryBeanRegistrySupport.java

protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    // 为单例模式且在缓存中存在
    if (factory.isSingleton() && containsSingleton(beanName)) {
        // <1> 单例锁
        synchronized (getSingletonMutex()) {
            // 从缓存中获取指定的 factoryBean
            Object object = this.factoryBeanObjectCache.get(beanName);
            if (object == null) {
                // <2> 缓存为空，则从 FactoryBean 中获取对象
                object = doGetObjectFromFactoryBean(factory, beanName);
                Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                if (alreadyThere != null) {
                    object = alreadyThere;
                }
                else {
                    // <3> 需要后续处理
                    if (shouldPostProcess) {
                        // 若该 Bean 处于创建中，则返回非处理对象，而不是存储它
                        if (isSingletonCurrentlyInCreation(beanName)) {
                            return object;
                        }
                        // 单例 Bean 的前置处理
                        beforeSingletonCreation(beanName);
                        try {
                            // 对从 FactoryBean 获取的对象进行后处理
                            // 将生成的对象将暴露给 bean 引用
                            object = postProcessObjectFromFactoryBean(object, beanName);
                        }
                        catch (Throwable ex) {
                            throw new BeanCreationException(beanName,
                                    "Post-processing of FactoryBean's singleton object failed", ex);
                        }
                        finally {
                            // 单例 Bean 的后置处理
                            afterSingletonCreation(beanName);
                        }
                    }
                    // <1.4> 添加到 factoryBeanObjectCache 中，进行缓存
                    if (containsSingleton(beanName)) {
                        this.factoryBeanObjectCache.put(beanName, object);
                    }
                }
            }
            return object;
        }
    }
    else {
        // 从 FactoryBean 中获取对象
        Object object = doGetObjectFromFactoryBean(factory, beanName);
        if (shouldPostProcess) {
            try {
                object = postProcessObjectFromFactoryBean(object, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
            }
        }
        return object;
    }
}
```

上面代码的主要工作：

1. `<1>`处，获取锁，锁住的对象都是 `this.singletonObjects`，因为在单例模式中必须要**保证全局唯一**。

   ```java
   //DefaultSingletonBeanRegistry.java
   
   private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
   
   public final Object getSingletonMutex() {
   	return this.singletonObjects;
   }
   ```

   

2. `<2>处`， 缓存为空，则从调用#`doGetObjectFromFactoryBean(factory, beanName)`方法,[详见2.2](#2.2 `doGetObjectFromFactoryBean`)。

3. `<3>处`，如果需要后续处理( `shouldPostProcess = true` )，则进行进一步处理：

   - 若该 Bean 处于创建中（`#isSingletonCurrentlyInCreation(String beanName)` 方法返回 `true` ），则返回**非处理的 Bean 对象**，而不是存储它。[详见2.3](#2.3 `isSingletonCurrentlyInCreation`)
   - 调用 `#beforeSingletonCreation(String beanName)` 方法，进行创建之前的处理。默认实现将该 Bean 标志为当前创建的。
   - 调用 `#postProcessObjectFromFactoryBean(Object object, String beanName)` 方法，对从 FactoryBean 获取的 Bean 实例对象进行后置处理。[详见2.4](#2.4 `postProcessObjectFromFactoryBean`)
   - 调用 `#afterSingletonCreation(String beanName)` 方法，进行创建 Bean 之后的处理，默认实现是将该 bean 标记为不再在创建中。

4. `<4>处`加入到 `factoryBeanObjectCache` 缓存中。

### 2.2 `doGetObjectFromFactoryBean`

   ```java
   // FactoryBeanRegistrySupport.java
   
   private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
           throws BeanCreationException {
   
       Object object;
       try {
           // 需要权限验证
           if (System.getSecurityManager() != null) {
               AccessControlContext acc = getAccessControlContext();
               try {
                   object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
               }
               catch (PrivilegedActionException pae) {
                   throw pae.getException();
               }
           }
           else {
               // 从 FactoryBean 中，获得 Bean 对象
               object = factory.getObject();
           }
       }
       catch (FactoryBeanNotInitializedException ex) {
           throw new BeanCurrentlyInCreationException(beanName, ex.toString());
       }
       catch (Throwable ex) {
           throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
       }
   
       if (object == null) {
           if (isSingletonCurrentlyInCreation(beanName)) {
               throw new BeanCurrentlyInCreationException(
                       beanName, "FactoryBean which is currently in creation returned null from getObject");
           }
           object = new NullBean();
       }
       return object;
   }
   ```

   上面代码重点就在第18行的`factory.getObject()`,通过调用 `FactoryBean#getObject()` 方法，获取 Bean 对象。



### 2.3 `isSingletonCurrentlyInCreation`

`#isSingletonCurrentlyInCreation(String beanName)` 方法，是用于检测当前 Bean 是否处于创建之中。代码如下：

```java
// DefaultSingletonBeanRegistry.java

//正在创建中的单例 Bean 的名字的集合
private final Set<String> singletonsCurrentlyInCreation =
        Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```

对应着两个重要的方法,`beforeSingletonCreation`和`afterSingletonCreation`

* 集合的元素,是在 `#beforeSingletonCreation(String beanName)` 方法中添加的。代码如下：

  ```java
  // DefaultSingletonBeanRegistry.java
  
  protected void beforeSingletonCreation(String beanName) {
  	if (!this.inCreationCheckExclusions.contains(beanName)
              && !this.singletonsCurrentlyInCreation.add(beanName)) { // 添加
  		throw new BeanCurrentlyInCreationException(beanName); // 如果添加失败，则抛出 BeanCurrentlyInCreationException 异常。
  	}
  }
  ```

* 对 `singletonsCurrentlyInCreation` 集合 remove 。代码如下：

  ```java
  // DefaultSingletonBeanRegistry.java
  
  protected void afterSingletonCreation(String beanName) {
  	if (!this.inCreationCheckExclusions.contains(beanName) &&
              !this.singletonsCurrentlyInCreation.remove(beanName)) { // 移除
  	    // 如果移除失败，则抛出 IllegalStateException 异常
  		throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
  	}
  }
  ```

### 2.4 `postProcessObjectFromFactoryBean`

`postProcessObjectFromFactoryBean(Object object, String beanName)` 方法，对从 FactoryBean 处获取的 Bean 实例对象进行后置处理。其默认实现是直接返回 object 对象，不做任何处理。代码如下：

```java
// DefaultSingletonBeanRegistry.java

protected Object postProcessObjectFromFactoryBean(Object object, String beanName) throws BeansException {
	return object;
}
```

Spring中获取bean的规则有这样一条：尽可能保证所有 bean 初始化后都会调用注册的 `BeanPostProcessor#postProcessAfterInitialization(Object bean, String beanName)` 方法进行处理，在实际开发过程中可以针对此特性设计自己的业务逻辑。
