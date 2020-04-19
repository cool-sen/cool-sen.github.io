# Spring-IOC-默认标签的解析


## 前言

Spring中的标签包括默认标签和自定义标签两种，而两种标签的用法以及解析方式存在着很大的不同。本篇文章主要分析默认标签的解析。

默认标签的解析是在`parseDefaultElement`方法中进行。

```java
// DefaultBeanDefinitionDocumentReader.java

private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		//对import标签的处理
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		//对alias标签的处理
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		//对bean标签的处理
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		//对beans标签的处理
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```

在以上四种标签的解析中，对bean标签的解析是最复杂的，也是最重要的。本篇文章就重点对bean标签的解析做一些分析。

## 1 总体流程

`processBeanDefinition`方法功能就是对bean标签就行解析，源码如下：

```java
// DefaultBeanDefinitionDocumentReader.java

protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		// 1. 如果解析成功，则返回 BeanDefinitionHolder 对象。如果解析失败，则返回 null 。
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			// 2. 进行自定义标签处理
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 3. 进行 BeanDefinition 的注册
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// 4. 发出响应事件，通知相关的监听器，这个bean已经解析完成了。
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

上面方法的主要步骤为：

1. 调用 `BeanDefinitionParserDelegate`类的`parseBeanDefinitionElement(Element ele, BeanDefinitionParserDelegate delegate)` 方法，进行元素解析。

- 如果解析**失败**，则返回 `null`，错误由` ProblemReporter `处理。
- 如果解析**成功**，则返回 `BeanDefinitionHolder` 实例 `bdHolder` 。`bdHolder`实例已经包含配置文件中配置的各种属性了，例如class、name、id、alias之类的属性。

2. 若实例 `bdHolder` 不为空,则调用 `BeanDefinitionParserDelegate`类的`decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder bdHolder)` 方法，进行自定义标签处理。
3. 对解析后的`bdHolder`进行注册,通过调用 `BeanDefinitionReaderUtils`类的`registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)` 方法。
4. 最后发出响应事件，通知相关的监听器，这个bean已经加载完成了。

下面就重点对上面四个步骤做个详细介绍。

## 2 `parseBeanDefinitionElement`方法分析

```java
//BeanDefinitionParserDelegate.java

