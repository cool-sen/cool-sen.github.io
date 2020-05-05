# Spring-IOC-创建Bean-属性填充


## 1 简介

在Spring 创建 bean 的流程中，Spring 先通过反射创建一个原始的 bean 对象，然后再向这个原始的 bean 对象中填充属性。对于填充属性这个过程，简单点来说，JavaBean 的每个属性通常都有 getter/setter 方法，我们可以直接调用 setter 方法将属性值设置进去。但是，填充属性的过程中还有许多事情要做。比如在 Spring 配置中，所有属性值都是以字符串的形式进行配置的，我们在将这些属性值赋值给对象的成员变量时，要根据变量类型进行相应的类型转换。对于一些集合类的配置，还要将这些配置转换成相应的集合对象才能进行后续的操作。除此之外，如果用户配置了自动注入（`autowire = byName/byType`），Spring 还要去为自动注入的属性寻找合适的注入项。由此可以见，属性填充的整个过程还是很复杂的，并非是简单调用 setter 方法设置属性值即可。

接下来，将深入到源码中，从源码中了解属性填充的整个过程。

## 2 源码分析

### 2.1 `populateBean `源码总览

在Spring中的属性填充，是`populateBean` 方法来实现的。该函数的作用是将 `BeanDefinition `中的属性值赋值给 `BeanWrapper` 实例对象。代码如下：

```java
// AbstractAutowireCapableBeanFactory.java

protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // 没有实例化对象
    if (bw == null) {
        // 有属性，则抛出 BeanCreationException 异常
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
            // 没有属性，直接 return 返回
        } else {
            return;
        }
    }

     /*
     * <1> 
     * 在属性被填充前，给 InstantiationAwareBeanPostProcessor 类型的后置处理器一个修改 
     * bean 状态的机会。关于这段后置引用，官方的解释是：让用户可以自定义属性注入。比如用户实现一
     * 个 InstantiationAwareBeanPostProcessor 类型的后置处理器，并通过 
     * postProcessAfterInstantiation 方法向 bean 的成员变量注入自定义的信息。当然，如果无
     * 特殊需求，直接使用配置中的信息注入即可。另外，Spring 并不建议大家直接实现 
     * InstantiationAwareBeanPostProcessor 接口，如果想实现这种类型的后置处理器，更建议
     * 通过继承 InstantiationAwareBeanPostProcessorAdapter 抽象类实现自定义后置处理器。
     */
    boolean continueWithPropertyPopulation = true;
    if (!mbd.isSynthetic()  // bean 不是"合成"的，即未由应用程序本身定义
            && hasInstantiationAwareBeanPostProcessors()) { // 是否持有 InstantiationAwareBeanPostProcessor
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) { 
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // postProcessAfterInstantiation方法返回值含义：是否继续填充 bean
                // 如果应该在 bean上面设置属性则返回 true，否则返回 false
                // 一般情况下，应该是返回true 。
                // 返回 false 的话，将会阻止在此 Bean 实例上调用任何后续的 InstantiationAwareBeanPostProcessor 实例。
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }
    /* 
     * 如果上面设置 continueWithPropertyPopulation = false，表明用户可能已经自己填充了
     * bean 的属性，不需要 Spring 帮忙填充了。此时直接返回即可
     */
    if (!continueWithPropertyPopulation) {
        return;
    }

    // bean 的属性值
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // <2>  根据名称或类型注入依赖
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
        // 将 PropertyValues 封装成 MutablePropertyValues 对象
        // MutablePropertyValues 允许对属性进行简单的操作，并提供构造函数以支持Map的深度复制和构造。
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // 根据名称自动注入
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // 根据类型自动注入
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    // 是否需要进行【依赖检查】
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    // <3> BeanPostProcessor 处理
    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // 对所有需要依赖检查的属性进行后处理
                PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    // 从 bw 对象中提取 PropertyDescriptor 结果集
                    // PropertyDescriptor：可以通过一对存取方法提取一个属性
                    if (filteredPds == null) {
                        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                    }
                    pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvsToUse == null) {
                        return;
                    }
                }
                pvs = pvsToUse;
            }
        }
    }
    
    // <4> 依赖检查
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        // 依赖检查，对应 depends-on 属性
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    // <5> 将属性应用到 bean 中
    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

这个方法的执行流程。如下：

1. `<1>`处 ，根据 `hasInstantiationAwareBeanPostProcessors` 属性来判断，是否需要在注入属性之前给 `InstantiationAwareBeanPostProcessors `最后一次改变 bean 的机会。**此过程可以控制 Spring 是否继续进行属性填充**。
2. `<2>处`，根据名称或类型解析相关依赖。
3. `<3>处`，进行 `BeanPostProcessor `处理。再次应用后置处理，用于动态修改属性列表 pvs 的内容
4. `<4>处`，依赖检测。
5. `<5>处`，将所有 `PropertyValues `中的属性，填充到 `BeanWrapper` 中。

注意第3步，也就是根据名称或类型解析相关依赖（autowire）。该逻辑只会解析依赖，并不会将解析出的依赖立即注入到 bean 对象中。所有的属性值是在 `applyPropertyValues `方法中统一被注入到 bean 对象中的。

下面将对`populateBean` 方法中比较重要的几个方法调用进行分析，也就是第2步和第5步中的三个方法。

### 2.2 `autowireByName`方法分析

该方法顾名思义根据**属性名称**，完成自动依赖注入的。代码如下：

```java
// AbstractAutowireCapableBeanFactory.java

