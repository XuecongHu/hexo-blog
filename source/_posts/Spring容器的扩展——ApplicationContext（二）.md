---
title: 读书笔记《Spring源码深度解析》| Spring容器的扩展ApplicationContext（下）
date: 2018-01-03 14:26:43
tags: Java Spring
---
本文会接着上一篇继续介绍ApplicationContext启动时的步骤。建议在阅读本文之前先阅读上一篇文章([读书笔记《Spring源码深度解析》| Spring容器的扩展ApplicationContext（上）](http://www.frankhu.org/2017/12/16/Spring%E5%AE%B9%E5%99%A8%E7%9A%84%E6%89%A9%E5%B1%95%E2%80%94%E2%80%94ApplicationContext/))

<!--more-->

先回顾ApplicationContext启动时的步骤：

1. 初始化前的环境准备，例如读取系统属性或环境变量并进行验证。
2. 初始化持有的BeanFactory
3. 对BeanFactory进行功能填充
4. 可自定义在BeanFactory完成后的行为动作
5. 激活各种BeanFactory处理器
6. 注册拦截Bean创建的bean处理器
7. 为上下文初始化国际化的源
8. 初始化应用消息广播源
9. 留给子类来初始化其他的bean
10. 在所有注册的bean中查找listener bean，注册到消息广播器中
11. 初始化剩下的非惰性单例
12. 通知生命周期处理器刷新过程，同时发出ContextRefreshEvent的事件

对应的关键源码如下：
```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
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
上一篇文章介绍了BeanFactory准备完毕的步骤(环境准备，初始化BeanFactory，BeanFactory功能填充)，下文接着介绍后续的步骤。

# 自定义BeanFactory后处理
在refresh方法中，第四个步骤就是执行postProcessBeanFactory方法。该步骤中允许ApplicationContext自定义BeanFactory后的行为处理，增加了对BeanFactory自定义修改的能力。因此在AbstractApplicationContext中该方法是空的，留给了子类实现。
在AbstractRefreshableWebApplicationContext类中，我们看到了其中的一个实现例子：
```java
/**
* Register request/session scopes, a {@link ServletContextAwareProcessor}, etc.
*/
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext, this.servletConfig));
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    beanFactory.ignoreDependencyInterface(ServletConfigAware.class);

    WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, 		this.servletContext);
    WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, 			this.servletContext, this.servletConfig);
}
```
在该Context中的postProcessBeanFactory方法中进行了与webApplication有关的行为，例如增加了ServletContextAwareProcessor的BeanPostProcessor。
如果有对BeanFactory的自定义行为需求，可以继承ApplicationContext类并覆盖postProcessBeanFactory方法，满足需求。但需要注意的是此时任何一个bean实例都没有被创建，可以更改的是BeanFactory中的bean定义。但一般我们都不会使用该方式来自定义对BeanFactory的修改，一般会使用BeanFactoryPostProcessor接口。我们在后文可以看到有关BeanFactory的修改实例。

# 激活各种BeanFactoryPostProcessor
假如我们不想继承ApplicationContext类覆写postProcessBeanFactory方法来自定义修改BeanFactory，我们还添加BeanFactoryPostProcessor接口的实现类。我们先看BeanFactoryPostProcessor接口的定义：
```java
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```
从源码中看，BeanFactoryPostProcessor简单来说就是增加了对BeanFactory自定义修改的能力，和ApplicationContext类中的postProcessBeanFactory方法相同。从refresh源码我们也知道执行postProcessBeanFactory时，BeanFactory经过前面三步已经准备好了，所以是"PostProcessor"。但此时还没有任何一个Bean已经被实例化，所以一般是BeanFactory中的Bean定义进行修改。

举书中的例子，典型应用就是PropertyPlaceHolderConfigurer类。我们在写bean配置的时候，有时会希望某些配置是从其他文件读取的，例如一些不同环境的数据库配置等。
```xml
<bean id="message" class="HelloMessage">
    <property name="mes">
        <value>${bean.message}</value>
    </property>
