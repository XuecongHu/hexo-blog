---
title: 读书笔记《Spring源码深度解析》| Spring容器的扩展ApplicationContext（上）
date: 2017-12-16 09:40:56
tags: Spring Java
---

尽管在Spring中BeanFactory接口及其实现类已经实现了IoC，但Spring还是提供了另外一个容器的实现——ApplicationContext，两者都是用于加载Bean的。但实际上，人们在应用开发实践中用得最多的是ApplicationContext，而不是BeanFactory。这是因为ApplicationContext在BeanFactory的基础上扩展了其他的功能，以便于在开发企业级应用时能够基于Ioc容器做更多自定义的动作，具有强大的可扩展性和定制性，是BeanFactory功能的超集。

究竟ApplicationContext相比BeanFactory扩展了什么？具体做了哪些事情？本文从源码分析的角度给出ApplicationContext的全景，更好地理解ApplicationContext。
（本文是阅读《Spring源码深度解析》的第6章——容器的功能扩展后的学习心得）
<!--more-->

# ApplicationContext Overview
ApplicationContext是只属于单个的容器管理公共接口，封装了IoC容器的实现，客户端只需通过ApplicationContext的接口就能Spring容器打交道，完成获取Bean、添加监听器、发布事件等等的事情。我们注意到ApplicationContext有一些在BeanFactory中没听过的名词，例如监听、事件、消息源等，而这些正是ApplicationContext提供的扩展功能。根据Spring API的官方文档，具体就是包括以下：
>- The ability to load file resources in a generic fashion. Inherited from the ResourceLoader nterface.
>- The ability to publish events to registered listeners. Inherited from the ApplicationEventPublisher interface.
>- The ability to resolve messages, supporting internationalization. Inherited from the MessageSource interface.
>- Inheritance from a parent context. Definitions in a descendant context will always take priority.

单个应用并不是在一个应用内只能启动一个AppliacationContext。实际上一个应用程序可以启动多个ApplicationContext，但不同ApplicationContext中的Bean是不同，而同一个ApplicationContext中的Bean总是相同。