@Nullable
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
		// 1. 解析 id 和 name 属性
		String id = ele.getAttribute(ID_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		// 计算别名集合
		List<String> aliases = new ArrayList<>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}
		// 3.1 beanName ，优先使用 id
		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			// 3.2 beanName ，其次使用 aliases 的第一个
			beanName = aliases.remove(0);
			if (logger.isTraceEnabled()) {
				logger.trace("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}
		// 检查 beanName 的唯一性
		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}
		// 2. 解析属性，构造 AbstractBeanDefinition 对象
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			// 3.beanName，最后使用 beanName 生成规则
			if (!StringUtils.hasText(beanName)) {
				try {
					if (containingBean != null) {
						// 3. 使用 beanName 生成规则，生成唯一的 beanName
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						// Register an alias for the plain bean class name, if still possible,
						// if the generator returned the class name plus a suffix.
						// This is expected for Spring 1.2/2.0 backwards compatibility.
						//3. 使用 beanName 生成规则，生成唯一的 beanName
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isTraceEnabled()) {
						logger.trace("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			// 4. 创建 BeanDefinitionHolder 对象
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}
```

上面方法主要分为四个步骤：

1. 提取元素中的id以及name属性。
2. 调用`parseBeanDefinitionElement(ele, beanName, containingBean)`方法对属性进行解析并封装成 `AbstractBeanDefinition` 实例 `beanDefinition` 。
3. 生成唯一的`beanName`,`beanName` 的命名规则为：
   * 3.1 处 ，如果 `id` 不为空，则 `beanName = id` 。
   * 3.2 处，如果 `id` 为空，但是 `aliases` 不空，则 `beanName` 为 `aliases` 的**第一个**元素。
   * 3.3 处，如果两者都为空，则根据**默认规则**来设置 `beanName `。

4. 根据所获取的信息（`beanName`、`aliases`、`beanDefinition`）构造 `BeanDefinitionHolder` 实例对象并返回。

### 2.1 `parseBeanDefinitionElement`分析

**注意**，这个`parseBeanDefinitionElement`与上一节的`parseBeanDefinitionElement`不同，这一小节的`parseBeanDefinitionElement`方法多了一个参数`beanName`。

```java
//BeanDefinitionParserDelegate.java

@Nullable
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		// 解析 class 属性
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
		// 解析 parent 属性
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
			// 创建用于承载属性的 AbstractBeanDefinition 实例
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);

			// 解析默认 bean 的各种属性
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			// 提取 description
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

			// 下面的一堆是解析 <bean>......</bean> 内部的子元素
			// 解析出来以后的信息都放到 bd 的属性中

			// 解析元数据 <meta />
			parseMetaElements(ele, bd);
			// 解析 lookup-method 属性 <lookup-method />
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
			// 解析 replaced-method 属性 <replaced-method />
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

			// 解析构造函数参数 <constructor-arg />
			parseConstructorArgElements(ele, bd);
			// 解析 property 子元素 <property />
			parsePropertyElements(ele, bd);
			// 解析 qualifier 子元素 <qualifier />
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}

```

该方法解析了bean标签的所有属性，有常用的，也有不常用的。

其中有两个重要的方法

* `createBeanDefinition(className, parent)`,该方法作用是创建 `AbstractBeanDefinition `对象。`AbstractBeanDefinition `有什么用，为什么要创建`AbstractBeanDefinition `对象呢？请看下面的补充分析。
* `parseBeanDefinitionAttributes(ele, beanName, containingBean, bd)`,该方法作用是解析默认 bean 的各种属性。

#### 2.1.1 创建`AbstractBeanDefinition `对象

讲创建`AbstractBeanDefinition `对象之前，先来补充下`BeanDefinition`的一些知识。

`org.springframework.beans.factory.config.BeanDefinition` ，是一个接口，它描述了一个 Bean 实例的定义，包括属性值、构造方法值和继承它的类的更多信息。

![image-20200419210842067](/images/IoC-load-BeanDefinitions-2/image-20200419210842067.png)

<center>BeanDefinition相关类图</center>

从父关系来看，`BeanDefinition` 继承 `AttributeAccessor` 和 `BeanMetadataElement` 接口。

* `org.springframework.cor.AttributeAccessor` 接口，定义了与其它对象的（元数据）进行连接和访问的约定，即对属性的修改，包括获取、设置、删除。
* `org.springframework.beans.BeanMetadataElement` 接口，Bean 元对象持有的配置元素可以通过 `getSource()` 方法来获取。

从子关系来看，`BeanDefinition`在Spring中有三个实现类。`ChildBeanDefinition`,`RootBeanDefinition`,`GenericBeanDefinition`。

* 三个实现类都继承 `AbstractBeanDefinition `抽象类

* 如果配置文件中定义了父 `<bean>` 和 子 `<bean>` ，则父`<bean>` 用 `RootBeanDefinition` 表示，子 `<bean>`用` ChildBeanDefinition` 表示。没有父 `<bean>` 的就使用`RootBeanDefinition `表示。

Spring通过`BeanDefinition`将配置文件中的<bean>配置信息转换为容器的内部表示，并将这些`BeanDefiniton`注册到`BeanDefinitonRegistry`中。Spring容器的`BeanDefinitionRegistry`就像是Spring配置信息的内存数据库，主要是以map的形式保存，后续操作直接从`BeanDefinitionRegistry`中读取配置信息。

解析属性首先要创建用于承载属性的实例，也就是创建`GenericBeanDefinition`类型的实例，`createBeanDefinition(className, parent)`就是实现这个功能。

```java
//BeanDefinitionParserDelegate.java

protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
			throws ClassNotFoundException {

		return BeanDefinitionReaderUtils.createBeanDefinition(
				parentName, className, this.readerContext.getBeanClassLoader());
	}
```

其中的`BeanDefinitionReaderUtils.createBeanDefinition()`方法如下：

```java
// BeanDefinitionReaderUtils.java

public static AbstractBeanDefinition createBeanDefinition(
			@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {

		GenericBeanDefinition bd = new GenericBeanDefinition();
		bd.setParentName(parentName);
		if (className != null) {
			if (classLoader != null) {
                // 如果classLoader不为空，使用传入的classLoader同一虚拟机加载类对象
				bd.setBeanClass(ClassUtils.forName(className, classLoader));
			}
			else {
                // classLoader为空,记录className
				bd.setBeanClassName(className);
			}
		}
		return bd;
	}
```

#### 2.1.2 `parseBeanDefinitionAttributes`

`arseBeanDefinitionAttributes`方法是对element所有元素属性进行解析：

```java
// BeanDefinitionParserDelegate.java

public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
        @Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {
    // 解析 scope 属性
    if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
        error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
    } else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
        bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
    } else if (containingBean != null) {
        // Take default from containing bean in case of an inner bean definition.
        bd.setScope(containingBean.getScope());
    }

    // 解析 abstract 属性
    if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
        bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
    }

    // 解析 lazy-init 属性
    String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
    if (DEFAULT_VALUE.equals(lazyInit)) {
        lazyInit = this.defaults.getLazyInit();
    }
    bd.setLazyInit(TRUE_VALUE.equals(lazyInit));

    // 解析 autowire 属性
    String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
    bd.setAutowireMode(getAutowireMode(autowire));

    // 解析 depends-on 属性
    if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
        String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
        bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
    }

    // 解析 autowire-candidate 属性
    String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
    if ("".equals(autowireCandidate) || DEFAULT_VALUE.equals(autowireCandidate)) {
        String candidatePattern = this.defaults.getAutowireCandidates();
        if (candidatePattern != null) {
            String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
            bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
        }
    } else {
        bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
    }

    // 解析 primary 标签
    if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
        bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
    }

    // 解析 init-method 属性
    if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
        String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
        bd.setInitMethodName(initMethodName);
    } else if (this.defaults.getInitMethod() != null) {
        bd.setInitMethodName(this.defaults.getInitMethod());
        bd.setEnforceInitMethod(false);
    }

    // 解析 destroy-method 属性
    if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
        String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
        bd.setDestroyMethodName(destroyMethodName);
    } else if (this.defaults.getDestroyMethod() != null) {
        bd.setDestroyMethodName(this.defaults.getDestroyMethod());
        bd.setEnforceDestroyMethod(false);
    }

    // 解析 factory-method 属性
    if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
        bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
    }
    if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
        bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
    }

    return bd;
}
```

#### 2.1.3 解析 `<bean>...</bean> `内部的子元素

<bean>......</bean> 内部的子元素有，`meta`,`lookup-method`,`replace-method`,`constructor-arg`,`property`,`qualifier`。这里我没有做深入研究，小伙伴们想要深入了解可以参考以下博文，讲解的很详细。

* [IOC 之解析 bean 标签：constructor-arg、property 子元素](http://cmsblogs.com/?p=2754)
* [IOC 之解析 bean 标签：meta、lookup-method、replace-method](http://cmsblogs.com/?p=2736)

## 3 `decorateBeanDefinitionIfRequired`方法分析

　`decorateBeanDefinitionIfRequired(ele, bdHolder)`方法，作用是解析默认标签中的自定义标签元素。

```java
//BeanDefinitionParserDelegate.java

