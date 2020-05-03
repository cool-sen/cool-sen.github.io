# 创建bean的实例




## 1 简介

本文将详细分析`doCreateBean`方法中的一个重要的调用，即`createBeanInstance`方法。先来了解一下方法的大致脉络。

```java
// AbstractAutowireCapableBeanFactory.java

protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    
    // 解析 bean ，将 bean 类名解析为 class 引用。
    Class<?> beanClass = resolveBeanClass(mbd, beanName);
	
    /*
     * 检测类的访问权限。默认情况下，对于非 public 的类，是允许访问的。
     * 若禁止访问，这里会抛出异常
     */
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) { // 校验
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    // <1> 如果存在 Supplier 回调，则使用给定的回调方法初始化策略
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }

    // <2> 使用 FactoryBean 的 factory-method 来创建bean 对象，支持静态工厂和实例工厂
    if (mbd.getFactoryMethodName() != null)  {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // <3> 快捷路径
    /*
     * 当多次构建同一个 bean 时，可以使用此处的快捷路径，即无需再次推断应该使用哪种方式构造实例，
     * 以提高效率。比如在多次构建同一个 prototype 类型的 bean 时，就可以走此处的捷径。
     * 这里的 resolved 和 mbd.constructorArgumentsResolved 将会在 bean 第一次实例
     * 化的过程中被设置。
     */
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        // constructorArgumentLock 构造函数的常用锁
        synchronized (mbd.constructorArgumentLock) {
            // 如果已缓存的解析的构造函数或者工厂方法不为空，则可以利用构造函数解析
            // 因为需要根据参数确认到底使用哪个构造函数，该过程比较消耗性能，所有采用缓存机制
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    // 已经解析好了，直接注入即可
    if (resolved) {
        // <3.1> autowire 自动注入，调用构造函数自动注入
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);
        } else {
            // <3.2> 使用默认构造函数构造
            return instantiateBean(beanName, mbd);
        }
    }

    // <4> 由后置处理器决定返回哪些构造方法
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    // <4.1> 有参数情况时，创建 Bean 。先利用参数个数，类型等，确定最精确匹配的构造方法。
    /*
     *	下面4个条件，只要有一个为 true，就会通过构造方法自动注入的方式构造 bean 实例
     * 
     *    条件1：ctors != null -> 后置处理器返回构造方法数组是否为空
     *    
     *    条件2：mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR 
     *              -> bean 配置中的 autowire 属性是否为 constructor    
     *    条件3：mbd.hasConstructorArgumentValues() 
     *              -> constructorArgumentValues 是否存在元素，即 bean 配置文件中
     *                 是否配置了 <construct-arg/>
     *    条件4：!ObjectUtils.isEmpty(args) 
     *              -> args 数组是否存在元素，args 是由用户调用 
     *                 getBean(String name, Object... args) 传入的
     */
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // <4.1> 选择构造方法，创建 Bean 。
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
        return autowireConstructor(beanName, mbd, ctors, null); // args = null
    }

    // <4.2> 有参数时，又没获取到构造方法，则只能调用无参构造方法来创建实例了(兜底方法)
    return instantiateBean(beanName, mbd);
}
```

整体的思路如下：

1. `<1>` 处，如果存在 Supplier 回调，则调用 `obtainFromSupplier` 方法，进行初始化。

2. `<2>` 处，如果存在工厂方法，则使用工厂方法进行初始化。

3. `<3>`处，如果**缓存中存在**，即已经解析过了，则直接使用已经解析了的。根据 `constructorArgumentsResolved` 参数来选择：

   * `<3.1>` 处，使用构造函数自动注入。调用`autowireConstructor(String beanName, RootBeanDefinition mbd, Constructor[] ctors, Object[] explicitArgs)` 方法
   * `<3.2>` 处，使用默认构造函数。调用 `instantiateBean(final String beanName, final RootBeanDefinition mbd)` 方法。

4. `<4>` 处，如果**缓存中没有**，则通过组合条件决定使用哪种方式构建 bean 对象。

   * `<4.1>` 处，如果存在参数，则使用相应的带有参数的构造函数。

   * `<4.2>` 处，否则，使用默认构造函数。

所以，这里有三种构造 bean 对象的方式，如下：

1. Supplier 回调。

2. 通过“工厂方法”的方式构造 bean 对象。

