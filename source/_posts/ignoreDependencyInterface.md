---
title: 打开BeanFactory ignoreDependencyInterface方法的正确姿势
date: 2018-01-13 18:40:08
tags: Java Spring
---

在阅读Spring容器扩展部分源码的过程中，我了解到BeanFactory接口中有个方法叫ignoreDependencyInterface。从官方文档的“字面”来看，其作用指定自动装配(autowiring)的时候忽略的接口。还有一个很相似的方法叫ignoreDependencyType，同样其官方字面意思是指自动装配(autowiring)的时候忽略的类。
究竟这两个方法是不是我们的理解相同呢？真的可以让指定的接口和类在自动装配的时候被忽略？有没有注意不到的坑？
<!--more-->
```java
/**
* Ignore the given dependency interface for autowiring.
* <p>This will typically be used by application contexts to register
* dependencies that are resolved in other ways, like BeanFactory through
* BeanFactoryAware or ApplicationContext through ApplicationContextAware.
* <p>By default, only the BeanFactoryAware interface is ignored.
* For further types to ignore, invoke this method for each type.
* @param ifc the dependency interface to ignore
* @see org.springframework.beans.factory.BeanFactoryAware
* @see org.springframework.context.ApplicationContextAware
*/
void ignoreDependencyInterface(Class<?> ifc);

/**
* Ignore the given dependency type for autowiring:
* for example, String. Default is none.
* @param type the dependency type to ignore
*/
void ignoreDependencyType(Class<?> type);
```

先上结论：
1. 自动装配时忽略指定接口或类的依赖注入，使用ignoreDependencyType已经足够
2. ignoreDependencyInterface的真正意思是在自动装配时忽略指定接口的实现类中，对外的依赖。

具体详述见下文的逐步踩坑。

# "自动装配(autowiring)"的坑
首先我们来试试水。我们试试能不能忽略一些我们常用的类。
假设有一个类，叫ListHolder，就只有一个用@Autowired注解的ArrayList对象。
```java
public class ListHolder {
	@Autowired
	private ArrayList<String> list;

	public ArrayList<String> getList() {
		return list;
	}

	public void setList(ArrayList<String> list) {
		this.list = list;
	}
}
```
xml配置文件配置了ArrayList bean和ListHolder bean：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
	   http://www.springframework.org/schema/context
	   http://www.springframework.org/schema/context/spring-context.xsd">

	<bean id="list" class="java.util.ArrayList">
		<constructor-arg>
			<list>
				<value>foo</value>
				<value>bar</value>
			</list>
		</constructor-arg>
	</bean>

	<bean id="listHolder" class="com.huxuecong.ignoreDependency.ListHolder"/>
	<bean class="com.huxuecong.autowire.IgnoreAutowiringProcessor"/>
    <context:annotation-config></context:annotation-config>
</beans>
```
我们定义一个BeanFactoryPostProcessor接口实现类，在接口实现类中调用ignoreDependencyType方法：
```java
public class IgnoreAutowiringProcessor implements BeanFactoryPostProcessor {
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		beanFactory.ignoreDependencyType(ArrayList.class);
	}
}
```
运行结果：
[foo, bar]

没有起作用。即使调用了ignoreDependencyType方法，ArrayList还是被注入了。说明ignoreDependencyType方法完全不起到该有的作用。
由于我对Spring的了解尚浅，我很怀疑自动装配是不是真的和我了解的相同，使用@Autowired注解就可以了呢？

经过一番中英文搜索之后，我终于发现有[博客](http://yangbolin.cn/2016/11/12/spring-work-01/)专门讲ignoreDependencyType和ignoreDependencyInterface方法。
在博客中提到了我之前没用过的自动装配方式：在beans标签中使用default-autowire属性来注入依赖。
于是我这次按照网友的说法，对我的例子进行改造：
(1)ListHolder中的list对象不使用@Autowired注解
```java
public class ListHolder {
	private ArrayList<String> list;

	public ArrayList<String> getList() {
		return list;
	}

	public void setList(ArrayList<String> list) {
		this.list = list;
	}
}
```
(2)xml配置文件中去掉<context:annotation-config/>标签，并且在beans标签添加default-autowire的属性，其值为“byType"，意思是按照对象的类型进行装配。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
	   http://www.springframework.org/schema/context
	   http://www.springframework.org/schema/context/spring-context.xsd"
	   default-autowire="byType">

	<bean id="list" class="java.util.ArrayList">
		<constructor-arg>
			<list>
				<value>foo</value>
				<value>bar</value>
			</list>
		</constructor-arg>
	</bean>

	<bean id="listHolder" class="com.huxuecong.ignoreDependency.ListHolder"/>
	<bean class="com.huxuecong.autowire.IgnoreAutowiringProcessor"/>
</beans>
```
这次运行之后，结果就符合期待了：
null