</bean>
```
${bean.message}在另外一个配置文件中定义其值：
bean.message=hello world
Spring如何找到该配置文件，并且将字段的值对应起来？这就是PropertyPlaceHolderConfigurer类的作用：
```xml
<bean id="mesHandler" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="location">
    <value>config/bean.properties</value>
    </property>
</bean>
```
PropertyPlaceHolderConfigurer类之所以改变bean属性值为配置文件的值是因为间接实现了BeanFactoryPostProcessor接口。ApplicationContext会检测所有实现了BeanFactoryPostProcessor接口的类，并一一执行该接口的postProcessBeanFactory方法。
我们看看PropertyPlaceHolderConfigurer类的postProcessBeanFactory方法做了什么。实际上PropertyPlaceHolderConfigurer类自身没有postProcessBeanFactory方法。它间接继承于PropertyResourceConfigurer类，该类实现了BeanFactoryPostProcessor接口并有postProcessBeanFactory方法。
```java
public abstract class PropertyResourceConfigurer extends PropertiesLoaderSupport implements BeanFactoryPostProcessor, PriorityOrdered {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        try {
            Properties mergedProps = mergeProperties();

            // Convert the merged properties, if necessary.
            convertProperties(mergedProps);

            // Let the subclass process the properties.
            processProperties(beanFactory, mergedProps);
        }
        catch (IOException ex) {
            throw new BeanInitializationException("Could not load properties", ex);
        }
    }
    ...
}
```
该方法中的mergeProperties得到了配置文件，covertProperties将得到的配置转换为合适的类型，processProperties将值转换。

正因为此时BeanFactory没有任何实例，所以BeanFactoryPostProcessor接口在bean实例化之前改变的是Bean定义，后续所有的bean实例化都会基于该已经被BeanFactoryPostProcessor接口修改过的定义。所以上文例子中的HelloMessage类所有实例的属性值都会是配置文件中的值。

既然我们知道了BeanFactoryPostProcessor接口可以自定义修改BeanFactory，那么假如定义了很多BeanFactoryPostProcessor接口，又会以什么样的顺序执行呢？
要确定BeanFactoryPostProcessor接口的执行顺序，我们还需要实现Ordered接口，Ordered接口中有order属性，通过给order属性设值，我们能确定BeanFactoryPostProcessor的顺序——order属性值越小，执行顺序越前。此外还可以实现PriorityOrdered接口，实现了该接口的永远比实现了Ordered接口要优先。PriorityOrdered接口也有order属性，同样也是order值越小越优先。

## ApplicationContext激活BeanFactoryPostProcessor过程
现在我们知道了BeanFactoryPostProcessor接口的作用，究竟ApplicationContext是如何激活BeanFactoryPostProcessor？
首先我们要知道ApplicationContext添加BeanFactoryPostProcessor有两种方式：一种是硬编码，即通过调用ApplicationContext的addBeanFactoryPostProcessor方法添加，如：
```java
AbstractApplicationContext applicationContext =
                new ClassPathXmlApplicationContext("beanfactorypostprocessor.xml");
        applicationContext.addBeanFactoryPostProcessor(new xxBeanFactoryPostProcessor());
```
另一种方式我们常用的配置方式，如在xml文件中定义一个BeanFactoryPostProcessor bean。

激活这些BeanFactoryPostProcessor就是在refresh方法的第五步：执行invokeBeanFactoryPostProcessors方法。该方法的真正逻辑是在PostProcessorRegistrationDelegate类的invokeBeanFactoryPostProcessors方法执行。
该方法比较长，我们不一一细看，只看与BeanFactoryPostProcessor相关的关键部分。

方法只有两个入参：
```java
public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors)
```
beanFactory对象是ApplicationContext中持有的BeanFactory，beanFactoryPostProcessors列表是通过硬编码添加的BeanFactoryPostProcessor。

首先激活的是硬编码的BeanFactoryPostProcessor接口实现类：
```java
List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<>();
...
/**
* 首先激活硬编码的BeanFactoryPostProcessor，按照硬编码的顺序执行，不管有无实现PriorityOrdered和Ordered接口
*/
for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
    ...
    regularPostProcessors.add(postProcessor);
    ...
}
...
invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
```
从这段代码，我们可以知道硬编码方式添加的BeanFactoryPostProcessor是不管有无实现PriorityOrdered和Ordered接口的，其顺序就是硬编码时的先后顺序。如：
```java
AbstractApplicationContext applicationContext =
                new ClassPathXmlApplicationContext("beanfactorypostprocessor.xml");