3. 通过“构造方法自动注入”的方式构造 bean 对象。

4. 通过“默认构造方法”的方式构造 bean 对象。

下面将会分析第1种和第3种构造bean对象的方法。

## 2 通过Supplier 回调创建 bean 对象

### 2.1 `Supplier`介绍

 Supplier 是什么呢？,Supplier是一个接口，位于`java.util.function`包下。代码如下：

```java
public interface Supplier<T> {

    T get();
    
}
```

Supplier 接口仅有一个功能性的 `#get()` 方法，该方法会返回一个 `<T>` 类型的对象，有点儿类似工厂方法。那这个接口有什么作用？用于指定创建 bean 的回调。如果我们设置了这样的回调，那么其他的构造器或者工厂方法都会没有用。

Spring 提供了相应的 setter 方法，用于设置Supplier 参数，代码如下：

```java
// AbstractBeanDefinition.java

/**
 * 创建 Bean 的 Supplier 对象
 */
@Nullable
private Supplier<?> instanceSupplier;

public void setInstanceSupplier(@Nullable Supplier<?> instanceSupplier) {
	this.instanceSupplier = instanceSupplier;
}
```

在构造 `BeanDefinition` 对象的时候，设置了 `instanceSupplier` 该值，代码如下（以 `RootBeanDefinition` 为例）：

```java
// RootBeanDefinition.java

public <T> RootBeanDefinition(@Nullable Class<T> beanClass, String scope, @Nullable Supplier<T> instanceSupplier) {
	super();
	setBeanClass(beanClass);
	setScope(scope);
	// 设置 instanceSupplier 属性
	setInstanceSupplier(instanceSupplier);
}
```

### 2.2 Supplier 回调

在`createBeanInstance`方法中，对应调用代码如下：

```java
Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
if (instanceSupplier != null) {
    return obtainFromSupplier(instanceSupplier, beanName);
}
```

先从 `BeanDefinition` 中获取 Supplier 对象。如果不为空，则调用 `obtainFromSupplier` 方法。代码如下：

```java
// AbstractAutowireCapableBeanFactory.java

/**
 * 当前线程，正在创建的 Bean 对象的名字
 */
private final NamedThreadLocal<String> currentlyCreatedBean = new NamedThreadLocal<>("Currently created bean");

protected BeanWrapper obtainFromSupplier(Supplier<?> instanceSupplier, String beanName) {
    Object instance;
    // 获得原创建的 Bean 的对象名
    String outerBean = this.currentlyCreatedBean.get();
    // 设置新的 Bean 的对象名，到 currentlyCreatedBean 中
    this.currentlyCreatedBean.set(beanName);
    try {
        // <1> 调用 Supplier 的 get()，返回一个 Bean 对象
        instance = instanceSupplier.get();
    } finally {
        // 设置原创建的 Bean 的对象名，到 currentlyCreatedBean 中
        if (outerBean != null) {
            this.currentlyCreatedBean.set(outerBean);
        } else {
            this.currentlyCreatedBean.remove();
        }
    }

    // 未创建 Bean 对象，则创建 NullBean 对象
    if (instance == null) {
        instance = new NullBean();
    }
    // <2> 创建 BeanWrapper 对象
    BeanWrapper bw = new BeanWrapperImpl(instance);
    // <3> 初始化 BeanWrapper 对象
    initBeanWrapper(bw);
    return bw;
}
```

流程如下：

1. `<1>` 处，调用 Supplier 的 `get()` 方法，获得一个 Bean 实例对象。

2. `<2>` 处，根据该实例对象构造一个` BeanWrapper `对象 `bw` 。

3. `<3>`处， 初始化该对象。

##  3 通过构造方法自动注入创建 bean 对象

这个初始化方法，我们可以简单理解为是**带有参数的构造方法**，来初始化 Bean 对象。代码逻辑较为复杂，需要大家耐心阅读。代码段如下：

```java
// AbstractAutowireCapableBeanFactory.java

protected BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {
    
    // 创建 ConstructorResolver 对象，并调用其 autowireConstructor 方法
    return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}
```