经过这次踩坑，发现英语中的autowiring特定指的是通过beans标签default-autowire属性来依赖注入的方式，而不是指使用@Autowired注解进行的依赖注入。区别在于，使用default-autowire会自动给所有的类都会从容器中查找匹配的依赖并注入，而使用@Autowired注解只会给这些注解的对象从容器查找依赖并注入。

自动装配和@Autowired注解的装配不是同一回事。

但从这次例子来看，ignoreDependencyType方法和我们期待的完全一致，可以在自动装配的时候忽略ArrayList类的对象。

# ignoreDependencyInterface的坑
在愉快地使用ignoreDependencyType方法后，立即如法炮制ignoreDependencyInterface方法，如无意外，应该效果一样，使得某个接口的实现类被忽略。
于是在上面的例子上进行改造：
(1)ListHolder类中的ArrayList类型改为List类型：
```java
public class ListHolder {
	private List<String> list;

	public List<String> getList() {
		return list;
	}

	public void setList(List<String> list) {
		this.list = list;
	}
}
```
(2)ignoreDependencyType改为使用ignoreDependencyInterface方法：
```java
public class IgnoreAutowiringProcessor implements BeanFactoryPostProcessor {
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		beanFactory.ignoreDependencyInterface(List.class);
	}
}
```
运行结果：
[foo, bar]

完全不起作用，ignoreDependencyInterface不按照我们的想法工作。曾经一度怀疑Spring是否有bug，但是从Spring的流行性和网上搜索结果来看，这种可能微乎其微，更多是对该方法的理解不对。
我突发奇想，能不能也把接口List.class传给ignoreDependencyType，而不是使用ArrayList.class呢？
于是又在上面的代码基础上，改为：
```java
public class IgnoreAutowiringProcessor implements BeanFactoryPostProcessor {
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		beanFactory.ignoreDependencyType(List.class);
	}
}
```
这次的运行结果：null
从这次的结果来看，如果真的要忽略某个接口的实现类，大不必使用ignoreDependencyInterface方法，ignoreDependencyType方法已经居家必用了。

迷之一样的ignoreDependencyInterface方法到底是用来干嘛的呢？

# ignoreDependencyInterface的真正奥义
谷歌苦寻无果，无论中英文都没有发现比较详细的阐述，也说明该方法甚少被使用。只能自行在源码中寻找答案。

经过若干个小时在咖啡厅的不断调试，终于发现最终的奥义，我们只看最终关键的代码。
调用ignoreDependencyInterface方法后，被忽略的接口会存储在BeanFactory的名为ignoredDependencyInterfaces的Set集合中，而调用ignoreDependencyType则存储在ignoredDependencyTypes的Set集合中：
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

    private final Set<Class<?>> ignoredDependencyInterfaces = new HashSet<>();
    private final Set<Class<?>> ignoredDependencyTypes = new HashSet<>();

    public void ignoreDependencyType(Class<?> type) {
        this.ignoredDependencyTypes.add(type);
    }
    
    public void ignoreDependencyInterface(Class<?> ifc) {
        this.ignoredDependencyInterfaces.add(ifc);
    }
...
}
```
ignoredDependencyInterfaces集合在同类中被使用仅在一处——isExcludedFromDependencyCheck方法中：
```java
protected boolean isExcludedFromDependencyCheck(PropertyDescriptor pd) {
    return (AutowireUtils.isExcludedFromDependencyCheck(pd) || this.ignoredDependencyTypes.contains(pd.getPropertyType()) || AutowireUtils.isSetterDefinedInInterface(pd, this.ignoredDependencyInterfaces));
}
```
isExcludedFromDependencyCheck方法的意思是判断给定的bean属性在依赖检测中要被排除，假如该方法返回true，也就是在依赖检测中这个bean的属性要被排除，在自动装配时就会被忽略。

通过这个方法的源码也就明白，实际上我们说的在自动装配时忽略某个类或者接口的实现，使用ignoreDependencyType方法已经足够了，因为在isExcludedFromDependencyCheck方法中使用ignoredDependencyTypes集合是否包含属性的类型来判断。
因此在我们例子中，ListHolder对象中的list属性是List接口的实现，而我们又把List.class参数传给ignoreDependencyType方法，自然就会在自动装配时被忽略。

而ignoredDependencyInterface的真正作用还得看AutowireUtils类的isSetterDefinedInInterface方法。
```java
public static boolean isSetterDefinedInInterface(PropertyDescriptor pd, Set<Class<?>> interfaces) {
    //获取bean中某个属性对象在bean类中的setter方法
    Method setter = pd.getWriteMethod();
    if (setter != null) {
        // 获取bean的类型
        Class<?> targetClass = setter.getDeclaringClass();
        for (Class<?> ifc : interfaces) {
            if (ifc.isAssignableFrom(targetClass) && // bean类型是否接口的实现类
                ClassUtils.hasMethod(ifc, setter.getName(), setter.getParameterTypes())) { // 接口是否有入参和bean类型完全相同的setter方法
                return true;
            }
        }
    }
    return false;
}
```
这个方法的意思就是判断这一堆接口中有没有某个接口是拥有该bean属性的setter方法的。在我的例子中就是判断List接口有没有list属性类型的setter方法，也就是有无自己本身类型的setter方法。List接口的方法当中当然没有setList(List list)的方法啊，因此也不会生效。

所以ignoredDependencyInterface方法并不是让我们在自动装配时直接忽略实现了该接口的依赖。
这个方法的真正意思是忽略该接口的实现类中和接口setter方法入参类型相同的依赖。
举个例子。首先定义一个要被忽略的接口。
```java
public interface IgnoreInterface {

