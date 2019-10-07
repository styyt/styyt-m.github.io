---
title: mybatis-spring的mapper注入方式浅谈
date: 2019-09-01 11:32:36
tags: mybatis
categories: mybatis
---

众所周知，Spring注入bean时都是将可实例化对象注入。但是我们在写myabtis时却从来没有写过它的mapper实现类，仅仅是写了一个接口就可以在service层使用这个bean，这是为什么呢？我们可以先看下spring最终给我们注入到mapperBean的是什么类呢？

![mybatis的mapper](/intro/0019.png)

![普通的bean](/intro/0020.png)

果然，它注入的是mybatis代理类：MapperProxy，那么这个实例是什么时候注入进入bean工厂的呢？我们继续往下看：

找问题从源头找，首先mybatis是怎么知道哪些mapper并且知道去哪里扫描呢？没错就是通过以下三种方式：

```java
//第一种注解方式，如
@MapperScan("com.demo.*.mapper")

//第二种方式，传统的xml方式，如：
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="**.mapper" />
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
</bean>

```

我们先从传统的方式说起

MapperScannerConfigurer就是入口，它是怎么实现的呢？我们可以看他的内部结构：

```java
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {
......
}
```

![MapperScannerConfigurer](/intro/0021.png)

可以看到主要实现的全是支持spring关于自定义设置bean各项值的相关接口类。主要看这个BeanDefinitionRegistryPostProcessor接口，这个接口作用是让我们可以自己制定bean并放入bean工厂，原来就是因为它，我们跟踪项目启动调用栈看看：

简略步骤为：ContextLoaderListener初始化--->XmlWebApplicationContext刷新(初始化)servletContext：

刷新（初始化）beanfactory---->获取BeanDefinitionRegistryPostProcessor的实现bean---->调用接口实现类的方法：

org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory, List<BeanFactoryPostProcessor>)------>org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors(Collection<? extends BeanDefinitionRegistryPostProcessor>, BeanDefinitionRegistry)

```java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<String>();

		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new LinkedList<BeanDefinitionRegistryPostProcessor>();

			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					registryProcessors.add(registryProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			..............................
	}

---------------------------------------------------------------------
//调用接口实现类的postProcessBeanDefinitionRegistry
private static void invokeBeanDefinitionRegistryPostProcessors(
			Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {

		for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
			postProcessor.postProcessBeanDefinitionRegistry(registry);
		}
	}
```

到了正题：spring帮我们调用了自定义的bean注入方法：

org.mybatis.spring.mapper.MapperScannerConfigurer.postProcessBeanDefinitionRegistry(BeanDefinitionRegistry)

```java
 public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }
	//扫描我们配置的pkg内的接口的方法类ClassPathMapperScanner
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    //设置我们配置的各种属性：sqlSessionFactory，applicationContext等等
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
     //根据我们的配置设置各种过滤器
    scanner.registerFilters();
     //重点 开始扫描basePackage下的接口类  调用spring的扫描方法
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }

//org.mybatis.spring.mapper.ClassPathMapperScanner.registerFilters()
public void registerFilters() {
    //默认接受所有接口类
    boolean acceptAllInterfaces = true;

    // if specified, use the given annotation and / or marker interface
    //只接受制定的annotation接口类
    if (this.annotationClass != null) {
      addIncludeFilter(new AnnotationTypeFilter(this.annotationClass));
      acceptAllInterfaces = false;
    }

    // override AssignableTypeFilter to ignore matches on the actual marker interface
    //只接受指定类
    if (this.markerInterface != null) {
      addIncludeFilter(new AssignableTypeFilter(this.markerInterface) {
        @Override
          //重写判断方法
        protected boolean matchClassName(String className) {
          return false;
        }
      });
      acceptAllInterfaces = false;
    }

    if (acceptAllInterfaces) {
      // default include filter that accepts all classes
      addIncludeFilter(new TypeFilter() {
        public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
          return true;
        }
      });
    }

    // exclude package-info.java
    addExcludeFilter(new TypeFilter() {
      public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        String className = metadataReader.getClassMetadata().getClassName();
        return className.endsWith("package-info");
      }
    });
  }



//org.springframework.context.annotation.ClassPathBeanDefinitionScanner.scan(String...)
public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

    	
		doScan(basePackages);

		// Register annotation config processors, if necessary.
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}
```

org.mybatis.spring.mapper.ClassPathMapperScanner.doScan(String...)