```java
@Test
public void test() {
  AbstractApplicationContext ctx1 = new ClassPathXmlApplicationContext("aware.xml");
  People people1 = (People) ctx1.getBean("p");
  People people2 = (People) ctx1.getBean("p");
  System.out.println(people1 == people2);

  AbstractApplicationContext ctx2 = new ClassPathXmlApplicationContext("aware.xml");
  People people3 = (People) ctx2.getBean("p");
  System.out.println(people1 == people3);
}
```
运行结果：
```
true
false
```
![全家桶](Spring容器的扩展——ApplicationContext/ApplicationContext全家桶.jpg)
从ApplicationContext的“全家福“图可以看出，ApplicationContext继承体系中有各种各样的叶子节点，例如ClassPathXmlApplicationContext、GenericGroovyApplicationContext、AnnotationConfigApplicationContext等，我目前认为代表的是不同形式的元数据(metadata）配置方式。最常用的是例如ClassPathXmlApplicationContext，bean的定义和依赖注入等等都在xml文件中定义。
顺便一提的是，ApplicationContext是有父子容器层级关系：
```
@Nullable
ApplicationContext getParent()
Return the parent context, or null if there is no parent and this is the root of the context hierarchy.
```
这是为了方便有统一的ApplicationContext。例如一个web应用可以有一个公共的父ApplicationContext，而应用内不同的Servlet则有各自独立的子ApplicationContext。

# 以ClassPathXmlApplicationContext为例
我们从最基本的加载配置文件入手。ApplicationContext加载xml文件的常用实现类是ClassPathXmlApplicationContext。和BeanFactory相比，除了不需要额外创建Resource类来作为构造器的入参之外，其他都大同小异。
```java
ApplicationContext beanFactory = new ClassPathXmlApplicationContext("applicationContext.xml");
TestBean testBean = (TestBean) beanFactory.getBean("testBean");
```
进一步查看ClassPathXmlApplicationContext的构造器源码：
```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}

public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent); // 设置父context
		setConfigLocations(configLocations); // 设置将要解析的xml文件路径
		if (refresh) {
			refresh();
		}
	}
```
从其构造器的源码得知，ClassPathXmlApplicationContext在创建实例的时候，主要做了三件事情：
1. 设置父ApplicationContext
2. 设置将要解析的xml文件路径
3. 执行refresh方法，所有的容器实现相关功能都在该方法完成，是最主要的逻辑部分。

启动ApplicationContext时的关键代码都在refresh方法中，我们只需看refresh方法。该方法并不在ClassPathXmlApplicationContext里实现，是继承自AbstractApplicationContext。AbstractApplicationContext是对ApplicationContext接口的基本实现，后面分析时的源码，也主要在此类。
```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			// 初始化前的准备，例如读取系统属性或环境变量并进行验证。
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			// 初始化持有的BeanFactory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			// 对BeanFactory进行功能填充，例如加载和解析@Qualifier和@Autowired注解的类和对象。
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 可自定义在BeanFactory完成后的行为动作
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				// 激活各种BeanFactory处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				// 注册拦截Bean创建的bean处理器
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				// 为上下文初始化国际化的源
				initMessageSource();

				// Initialize event multicaster for this context.
				// 初始化应用消息广播源
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				// 留给子类来初始化其他的bean
				onRefresh();

				// Check for listener beans and register them.
				// 在所有注册的bean中查找listener bean，注册到消息广播器中
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				// 初始化剩下的非惰性单例
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				// 通知生命周期处理器刷新过程，同时发出ContextRefreshEvent的事件
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```
就算不看具体的细节，从refresh方法也能看出ApplicationContext的底层大概原理。ApplicationContext持有一个ConfigurableListBeanFactory类型的对象beanFactory，利用该beanFactory来实现bean的解析和加载，在beanFactory对象的基础上，做了额外的功能扩展。根据源码，refresh方法中的执行过程步骤大致如下：
1. 初始化前的环境准备，例如读取系统属性或环境变量并进行验证。
2. 初始化持有的BeanFactory
3. 对BeanFactory进行功能填充，例如加载和解析@Qualifier和@Autowired注解的类和对象。
4. 可自定义在BeanFactory完成后的行为动作
5. 激活各种BeanFactory处理器
6. 注册拦截Bean创建的bean处理器
7. 为上下文初始化国际化的源
8. 初始化应用消息广播源
9. 留给子类来初始化其他的bean
10. 在所有注册的bean中查找listener bean，注册到消息广播器中
11. 初始化剩下的非惰性单例
12. 通知生命周期处理器刷新过程，同时发出ContextRefreshEvent的事件

后文将详细讲解每一个步骤。但由于步骤内容都很多，将本章读后感分为多个系列。

本文介绍的是refresh方法中的第1——3步，即初始化前的环境准备、初始化持有的BeanFactory和BeanFactory的功能填充。



# 环境准备
```java
/**
* Prepare this context for refreshing, setting its startup date and
* active flag as well as performing any initialization of property sources.
*/
protected void prepareRefresh() {
  this.startupDate = System.currentTimeMillis();
  this.closed.set(false);
  this.active.set(true);

  if (logger.isInfoEnabled()) {
  logger.info("Refreshing " + this);
  }

  // Initialize any placeholder property sources in the context environment
  initPropertySources();

  // Validate that all properties marked as required are resolvable
  // see ConfigurablePropertyResolver#setRequiredProperties
  getEnvironment().validateRequiredProperties();

  // Allow for the collection of early ApplicationEvents,
  // to be published once the multicaster is available...
  this.earlyApplicationEvents = new LinkedHashSet<>();
}
```
环境准备方法中执行的是对系统属性以及环境变量的初始化和验证。initPropertySources方法就是对属性进行初始化，然而我们发现方法是空的，它留给了其子类来实现，体现了Spring的开放扩展能力。
```java
/**
	Replace any stub property sources with actual instances.
**/
protected void initPropertySources() {
	// For subclasses: do nothing by default.
}
```
验证属性则是调用了ConfigurableEnvironment接口的validateRequiredProperties方法。AbstractApplicationContext类的getEnviroment方法返回的是StandardEnvironment类实例。StandardEnvironment类间接实现了ConfigurableEnvironment接口，通过查看相关源码，最终可知这里的validateRequiredProperties方法实际上是对所有必要的属性进行验证，假如有必要的属性没有解析到值，则不通过验证，抛出异常。
```java
@Override
	public void validateRequiredProperties() {
		MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
		for (String key : this.requiredProperties) {
			if (this.getProperty(key) == null) {
				ex.addMissingRequiredProperty(key);
			}
		}
		if (!ex.getMissingRequiredProperties().isEmpty()) {
			throw ex;
		}
	}
```
因此，当我们要规定某个工程必须用到某个设置，该设置是从系统环境变量读取的，假如用户没有设置则项目无法启动。此时我们可以继承ClassPathXmlApplicationContext，实现自定义的initPropertySources方法，实现该需求。
```java
public class MyClassPathXmlApplicationContext
    extends ClassPathXmlApplicationContext {
    public MyClassPathXmlApplicationContext(String configLocation) throws BeansException {
        super(configLocation);
    }

    @Override
    protected void initPropertySources() {
        getEnvironment().setRequiredProperties("VAR");
    }
}
```
假如系统变量没有"VAR"这个变量，在启动ApplicationContext的时候就会抛出异常，中止启动。

# 加载BeanFactory
在准备完必要的属性变量之后，AbstractApplicationContext就通过obtainFreshBeanFactory方法获取BeanFactory。
```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```
在ClassPathXmlApplicationContext的继承体系中，refreshBeanFactory方法和getBeanFactory方法都在AbstractRefreshableApplicationContext类中实现：
```java
protected final void  refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory); // 定制BeanFactory
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
	
public final ConfigurableListableBeanFactory getBeanFactory() {
		synchronized (this.beanFactoryMonitor) {
			if (this.beanFactory == null) {
				throw new IllegalStateException("BeanFactory not initialized or already closed - " +
						"call 'refresh' before accessing beans via the ApplicationContext");
			}
			return this.beanFactory;
		}
	}	
```
在refreshBeanFactory方法按顺序大概完成以下几件事情：
1. 如果以前曾经创建BeanFactory，则完全销毁；
2. 创建DefaultListableBeanFactory实例
3. 定制BeanFactory
4. 加载BeanDefinitions

## 定制BeanFactory
在定制BeanFactory化时就是对BeanFactory的扩展。
```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
		if (this.allowBeanDefinitionOverriding != null) {
	beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		if (this.allowCircularReferences != null) {
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}
```
ApplicationContext可以自定义设置是否允许BeanDefinition被覆盖以及是否允许循环依赖。
在Spring3.0中本来该方法还有提供了注解@Qualifier和@Autowire的支持，但当前已将这部分移到CustomAutowireConfigurer类的postProcessBeanFactory方法实现，而CustomAutowireConfigurer类实现了BeanFactoryPostProcessor接口，将在激活BeanFactoryPostProcessor的步骤中执行。
```java
public class CustomAutowireConfigurer implements BeanFactoryPostProcessor ... {
  	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
  		...
  		DefaultListableBeanFactory dlbf = (DefaultListableBeanFactory) beanFactory;
        if (!(dlbf.getAutowireCandidateResolver() instanceof QualifierAnnotationAutowireCandidateResolver)) {
        dlbf.setAutowireCandidateResolver(new QualifierAnnotationAutowireCandidateResolver());
        }
        ...
  	}
}
```
那要如何定制BeanFactory呢？假设要不允许BeanDefinition被覆盖（Spring默认允许），同样Spring的强大扩展功能允许我们自行实现customizeBeanFactory方法，在实现的方法中设置。
```java
public class DisAllowBeanDefinitionClassPathXmlApplicationContext
    extends ClassPathXmlApplicationContext {
   ...
    @Override
    protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
        super.setAllowBeanDefinitionOverriding(false);
        super.customizeBeanFactory(beanFactory);
    }
}
```
使用新继承的类来创建ApplicationContext，在解析Bean的时候就不允许有ID相同的bean配置。

## 加载BeanDefinition
定制完BeanFactory之后，就要加载BeanDefinition了。加载BeanDefintion的逻辑是在AbstractRefreshableApplicationContext的间接子类AbstractXmlApplicationContext中实现的。
```java
@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}
	
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}	
```
其实这块的实现和XmlBeanFactory的实现非常相似，只是多了一些验证和资源解析。底层还是使用XmlBeanDefinitionReader来解析xml文件。经过这个方法，AbstractXmlApplicationContext中的BeanFactory就已经包含了所有解析好的配置了。

# 功能扩展
回到AbstractApplicationContext的refresh方法，经过prepareRefresh和obtainFreshBeanFactory方法，ApplicationContext已经完成了对xml文件的解析。接下来的prepareBeanFactory方法就是对BeanFactory的扩展。
```java
/**
	 * Configure the factory's standard context characteristics,
	 * such as the context's ClassLoader and post-processors.
	 * @param beanFactory the BeanFactory to configure
	 */
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		// 设置SPEL的表达式处理器
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		// 增加默认的PropertyEditor
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		
		// 忽略自动装配的接口
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		// 设置自动装配的特殊规则
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		// 增加对AsplectJ的支持
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		// 设置默认系统环境beans
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```
在该方法中，主要对BeanFactory做了以下几个方面的扩展：
1. 对SpEL的支持
2. 属性编辑器的支持
3. 添加内置类，例如EnvironmentAware
4. 添加依赖功能可忽略的接口
5. 注册一些固定依赖的属性
6. 增加AspectJ的支持
7. 注册相关环境变量及属性注册

## SpEL语言支持
SpEL是Spring表达式语言，具体解释引用网友的说法[表达式语言SpEL](http://blog.csdn.net/zhoudaxia/article/details/38174169)：
>能够在运行时构建复杂表达式、存取对象图属性、调用对象方法等等，并且能与Spring功能完美整合，如能用来配置Bean定义。表达式语言给静态Java语言增加了动态功能。

可以在bean定义中使用SpEL，例如：
```
<bean id="tb" class="xxx"/>
<bean id="test" class="yyy">
	<property name="a" ref="#{tb}"/>
<bean/>	
```
这里不详细介绍SpEL。prepareBeanFactory方法中通过调用持有BeanFactory对象的setBeanExpressionResolver方法来设置SpEL的支持。
```java
beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
```
设置后，ApplicationContext就有了对SpEL进行解析的能力。
而ApplicationContext又会在加载Bean、给bean初始化时，调用AbstractAutowireCapableBeanFactory类的applyPropertyValues方法来完成Bean的属性填充。在applyPropertyValues方法中，会创建一个BeanDefinitionValueResolver的实例，然后调用该实例的resolveValueIfNecessary方法。

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
	...
 	BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter); 
  	...
    String propertyName = pv.getName();
  	Object originalValue = pv.getValue();
  	Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);  
    ...
}
```

最终会调用AbstractBeanFactory的evaluateBeanDefinitionString来解析属性值：

```java
protected Object evaluateBeanDefinitionString(@Nullable String value, @Nullable BeanDefinition beanDefinition) {
		if (this.beanExpressionResolver == null) {
			return value;
		}
		Scope scope = null;
		if (beanDefinition != null) {
			String scopeName = beanDefinition.getScope();
			if (scopeName != null) {
				scope = getRegisteredScope(scopeName);
			}
		}
		return this.beanExpressionResolver.evaluate(value, new BeanExpressionContext(this, scope));
	}