```java
// ConstructorResolver.java

public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd,
        @Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {
    
    // 封装 BeanWrapperImpl 对象，并完成初始化
    BeanWrapperImpl bw = new BeanWrapperImpl();
    this.beanFactory.initBeanWrapper(bw);

    // 获得 constructorToUse、argsHolderToUse、argsToUse
    Constructor<?> constructorToUse = null; // 构造函数
    ArgumentsHolder argsHolderToUse = null; // 构造参数
    Object[] argsToUse = null; // 构造参数

    // 确定构造参数(argsToUse)
    // 如果 getBean() 已经传递，则直接使用
    if (explicitArgs != null) {
        argsToUse = explicitArgs;
    } else {
        // 尝试从缓存中获取
        Object[] argsToResolve = null;
        synchronized (mbd.constructorArgumentLock) {
            // 获取已解析的构造方法
            constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse != null && mbd.constructorArgumentsResolved) {
                // 获取已解析的构造方法参数列表
                argsToUse = mbd.resolvedConstructorArguments;
                if (argsToUse == null) {
                    // 若 argsToUse 为空，则获取未解析的构造方法参数列表
                    argsToResolve = mbd.preparedConstructorArguments;
                }
            }	
        }
        // 缓存中存在,则解析存储在 BeanDefinition 中的参数
        // 如给定方法的构造函数 A(int ,int )，则通过此方法后就会把配置文件中的("1","1")转换为 (1,1)
        // 缓存中的值可能是原始值也有可能是最终值
        if (argsToResolve != null) {
            argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve, true);
        }
    }

    // 没有缓存，则尝试从配置文件中获取参数
    if (constructorToUse == null || argsToUse == null) {
        // 如果 chosenCtors 未传入，则获取构造方法们
        Constructor<?>[] candidates = chosenCtors;
        if (candidates == null) {
            Class<?> beanClass = mbd.getBeanClass();
            try {
                candidates = (mbd.isNonPublicAccessAllowed() ?
                        beanClass.getDeclaredConstructors() : beanClass.getConstructors());
            } catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Resolution of declared constructors on bean Class [" + beanClass.getName() +
                        "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
            }
        }

        // 创建 Bean
        if (candidates.length == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
            Constructor<?> uniqueCandidate = candidates[0];
            if (uniqueCandidate.getParameterCount() == 0) {
                synchronized (mbd.constructorArgumentLock) {
                    mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
                    mbd.constructorArgumentsResolved = true;
                    mbd.resolvedConstructorArguments = EMPTY_ARGS;
                }
                bw.setBeanInstance(instantiate(beanName, mbd, uniqueCandidate, EMPTY_ARGS));
                return bw;
            }
        }

        // 是否需要解析构造器
        boolean autowiring = (chosenCtors != null ||
                mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
        // 用于承载解析后的构造函数参数的值
        ConstructorArgumentValues resolvedValues = null;
        int minNrOfArgs;
        if (explicitArgs != null) {
            minNrOfArgs = explicitArgs.length;
        } else {
            // 从 BeanDefinition 中获取构造参数，也就是从配置文件中提取构造参数
            ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
            resolvedValues = new ConstructorArgumentValues();
            /*
             * 确定构造方法参数数量，比如下面的配置：
             *     <bean id="person" class="com.coolsen.demo.autowire.Person">
             *         <constructor-arg index="0" value="xiaoming"/>
             *         <constructor-arg index="1" value="1"/>
             *         <constructor-arg index="2" value="man"/>
             *     </bean>
             *
             * 此时 minNrOfArgs = maxIndex + 1 = 2 + 1 = 3，除了计算 minNrOfArgs，
             * 下面的方法还会将 cargs 中的参数数据转存到 resolvedValues 中
             */
            minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
        }

        // 对构造函数进行排序处理
        // public 构造函数优先参数数量降序，非public 构造函数参数数量降序
        AutowireUtils.sortConstructors(candidates);

        // 最小参数类型权重
        int minTypeDiffWeight = Integer.MAX_VALUE;
        Set<Constructor<?>> ambiguousConstructors = null;
        LinkedList<UnsatisfiedDependencyException> causes = null;

        // 迭代所有构造函数
        for (Constructor<?> candidate : candidates) {
            // 获取该构造函数的参数类型
            Class<?>[] paramTypes = candidate.getParameterTypes();

            // 如果已经找到选用的构造函数或者需要的参数个数小于当前的构造函数参数个数，则终止。
            /*
             * 下面的 if 分支的用途是：若匹配到到合适的构造方法了，提前结束 for 循环
             * constructorToUse != null 这个条件比较好理解，下面分析一下条件 argsToUse.length > paramTypes.length：
             * 前面说到 AutowireUtils.sortConstructors(candidates) 用于对构造方法进行
             * 排序，排序规则如下：
             *   1. 具有 public 访问权限的构造方法排在非 public 构造方法前
             *   2. 参数数量多的构造方法排在前面
             *
             * 假设现在有一组构造方法按照上面的排序规则进行排序，排序结果如下（省略参数名称）：
             *
             *   1. public Hello(Object, Object, Object)
             *   2. public Hello(Object, Object)
             *   3. public Hello(Object)
             *   4. protected Hello(Integer, Object, Object, Object)
             *   5. protected Hello(Integer, Object, Object)
             *   6. protected Hello(Integer, Object)
             *
             * argsToUse = [num1, obj2]，可以匹配上的构造方法2和构造方法6。由于构造方法2有
             * 更高的访问权限，所以没理由不选他（尽管后者在参数类型上更加匹配）。由于构造方法3
             * 参数数量 < argsToUse.length，参数数量上不匹配，也不应该选。所以 
             * argsToUse.length > paramTypes.length 这个条件用途是：在条件 
             * constructorToUse != null 成立的情况下，通过判断参数数量与参数值数量
             * （argsToUse.length）是否一致，来决定是否提前终止构造方法匹配逻辑。
             */
            
            if (constructorToUse != null && argsToUse.length > paramTypes.length) {
                break;
            }
            
            /*
             * 构造方法参数数量低于配置的参数数量，则忽略当前构造方法，并重试。比如 
             * argsToUse = [obj1, obj2, obj3, obj4]，上面的构造方法列表中，
             * 构造方法1、2和3显然不是合适选择，忽略之。
             */
            if (paramTypes.length < minNrOfArgs) {
                continue;
            }

            // 参数持有者 ArgumentsHolder 对象	
            ArgumentsHolder argsHolder;
            if (resolvedValues != null) {
                try {
                    // 判断否则方法是否有 ConstructorProperties 注解，若有，则取注解中的值
                    String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
                    if (paramNames == null) {
                        // 获取构造函数、方法参数的探测器
                        ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                        if (pnd != null) {
                            // 通过探测器获取构造函数的参数名称
                            paramNames = pnd.getParameterNames(candidate);
                        }
                    }
                    
                     /* 
                     * 创建参数值列表，返回 argsHolder 会包含进行类型转换后的参数值，比如下
                     * 面的配置:
                     *
                     *  <bean id="person" class="com.coolsen.demo.autowire.Person">
                     *         <constructor-arg name="name" value="xiaoming"/>
                     *         <constructor-arg name="age" value="1"/>
                     *         <constructor-arg name="sex" value="man"/>
                     *  </bean>
                     *
                     * Person 的成员变量 age 是 Integer 类型的，但由于在 Spring 配置中
                     * 只能配成 String 类型，所以这里要进行类型转换。
                     */
                    argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
                            getUserDeclaredConstructor(candidate), autowiring, candidates.length == 1);
                } catch (UnsatisfiedDependencyException ex) {
                    if (logger.isTraceEnabled()) {
                        logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
                    }
                    // Swallow and try next constructor.
                    if (causes == null) {
                        causes = new LinkedList<>();
                    }
                    causes.add(ex);
                    continue; // continue ，继续执行
                }
            } else {
                if (paramTypes.length != explicitArgs.length) {
                    continue;
                }
                argsHolder = new ArgumentsHolder(explicitArgs);
            }

             /*
             * 计算参数值（argsHolder.arguments）每个参数类型与构造方法参数列表
             * （paramTypes）中参数的类型差异量，差异量越大表明参数类型差异越大。参数类型差异
             * 越大，表明当前构造方法并不是一个最合适的候选项。引入差异量（typeDiffWeight）
             * 变量目的：是将候选构造方法的参数列表类型与参数值列表类型的差异进行量化，通过量化
             * 后的数值筛选出最合适的构造方法。
             * 
             * 讲完差异量，再来说说 mbd.isLenientConstructorResolution() 条件。
             * 官方的解释是：返回构造方法的解析模式，有宽松模式（lenient mode）和严格模式
             * （strict mode）两种类型可选。
             * 严格模式：解析构造函数时，必须所有的都需要匹配，否则抛出异常
             * 宽松模式：使用具有"最接近的模式"进行匹配
             * typeDiffWeight：类型差异权重
             */
            int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
                    argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
            // 如果它代表着当前最接近的匹配，则选择其作为构造函数
            if (typeDiffWeight < minTypeDiffWeight) {
                constructorToUse = candidate;
                argsHolderToUse = argsHolder;
                argsToUse = argsHolder.arguments;
                minTypeDiffWeight = typeDiffWeight;
                ambiguousConstructors = null;
            } 
            
            /* 
             * 如果两个构造方法与参数值类型列表之间的差异量一致，那么这两个方法都可以作为
             * 候选项，这个时候就出现歧义了，这里先把有歧义的构造方法放入 
             * ambiguousConstructors 集合中
             */
            else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
                if (ambiguousConstructors == null) {
                    ambiguousConstructors = new LinkedHashSet<>();
                    ambiguousConstructors.add(constructorToUse);
                }
                ambiguousConstructors.add(candidate);
            }
        }

        // 若上面未能筛选出合适的构造方法，这里将抛出 BeanCreationException 异常
        if (constructorToUse == null) {
            if (causes != null) {
                UnsatisfiedDependencyException ex = causes.removeLast();
                for (Exception cause : causes) {
                    this.beanFactory.onSuppressedException(cause);
                }
                throw ex;
            }
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Could not resolve matching constructor " +
                    "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
        } 
        
         /*
         * 如果 constructorToUse != null，且 ambiguousConstructors 也不为空，表明解析
         * 出了多个的合适的构造方法，此时就出现歧义了。Spring 不会擅自决定使用哪个构造方法，
         * 所以抛出异常。
         */
        else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Ambiguous constructor matches found in bean '" + beanName + "' " +
                    "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
                    ambiguousConstructors);
        }

        if (explicitArgs == null) {
             /*
             * 缓存相关信息，比如：
             *   1. 已解析出的构造方法对象 resolvedConstructorOrFactoryMethod
             *   2. 构造方法参数列表是否已解析标志 constructorArgumentsResolved
             *   3. 参数值列表 resolvedConstructorArguments 或 preparedConstructorArguments
             *
             * 这些信息可用在其他地方，用于进行快捷判断
             */
            argsHolderToUse.storeCache(mbd, constructorToUse);
        }
    }

    // 创建 Bean 对象，并设置到 bw 中
    bw.setBeanInstance(instantiate(beanName, mbd, constructorToUse, argsToUse));
    return bw;
}
```