applicationContext.addBeanFactoryPostProcessor(new ABeanFactoryPostProcessor()); //ABeanFactoryPostProcessor的执行顺序永远比BBeanFactoryPostProcessor要更前，不管BBeanFactoryPostProcessor有无实现PriorityOrdered或者Ordered接口，其order属性值是否更小。
applicationContext.addBeanFactoryPostProcessor(new BBeanFactoryPostProcessor());
```

激活完硬编码的BeanFactoryPostProcessor之后，激活通过配置方式添加的BeanFactoryPostProcessor。
```java
String[] postProcessorNames =
beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
List<String> orderedPostProcessorNames = new ArrayList<>();
List<String> nonOrderedPostProcessorNames = new ArrayList<>();
for (String ppName : postProcessorNames) {
    if (processedBeans.contains(ppName)) {
        // skip - already processed in first phase above
    }
    else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
    }
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
        orderedPostProcessorNames.add(ppName);
    }
    else {
        nonOrderedPostProcessorNames.add(ppName);
    }
}

// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
for (String postProcessorName : orderedPostProcessorNames) {
    orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
}
sortPostProcessors(orderedPostProcessors, beanFactory);
invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

// Finally, invoke all other BeanFactoryPostProcessors.
List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
for (String postProcessorName : nonOrderedPostProcessorNames) {
    nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
}
invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);
```
通过源码可以知道Spring会通过BeanFactory的getBeanNamesForType方法来获取所有实现了BeanFactoryPostProcessor接口类的beanName，然后根据BeanFactory的另外一个方法isTypeMatch来判断这些bean是否也实现了PriorityOrdered或Ordered接口。
按照我们的预想，spring果然是先执行实现了PriorityOrdered接口的类，而且会对这些实现类排序，按照排序的顺序激活，然后再按照同样方式执行Ordered接口实现类，最后才是执行两个接口都没有实现的普通类。

# 注册BeanPostProcessor
如果你之前没有了解过BeanFactoryPostProcessor和BeanPostProcessor，到这里你可能会很迷茫，怎么又有一个BeanPostProcessor，与BeanFactoryPostProcessor只有一词之差，两者有何区别？这里并不打算着重介绍BeanPostProcessor与BeanFactoryPostProcessor之间的区别，但是可以有一个初步的理解：
BeanFactoryPostProcessor修改的是还未创建任何Bean实例的BeanFactory，一般修改的是BeanDefinition；
BeanPostProcessor修改的是Bean实例，增加了对Bean实例定制化的能力，用源码说话：
```java
public interface BeanPostProcessor {
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```
BeanPostProcessor有两个方法，postProcessBeforeInitialization的执行时机是在bean的init-method方法之前，postProcessAfterInitialization则是在init-method方法之后调用。

回到spring容器的启动，refresh方法中的第六步registerBeanPostProcessors方法就是将自定义的BeanPostProcessor实现类添加到容器中，但是注意的是并不在此阶段激活，激活是在创建bean实例时，此阶段仅仅是将BeanPostProcessor按照与BeanFactoryPostProcessor类似的顺序（实现了PriorityOrdered接口的比Ordered接口更优先，相同接口的order属性值更小的更优先）注册给容器。
我们且看registerBeanPostProcessors方法中是如何做到的，同样只关注与BeanPostProcessor相关的关键源码。

registerBeanPostProcessors方法同样的真正实现是在PostProcessorRegistrationDelegate类的同名方法中：
```java
public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
```
方法有两入参，beanFactory对象表示的是ApplicationContext中的持有BeanFactory。

接下来的步骤与BeanFactoryPostProcessor非常相似。同样地通过getBeanNamesForType获取BeanPostProcessor实现类bean和isTypeMatch判断bean是否实现了PriorityOrdered或Ordered接口。PriorityOrdered接口的实现类比Ordered接口实现类更优先，相同接口则按照order属性大小比较。
```java
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
...
// Separate between BeanPostProcessors that implement PriorityOrdered,
// Ordered, and the rest.
List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
List<String> orderedPostProcessorNames = new ArrayList<>();
List<String> nonOrderedPostProcessorNames = new ArrayList<>();

for (String ppName : postProcessorNames) {
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, 		BeanPostProcessor.class);
        priorityOrderedPostProcessors.add(pp);
        ...
    }
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
        orderedPostProcessorNames.add(ppName);
    }
    else {
        nonOrderedPostProcessorNames.add(ppName);
    }
}

