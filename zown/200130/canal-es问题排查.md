---
title: 通过一次cannal监听问题浅读spring-dada源码及其在es中的应用一
date: 2020-01-12 16:54:13
tags: 
- es
- spring
categories: es
---

# 写在前面

​	系统需要与其他系统对接，并用到其他系统数据，但是其他系统又很难改造，所以使用canal来对其他系统数据库进行监控，并将变更的信息写入es，项目中使用spring-data-es的组件与es进行交互

# 问题

​	在测试环境突然报了一个很奇怪的错误

​	![java.lang.IllegalArgumentException](/intro/0052.png)

# 解决

​	经过分析异常堆栈，发现是因为用了spring-data的ElasticsearchRepository查询组件引起的，我们一步步来看，先看调用的地方

![调用的地方](/intro/0053.png)

​	神奇的地方就在这里，我们不需要写任何逻辑代码，spring就帮我们实现了es的查询，这就归功于spring-data组件的功能了，缺点就是排查问题难度加大

​	现在可以好好跟踪代码看一下了，被代理的bean是spring-data-es为该接口动态注入的bean即SimpleElasticsearchRepository的动态代理类，即下图中spring-es帮我们实现的几个通用类中的一个代理对象

![SimpleElasticsearchRepository1](/intro/0055.png)

，关于这个bean如何被注册到spring内的呢，具体过程如下：

- **ElasticsearchRepositoriesAutoConfiguration**

  ![ElasticsearchRepositoriesAutoConfiguration](/intro/0056.png)

  自动化配置类会导入ElasticsearchRepositoriesRegistrar这个ImportBeanDefinitionRegistrar。

- ElasticsearchRepositoriesRegistrar继承自AbstractRepositoryConfigurationSourceSupport，是个ImportBeanDefinitionRegistrar接口的实现类，会被Spring容器调用registerBeanDefinitions进行自定义bean的注册。

  ![AbstractRepositoryConfigurationSourceSupport](/intro/0057.png)