protected void autowireByName(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
     /*
     * 获取非简单类型属性的名称，且该属性未被配置在配置文件中。这里从反面解释一下什么是"非简单类型"
     * 属性，我们先来看看 Spring 认为的"简单类型"属性有哪些，如下：
     *   1. CharSequence 接口的实现类，比如 String
     *   2. Enum
     *   3. Date
     *   4. URI/URL
     *   5. Number 的继承类，比如 Integer/Long
     *   6. byte/short/int... 等基本类型
     *   7. Locale
     *   8. 以上所有类型的数组形式，比如 String[]、Date[]、int[] 等等
     * 
     * 除了要求非简单类型的属性外，还要求属性未在配置文件中配置过，也就是 pvs.contains(pd.getName()) = false。
     */
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        // 检测是否存在与 propertyName 相关的 bean 或 BeanDefinition。若存在，则调用 BeanFactory.getBean 方法获取 bean 实例
        if (containsBean(propertyName)) {
            // 从容器中获取相应的 bean 实例
            Object bean = getBean(propertyName);
            // 将解析出的 bean 存入到属性值列表（pvs）中
            pvs.add(propertyName, bean);
            // 属性依赖注入
            registerDependentBean(propertyName, beanName);
            if (logger.isTraceEnabled()) {
                logger.trace("Added autowiring by name from bean name '" + beanName +
                        "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
            }
        } else {
            if (logger.isTraceEnabled()) {
                logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                        "' by name: no matching bean found");
            }
        }
    }
}
```

`autowireByName` 方法的逻辑比较简单，该方法首先获取非简单类型属性的名称，然后再根据名称到容器中获取相应的 bean 实例，最后再将获取到的 bean 添加到属性列表中即可。

### 2.3  `autowireByType `方法分析

相较于 `autowireByName`，`autowireByType` 则要复杂一些，复杂之处在于解析依赖的过程。代码如下：

```java
// AbstractAutowireCapableBeanFactory.java

protected void autowireByType(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

    // 获取 TypeConverter 实例
    // 使用自定义的 TypeConverter，用于取代默认的 PropertyEditor 机制
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }

    Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
    // 获取非简单类型的属性
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        try {
            PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
            // 如果属性类型为 Object，则忽略，不做解析
            if (Object.class != pd.getPropertyType()) {
                /*
                 * 获取 setter 方法（write method）的参数信息，比如参数在参数列表中的
                 * 位置，参数类型，以及该参数所归属的方法等信息
                 */
                MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
                boolean eager = !PriorityOrdered.class.isInstance(bw.getWrappedInstance());
                // 创建依赖描述对象
                DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                /*
                 * 下面的方法用于解析依赖。过程比较复杂，先把这里看成一个黑盒，我们只要知道这
                 * 个方法可以帮我们解析出合适的依赖即可。
                 */
                Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
                if (autowiredArgument != null) {
                    // 将解析出的 bean 存入到属性值列表（pvs）中
                    pvs.add(propertyName, autowiredArgument);
                }
                // 遍历 autowiredBeanName 数组
                for (String autowiredBeanName : autowiredBeanNames) {
                    // 属性依赖注入
                    registerDependentBean(autowiredBeanName, beanName);
                    if (logger.isTraceEnabled()) {
                        logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
                                propertyName + "' to bean named '" + autowiredBeanName + "'");
                    }
                }
                // 清空 autowiredBeanName 数组
                autowiredBeanNames.clear();
            }
        } catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
        }
    }
}
```

和 autowireByName 一样，autowireByType 首先也是获取非简单类型属性的名称。然后再根据属性名获取属性描述符，并由属性描述符获取方法参数对象 MethodParameter，随后再根据 MethodParameter 对象获取依赖描述符对象，整个过程为 `beanName → PropertyDescriptor → MethodParameter → DependencyDescriptor`。在获取到依赖描述符对象后，再根据依赖描述符解析出合适的依赖。最后将解析出的结果存入属性列表 pvs 中即可。

相对于 `autowireByName` 方法而言，根据类型寻找相匹配的 bean 过程比较复杂。即`resolveDependency`方法，下面分析该方法。

#### 2.3.1 `resolveDependency`方法分析

```java
// DefaultListableBeanFactory.java

