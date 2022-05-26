[toc]

# Spring IOC

> Spring：5.3.15
>
> SpringBoot：2.6.3
>
> [Spring Framework Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/)
>
> [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/2.6.3/reference/html/)

## IOC 概念

**IOC**：Inversion of Control，控制反转，是面向对象编程中的一种设计原则，对象的生命周期由IOC容器控制，业务代码只需要去关注业务流程，而不用去关心对象生命周期的控制，因此可以用来**减低代码之间的耦合度**。最常见的方式叫做依赖注入（DI）。对象在被创建的时候，由IOC容器将对象所依赖的对象的引用传递给它，即将依赖注入到对象中。是一种思想。

**DI**：Dependency Injection，依赖注入，是IOC的一种实现。

> [浅谈IOC-说清楚IOC是什么](https://www.cnblogs.com/DebugLZQ/archive/2013/06/05/3107957.html)

IOC本身是一个容器，可以认为是一个很强大的HashMap，除了能存对象，还能对对象进行各种装配修饰。

## Spring IOC 整体流程

## 重要类/接口

### BeanDefinition

`BeanDefinition`是Spring定义一个Bean的最小接口，主要的目的是允许`BeanFactoryPostProcessor`后置处理器去审视和修改属性值和其他bean元信息。

总结`BeanDefinition`保存了对一个Bean的定义信息，包括：

- 对Bean实例的定义信息
    - Bean的类名
    - 构造参数、有哪些属性（Field）
- 对Spring IOC的支持信息
    - Bean的角色信息（APPLICATION、SUPPORT、INFRASTRUCTURE）
    - 是否懒加载该Bean
    - 该Bean的作用域（SINGLETON、PROTOTYPE）
    - 该Bean的依赖关系（依赖的Bean）
    - 是将该Bean作为依赖注入的候选者（是否是Autowired的候选`void setAutowireCandidate(boolean)`）
    - 是否是`@Autowired`的主要候选`void setPrimary(boolean)`
    - 设置父Bean的名称
    - 设置FactoryBean名称、工厂方法名称
    - 设置初始化、销毁方法名称

```java
/**
 * A BeanDefinition describes a bean instance, which has property values,
 * constructor argument values, and further information supplied by
 * concrete implementations.
 * 
 * BeanDefinition 描述了一个bean实例，具有属性值、构造函数参数值以及具体实现所提供的更多信息。
 *
 * <p>This is just a minimal interface: The main intention is to allow a
 * {@link BeanFactoryPostProcessor} to introspect and modify property values
 * and other bean metadata.
 * 
 * 这仅仅是一个最小的接口：主要的目的是允许BeanFactoryPostProcessor后置处理器去审视和修改属性值和其他bean元信息。
 */
 // AttributeAccessor 
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    ...
}
```

`AttributeAccessor`接口主要定义了对任意对象添加和访问元数据的方法，

```java
/**
 * Interface defining a generic contract for attaching and accessing metadata
 * to/from arbitrary objects.
 */
public interface AttributeAccessor {
  void setAttribute(String name, @Nullable Object value);
  @Nullable
  Object getAttribute(String name);
  
  @SuppressWarnings("unchecked")
  default <T> T computeAttribute(String name, Function<String, T> computeFunction) {
    ...
  }
  
  @Nullable
  Object removeAttribute(String name);
  boolean hasAttribute(String name);
  String[] attributeNames();

}
```

`BeanMetadataElement`定义了获取配置源对象的方法。

```java
/**
 * Interface to be implemented by bean metadata elements
 * that carry a configuration source object.
public interface BeanMetadataElement {
  /**
   * Return the configuration source {@code Object} for this metadata element
   * (may be {@code null}).
   */
  @Nullable
  default Object getSource() {
    return null;
  }
}
```

`BeanDefinition`中的属性：

```java
  // 域是单例还是原型
  String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
  String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;
  
  /* 角色信息 */
  // 一般用户定义的Bean会是这个角色
  int ROLE_APPLICATION = 0;
  // 该角色表明该Bean是一些较大配置的支持部分，一般是ComponentDefinition得到的配置（一般XML）
  int ROLE_SUPPORT = 1;
  // 该角色表示它是Spring内部使用的Bean
  int ROLE_INFRASTRUCTURE = 2;
```

`BeanDefinition`中的方法：

```java
  // 父Bean名称操作
  void setParentName(@Nullable String parentName);
  @Nullable
  String getParentName();
  
  // 设置类名，可能在后置处理过程中被替换
  void setBeanClassName(@Nullable String beanClassName);
  // 获取类名，在运行时不一定是实际类名
  @Nullable
  String getBeanClassName();

  // 设置Bean的Scope
  void setScope(@Nullable String scope);
  @Nullable
  String getScope();

  // 是否懒加载
  void setLazyInit(boolean lazyInit);
  boolean isLazyInit();
  
  // 设置当前Bean依赖的其它Bean，BeanFactory会保证它们先被初始化
  void setDependsOn(@Nullable String... dependsOn);
  @Nullable
  String[] getDependsOn();

  // 设置当前Bean是否是一个Autowire的候选者，如果是，就会对当前Bean进行@Autowired解析 
  // 即当前Bean是否可以作为候选者注入到其它的Bean
  // TODO: 不确定是否还有别的注解跟这里有关系 @Resource?
  void setAutowireCandidate(boolean autowireCandidate);
  boolean isAutowireCandidate();

  // 设置当前Bean是否是主要的Autowired候选Bean，如果是，进行注入时会优先注入当前Bean
  void setPrimary(boolean primary);
  boolean isPrimary();

  // 设置当前Bean的FactoryBean对象（用来生成真正Bean实例）的名称
  void setFactoryBeanName(@Nullable String factoryBeanName);
  @Nullable
  String getFactoryBeanName();

  // 设置工厂方法名称
  void setFactoryMethodName(@Nullable String factoryMethodName);
  @Nullable
  String getFactoryMethodName();

  // 获取当前Bean的构造器参数列表
  ConstructorArgumentValues getConstructorArgumentValues();

  default boolean hasConstructorArgumentValues() {
    return !getConstructorArgumentValues().isEmpty();
  }

  // 返回当前Bean的PropertyValues，PropertyValue是Bean某个Field的包装，保存了name和value以及其它方法
  MutablePropertyValues getPropertyValues();
  default boolean hasPropertyValues() {
    return !getPropertyValues().isEmpty();
  }

  // 设置Init方法的名称
  void setInitMethodName(@Nullable String initMethodName);
  @Nullable
  String getInitMethodName();

  // 设置销毁方法
  void setDestroyMethodName(@Nullable String destroyMethodName);
  @Nullable
  String getDestroyMethodName();

  // 设置Bean的角色
  void setRole(int role);
  int getRole();

  /**
   * Set a human-readable description of this bean definition.
   * 设置一个人类能看懂的对当前bean definition的描述信息？ 😂🤣😅
   * @since 5.1
   */  
  void setDescription(@Nullable String description);
  @Nullable
  String getDescription();


  // Read-only attributes
  ResolvableType getResolvableType();

  // 当前Bean是否是单例、原型、Abstract
  boolean isSingleton();
  boolean isPrototype();
  boolean isAbstract();

  @Nullable
  String getResourceDescription();
  @Nullable
  BeanDefinition getOriginatingBeanDefinition();

}
```

### AbstractBeanDefinition

主要是对`BeanDefinition`中的方法进行了实现，此外还添加了以下新功能：

- 添加了装配策略的描述：不进行依赖注入、通过名称、类型、构造器和自动选择。这个属性是和当前Bean的Field相关的，比如假设当前bean的`BeanDefinition`实例的装配策略是`BY_NAME`的话，容器会找到一个匹配的bean给当前bean的实例进行注入，就算没有`@Autowired`、`@Resource`注解（和这俩没啥关系其实）。
- 添加了属性注入检测机制`dependencyCheck`
- 添加了被重写方法的支持
- 添加了是否是合成的`BeanDefinition`的标识

一些常量和属性：

```java
  /* 下面是定义的一些常量 */
  // 默认Bean的Scope是单例
  public static final String SCOPE_DEFAULT = "";
  
  /* 下面的几种依赖注入策略是相对于当前Bean来说的，也就是说当前Bean中的属性会适用下面的策略，
   * 例如，如果当前BeanDefinition实例的注入策略是BY_NAME的话，会在容器里找有没有这个名称的
   * bean，如果有，会给当前bean进行注入（和@Autowired注解没啥关系）。
   */
  // 不进行依赖注入
  // TODO:基于注解的依赖注入好像是这种模式
  public static final int AUTOWIRE_NO = AutowireCapableBeanFactory.AUTOWIRE_NO;
  // 通过名称进行依赖注入
  // TODO:通过set方法注入，set的名称需要和bean的名称一致
  public static final int AUTOWIRE_BY_NAME = AutowireCapableBeanFactory.AUTOWIRE_BY_NAME;
  // 通过类型
  // 同样是通过set方法注入
  public static final int AUTOWIRE_BY_TYPE = AutowireCapableBeanFactory.AUTOWIRE_BY_TYPE;
  // 通过构造器进行依赖注入
  public static final int AUTOWIRE_CONSTRUCTOR = AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR;
  // 由Spring检测选择合适的注入策略，从3.0版本开始已经被废弃
  @Deprecated
  public static final int AUTOWIRE_AUTODETECT = AutowireCapableBeanFactory.AUTOWIRE_AUTODETECT;

  // 下面几个是关于检查Bean的依赖注入情况的策略
  // 不进行检查
  public static final int DEPENDENCY_CHECK_NONE = 0;
  // 仅检查依赖的对象
  public static final int DEPENDENCY_CHECK_OBJECTS = 1;
  // 仅检查依赖的基本类型、String、集合类型
  public static final int DEPENDENCY_CHECK_SIMPLE = 2;
  // 检查全部的属性
  public static final int DEPENDENCY_CHECK_ALL = 3;

  // 关闭上下文时调用的方法，未指定的情况下由容器推断，目前可能是以下两种可能
  // {close(), shutdown()}
  public static final String INFER_METHOD = "(inferred)";

  // 指向当前Bean的Class对象
  @Nullable
  private volatile Object beanClass;
  // Bean的作用范围默认是单例
  @Nullable
  private String scope = SCOPE_DEFAULT;
  // 是否是abstract的，默认false
  private boolean abstractFlag = false;
  // 是否懒加载，默认false
  @Nullable
  private Boolean lazyInit;

  // 当前Bean的依赖注入策略，默认不自动进行依赖注入，仅对当前Bean的属性有效，详细情况看上面策略处的解析
  private int autowireMode = AUTOWIRE_NO;
  // 默认不进行依赖检测
  private int dependencyCheck = DEPENDENCY_CHECK_NONE;
  // 依赖的Bean数组
  @Nullable
  private String[] dependsOn;
  // 是否可以作为自动装配的后选择
  private boolean autowireCandidate = true;
  // 默认不作为主要候选者
  private boolean primary = false;
  // 用于解析autowired 候选者的
  private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>();

  // Supplier是jdk中的函数式接口，里面方法的签名：T get()，也就是说调用instanceSupplier会返回一个T对象
  // 这个属性是创建Bean实例的回调函数，作为声明式的指定工厂方法的替代。
  // 如果这个回调方法被设置了，那么它会覆盖所有其他创建者或者工厂方法的元信息。但是，bean属性填充和
  // 潜在的注解注入仍然会照常适用。
  // TODO: 不是很清楚具体作用，上面是doc的翻译
  @Nullable
  private Supplier<?> instanceSupplier;

  // 对于指向非public的构造函数和方法的外部元数据，指定是否允许访问它们。
  // 默认true，false代表仅允许访问public修饰的。
  // 适用于构造函数解析、工厂方法解析以及初始化/销毁方法。
  // 对于bean的属性访问器在任何情况下都必须是public的，并且不受此设置的影响。
  // 需要注意的是，注解驱动的配置仍然能够访问已被注解的非public成员。
  // 此设置仅适用于此bean定义汇总的外部化元数据。
  // TODO: 不是很清楚具体作用，上面是doc的翻译
  private boolean nonPublicAccessAllowed = true;
  
  // 指定是在宽松模式下解析构造函数(默认是true)还是切换到严格解析
  // （如果在转换参数时所有匹配的构造函数都不明确，则抛出异常，而宽松模式下
  //  将使用“最接近”类型匹配的构造函数）
  private boolean lenientConstructorResolution = true;
  
  // 对应的factoryBean的名称
  @Nullable
  private String factoryBeanName;
  // 工厂方法名
  @Nullable
  private String factoryMethodName;
  // 构造器参数值
  @Nullable
  private ConstructorArgumentValues constructorArgumentValues;
  // 存储Bean的属性的名称以及对应的值
  @Nullable
  private MutablePropertyValues propertyValues;
  // 存储被IOC容器覆盖的方法的信息
  private MethodOverrides methodOverrides = new MethodOverrides();
  // init方法名称
  @Nullable
  private String initMethodName;
  // 销毁方法名称
  @Nullable
  private String destroyMethodName;

  // Specify whether or not the configured initializer method is the default.
  private boolean enforceInitMethod = true;
  // Specify whether or not the configured destroy method is the default.
  private boolean enforceDestroyMethod = true;

  // 是否是合成类
  private boolean synthetic = false;
  // 角色
  private int role = BeanDefinition.ROLE_APPLICATION;
  // 人能看得懂的描述信息
  @Nullable
  private String description;
  //
  @Nullable
  private Resource resource;
```

### RootBeanDefinition

`RootBeanDefinition`表示在运行时Spring `BeanFactory`中合并的bean definition的特殊的`BeanDefinition`。它可能是从相互继承的多个原始`BeanDefinition`创建的，通常是注册为`GenericBeanDefinition`的beans。`RootBeanDefinition`本质上是运行时的“统一”`BeanDefinition`视图。`RootBeanDefinition`也可以用于配置阶段注册单个bean definition。然而，自从Spring2.5依赖，以编程方式注册bean definition的首选是`GenericBeanDefinition`。`GenericBeanDefinition`的优点是它允许动态定义父依赖项，而不是将角色“硬编码”为`RootBeanDefinition`。它不能作为其它`BeanDefinition`的子定义，`setParentName(parent)`如果`parent`不为`null`就会报错。

相对于`AbstractBeanDefinition`新添加的功能：

- 增加存储`BeanDefinition`跟`name`、`alias`对应关系的属性
- 增加`AnnotatedElement`属性，可以解析注解信息。
- 增加了缓存相关属性 // TODO:
- 

继承视图：

![image-20220525194611795](../../../Library/Application%20Support/typora-user-images/image-20220525194611795.png)

定义了更多的属性：

```java
  // BeanDefinitionHolder是用来存储name、alias和BeanDefinition关系的
  // 内部三个属性: 1. BeanDefinition beanDefinition  => BeanDefinition
  //              2. String beanName                => 它的名字
  //              3. String[] aliases;              => 这个BeanDefinition的所有别名
  @Nullable
  private BeanDefinitionHolder decoratedDefinition;

  // AnnotatedElement 是java.lang.reflect下的包，反射用的，可以获取注解信息
  @Nullable
  private AnnotatedElement qualifiedElement;

  // BeanDefinition是否需要重新合并，默认false即不需要
  volatile boolean stale;
  
  // 是否允许缓存
  boolean allowCaching = true;
  // 工厂方法是否唯一
  boolean isFactoryMethodUnique;

  // 可以解析类的超类、父接口、泛型参数相关的信息
  @Nullable
  volatile ResolvableType targetType;

  /** Package-visible field for caching the determined Class of a given bean definition. */
  @Nullable
  volatile Class<?> resolvedTargetType;
  // 是否是一个FactoryBean
  /** Package-visible field for caching if the bean is a factory bean. */
  @Nullable
  volatile Boolean isFactoryBean;
  // 返回值类型
  /** Package-visible field for caching the return type of a generically typed factory method. */
  @Nullable
  volatile ResolvableType factoryMethodReturnType;

  /** Package-visible field for caching a unique factory method candidate for introspection. */
  @Nullable
  volatile Method factoryMethodToIntrospect;

  /** Package-visible field for caching a resolved destroy method name (also for inferred). */
  @Nullable
  volatile String resolvedDestroyMethodName;

  // 构造器锁
  /** Common lock for the four constructor fields below. */
  final Object constructorArgumentLock = new Object();
  // 缓存已经解析的构造方法或者工厂方法
  /** Package-visible field for caching the resolved constructor or factory method. */
  @Nullable
  Executable resolvedConstructorOrFactoryMethod;
  // 构造参数解析
  /** Package-visible field that marks the constructor arguments as resolved. */
  boolean constructorArgumentsResolved = false;
  // 用于缓存完全解析的构造方法参数
  /** Package-visible field for caching fully resolved constructor arguments. */
  @Nullable
  Object[] resolvedConstructorArguments;
   // 缓存待解析的构造方法参数
  /** Package-visible field for caching partly prepared constructor arguments. */
  @Nullable
  Object[] preparedConstructorArguments;

  // 后置处理器锁
  /** Common lock for the two post-processing fields below. */
  final Object postProcessingLock = new Object();

  // 是否被 MergedBeanDefinitionPostProcessor 处理过
  /** Package-visible field that indicates MergedBeanDefinitionPostProcessor having been applied. */
  boolean postProcessed = false;
  // 是否初始化前的后置处理器已经启用
  /** Package-visible field that indicates a before-instantiation post-processor having kicked in. */
  @Nullable
  volatile Boolean beforeInstantiationResolved;
  //
  @Nullable
  private Set<Member> externallyManagedConfigMembers;
  // 初始化的回调函数
  @Nullable
  private Set<String> externallyManagedInitMethods;
  // 销毁时的回调函数
  @Nullable
  private Set<String> externallyManagedDestroyMethods;
```



### GenericBeanDefinition

`GenericBeanDefinition`是用于标准bean definition的一站式服务。与任何bean definition一样，它允许指定类以及可选的构造函数参数值和属性值。此外，可以通过`parentName`属性灵活配置从父bean definition派生的内容。

通常，使用这个GenericBeanDefinition类来注册用户可见的bean definition（后处理器可能会对其进行操作，甚至可能会重新配置父名称）。在父/子关系恰好是预先确定的情况下，使用`RootBeanDefinition`/`ChildBeanDefinition`。

继承关系：

![image-20220525194638746](../../../Library/Application%20Support/typora-user-images/image-20220525194638746.png)

### BeanDefinitionReader

该接口定义了读取`BeanDefinition`的API。

### BeadFactory

The root interface for accessing a Spring bean container. 

访问Spring bean容器的顶级接口。该接口中主要定义了从容器中获取bean实例的方法。

The [`BeanFactory`](https://docs.spring.io/spring-framework/docs/5.3.20/javadoc-api/org/springframework/beans/factory/BeanFactory.html) interface provides an advanced configuration mechanism capable of managing any type of object.[^1]

**有以下特点：**

- 不会主动调用BeanFactory后置处理器
- 不会主动添加Bean后置处理器
- 不会主动初始化单例
- 不会解析beanFactory，不会解析${}与#{}

IOC（DI）、Bean生命周期的各种功能，都由该接口的实现类`DefaultListableBeanFactory`提供。

![image-20220520165329854](../../../Library/Application%20Support/typora-user-images/image-20220520165329854.png)



### ApplicationContext

![image-20220520165424677](../../../Library/Application%20Support/typora-user-images/image-20220520165424677.png)

`BeadFactory`的子接口，相对于`BeadFactory`，功能更强大。

- `MessageSource` ：国际化相关，用于多语言支持。
- `ResourcePatternResolver`：通配符匹配资源，返回一个`Resource`（对文件、classpath资源的抽象）数组。
- `ApplicationEventPublisher`：事件发布，支持发布订阅。通过`context.publishEvent(Obj)`来发布消息，`@EventListener`注解的方法接收。
- `EnvironmentCapable`：环境相关支持，可以用于获取环境变量等配置信息。

实现类：

- `ClassPathXmlApplicationContext`
- `FileSystemXmlApplicationContext`
- `AnnotationConfigApplicationContext`

- `AnnotationConfigServletWebApplicationContext`：提供基于Web支持

### BeanPostProcessor

常见后置处理器：

- ``

### BeanFactoryPostProcessor

### DefaultSingletonRegistry

解决循环依赖。

```java
  /** Cache of singleton objects: bean name to bean instance. */
  // 第1级缓存，用于存储已经完全创建完成的Bean对象
  // （已经执行构造方法，属性已经注入）
  private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

  /** Cache of singleton factories: bean name to ObjectFactory. */
  // 第3级缓存，用于存储已经实例化，但是存在对象依赖的Bean的包装类ObjectFactory<?>的Bean
  private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

  /** Cache of early singleton objects: bean name to bean instance. */
  // 第2级缓存，用于存储已经实例化，但是还未进行依赖注入的Bean实例
  private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

  /** Set of registered singletons, containing the bean names in registration order. */
  private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

  /** Names of beans that are currently in creation. */
  private final Set<String> singletonsCurrentlyInCreation =
      Collections.newSetFromMap(new ConcurrentHashMap<>(16));

  /** Names of beans currently excluded from in creation checks. */
  private final Set<String> inCreationCheckExclusions =
      Collections.newSetFromMap(new ConcurrentHashMap<>(16));

  /** Collection of suppressed Exceptions, available for associating related causes. */
  @Nullable
  private Set<Exception> suppressedExceptions;

  /** Flag that indicates whether we're currently within destroySingletons. */
  private boolean singletonsCurrentlyInDestruction = false;

  /** Disposable bean instances: bean name to disposable instance. */
  private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

  /** Map between containing bean names: bean name to Set of bean names that the bean contains. */
  private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<>(16);

  /** Map between dependent bean names: bean name to Set of dependent bean names. */
  private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);

  /** Map between depending bean names: bean name to Set of bean names for the bean's dependencies. */
  private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);
```



## Scope

### 常见的scope

- `singleton` 单例
- `prototype` 原型
- `request`     请求域，只在当前请求中有效
- `session`     会话域，只在当前会话中有效
- `application`  应用域，只在当前应用程序有效

## AOP

### AspetJ（ajc）编译器增强

可以增强静态方法。静态方法属于类，是不能被继承重写的，因此无法通过代理的方式进行增强。

原理是**编译时**直接修改class文件，因此可以做到增强类的所有方法包括静态方法。

### agent类加载时增强

VM options添加参数：`-javaagent:{aptectjweaver-*.jar}`

### 代理方式增强

- proxyTargetClass = false， 目标实现了接口，用jdk实现
- proxyTargetClass = false，目标没有实现接口，用cglib实现
- proxyTargetClass = true，使用cglib实现

#### JDK

只能针对接口代理，代理类必须实现接口。

#### Cglib

通过继承实现，因此被代理类不能是final的。

#### ASM (JDK/Cglib底层原理)



____

[^1]:[Core Technologies](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans)
