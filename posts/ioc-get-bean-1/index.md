# ﻿IoC -加载 Bean-总览


## 2.1 获取 beanName

代码如下：

```Java
// AbstractBeanFactory.java

final String beanName = transformedBeanName(name);
```

这段代码的作用：这里传递的是 `name` 方法，不一定就是 beanName，可能是 aliasName ，也有可能是 FactoryBean ，所以这里需要调用 `#transformedBeanName(String name)` 方法，对 `name` 进行一番转换。

```java
// AbstractBeanFactory.java

protected String transformedBeanName(String name) {
	return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}
```



这里的转换过程包括两部分，一是去除 FactoryBean 的修饰符，二是取指定的 `alias` 所表示的最终 beanName 。详细分析如下：

1. 调用 `BeanFactoryUtils#transformedBeanName(String name)` 方法，去除 FactoryBean 的修饰符。代码如下：

   ```java
   // BeanFactoryUtils.java
   
   private static final Map<String, String> transformedBeanNameCache = new ConcurrentHashMap<>();
   
   public static String transformedBeanName(String name) {
   		Assert.notNull(name, "'name' must not be null");
   		if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
   			return name;
   		}
       	// BeanFactory.FACTORY_BEAN_PREFIX = "&"
   		// 就是去除传入 name 参数的 "&" 的前缀。
   		// computeIfAbsent 方法是jdk的代码，分成两种情况：
   		//  1. 未存在，则进行计算执行，并将结果添加到缓存。
   		//  2. 已存在，则直接返回，无需计算。
   		return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
   			do {
   				beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
   			}
   			while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
   			return beanName;
   		});
   	}
   ```

   上面代码的作用就是，去除`FACTORYBEAN`的修饰符，就是去除传入 `name` 参数的 `"&"` 的前缀。比如name="&test",那么将得到name="test"。

   2. 调用 `#canonicalName(String name)` 方法，取指定的 `alias` 所表示的最终 beanName 。代码如下：

      ```java
      public String canonicalName(String name) {
      		String canonicalName = name;
      		String resolvedName;
      		// 循环，从 aliasMap 中，获取到最终的 beanName
      		do {
      			resolvedName = this.aliasMap.get(canonicalName);
      			if (resolvedName != null) {
      				canonicalName = resolvedName;
      			}
      		}
      		while (resolvedName != null);
      		return canonicalName;
      	}
      ```
      
主要是一个循环获取 beanName 的过程，例如，别名 A 指向名称为 B 的 bean 则返回 B；若 别名 A 指向别名 B，别名 B 指向名称为 C 的 bean，则返回 C。
