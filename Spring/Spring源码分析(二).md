# Spring源码分析(二）

## 一、上文回顾

上节基于可达性分析讲了Spring的BeanFactory的初始化，现在来回顾一下BeanFactory创建过程中关键的方法refresh,另外已经把源码更新到spring5.x版本了

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

上节讲到了这一句：

```java
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

再进入obtainFreshBeanFactory方法中的关键步骤refreshBeanFactory方法

```java
protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

可以发现其实代码很简单，createBeanFactory()创建了ConfigurableListableBeanFactory，到此BeanFactory就正式创建了

然后再看一下后面的第二句

```java
customizeBeanFactory(beanFactory)
```

翻译一下，定制BeanFactory，这啥意思啊，我的BeanFactory不是创建出来了吗，还要定制？定制什么？点进去看看把……

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

哦！原来是这样，由于BeanFactory是用来存放Bean的，因此难免会碰到**Bean定义重写**（这应该很重要，我想增强BeanDefinition就靠这个）和**Bean循环引用**的情况（这有时可能会产生问题），ok，问题解决，接下来回顾及拓展结束，进入本节的正题



## 二、思考

现在容器已经产生了，那么，创建容器是用来解决什么问题的呢？既然是一个Bean工厂——当然是用来放Bean的咯，准确的来说——是用来放Bean的定义的。

其实这也不难理解，如果BeanFactory存放的是实例Bean，那么即代表在容器创建完成后需要将应用中定义的所有Bean都进行实例化，但实际上可能大多数Bean都不是我们立即想使用的，这会导致应用**启动缓慢**，且非常**影响性能**，另外，倘若我想在程序运行时对Bean进行一些增强，这样做也并**不具有可拓展性**。

所以拍脑袋一想就不难发现，如果BeanFactory中存放的只是一个Bean的定义，那么上述的问题都可以得到解决，BeanFactory只需要在用户需要使用Bean的时候为其创建（当然有些Bean可能需要应用运行时即创建），且能够通过修改Bean定义对其进行增强，这里我不禁要感叹这是多棒的设计啊哈哈~

这里所说的Bean定义，在Spring对应的即为**BeanDefinition**，在上面的refreshBeanFactory方法中，可以发现，当BeanFactory被创建之后，他会在第一时间做**loadBeanDefinitions**操作，即将Bean定义加载到Bean工厂中，精彩内容要开始了！



## 三、loadBeanDefinitions()

AbstractRefreshableApplicationContext下的loadBeanDefinitions是一个抽象方法，这也很好理解，一个Spring应用可以有很多定义Bean的方法，如xml、注解方式，甚至还可以有远程获取Bean的方法，因此也有很多获取Bean定义的方法，简单看一下重写了**loadBeanDefinitions**方法的类有哪些

![img](http://kylescloud.top/site/pic/loadDefinition.png)

这些类都重写了loadBeanDefinitions方法，不难通过类名判断各个类的意思，本节会先通过AbstractXmlApplicaitonContext来将如何加载bean定义，虽然现在普遍流行的是springboot通过注解形式来加载bean，但是AbstractXmlApplicaitonContext的loadBeanDefinitions方法下有很多值得关注的内容，且新手入门时接触的可能更多的是XML，因此也更容易理解。下面贴源码~在阅读源码之前，不难想到，如果需要从XML文件中读取配置，那么就需要通过IO来获取文件数据流，带着这个概念看下面的代码就不难理解

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
			beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}
```

这个方法有趣的地方在于方法**最后调用了重载的**loadBeanDefinitions方法，传入了beanDefinitionReader，再看上面的一些操作，可以推断，当前的loadBeanDefinition方法实际上是对BeanDefinitionReader**做了一些初始化和赋值操作**，让其能在加载bean的时候能正常工作，这一部分涉及了很多很重要的概念，下面重点介绍BeanDefinitionReader的相关操作

### **1.**创建BeanDefinitonReader，初始化Bean注册器

```java
XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
```

直接点进去

```java
/**
 * Create new XmlBeanDefinitionReader for the given bean factory.
 * @param registry the BeanFactory to load bean definitions into,
 * in the form of a BeanDefinitionRegistry
 */
public XmlBeanDefinitionReader(BeanDefinitionRegistry registry) {
   super(registry);
}
```

发现调用了XmlBeanDefinitionReader的构造器的父类构造方法，这里令我感到奇怪的是，为什么参数从ConfigurableListableBeanFactory变成了BeanDefinitionRegistry……后来仔细一看，发现ConfigurableListableBeanFactory原来实现了BeanDefinitionRegistry接口，那么BeanDefinitionRegistry为其提供了什么功能？看一下代码吧

```java
public interface BeanDefinitionRegistry extends AliasRegistry {
	//注册Bean定义
	void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException;
	//删除Bean定义
	void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
	//获取Bean定义
	BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
	//判断Bean定义是否存在
	boolean containsBeanDefinition(String beanName);
	//获取所有Bean定义的names
	String[] getBeanDefinitionNames();
	//获取Bean定义的数目
	int getBeanDefinitionCount();
	//是否使用Bean name存储的定义
	boolean isBeanNameInUse(String beanName);
}
```

这样BeanDefinitionRegistry接口提供的功能就一目了然了，

回到上面XmlBeanDefinitionReader的构造方法的super(registry)中，可以看到下面的代码片段

```java
protected AbstractBeanDefinitionReader(BeanDefinitionRegistry registry) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		this.registry = registry;

		// Determine ResourceLoader to use.
		if (this.registry instanceof ResourceLoader) {
			this.resourceLoader = (ResourceLoader) this.registry;
		}
		else {
			this.resourceLoader = new PathMatchingResourcePatternResolver();
		}

		// Inherit Environment if possible
		if (this.registry instanceof EnvironmentCapable) {
			this.environment = ((EnvironmentCapable) this.registry).getEnvironment();
		}
		else {
			this.environment = new StandardEnvironment();
		}
}
```

BeanDefinitionReader通过传入的参数为自己的Bean注册器赋值，这样在读取XML文件后就有了注册Bean的工具，这比较好理解。然后是接下来比较重要的两部



### 2.初始化ResourceLoader&Environment