public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder originalDef) {
		return decorateBeanDefinitionIfRequired(ele, originalDef, null);
	}
	
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
			Element ele, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {

		BeanDefinitionHolder finalDefinition = originalDef;

		// Decorate based on custom attributes first.
		NamedNodeMap attributes = ele.getAttributes();
		for (int i = 0; i < attributes.getLength(); i++) {
			Node node = attributes.item(i);
			finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
		}

		// Decorate based on custom nested elements.
		NodeList children = ele.getChildNodes();
		for (int i = 0; i < children.getLength(); i++) {
			Node node = children.item(i);
			if (node.getNodeType() == Node.ELEMENT_NODE) {
				finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
			}
		}
		return finalDefinition;
	}
```

这段代码的作用是，寻找自定义标签并根据自定义标签寻找命名空间处理器，并进行进一步的解析。这里没有做具体的分析，感兴趣的小伙伴可以参考其他资料。

## 4 `registerBeanDefinition`方法分析

`registerBeanDefinition`方法的作用：注册`BeanDefinition`。见如下代码：

```java
//BeanDefinitionReaderUtils.java

public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// 使用beanName做唯一标识注册
		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// 注册所有的别名
		// Register aliases for bean name, if any.
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

解析的`beanDefinition`都会被注册到`BeanDefinitionRegistry`类型的实例registry中，对于`beanDefinition`的注册分成了两部分：通过`beanName`注册和通过别名注册。