```

上面代码中的beanExpressionResolver正是StandardBeanExpressionResolver的实例。StandardBeanExpressionResolver类持有一个SpelExpressionParser类的对象，解析属性值都是基于该对象。
```java
public StandardBeanExpressionResolver(@Nullable ClassLoader beanClassLoader) {
		this.expressionParser = new SpelExpressionParser(new SpelParserConfiguration(null, beanClassLoader));
	}
```
通过查看evaluateBeanDefinitionString方法的调用者，发现SpEL表达式的解析在Spring容器主要用在依赖注入bean和bean属性填充的时候。

## 属性编辑器的支持
我们首先来看属性编辑器的应用。
在bean定义的时候，我们可能会发现需要对bean定义中的属性值进行转换的需求。例如People的birthDate属性，是LocalDate对象。
```java
public class People {
    private String name;
    private LocalDate birthDate;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public LocalDate getBirthDate() {
        return birthDate;
    }

    public void setBirthDate(LocalDate birthDate) {
        this.birthDate = birthDate;
    }

    @Override
    public String toString() {
        return "People{"
                + "name='" + name + '\''
                + ", birthDate=" + birthDate
                + '}';
    }
}
```
然而我们在定义bean的时候，希望可以直接写一个日期，让容器启动时可以自动转换，而无须再定义一个LocalDate的bean。
```xml
<bean id="p" class="com.huxuecong.springpractice.bean.People">
        <property name="name" value="spring"/>
        <property name="birthDate">
            <value>2017-12-17</value>
        </property>