// First, register the BeanPostProcessors that implement PriorityOrdered.
sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

// Next, register the BeanPostProcessors that implement Ordered.
List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
for (String ppName : orderedPostProcessorNames) {
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    orderedPostProcessors.add(pp);
    ...
}

sortPostProcessors(orderedPostProcessors, beanFactory);
registerBeanPostProcessors(beanFactory, orderedPostProcessors);

// Now, register all regular BeanPostProcessors.
List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
for (String ppName : nonOrderedPostProcessorNames) {
    BeanPostProcessor pp = beanFactory.getBean(ppName, 	BeanPostProcessor.class);
    nonOrderedPostProcessors.add(pp);
}
registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

// Finally, re-register all internal BeanPostProcessors.
sortPostProcessors(internalPostProcessors, beanFactory);
...
```
另外一提的是ApplicationContext并没有提供硬编码的方式来添加BeanPostProcessor，只能通过配置文件的方式添加。因此在该registerBeanPostProcessors方法中，不像BeanFactoryPostProcessor还会有硬编码方式的对应处理代码。

# 初始化消息资源
消息资源的初始化是ApplicationContext比BeanFactory多扩展的一个重要部分，主要用于资源的国际化。例如正在开发一个国际化的web应用，要求支持多国语言，要求应用可以根据系统的语言展示不同语言的资源。ApplicationContext要做的就是定义一种规范，为每一种语言都定义一个资源文件，在ApplicationContext启动的时候根据系统语言加载资源文件，从而展示对应的语言资源。

这一部分内容由于非常少用，不打算展开介绍，有兴趣的读者可以查阅AbstractApplicationContext类的initMessageSource方法和专门抽象了用于国际化信息访问的MessageSource接口。

# 初始化事件广播器和注册监听器
首先我们了解一下Spring中的事件机制。
spring事件机制是spring中发布——订阅(观察者)设计模式的实现。通过ApplicationEvent和ApplicationListener接口就能完成。ApplicationEvent接口代表要发布的事件，ApplicationListener则是发布事件的监听器。

更清晰的描述见[网友的说法](http://www.cnblogs.com/ArtsCrafts/p/Spring_Event.html)
>实际上，Spring的事件机制与所有的事件机制都基本类似，他们都需要事件源、 事件和事件监听器组成。只是此处的事件源是ApplicationContext。
>下图简单示范了ApplicationContext事件
>![ApplicationContext事件](https://images0.cnblogs.com/blog/331136/201309/13095604-5a10a5126e7744468ed1e37f8e731882.jpg)

举书中例子：
```java
public class TestEvent extends ApplicationEvent {

    String msg;
    
    public TestEvent(Object source) {
        super(source);
    }

    public TestEvent(Object source, String msg) {
        super(source);
        this.msg = msg;
    }