上面的方法逻辑比较复杂，做了不少事情，该方法的核心逻辑是根据参数值类型筛选合适的构造方法。解析出合适的构造方法后，剩下的工作就是构建 bean 对象了，这个工作交给了实例化策略去做。上面方法的整体流程为：

1. 创建 `BeanWrapperImpl `对象。

2. 解析构造方法参数，并算出 `minNrOfArgs`。

3. 获取构造方法列表，并排序。

4. 遍历排序好的构造方法列表，筛选合适的构造方法。

   * 获取构造方法参数列表中每个参数的名称。

   * 再次解析参数，此次解析会将value 属性值进行类型转换，由 String 转为合适的类型。

   * 计算构造方法参数列表与参数值列表之间的类型差异量，以筛选出更为合适的构造方法。

5. 缓存已筛选出的构造方法以及参数值列表，若再次创建 bean 实例时，可直接使用，无需再次进行筛选。

6. 使用初始化策略创建 bean 对象。

7. 将 bean 对象放入 `BeanWrapperImpl `对象中，并返回该对象。

下一小节将分析`instantiate`方法，以及通过反射和CGLIB 创建 Bean 对象。

## 4  `instantiate`方法

`instantiate`方法，对应上面代码的278行，代码如下：

```java
//ConstructorResolver.java

private Object instantiate(
			String beanName, RootBeanDefinition mbd, Constructor<?> constructorToUse, Object[] argsToUse) {

		try {
			InstantiationStrategy strategy = this.beanFactory.getInstantiationStrategy();
			if (System.getSecurityManager() != null) {
				return AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						strategy.instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse),
						this.beanFactory.getAccessControlContext());
			}
			else {
                /*
             	* 调用实例化策略创建实例，默认情况下使用反射创建实例。如果 bean 的配置信息中
             	* 包含 lookup-method 和 replace-method，则通过 CGLIB 增强 bean 实例
             	*/
				return strategy.instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean instantiation via constructor failed", ex);
		}
	}
```