</bean>
```
解决这个问题都需要利用CustomEditorConfigurer类，但可以使用该类的customEditor对象或propertyEditorRegistrars对象两种方式达成相同的作用。
但不管哪种方式，都需要首先编写自定义的属性编辑器：
```java
public class DatePropertyEditor extends PropertyEditorSupport {

    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        System.out.println("text: " + text);
        DateTimeFormatter df = DateTimeFormatter.ofPattern("yyyy-MM-dd");
        this.setValue(LocalDate.parse(text, df));
    }
}
```
第一种方式是直接将自定义的属性编辑器注册给CustomEditorConfigurer类的customEditors对象，该对象类型是Map<Class<?>, Class<? extends PropertyEditor>>。
```xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="customEditors">
        <map>
            <entry key="java.time.LocalDate" value="com.huxuecong.springpractice.converter.DatePropertyEditor">
            </entry>
        </map>
    </property>
</bean>
```
配置后，当ApplicationContext注入Bean属性的时候，发现其属性类型为java.time.LocalDate，就会自动调用自定义编辑器的setAsText方法，进行值的转换。

第二种方式首先要实现PropertyEditorRegistrar接口。
```java
public class DatePropertyEditorRegistrar implements PropertyEditorRegistrar {
    public void registerCustomEditors(PropertyEditorRegistry propertyEditorRegistry) {
        propertyEditorRegistry.registerCustomEditor(LocalDate.class, new DatePropertyEditor());
    }
}
```
然后在xml配置文件中，将自定义的PropertyEditorRegistrar实现类注册给CustomEditorConfigurer类的propertyEditorRegistrars对象。
```xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="propertyEditorRegistrars">
        <list>
            <bean class="com.huxuecong.springpractice.converter.DatePropertyEditorRegistrar"/>
        </list>
    </property>