    public void print() {
        System.out.println(msg);
    }
}
```
```java
public class TestListener implements ApplicationListener {
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof TestEvent) {
            TestEvent testEvent = (TestEvent)event;
            testEvent.print();
        }
    }
}
```
```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("testevent.xml");
TestEvent testEvent = new TestEvent("hello", "this is a message");
ctx.publishEvent(testEvent);
```
在上面的例子中自定义一个TestEvent和TestListener，通过ApplicationContext发布TestEvent，在容器中的TestListener会收到该Event，并执行其onApplicationEvent方法。

## 初始化ApplicationEventMulticaster
ApplicationContext中执行该事件发布的则是由其持有的ApplicationEventMulticaster对象负责。在ApplicationContext启动的时候，就是要初始化该ApplicationEventMulticaster对象。
```java
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
        beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        if (logger.isDebugEnabled()) {
            logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
    } 
    else {
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
        if (logger.isDebugEnabled()) {
            logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
                   APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
                   "': using default [" + this.applicationEventMulticaster + "]");
        }
    }
}
```

通过源码可以看到在这一步做的就是初始化一个ApplicationEventMulticaster对象，如果用户有自定义的ApplicationEventMulticaster接口实现类，则使用用户的自定义类；如果没有，则使用默认的SimpleApplicationEventMulticaster类。

我们可以顺便一看SimpleApplicationEventMulticaster类中的事件发布源码。
```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
    	Executor executor = getTaskExecutor();
    	if (executor != null) {
    		executor.execute(() -> invokeListener(listener, event));
    	}
    	else {
    		invokeListener(listener, event);
    	}
    }
}
```
可以看出发布一个Event，该类会将匹配该Event类型的的Listener都发布出去，所以是一种广播器。

## 注册监听器
ApplicationContext初始化广播器之后，执行registerListeners方法来注册自定义的监听器。
```java
protected void registerListeners() {
		// Register statically specified listeners first.
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
  }

    // Publish early application events now that we finally have a multicaster...
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (earlyEventsToProcess != null) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}
```
首先注册的是硬编码的ApplicationListener，然后注册通过配置文件定义的ApplicationListener Bean。最后由于此时已经有了广播器，还可以发布一些早期的应用事件。

# 完成BeanFactory初始化
接下来到了完成BeanFactory的初始化，这一步主要包括ConversionService的配置、Bean定义的冻结和非延迟加载的bean初始化三件事情。
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Initialize conversion service for this context.
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
    	beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
    	beanFactory.setConversionService(
    	beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no bean post-processor
    // (such as a PropertyPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    if (!beanFactory.hasEmbeddedValueResolver()) {
    	beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
    	getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);

    // Allow for caching all bean definition metadata, not expecting further changes.
    beanFactory.freezeConfiguration();

    // Instantiate all remaining (non-lazy-init) singletons.
    beanFactory.preInstantiateSingletons();
}
```