上面第18行的`instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse)`代码如下：

```java
// SimpleInstantiationStrategy.java

@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,final Constructor<?> ctor, Object... args) {
    // <1> 没有覆盖，直接使用反射实例化即可
    if (!bd.hasMethodOverrides()) {
        if (System.getSecurityManager() != null) {
            // 设置构造方法，可访问
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                ReflectionUtils.makeAccessible(ctor);
                return null;
            });
        }
        // 通过 BeanUtils 直接使用构造器对象实例化 Bean 对象
        return BeanUtils.instantiateClass(ctor, args);
    } else {
        // <2> 生成 CGLIB 创建的子类对象
        return instantiateWithMethodInjection(bd, beanName, owner, ctor, args);
    }
}
```

* `<1>`处，如果该 bean 没有配置 `lookup-method`、`replaced-method` 标签或者 `@Lookup` 注解，则直接通过**反射**的方式实例化 Bean 对象即可。
* `<2>处` ，使用 CGLIB 进行动态代理，因为可以在创建代理的同时将动态方法织入类中。

### 4.1 反射创建 Bean 对象

调用工具类 `BeanUtils` 的 `instantiateClass(Constructor ctor, Object... args)` 方法，完成反射工作，创建对象。代码如下：

```java
// BeanUtils.java

public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
    Assert.notNull(ctor, "Constructor must not be null");
    try {
        // 设置构造方法，可访问
        ReflectionUtils.makeAccessible(ctor);
        // 使用构造方法，创建对象
        return (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass()) ?
                KotlinDelegate.instantiateClass(ctor, args) : ctor.newInstance(args));
    // 各种异常的翻译，最终统一抛出 BeanInstantiationException 异常
    } catch (InstantiationException ex) {
        throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
    } catch (IllegalAccessException ex) {
        throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
    } catch (IllegalArgumentException ex) {
        throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
    } catch (InvocationTargetException ex) {
        throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
    }
}
```