</bean>
```
最终还是会调用自定义的属性编辑器来实现值的转换。



回到AbstractApplicationContext的prepareBeanFactory方法。该方法中设置属性注册编辑器支持的是语句是：
```java
beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
```
而这里调用的是AbstractBeanFactory的addPropertyEditorRegistrar方法：
```java
public void addPropertyEditorRegistrar(PropertyEditorRegistrar registrar) {
	Assert.notNull(registrar, "PropertyEditorRegistrar must not be null");
	this.propertyEditorRegistrars.add(registrar);
}
```
而ResourceEditorRegistrar类实现了PropertyEditorRegistrar接口，关键逻辑在其registerCustomEditors方法中。

```java
public void registerCustomEditors(PropertyEditorRegistry registry) {
	ResourceEditor baseEditor = new ResourceEditor(this.resourceLoader, this.propertyResolver);
	doRegisterEditor(registry, Resource.class, baseEditor);
	doRegisterEditor(registry, ContextResource.class, baseEditor);
	doRegisterEditor(registry, InputStream.class, new InputStreamEditor(baseEditor));
	doRegisterEditor(registry, InputSource.class, new InputSourceEditor(baseEditor));
	doRegisterEditor(registry, File.class, new FileEditor(baseEditor));
	doRegisterEditor(registry, Path.class, new PathEditor(baseEditor));
	doRegisterEditor(registry, Reader.class, new ReaderEditor(baseEditor));
	doRegisterEditor(registry, URL.class, new URLEditor(baseEditor));

	ClassLoader classLoader = this.resourceLoader.getClassLoader();
	doRegisterEditor(registry, URI.class, new URIEditor(classLoader));
	doRegisterEditor(registry, Class.class, new ClassEditor(classLoader));
	doRegisterEditor(registry, Class[].class, new ClassArrayEditor(classLoader));

	if (this.resourceLoader instanceof ResourcePatternResolver) {
		doRegisterEditor(registry, Resource[].class,
				new ResourceArrayPropertyEditor((ResourcePatternResolver) this.resourceLoader, this.propertyResolver));
	}
}