private static Class<?> javaxInjectProviderClass;

static {
	try {
		javaxInjectProviderClass = ClassUtils.forName("javax.inject.Provider", DefaultListableBeanFactory.class.getClassLoader());
	} catch (ClassNotFoundException ex) {
		javaxInjectProviderClass = null;
	}
}

public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
    // 初始化参数名称发现器，该方法并不会在这个时候尝试检索参数名称
    // getParameterNameDiscoverer 返回 parameterNameDiscoverer 实例，parameterNameDiscoverer 方法参数名称的解析器
    descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
    // 依赖类型为 Optional 类型
    if (Optional.class == descriptor.getDependencyType()) {
        return createOptionalDependency(descriptor, requestingBeanName);
    // 依赖类型为ObjectFactory、ObjectProvider
    } else if (ObjectFactory.class == descriptor.getDependencyType() ||
            ObjectProvider.class == descriptor.getDependencyType()) {
        return new DependencyObjectProvider(descriptor, requestingBeanName);
    // javaxInjectProviderClass 类注入的特殊处理
    } else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
        return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
    } else {
        // 为实际依赖关系目标的延迟解析构建代理
        // 默认实现返回 null
        Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(descriptor, requestingBeanName);
        if (result == null) {
            // 通用处理逻辑
            result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
        }
        return result;
    }
}
```

这里我们关注**通用处理逻辑** `doResolveDependency`方法，代码如下：

```java
// DefaultListableBeanFactory.java

public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
    @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
    // 注入点
    InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
    try {
        // 针对给定的工厂给定一个快捷实现的方式，例如考虑一些预先解析的信息
        // 在进入所有bean的常规类型匹配算法之前，解析算法将首先尝试通过此方法解析快捷方式。
        // 子类可以覆盖此方法
        Object shortcut = descriptor.resolveShortcut(this);
        if (shortcut != null) {
            // 返回快捷的解析信息
            return shortcut;
        }
        // 依赖的类型
        Class<?> type = descriptor.getDependencyType();
        // 支持 Spring 的注解 @value
        Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
        if (value != null) {
            if (value instanceof String) {
                String strVal = resolveEmbeddedValue((String) value);
                BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
                value = evaluateBeanDefinitionString(strVal, bd);
            }
            TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
            return (descriptor.getField() != null ?
                    converter.convertIfNecessary(value, type, descriptor.getField()) :
                    converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
        }
        // 解析复合 bean，其实就是对 bean 的属性进行解析
        // 包括：数组、Collection 、Map 类型
        Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
        if (multipleBeans != null) {
            return multipleBeans;
        }
        // 查找与类型相匹配的 bean
        // 返回值构成为：key = 匹配的 beanName，value = beanName 对应的实例化 bean
        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
        // 没有找到，检验 @autowire  的 require 是否为 true
        if (matchingBeans.isEmpty()) {
            // 如果 @autowire 的 require 属性为 true ，但是没有找到相应的匹配项，则抛出异常
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            return null;
        }
        String autowiredBeanName;
        Object instanceCandidate;
        if (matchingBeans.size() > 1) {
            // 确认给定 bean autowire 的候选者
            // 按照 @Primary 和 @Priority 的顺序
            autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
            if (autowiredBeanName == null) {
                if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                    // 唯一性处理
                    return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
                }
                else {
                    // 在可选的Collection / Map的情况下，默默地忽略一个非唯一的情况：可能它是一个多个常规bean的空集合
                    return null;
                }
            }
            instanceCandidate = matchingBeans.get(autowiredBeanName);
        } else {
            Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
            autowiredBeanName = entry.getKey();
            instanceCandidate = entry.getValue();
        }
        if (autowiredBeanNames != null) {
            autowiredBeanNames.add(autowiredBeanName);
        }
        if (instanceCandidate instanceof Class) {
            instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
        }
        Object result = instanceCandidate;
        if (result instanceof NullBean) {
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            result = null;
        }
        if (!ClassUtils.isAssignableValue(type, result)) {
            throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
        }
        return result;
    } finally {
        ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
    }
}
```

总结一下`doResolveDependency `方法的大概流程。

1. 首先将 beanName 和 requiredType 作为参数，并尝试从 `BeanFactory `中获取与此对于的 bean。若获取成功，就可以提前结束 `doResolveDependency `方法的逻辑。
2. 处理 @value 注解。
3. 解析数组、List、Map 等类型的依赖，如果解析结果不为空，则返回结果。
4. 根据类型查找合适的候选项。
5. 如果候选项的数量为0，则抛出异常。为1，直接从候选列表中取出即可。若候选项数量 > 1，则在多个候选项中确定最优候选项，若无法确定则抛出异常。
6. 若候选项是 Class 类型，表明候选项还没实例化，此时通过 `BeanFactory.getBean` 方法对其进行实例化。若候选项是非 Class 类型，则表明已经完成了实例化，此时直接返回即可。

### 2.4 `applyPropertyValues` 方法分析



获取的属性封装在 PropertyValues 的实例对象 `pvs` 中，并没有应用到已经实例化的 bean 中。因为在 Spring 配置文件中属性值都是以 String 类型进行配置的，所以 Spring 框架需要对 String 类型进行转换。除此之外，对于 ref 属性，这里还需要根据 ref 属性值解析依赖。而 `applyPropertyValues`方法，则是完成这一步骤的。代码如下：

```java
// AbstractAutowireCapableBeanFactory.java

protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    if (pvs.isEmpty()) {
        return;
    }

    // 设置 BeanWrapperImpl 的 SecurityContext 属性
    if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
        ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
    }

    // MutablePropertyValues 类型属性
    MutablePropertyValues mpvs = null;

    // 原始类型
    List<PropertyValue> original;
    // 获得 original
    if (pvs instanceof MutablePropertyValues) {
        mpvs = (MutablePropertyValues) pvs;
        // 属性值已经转换
        if (mpvs.isConverted()) {
            // Shortcut: use the pre-converted values as-is.
            try {
                // 为实例化对象设置属性值 ，依赖注入真真正正地实现在此！！！！！
                bw.setPropertyValues(mpvs);
                return;
            } catch (BeansException ex) {
                throw new BeanCreationException(
                        mbd.getResourceDescription(), beanName, "Error setting property values", ex);
            }
        }
        original = mpvs.getPropertyValueList();
    } else {
        // 如果 pvs 不是 MutablePropertyValues 类型，则直接使用原始类型
        original = Arrays.asList(pvs.getPropertyValues());
    }

    // 获取 TypeConverter = 获取用户自定义的类型转换
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }

    // 获取对应的解析器
    BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

    // Create a deep copy, resolving any references for values.
    List<PropertyValue> deepCopy = new ArrayList<>(original.size());
    boolean resolveNecessary = false;
    // 遍历属性，将属性转换为对应类的对应属性的类型
    for (PropertyValue pv : original) {
        // 属性值不需要转换
        if (pv.isConverted()) {
            deepCopy.add(pv);
        // 属性值需要转换
        } else {
            String propertyName = pv.getName();
            Object originalValue = pv.getValue(); // 原始的属性值，即转换之前的属性值
            /*
             * 解析属性值。举例说明，先看下面的配置：
             * 
             *   <bean id="macbook" class="MacBookPro">
             *       <property name="manufacturer" value="Apple"/>
             *       <property name="width" value="280"/>
             *       <property name="cpu" ref="cpu"/>
             *       <property name="interface">
             *           <list>
             *               <value>USB</value>
             *               <value>HDMI</value>
             *               <value>Thunderbolt</value>
             *           </list>
             *       </property>
             *   </bean>
             *
             * 上面是一款电脑的配置信息，每个 property 配置经过下面的方法解析后，返回如下结果：
             *   propertyName = "manufacturer", resolvedValue = "Apple"
             *   propertyName = "width", resolvedValue = "280"
             *   propertyName = "cpu", resolvedValue = "CPU@1234"  注：resolvedValue 是一个对象
             *   propertyName = "interface", resolvedValue = ["USB", "HDMI", "Thunderbolt"]
             *
             * 如上所示，resolveValueIfNecessary 会将 ref 解析为具体的对象，将 <list> 
             * 标签转换为 List 对象等。对于 int 类型的配置，这里并未做转换，所以 
             * width = "280"，还是字符串。除了解析上面几种类型，该方法还会解析 <set/>、
             * <map/>、<array/> 等集合配置
             */
            
            Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue); 
            Object convertedValue = resolvedValue; 
            
            /*
             * convertible 表示属性值是否可转换，由两个条件合成而来。第一个条件不难理解，解释
             * 一下第二个条件。第二个条件用于检测 propertyName 是否是 nested 或者 indexed，
             * 直接举例说明吧：
             * 
             *   public class Room {
             *       private Door door = new Door();
             *   }
             *
             * room 对象里面包含了 door 对象，如果我们想向 door 对象中注入属性值，则可以这样配置：
             *
             *   <bean id="room" class="xyz.coolblog.Room">
             *      <property name="door.width" value="123"/>
             *   </bean>
             * 
             * isNestedOrIndexedProperty 会根据 propertyName 中是否包含 . 或 [  返回 
             * true 和 false。包含则返回 true，否则返回 false。
             * 关于 nested 类型的属性，大家还可以参考 Spring 的官方文档：
             *     https://docs.spring.io/spring/docs/4.3.17.RELEASE/spring-framework-reference/htmlsingle/#beans-beans-conventions
             */
            boolean convertible = bw.isWritableProperty(propertyName) &&
                    !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName); 
            // 对于一般的属性，convertible 通常为 true
            if (convertible) {
                // 对属性值的类型进行转换，比如将 String 类型的属性值 "123" 转为 Integer 类型的 123
                convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
            }
            
            /*
             * 如果 originalValue 是通过 autowireByType 或 autowireByName 解析而来，
             * 那么此处条件成立，即 (resolvedValue == originalValue) = true
             */
            if (resolvedValue == originalValue) {
                if (convertible) {
                    // 将 convertedValue 设置到 pv 中，后续再次创建同一个 bean 时，就无需再次进行转换了
                    pv.setConvertedValue(convertedValue);
                }
                deepCopy.add(pv);
            /*
             * 如果原始值 originalValue 是 TypedStringValue，且转换后的值 
             * convertedValue 不是 Collection 或数组类型，则将转换后的值存入到 pv 中。
             */
            } else if (convertible && originalValue instanceof TypedStringValue &&
                    !((TypedStringValue) originalValue).isDynamic() &&
                    !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
                pv.setConvertedValue(convertedValue);
                deepCopy.add(pv);
            } else {
                resolveNecessary = true;
                // 重新封装属性的值
                deepCopy.add(new PropertyValue(pv, convertedValue));
            }
        }
    }
    // 标记属性值已经转换过
    if (mpvs != null && !resolveNecessary) {
        mpvs.setConverted();
    }

    // 进行属性依赖注入，依赖注入的真真正正实现依赖的注入方法在此！！！
    try {
        bw.setPropertyValues(new MutablePropertyValues(deepCopy));
    } catch (BeansException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Error setting property values", ex);
    }
}
```

上面方面的流程如下：

1. 检测属性值列表是否已转换过的，若转换过，则直接填充属性，无需再次转换。
2. 遍历属性值列表 pvs，解析原始值 originalValue，得到解析值 resolvedValue。
3. 对解析后的属性值 resolvedValue 进行类型转换。
4. 将类型转换后的属性值设置到 PropertyValue 对象中，并将 PropertyValue 对象存入 deepCopy 集合中
5. 将 deepCopy 中的属性信息注入到 bean 对象中。

## 3 小结

至此，`doCreateBean` 方法的第二个过程：**属性填充**已经分析完成了。

## 参考

* [《Spring 源码深度解析》- 郝佳著](https://book.douban.com/subject/25866350/)
* [Spring IOC 容器源码分析 - 填充属性到 bean 原始对象]([http://www.tianxiaobo.com/2018/06/11/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E5%A1%AB%E5%85%85%E5%B1%9E%E6%80%A7%E5%88%B0-bean-%E5%8E%9F%E5%A7%8B%E5%AF%B9%E8%B1%A1/#23-autowirebytype-%E6%96%B9%E6%B3%95%E5%88%86%E6%9E%90](http://www.tianxiaobo.com/2018/06/11/Spring-IOC-容器源码分析-填充属性到-bean-原始对象/#23-autowirebytype-方法分析))