### 4.1 通过`beanName`注册

```java
// DefaultListableBeanFactory.java

/** Whether to allow re-registration of a different definition with the same name. */
private boolean allowBeanDefinitionOverriding = true;

/** Map of bean definition objects, keyed by bean name. */
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
/** List of bean definition names, in registration order. */
private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
/** List of names of manually registered singletons, in registration order. */
private volatile Set<String> manualSingletonNames = new LinkedHashSet<>(16);
/** Cached array of bean definition names in case of frozen configuration. */
@Nullable
private volatile String[] frozenBeanDefinitionNames;

@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException {

    // 校验 beanName 与 beanDefinition 非空
    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    // 1. 校验 BeanDefinition 。
    // 这是注册前的最后一次校验了，主要是对属性 methodOverrides 进行校验。
    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        } catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                    "Validation of bean definition failed", ex);
        }
    }

    // 2. 从缓存中获取指定 beanName 的 BeanDefinition
    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    // 3. 如果已经存在
    if (existingDefinition != null) {
        // 如果存在但是不允许覆盖，抛出异常
        if (!isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
        // 覆盖 beanDefinition 大于 被覆盖的 beanDefinition 的 ROLE ，打印 info 日志
        } else if (existingDefinition.getRole() < beanDefinition.getRole()) {
            // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
            if (logger.isInfoEnabled()) {
                logger.info("Overriding user-defined bean definition for bean '" + beanName +
                        "' with a framework-generated bean definition: replacing [" +
                        existingDefinition + "] with [" + beanDefinition + "]");
            }
        // 覆盖 beanDefinition 与 被覆盖的 beanDefinition 不相同，打印 debug 日志
        } else if (!beanDefinition.equals(existingDefinition)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Overriding bean definition for bean '" + beanName +
                        "' with a different definition: replacing [" + existingDefinition +
                        "] with [" + beanDefinition + "]");
            }
        // 其它，打印 debug 日志
        } else {
            if (logger.isTraceEnabled()) {
                logger.trace("Overriding bean definition for bean '" + beanName +
                        "' with an equivalent definition: replacing [" + existingDefinition +
                        "] with [" + beanDefinition + "]");
            }
        }
        // 允许覆盖，直接覆盖原有的 BeanDefinition 到 beanDefinitionMap 中
        this.beanDefinitionMap.put(beanName, beanDefinition);
    // 4. 如果未存在
    } else {
        // 检测创建 Bean 阶段是否已经开启，如果开启了则需要对 beanDefinitionMap 进行并发控制
        if (hasBeanCreationStarted()) {
            // beanDefinitionMap 为全局变量，存在并发访问的情况
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                // 添加到 BeanDefinition 到 beanDefinitionMap 中
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                // 添加 beanName 到 beanDefinitionNames 中
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                // 从 manualSingletonNames 移除 beanName
               removeManualSingletonName(beanName);
            }	
        } else {
            // Still in startup registration phase
            // 添加到 BeanDefinition 到 beanDefinitionMap 中
            this.beanDefinitionMap.put(beanName, beanDefinition);
            // 添加 beanName 到 beanDefinitionNames 中
            this.beanDefinitionNames.add(beanName);
            // 从 manualSingletonNames 移除 beanName
            this.manualSingletonNames.remove(beanName);
        }
        
        this.frozenBeanDefinitionNames = null;
    }

    // 5. 重新设置 beanName 对应的缓存
    if (existingDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
		else if (isConfigurationFrozen()) {
			clearByTypeCache();
		}
}
```