### 4.2 CGLIB 创建 Bean 对象

```java
// SimpleInstantiationStrategy.java

protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
	throw new UnsupportedOperationException("Method Injection not supported in SimpleInstantiationStrategy");
}
```

方法默认是**没有实现**的，具体过程由其子类`CglibSubclassingInstantiationStrategy` 来实现。代码如下：

```java
// CglibSubclassingInstantiationStrategy.java

@Override
protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
    return instantiateWithMethodInjection(bd, beanName, owner, null);
}
@Override
protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner, @Nullable Constructor<?> ctor, Object... args) {
    // Must generate CGLIB subclass...
    // 通过CGLIB生成一个子类对象
    return new CglibSubclassCreator(bd, owner).instantiate(ctor, args);
}
```

创建一个 `CglibSubclassCreator` 对象，后调用其 `#instantiate(Constructor ctor, Object... args)` 方法，生成其子类对象。代码如下：

```java
// CglibSubclassingInstantiationStrategy.java

public Object instantiate(@Nullable Constructor<?> ctor, Object... args) {
    // 通过 Cglib 创建一个代理类
    Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
    Object instance;
    // 没有构造器，通过 BeanUtils 使用默认构造器创建一个bean实例
    if (ctor == null) {
        instance = BeanUtils.instantiateClass(subclass);
    } else {
        try {
            // 获取代理类对应的构造器对象，并实例化 bean
            Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
            instance = enhancedSubclassConstructor.newInstance(args);
        } catch (Exception ex) {
            throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
                    "Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
        }
    }
    // 为了避免 memory leaks 异常，直接在 bean 实例上设置回调对象
    Factory factory = (Factory) instance;
    factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
            new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
            new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
    return instance;
}
```

## 小结

`createBeanInstance`已经分析完毕了,其中工厂方法初始化和默认构造函数注入没有分析。有一个很重要的原因就是，构造函数自动注入初始化即`autowireConstructor`的方法实在是太长了，逻辑很复杂，分析完已经晕了，哈哈。很感谢一些博主，因为他们的博文，我看起源码来才能更快的理解。继续加油！