- ElasticsearchRepositoriesRegistrar委托给RepositoryConfigurationDelegate完成bean的解析。

  具体解析过程如下：

  ![org.springframework.data.repository.config.RepositoryConfigurationDelegate#registerRepositoriesIn](/intro/0058.png)

  ```java
  /*1、找出模块中的org.springframework.data.repository.Repository接口的实现类或者org.springframework.data.repository.RepositoryDefinition注解的修饰类，并会过滤掉org.springframework.data.repository.NoRepositoryBean注解的修饰类。找出后封装到RepositoryConfiguration中
  2、遍历这些RepositoryConfiguration，然后构造成BeanDefinition并注册到Spring容器中。需要注意的是这些RepositoryConfiguration会以beanClass为ElasticsearchRepositoryFactoryBean这个类的方式被注册，并把对应的Repository接口当做构造参数传递给ElasticsearchRepositoryFactoryBean，还会设置相应的属性比如elasticsearchOperations、evaluationContextProvider、namedQueries、repositoryBaseClass、lazyInitqueryLookupStrategyKey
  3、ElasticsearchRepositoryFactoryBean被实例化的时候设置对应的构造参数和属性。设置完毕以后调用afterPropertiesSet方法(实现了InitializingBean接口)。在afterPropertiesSet方法内部会去创建RepositoryFactorySupport类，并进行一些初始化，比如namedQueries、repositoryBaseClass等。然后通过这个RepositoryFactorySupport的getRepository方法基于Repository接口创建出代理类，并使用AOP添加了几个MethodInterceptor*/
      
      
      // 遍历基于第1步条件得到的RepositoryConfiguration集合
  for (RepositoryConfiguration<? extends RepositoryConfigurationSource> configuration : extension
      .getRepositoryConfigurations(configurationSource, resourceLoader, inMultiStoreMode)) {
    // 构造出BeanDefinitionBuilder
    BeanDefinitionBuilder definitionBuilder = builder.build(configuration);
  
    extension.postProcess(definitionBuilder, configurationSource);
  
    if (isXml) {
      // 设置elasticsearchOperations属性
      extension.postProcess(definitionBuilder, (XmlRepositoryConfigurationSource) configurationSource);
    } else {
      // 设置elasticsearchOperations属性
      extension.postProcess(definitionBuilder, (AnnotationRepositoryConfigurationSource) configurationSource);
    }
  
    // 使用命名策略生成bean的名字
    AbstractBeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();
    String beanName = beanNameGenerator.generateBeanName(beanDefinition, registry);
  
    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug(REPOSITORY_REGISTRATION, extension.getModuleName(), beanName,
          configuration.getRepositoryInterface(), extension.getRepositoryFactoryClassName());
    }
  
    beanDefinition.setAttribute(FACTORY_BEAN_OBJECT_TYPE, configuration.getRepositoryInterface());
    // 注册到Spring容器中
    registry.registerBeanDefinition(beanName, beanDefinition);
    definitions.add(new BeanComponentDefinition(beanDefinition, beanName));
  }
  
  // build方法
  public BeanDefinitionBuilder build(RepositoryConfiguration<?> configuration) {
  
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null!");
    Assert.notNull(resourceLoader, "ResourceLoader must not be null!");
    // 得到factoryBeanName，这里会使用extension.getRepositoryFactoryClassName()去获得
    // extension.getRepositoryFactoryClassName()返回的正是ElasticsearchRepositoryFactoryBean
    String factoryBeanName = configuration.getRepositoryFactoryBeanName();
    factoryBeanName = StringUtils.hasText(factoryBeanName) ? factoryBeanName
        : extension.getRepositoryFactoryClassName();
    // 基于factoryBeanName构造BeanDefinitionBuilder
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(factoryBeanName);
  
    builder.getRawBeanDefinition().setSource(configuration.getSource());
    // 设置ElasticsearchRepositoryFactoryBean的构造参数，这里是对应的Repository接口
    // 设置一些的属性值
    builder.addConstructorArgValue(configuration.getRepositoryInterface());
    builder.addPropertyValue("queryLookupStrategyKey", configuration.getQueryLookupStrategyKey());
    builder.addPropertyValue("lazyInit", configuration.isLazyInit());
    builder.addPropertyValue("repositoryBaseClass", configuration.getRepositoryBaseClassName());
  
    NamedQueriesBeanDefinitionBuilder definitionBuilder = new NamedQueriesBeanDefinitionBuilder(
        extension.getDefaultNamedQueryLocation());
  
    if (StringUtils.hasText(configuration.getNamedQueriesLocation())) {
      definitionBuilder.setLocations(configuration.getNamedQueriesLocation());
    }
  
    builder.addPropertyValue("namedQueries", definitionBuilder.build(configuration.getSource()));
    // 查找是否有对应Repository接口的自定义实现类
    String customImplementationBeanName = registerCustomImplementation(configuration);
    // 存在自定义实现类的话，设置到属性中
    if (customImplementationBeanName != null) {
      builder.addPropertyReference("customImplementation", customImplementationBeanName);
      builder.addDependsOn(customImplementationBeanName);
    }
  
    RootBeanDefinition evaluationContextProviderDefinition = new RootBeanDefinition(
        ExtensionAwareEvaluationContextProvider.class);
    evaluationContextProviderDefinition.setSource(configuration.getSource());
    // 设置一些的属性值
    builder.addPropertyValue("evaluationContextProvider", evaluationContextProviderDefinition);
  
    return builder;
  }
  
  // RepositoryFactorySupport的getRepository方法，获得Repository接口的代理类
  public <T> T getRepository(Class<T> repositoryInterface, Object customImplementation) {
  
    // 获取Repository的元数据
    RepositoryMetadata metadata = getRepositoryMetadata(repositoryInterface);
    // 获取Repository的自定义实现类
    Class<?> customImplementationClass = null == customImplementation ? null : customImplementation.getClass();
    // 根据元数据和自定义实现类得到Repository的RepositoryInformation信息类
    // 获取信息类的时候如果发现repositoryBaseClass是空的话会根据meta中的信息去自动匹配
    // 具体匹配过程在下面的getRepositoryBaseClass方法中说明
    RepositoryInformation information = getRepositoryInformation(metadata, customImplementationClass);
    // 验证
    validate(information, customImplementation);
    // 得到最终的目标类实例，会通过repositoryBaseClass去查找
    Object target = getTargetRepository(information);
  
    // 创建代理工厂
    ProxyFactory result = new ProxyFactory();
    result.setTarget(target);
    result.setInterfaces(new Class[] { repositoryInterface, Repository.class });
    // 进行aop相关的设置
      //重要
    result.addAdvice(SurroundingTransactionDetectorMethodInterceptor.INSTANCE);
    result.addAdvisor(ExposeInvocationInterceptor.ADVISOR);
  
    if (TRANSACTION_PROXY_TYPE != null) {
      result.addInterface(TRANSACTION_PROXY_TYPE);
    }
    // 使用RepositoryProxyPostProcessor处理
    for (RepositoryProxyPostProcessor processor : postProcessors) {
      processor.postProcess(result, information);
    }
  
    if (IS_JAVA_8) {
      // 如果是JDK8的话，添加DefaultMethodInvokingMethodInterceptor
      result.addAdvice(new DefaultMethodInvokingMethodInterceptor());
    }
  
    // 添加QueryExecutorMethodInterceptor
    result.addAdvice(new QueryExecutorMethodInterceptor(information, customImplementation, target));
    // 使用代理工厂创建出代理类，这里是使用jdk内置的代理模式
    return (T) result.getProxy(classLoader);
  }
  
  // 目标类的获取
  protected Class<?> getRepositoryBaseClass(RepositoryMetadata metadata) {
    // 如果Repository接口属于QueryDsl，抛出异常。目前还不支持
    if (isQueryDslRepository(metadata.getRepositoryInterface())) {
      throw new IllegalArgumentException("QueryDsl Support has not been implemented yet.");
    }
    // 如果主键是数值类型的话，repositoryBaseClass为NumberKeyedRepository
    if (Integer.class.isAssignableFrom(metadata.getIdType())
        || Long.class.isAssignableFrom(metadata.getIdType())
        || Double.class.isAssignableFrom(metadata.getIdType())) {
      return NumberKeyedRepository.class;
    } else if (metadata.getIdType() == String.class) {
      // 如果主键是String类型的话，repositoryBaseClass为SimpleElasticsearchRepository
      return SimpleElasticsearchRepository.class;
    } else if (metadata.getIdType() == UUID.class) {
      // 如果主键是UUID类型的话，repositoryBaseClass为UUIDElasticsearchRepository
      return UUIDElasticsearchRepository.class;
    } else {
      // 否则报错
      throw new IllegalArgumentException("Unsupported ID type " + metadata.getIdType());
    }
  }
  
  
  //重点在这个代码上  QueryExecutorMethodInterceptor
  
  //result.addAdvice(new QueryExecutorMethodInterceptor(information, customImplementation, target));
  ```

  上面代码向bean添加了QueryExecutorMethodInterceptor拦截，我们可以看到它的invoke方法，就一目了然了

  ![QueryExecutorMethodInterceptor](/intro/0059.png)

  ​																										未完待续... ...