这个方法真的很复杂，简要概括逻辑为：

1. 对` BeanDefinition` 进行校验，该校验也是注册过程中的最后一次校验了，主要是对 `AbstractBeanDefinition `的 `methodOverrides` 属性进行校验。注意这个与对于XML格式的校验不同。
2. 根据 `beanName` 从缓存中获取 `BeanDefinition `对象。
3. 如果缓存中存在，则根据 `allowBeanDefinitionOverriding` 标志来判断是否允许覆盖。如果允许则直接覆盖。否则，抛出 `BeanDefinitionStoreException` 异常。
4.  若缓存中没有指定 `beanName` 的 `BeanDefinition`，则判断当前阶段是否已经开始了 Bean 的创建阶段。如果是，则需要对 `beanDefinitionMap` 进行加锁控制并发问题，否则直接设置即可。
5. 若缓存中存在该 `beanName` 或者单例 bean 集合中存在该 `beanName` ，则调用 `#resetBeanDefinition(String beanName)` 方法，重置 `BeanDefinition `缓存。

这段代码核心就是`this.beanDefinitionMap.put(beanName, beanDefinition)`。

### 4.2 通过别名注册

```java
// SimpleAliasRegistry.java

/** Map from alias to canonical name. */
// key: alias
// value: beanName
private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);

public void registerAlias(String name, String alias) {
		// 校验 name 、 alias
		Assert.hasText(name, "'name' must not be empty");
		Assert.hasText(alias, "'alias' must not be empty");
		synchronized (this.aliasMap) {
			// name 与alias 则去掉alias
			if (alias.equals(name)) {
				this.aliasMap.remove(alias);
				if (logger.isDebugEnabled()) {
					logger.debug("Alias definition '" + alias + "' ignored since it points to same name");
				}
			}
			else {
				String registeredName = this.aliasMap.get(alias);
				if (registeredName != null) {
					// 相同，则 return ，无需重复注册
					if (registeredName.equals(name)) {
						// An existing alias - no need to re-register
						return;
					}
					// 如果alias不允许被覆盖则抛出异常
					if (!allowAliasOverriding()) {
						throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" +
								name + "': It is already registered for name '" + registeredName + "'.");
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Overriding alias '" + alias + "' definition for registered name '" +
								registeredName + "' with new target name '" + name + "'");
					}
				}
				// 校验，是否存在循环指向。如：当A->B存在时，若再次出现A->C->B时候则会抛出异常
				checkForAliasCircle(name, alias);
				// 注册 alias
				this.aliasMap.put(alias, name);
				if (logger.isTraceEnabled()) {
					logger.trace("Alias definition '" + alias + "' registered for name '" + name + "'");
				}
			}
		}
	}
```

`checkForAliasCircle`方法来对别名进行了**循环**检测。代码如下：

```java
//SimpleAliasRegistry.java

protected void checkForAliasCircle(String name, String alias) {
    if (hasAlias(alias, name)) {
        throw new IllegalStateException("Cannot register alias '" + alias +
                "' for name '" + name + "': Circular reference - '" +
                name + "' is a direct or indirect alias for '" + alias + "' already");
    }
}
public boolean hasAlias(String name, String alias) {
    for (Map.Entry<String, String> entry : this.aliasMap.entrySet()) {
        String registeredName = entry.getValue();
        if (registeredName.equals(name)) {
            String registeredAlias = entry.getKey();
            if (registeredAlias.equals(alias) || hasAlias(registeredAlias, alias)) {
                return true;
            }
        }
    }
    return false;
}
```

## 5 　通知监听器解析及注册完成

通过代码`getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder))`完成此工作，这里的实现只为扩展，前在Spring中并没有对此事件做任何逻辑处理。当开发人员需要对注册`BeanDefinition`事件进行监听时可以注册监听器,将处理逻辑写入监听器中。

## 参考

* [《Spring 源码深度解析》- 郝佳著](https://book.douban.com/subject/25866350/)

* [IOC 之注册解析的 `BeanDefinition`](http://cmsblogs.com/?p=2763)
* [IOC 之解析 bean 标签：开启解析进程](http://cmsblogs.com/?p=2731)


