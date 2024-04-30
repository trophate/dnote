# 创建

指定类创建：

```java
AnnotationConfigApplicationContext context1 = new AnnotationConfigApplicationContext(Apple.class, Banana.class);
System.out.println(context1.getBean(Apple.class));
```

类不需要用注解标识。

扫描包创建：

```java
AnnotationConfigApplicationContext context2 = new AnnotationConfigApplicationContext("com.test.beans");
System.out.println(context2.getBean(Mango.class));
```

类需要用注解标识。



## 创建流程

以指定类创建为例。

进入构造方法：

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this();
    // 注册
    register(componentClasses);
    // 刷新
    refresh();
}
```

this是无参构造方法，该方法创建了reader和scanner对象。reader用于显示注册bean，scanner用于路径扫描注册bean。两者都可以解析bean注解。

```java
public AnnotationConfigApplicationContext() {
    StartupStep createAnnotatedBeanDefReader = this.getApplicationStartup().start("spring.context.annotated-bean-reader.create");
    // this——将自身作为注册表
    this.reader = new AnnotatedBeanDefinitionReader(this);
    createAnnotatedBeanDefReader.end();
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

register用于将bean定义解析生成BeanDefinition并存入集合中。refresh用于通过定义实例化bean。

register调用reader.register进行注册。

```java
// reader.register -> doRegisterBean
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
                                @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
                                @Nullable BeanDefinitionCustomizer[] customizers) {

    // 生成bean定义
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    // 通过@Conditional判断是否注册
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    // 设置回调代替构造函数/工厂方法创建实例
    abd.setInstanceSupplier(supplier);
    // 设置作用域
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());

    // 获取名称
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    // 处理#通用注解
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    if (customizers != null) {
        for (BeanDefinitionCustomizer customizer : customizers) {
            customizer.customize(abd);
        }
    }

    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    // 设置作用域代理（scoped-proxy）
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 注册
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

```java
AnnotatedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition
```

GenericBeanDefinition是用于标准bean定义的一站式工具，一般用于定义用户可见（即自定义）bean。

AnnotatedBeanDefinition能够暴露定义类的元数据。

@Conditional用于有条件的允许注册bean。@Profile是通过@Conditional实现的。

自定义回调示例：

```java

```

\#通用注解：@Lazy、@Primary、@DependsOn、@Role、@Description

bean注册到定义集合中，集合的位置如下：

![](../img/bean定义集合位置.svg)

refresh

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

        // 激活上下文并设置属性源
        // Prepare this context for refreshing.
        prepareRefresh();

        // 以子类实现刷新工厂
        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 为工厂的使用做准备，包括设置类加载器、工厂配置、早期后置处理器、环境变量bean等。
        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // 以子类实现注册特殊的后置处理器
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // 处理后置处理器
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            // 实例化并调用工厂后置处理器
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // 实例化所有后置处理器并在后置处理器集合中加入
            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();

            // 初始化消息源
            // Initialize message source for this context.
            initMessageSource();

            // 初始化事件组播器
            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // 以子类实现初始化特殊bean
            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // 检查监听器并注册它们
            // Check for listener beans and register them.
            registerListeners();

            // 实例化剩余非懒加载单例bean
            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // 发布相应事件
            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
            }

            // 销毁所有创建的单例bean
            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // 修改上下文状态为未激活
            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
            contextRefresh.end();
        }
    }
}
```

prepareRefresh

```java
protected void prepareRefresh() {
    // 激活上下文
    // Switch to active.
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    if (logger.isDebugEnabled()) {
        if (logger.isTraceEnabled()) {
            logger.trace("Refreshing " + this);
        }
        else {
            logger.debug("Refreshing " + getDisplayName());
        }
    }

    // 初始化属性源（属性详见spring环境抽象）
    // Initialize any placeholder property sources in the context environment.
    initPropertySources();

    // 验证必需属性是否可解析
    // Validate that all properties marked as required are resolvable:
    // see ConfigurablePropertyResolver#setRequiredProperties
    getEnvironment().validateRequiredProperties();

    // Store pre-refresh ApplicationListeners...
    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
    }
    else {
        // Reset local application listeners to pre-refresh state.
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    // Allow for the collection of early ApplicationEvents,
    // to be published once the multicaster is available...
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

可重写initPropertySources来替换所有根属性源。

obtainFreshBeanFactory是子类实现的，可以通过重写自定义工厂刷新方法。

prepareBeanFactory

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 告诉工厂使用上下文中的类加载器
    // Tell the internal bean factory to use the context's class loader etc.
    beanFactory.setBeanClassLoader(getClassLoader());
    // 是否忽略初始化spel基础架构
    if (!shouldIgnoreSpel) {
        beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    }
    // 添加属性编辑器注册器（详见数据绑定）
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 添加用来配置工厂的后置处理器
    // Configure the bean factory with context callbacks.
    // ApplicationContextAwareProcessor
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    // 忽略自动装配（autowire）
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);

    // 设置依赖关系
    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    // registerResolvableDependency
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
    if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {
        beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, getApplicationStartup());
    }
}
```

ApplicationContextAwareProcessor是一个后置处理器实现类，它的作用是将ApplicationContext、Environment、StringValueResolver提供给实现了EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、and/or ApplicationContextAware的bean。（XxxAware的作用即是用来获取上下文中的Xxx对象）

registerResolvableDependency用于给依赖手动赋值。该方法适用于工厂/上下文中被标识为自动装配但却没有声明为bean的类。