private void doRegisterEditor(PropertyEditorRegistry registry, Class<?> requiredType, PropertyEditor editor) {
	if (registry instanceof PropertyEditorRegistrySupport) {
		((PropertyEditorRegistrySupport) registry).overrideDefaultEditor(requiredType, editor);
	}
	else {
		registry.registerCustomEditor(requiredType, editor);
	}
}
```
从该方法的源码看出，ResourceEditorRegistrar类对作为入参的PropertyEditorRegistry类对象注册了编辑器，和我们上文中的例子DatePropertyEditorRegistrar类似。不同的是，ResourceEditorRegistrar类注册了很多编辑器，如File、URL、Class的编辑器。这样，当给bean属性填充的时候，ApplicationContext遇到了File、URL、Class等的属性时，会自动将String类转为对应的File类、URL类或者Class类。说明ApplicationContext已经实现了很多默认的属性编辑器。



但是何时调用registerCustomEditors方法呢？不在AbstractApplicationContext的prepareBeanFactory方法中，而是加载bean转换值的时候——initBeanWrapper方法中。initBeanWrapper方法是bean初始化时，将BeanDefinition转换为BeanWrapper进行属性填充时调用的方法。

```java
protected void initBeanWrapper(BeanWrapper bw) {
	bw.setConversionService(getConversionService());
	registerCustomEditors(bw);
}