## ConversionService的设置
在[上一篇文章中](http://www.frankhu.org/2017/12/16/Spring%E5%AE%B9%E5%99%A8%E7%9A%84%E6%89%A9%E5%B1%95%E2%80%94%E2%80%94ApplicationContext/#%E5%B1%9E%E6%80%A7%E7%BC%96%E8%BE%91%E5%99%A8%E7%9A%84%E6%94%AF%E6%8C%81)，我们提过了ApplicationContext有提供类型转换的途径。实际上Spring还可以通过ConversionService来转换类型。
我们先举一个具体的例子： 假如有一个People类的birthDate属性是LocalDate，在xml配置文件中该字段是String类型，我们希望在ApplicationContext中可以自动将String类型转换为LocalDate类型，简约xml配置。
首先需要自定义一个Converter接口实现类，在该类中实现转换类型的逻辑：
```java
public class String2DateConverter implements Converter<String, LocalDate> {

    public LocalDate convert(String source) {
        final DateTimeFormatter dff = DateTimeFormatter.ofPattern("yyyy-MM-dd");
        final LocalDate date = LocalDate.parse(source, dff);
        return date;
    }
}
```
然后在xml配置文件中将converter注册给ConversionServiceFactoryBean：
```xml
<bean id="conversionService" class="org.springframework.context.support.ConversionServiceFactoryBean">
  <property name="converters">
    <list>
    	<bean class="com.huxuecong.springpractice.converter.String2DateConverter"/>
    </list>
  </property>
</bean>

<bean id="pp" class="com.huxuecong.springpractice.bean.People">
	<property name="birthDate" value="1992-10-09"/>
</bean>
```
启动ApplicationContext时就会自动将xml文件中People的birthDate属性从String类型转换为LocalDate类型：
```java
AbstractApplicationContext context = new ClassPathXmlApplicationContext("conversionservice.xml");
System.out.println((People) context.getBean("pp"));
```

为什么Spring还要提供ConversionSevice来转换类型？
实际上在上篇文章中提过的两种转换方式都是基于jdk的PropertyEditor接口，自Spring3之后，Spring就提供了统一的抽象规范方式来进行类型转换：ConversionService接口。我们先看ConversionService的源码：
```java
public interface ConversionService {
    boolean canConvert(@Nullable Class<?> sourceType, Class<?> targetType);

    boolean canConvert(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType);

    <T> T convert(@Nullable Object source, Class<T> targetType);

    Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType);

}
```
如同其名字，ConversionService提供的是类型转换的服务，而且不仅是某两种种类型之间的“单一”转换，而是包含了多种类型之间的转换。而负责具体执行“单一“转换的是Converter接口：
```java
public interface Converter<S, T> {
    T convert(S source);
}
```
具体有关ConversionService的信息不过多介绍，有兴趣的读者可以继续查阅。

回到完成BeanFactory初始化这一步中，做的事情就是给BeanFactory设置ConversionService：
```java
if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
    beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
    beanFactory.setConversionService(
    beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
}
```
由于我们在xml配置文件中设置了ConversionServiceFactoryBean，而通过ConversionServiceFactoryBean源码我们可以知道ApplicationContext默认创建的是DefaultConversionService。
```java
public class ConversionServiceFactoryBean implements FactoryBean<ConversionService>, InitializingBean {

    private Set<?> converters;

    private GenericConversionService conversionService;

    public void setConverters(Set<?> converters) {
        this.converters = converters;
    }

    @Override
    public void afterPropertiesSet() {
        this.conversionService = createConversionService();
        ConversionServiceFactory.registerConverters(this.converters, this.conversionService);
    }

    protected GenericConversionService createConversionService() {
        return new DefaultConversionService();
    }
    ...
}
```
通过这段源码也知道ConversionService持有Converter集合，通过调用Converter来完成类型转换。

##  冻结Bean定义
至此类型转换完成了，ApplicationContext将调用BeanFactory的freezeConfiguration方法对Bean定义进行冻结，使得任何的BeanDefinition不能被修改或进一步处理。
```java
public void freezeConfiguration() {
    this.configurationFrozen = true;
    this.frozenBeanDefinitionNames = StringUtils.toStringArray(this.beanDefinitionNames);
}
```

## 初始化非延迟加载
ApplicationContext接下来会将所有非惰性单例bean提前实例化，这样做的好处是可以在启动容器时就能知道配置的错误，而无需等到真正使用时才出错。
```java
public void preInstantiateSingletons() throws BeansException {
	...
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    for (String beanName : beanNames) {
    	RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
    	// 提前实例化非惰性的单例bean
    	if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
    		if (isFactoryBean(beanName)) {
      			...
                if (isEagerInit) {
                	getBean(beanName);
                }
    		}
    		else {
    			getBean(beanName);
    		}
    	}
    }
	...
}
```


# finishRefresh
终于来到了ApplicationContext启动的最后一步：执行finishRefresh方法。经过这一步，ApplicationContext的所有启动动作都执行完毕。在该方法中主要包含了两个事情：
1. 初始化LifecycleProcessor并调用其onRefresh方法来启动Lifecycle bean的生命周期
2. 发布ContextRefreshedEvent事件，让感兴趣的Listener收到
```java
protected void finishRefresh() {
    // Clear context-level resource caches (such as ASM metadata from scanning).
    clearResourceCaches();

    // Initialize lifecycle processor for this context.
    initLifecycleProcessor();

    /**
    * 启动LifecycleProcessor的refresh方法，启动所有Lifecycle
    */
    // Propagate refresh to lifecycle processor first.
    getLifecycleProcessor().onRefresh();

    // Publish the final event.
    publishEvent(new ContextRefreshedEvent(this));

    // Participate in LiveBeansView MBean, if active.
    LiveBeansView.registerApplicationContext(this);
}
```

## 启动Lifecycle声明周期
Spring提供了Lifecycle接口来使得bean的启动和结束生命周期和ApplicationContext一致。例如可以用来配置异步的后台程序，在启动容器后一直执行(如对消息队列的轮询），在容器关闭时也同样关闭。
```java
public interface Lifecycle {

	void start();

	void stop();

	boolean isRunning();
}
```
Spring将容器启动启动Lifecycle生命周期和关闭时结束Lifecycle生命周期的具体工作委托给LifecycleProcessor接口来完成：
```java
public interface LifecycleProcessor extends Lifecycle {
	/**
	 * Notification of context refresh, e.g. for auto-starting components.
	 */
	void onRefresh();

	/**
	 * Notification of context close phase, e.g. for auto-stopping components.
	 */
	void onClose();
}
```

ApplicationContext的initLifecycleProcessor方法则是初始化一个默认的LifecycleProcessor(假如用户没有自定义)——DefaultLifecycleProcessor。
```java
protected void initLifecycleProcessor() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
      this.lifecycleProcessor =
        beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
      if (logger.isDebugEnabled()) {
        logger.debug("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
      }
    }
    else {
      DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
      defaultProcessor.setBeanFactory(beanFactory);
      this.lifecycleProcessor = defaultProcessor;
      beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
      if (logger.isDebugEnabled()) {
        logger.debug("Unable to locate LifecycleProcessor with name '" +
                     LIFECYCLE_PROCESSOR_BEAN_NAME +
                     "': using default [" + this.lifecycleProcessor + "]");
      }
    }
}
```
因此，默认调用DefaultLifecycleProcessor的onRefresh方法。
```java
public class DefaultLifecycleProcessor implements LifecycleProcessor, BeanFactoryAware {
	...
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	}

	private void startBeans(boolean autoStartupOnly) {
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<>();
		lifecycleBeans.forEach((beanName, bean) -> {
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);
				LifecycleGroup group = phases.get(phase);
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(beanName, bean);
			}
		});
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<>(phases.keySet());
			Collections.sort(keys);
			for (Integer key : keys) {
				phases.get(key).start();
			}
		}
	}
```
但通过看源码，我们发现光实现了Lifecycle接口的bean并不会在ApplicationContext启动的时候自动启动，而是SmartLifecycle接口才会。关于这点，在Lifecycle接口的定义中也有说明：
> A common interface defining methods for start/stop lifecycle control.
>   The typical use case for this is to control asynchronous processing.
>   NOTE: This interface does not imply specific auto-startup semantics.
>   Consider implementing SmartLifecycle for that purpose.

关于Lifecycle接口的更多详细信息，读者可以自行查阅更多专门的资料。

## 发布ContextFreshedEvent事件
当ApplicationContext启动完毕，Spring会发布ContextFreshedEvent事件使得感兴趣的监听器收到并做相应的处理。
```java
publishEvent(new ContextRefreshedEvent(this));
```

# The Ending
本系列从源码的角度介绍了ApplicationContext启动时所完成的工作，这章所介绍的内容基本都是比BeanFactory所扩展的功能。每一个扩展的功能都可以作为一个专题来介绍，但本文仅关注于ApplicationContext启动时的主要步骤。

总而言之，ApplicationContext在BeanFactory的基础上做了很多额外的事情，并都具有强大的扩展性，留给用户自定义，如BeanFactoryPostProcessor、类型转换、Aware接口、Lifecycle等。这些良好的设计都很值得我们慢慢学习。