```java
//mybatis重写了spring的扫描方法，就是这里将spring默认的jdk代理或者cg代理改写为自己的mapperproxy
@Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {、
      //先调用父类的扫描，配合前面的Filter获取包下面所有的BeanDefinitionHolder
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
      for (BeanDefinitionHolder holder : beanDefinitions) {
          //循环完善BeanDefinitionHolder的各类信息
        GenericBeanDefinition definition = (GenericBeanDefinition) holder.getBeanDefinition();

        if (logger.isDebugEnabled()) {
          logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() 
              + "' and '" + definition.getBeanClassName() + "' mapperInterface");
        }

        // the mapper interface is the original class of the bean
        // but, the actual class of the bean is MapperFactoryBean
         //补充信息：mapperInterface=getBeanClassName，也就是mapper的接口类名
        definition.getPropertyValues().add("mapperInterface", definition.getBeanClassName());
           //重点，英文解释很清楚了，MapperFactoryBean作为实际bean的提供方，
          //MapperFactoryBean实现了FactoryBean
        definition.setBeanClass(MapperFactoryBean作为实际bean的提供方，.class);

        definition.getPropertyValues().add("addToConfig", this.addToConfig);

        boolean explicitFactoryUsed = false;
        if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
          definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
          explicitFactoryUsed = true;
        } else if (this.sqlSessionFactory != null) {
          definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
          explicitFactoryUsed = true;
        }

        if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
          if (explicitFactoryUsed) {
            logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
          }
          definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
          explicitFactoryUsed = true;
        } else if (this.sqlSessionTemplate != null) {
          if (explicitFactoryUsed) {
            logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
          }
          definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
          explicitFactoryUsed = true;
        }

        if (!explicitFactoryUsed) {
          if (logger.isDebugEnabled()) {
            logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
          }
          definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
        }
      }
    }

    return beanDefinitions;
  }

```

FactoryBean最大作用就是：可以让我们自定义Bean的创建过程，创建过程就是调用接口的实现类：getObject，如下图：

![FactoryBean](/intro/0023.png)

看到这里就明了了。先把MapperFactoryBean源码贴出来

```java
//org.mybatis.spring.mapper.MapperFactoryBean<T>
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {

  private Class<T> mapperInterface;

  private boolean addToConfig = true;

  /**
   * Sets the mapper interface of the MyBatis mapper
   *
   * @param mapperInterface class of the interface
   */
  public void setMapperInterface(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  /**
   * If addToConfig is false the mapper will not be added to MyBatis. This means
   * it must have been included in mybatis-config.xml.
   * <p>
   * If it is true, the mapper will be added to MyBatis in the case it is not already
   * registered.
   * <p>
   * By default addToCofig is true.
   *
   * @param addToConfig
   */
  public void setAddToConfig(boolean addToConfig) {
    this.addToConfig = addToConfig;
  }

  /**
   * {@inheritDoc}
   */
  @Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        configuration.addMapper(this.mapperInterface);
      } catch (Throwable t) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", t);
        throw new IllegalArgumentException(t);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }

  /**
   * {@inheritDoc}
   */
  public T getObject() throws Exception {
     //返回了之前在doScan设置的接口类的代理类
    return getSqlSession().getMapper(this.mapperInterface);
  }

  /**
   * {@inheritDoc}
   */
  public Class<T> getObjectType() {
    return this.mapperInterface;
  }

  /**
   * {@inheritDoc}
   */
  public boolean isSingleton() {
    return true;
  }

}
```

getObject逻辑如下：org.mybatis.spring.SqlSessionTemplate.getMapper(Class<T>)--》org.apache.ibatis.session.Configuration.getMapper(Class<T>, SqlSession)-----》org.apache.ibatis.binding.MapperRegistry.getMapper(Class<T>, SqlSession)-----》org.apache.ibatis.binding.MapperProxyFactory.newInstance(SqlSession)----->	org.apache.ibatis.binding.MapperProxyFactory.newInstance(MapperProxy<T>)

```java
 public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }


-----------------------------
public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
-------------
    //org.apache.ibatis.binding.MapperProxyFactory.newInstance(MapperProxy<T>)
    //利用jdk代理返回了代理类，加强（实现类为）mapperProxy
protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
```

到此整个mapperbean的代理MapperProxy构建完成并返回给beanfactory进行管理和注入其他bean。后续由mapperProxy来负责mapper的与数据库的交互。

里面还有很多没有仔细研究到、写到的地方啊，太复杂了~~！！