protected void registerCustomEditors(PropertyEditorRegistry registry) {
	PropertyEditorRegistrySupport registrySupport =
			(registry instanceof PropertyEditorRegistrySupport ? (PropertyEditorRegistrySupport) registry : null);
	if (registrySupport != null) {
		registrySupport.useConfigValueEditors();
	}
	// this.propertyEditorRegistrars在AbstractApplicationContext类的prepareBeanFactory方法中添加了ResourceEditorRegistrar对象
	if (!this.propertyEditorRegistrars.isEmpty()) {
		for (PropertyEditorRegistrar registrar : this.propertyEditorRegistrars) {
			try {
				registrar.registerCustomEditors(registry);
			}
			catch (BeanCreationException ex) {
				Throwable rootCause = ex.getMostSpecificCause();
				if (rootCause instanceof BeanCurrentlyInCreationException) {
					BeanCreationException bce = (BeanCreationException) rootCause;
					String bceBeanName = bce.getBeanName();
					if (bceBeanName != null && isCurrentlyInCreation(bceBeanName)) {
						if (logger.isDebugEnabled()) {
							logger.debug("PropertyEditorRegistrar [" + registrar.getClass().getName() +
									"] failed because it tried to obtain currently created bean '" +
									ex.getBeanName() + "': " + ex.getMessage());
						}
						onSuppressedException(ex);
						continue;
					}
				}
				throw ex;
			}
		}
	}
	if (!this.customEditors.isEmpty()) {
		this.customEditors.forEach((requiredType, editorClass) ->
				registry.registerCustomEditor(requiredType, BeanUtils.instantiateClass(editorClass)));
	}
}
```
initBeanWrapper方法的入参是BeanWrapper接口，BeanWrapper接口间接继承了PropertyEditorRegistry接口。在registerCustomEditors方法中，所有注册到BeanFactory的PropertyEditorRegistrar接口实现类和PropertyEditor实现类都会注册到BeanWrapper类（实际上上是PropertyEditorRegistry接口）中。

而后，在转换值的时候，则是由BeanWrapper接口的默认实现类BeanWrapperImpl来负责，在BeanWrapperImpl类中会调用其covertForProperty方法，最终会调用到PropertyEditorRegistry接口的findCustomEditor方法，并根据属性编辑器进行值转换。

在上文例子中的日期转换器都是注入依赖到CustomEditorConfigurer类，而该类实现了BeanFactoryPostProcessor接口。

```java
public class CustomEditorConfigurer implements BeanFactoryPostProcessor, Ordered {
  ...
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    if (this.propertyEditorRegistrars != null) {
    	for (PropertyEditorRegistrar propertyEditorRegistrar : this.propertyEditorRegistrars) {
    	beanFactory.addPropertyEditorRegistrar(propertyEditorRegistrar);
    	}
    }
    if (this.customEditors != null) {
    	this.customEditors.forEach(beanFactory::registerCustomEditor);
    }
}
```

该方法将用户自己定义的PropertyEditorRegistrar接口实现类和PropertyEditor接口实现类注册到BeanFactory中，使得在加载Bean的时候在registerCustomEditors方法也会调用到。



## 激活ApplicationContextAwareProcessor
回到prepareBeanFactory方法，接下来就是在BeanFactory中添加ApplicationContextAwareProcessor。ApplicationContextAwareProcessor实现了BeanPostProcessor接口，在实例化Bean对象时调用init-method前后，会调用该接口的相关方法。我们首先看ApplicationContextAwareProcessor的重要源码。
```java
@Override
	@Nullable
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
```
其实最终要就是invokeAwareInterfaces方法，该方法就是为各种Aware接口设置其所需的对象。
xxAware接口的作用由Spring容器统一通知这些Aware实现类对应的对象。例如EnvironmentAware接口即由容器通知其实现类目前容器中的Environment是哪个对象。ApplicationContextAware接口则是由容器通知其实现类目前容器中的ApplicationContext是哪个对象，BeanNameAware接口则是由容器通知其实现类在容器中的name是什么。

## 忽略依赖和设置依赖
接下来Spring容器使得ApplicationContextAware等Aware接口在自动装配的时候被忽略。
ignoreDependencyInterface方法是ConfigurableListableBeanFactory类中的方法，通过调用该方法将使得作为入参的类或接口无法被自动装配。
我理解之所以这么做是让Spring容器在注入xxAware接口时，在ApplicationContextAwareProcessor中统一调用其setXXX方法。由于Spring在初始化bean的时候会递归地初始化其属性，而假如该属性是xxAware接口，那么也会被初始化。Spring希望在ApplicationContextAwareProcessor类中统一为xxAware的实现类设置依赖。
```java
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

由于上面的xxAware接口被忽略依赖，因此容器需要特定地为其指定依赖：
```java
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

# 总结
refresh方法中的前三步到此已结束。经过此三步，ApplicationContext已经拥有了很多BeanFactory不一样的特性：可定制化BeanFactory、SpEL的支持、属性编辑器、Aware接口的设置、AOP的支持等等。但所有的扩展都是基于BeanFactory，所以核心还是基于BeanFactory的实现。
后续的步骤在后面的文章中再详细叙述。