	void setList(List<String> list);

	void setSet(Set<String> set);
}
```
然后需要实现该接口，在实现类中注意要有setter方法入参相同类型的域对象，在例子中就是List<String>和Set<String>。
```java
public class IgnoreInterfaceImpl implements IgnoreInterface {

	private List<String> list;
	private Set<String> set;

	@Override
	public void setList(List<String> list) {
		this.list = list;
	}

	@Override
	public void setSet(Set<String> set) {
		this.set = set;
	}

	public List<String> getList() {
		return list;
	}

	public Set<String> getSet() {
		return set;
	}
}
```
定义xml配置文件：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
	   http://www.springframework.org/schema/context
	   http://www.springframework.org/schema/context/spring-context.xsd"
default-autowire="byType">


	<bean id="list" class="java.util.ArrayList">
		<constructor-arg>
			<list>
				<value>foo</value>
				<value>bar</value>
			</list>
		</constructor-arg>
	</bean>

	<bean id="set" class="java.util.HashSet">
		<constructor-arg>
			<list>
				<value>foo</value>
				<value>bar</value>
			</list>
		</constructor-arg>
	</bean>

	<bean id="ii" class="com.huxuecong.ignoreDependency.IgnoreInterfaceImpl"/>
	<bean class="com.huxuecong.autowire.IgnoreAutowiringProcessor"/>
</beans>
```
最后调用ignoreDependencyInterface:
```java
beanFactory.ignoreDependencyInterface(IgnoreInterface.class);
```
运行结果：
null
null
而如果不调用ignoreDependencyInterface，则是：
[foo, bar]
[bar, foo]

忽略接口生效。但其意思和我们最初理解的存在一定的差距。我们最初理解是在自动装配时忽略该接口的实现，实际上是在自动装配时忽略该接口实现类中和setter方法入参相同的类型，也就是忽略该接口实现类中存在依赖外部的bean属性注入。

典型应用就是BeanFactoryAware和ApplicationContextAware接口。
首先看该两个接口的源码：
```java
public interface BeanFactoryAware extends Aware {
	void setBeanFactory(BeanFactory beanFactory) throws BeansException;
}

public interface ApplicationContextAware extends Aware {
	void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```
在Spring源码中在不同的地方忽略了该两个接口：
```java
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
ignoreDependencyInterface(BeanFactoryAware.class);
```
使得我们的BeanFactoryAware接口实现类在自动装配时不能被注入BeanFactory对象的依赖：
```java
public class MyBeanFactoryAware implements BeanFactoryAware {
	private BeanFactory beanFactory; // 自动装配时忽略注入

	@Override
	public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
		this.beanFactory = beanFactory;
	}

	public BeanFactory getBeanFactory() {
		return beanFactory;
	}
}
```
ApplicationContextAware接口实现类中的ApplicationContext对象的依赖同理：
```java
public class MyApplicationContextAware implements ApplicationContextAware {
	private ApplicationContext applicationContext; // 自动装配时被忽略注入

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext;
	}

	public ApplicationContext getApplicationContext() {
		return applicationContext;
	}
}
```

这样的做法使得ApplicationContextAware和BeanFactoryAware中的ApplicationContext或BeanFactory依赖在自动装配时被忽略，而统一由框架设置依赖，如ApplicationContextAware接口的设置会在ApplicationContextAwareProcessor类中完成：
```java
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
通过这种方式保证了ApplicationContextAware和BeanFactoryAware中的容器保证是生成该bean的容器。

但在实践中我们什么时候会使用ignoreDependencyInterface接口？
笔者使用Spring经验有限，只能给出目前的应用场景很少，但起码想到一个：假如我们想自定义一个类似的xxAware接口，比如ApplicationEventMulticasterAware。那么调用ignoreDependencyInterface方法可以保证获取到的ApplicationEventMulticaster对象就是生成该bean容器中的ApplicationEventMulticaster对象。

