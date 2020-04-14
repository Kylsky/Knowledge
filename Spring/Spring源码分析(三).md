# Spring源码分析(三)

## 一、回顾

上回讲到refresh方法中通过obtainFreshBeanFactory创建了beanFactory并且将配置文件中的bean转换为BeanDefinition再注册到BeanFactory中。在BeanFactory创建和BeanDefinitions注册完成后，本篇将对BeanFactory相关的后续操作进行介绍，直到初始化单例bean为止。先贴一下代码看个大概

```java
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			prepareRefresh();
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
            //本篇文章从这里开始讲
			prepareBeanFactory(beanFactory);
			try {
                //重要，点进去看一下，后面会介绍
				postProcessBeanFactory(beanFactory);
                //重要，之后有介绍
				invokeBeanFactoryPostProcessors(beanFactory);
                //注册BeanPostProcessors，不展开
				registerBeanPostProcessors(beanFactory);
                //初始化当前 ApplicationContext 的 MessageSource，是国际化的相关操作，不展开，可以看看这篇文章：https://blog.csdn.net/sid1109217623/article/details/84065725
				initMessageSource();
                //初始化当前 ApplicationContext 的事件广播器，不展开
				initApplicationEventMulticaster();
                //钩子函数，不展开
				onRefresh();
                //注册监听器，不展开
				registerListeners();
                //这段代码非常重要，因为篇幅较长，所以请看下回分解
				finishBeanFactoryInitialization(beanFactory);
				finishRefresh();
			}
			catch (BeansException ex) {
				……
			}
			finally {
				……
				resetCommonCaches();
			}
		}
    }
```



## 二、prepareBeanFactory(beanFactory)

主要内容：设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// 设置 BeanFactory 的类加载器，我们知道 BeanFactory 需要加载类，也就需要类加载器，这里设置为加载当前 ApplicationContext 类的类加载器
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// 这句话比较重要，点进去看看，后面有详细介绍，看完记得跳回来~
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

//下面几行的意思就是，如果某个 bean 依赖于以下几个接口的实现类，在自动装配的时候忽略它们，Spring 会通过其他方式来处理这些依赖。 
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

/**
 * 下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值，
 * 这里解释一下第一行，对于一个应用上下文，即整个spring实例，beanfactory也只是其持有的一个bean
 * ApplicationContext 还继承了 ResourceLoader、ApplicationEventPublisher、MessageSource
 * 所以对于这几个依赖，可以赋值为 this，注意 this 是一个 ApplicationContext
 * 那这里怎么没看到为 MessageSource 赋值呢？那是因为 MessageSource 被注册成为了一个普通的 bean
 */		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// 这个 BeanPostProcessor 也很简单，在 bean 初始化后，如果是 ApplicationListener 的子类，
   		// 那么将其添加到 listener列表中，可以理解成注册事件监听器
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		//这里涉及到特殊的 bean，名为：loadTimeWeaver，是AspectJ的内容，会执行运行时织入，暂时忽略
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		//如果没有定义以下bean，spring会进行初始化
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

### beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this))

这句话字面意思理解起来不难，给beanFactory添加一个BeanPostProcessor，叫ApplicationContextAwareProcessor，这里需要理解两个概念——**什么是BeanPostProcessor，什么是(ApplicationContext)Aware**

分别来看看两者的代码，先是BeanPostProcessor

```java
/* 将此BeanPostProcessor应用于给定的
 * 显式初始化方法回调之前的新bean实例
 */
public interface BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

官方的翻译理解起来是真的困难，简单来说，就是在beanDefinition被实例化，并完成了各项属性赋值后，这个bean仍会有一个初始化(init)操作，BeanPostProcessor就是负责在这个初始化操作之前和之后对bean进行一系列处理。

接下来再看看Aware接口

```java
/**
 * Marker superinterface indicating that a bean is eligible to be
 * notified by the Spring container of a particular framework object
 * through a callback-style method. Actual method signature is
 * determined by individual subinterfaces, but should typically
 * consist of just one void-returning method that accepts a single
 * argument.
 */
public interface Aware {

}
```

Aware，翻译过来是知道的，已感知的，意识到的，所以这些接口从字面意思应该是能感知到所有Aware前面的含义。不过这个接口下面啥都没有，真够抽象的，看来是把感知这一任务都交给自己的实现类了，就拿这回说的ApplicationContextAware举例吧：

```java
public interface ApplicationContextAware extends Aware {
	void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

看来所谓的感知也就是get、set啊……



现在可以回到之前要探讨的内容了,如下：

```
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
```

其实这段代码的意思就是添加了一个BeanPostProcessor，这个处理器用来处理哪个bean的初始化前后相关操作呢？答案是ApplicationContextAware。

不难想象，当我们需要通过ApplicationContextAware来对ApplicationContext进行get、set操作时(这是必然会做的)，就会对它进行初始化，为了在其初始化（注意不是实例化）前后能做些操作，比如bean初始化前后的日志等，就可以为其增加一个BeanPostProcessor。

看完这部分代码，prepareBeanFactory方法的重要内容就差不多了，继续跳回到prepareBeanFactory()看后续吧。



## 三、postProcessBeanFactory(beanFactory)

postProcessBeanFactory后处理beanFactory。时机是在所有的beanDenifition加载完成之后，bean实例化之前执行。比如，在beanfactory加载完成所有的bean后，想修改其中某个bean的定义，或者对beanFactory做一些其他的配置，就可以用此方法。这里贴一下AbstractRefreshableWebApplicationContext对postProcessBeanFactory的实现,比较简单，就不细说了：

```java
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    	//添加一个BeanPostProcessor，用于处理ServletContextAware
        beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext, this.servletConfig));

beanFactory.ignoreDependencyInterface(ServletContextAware.class);
        beanFactory.ignoreDependencyInterface(ServletConfigAware.class);
 //注册应用域，如request、session、application等
WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
//注册EnvironmentBeans，如servletContext、contextParameters等
WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext, this.servletConfig);
}
```



## 四、invokeBeanFactoryPostProcessors(beanFactory)

在看代码之前，先来了解一下什么是BeanFactoryPostProcessor，发现这玩意儿和BeanPostProcessor像得很，前面讲到，BeanPostProcessor是在bean已经实例化时对其初始化操作前后进行处理，而对于BeanFactoryPostProcessor，他在后者的基础上加上了“Factory”这一词语，那么是否可以这么理解——BeanPostProcessor是对Bean(已实例化的Bean)的处理，而BeanFactoryPostProcessor是对BeanFactory的处理。来看看BeanFactoryPostProcessor，这样就一目了然了

```java
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

有了BeanFactoryPostProcessor，其实甚至可以做到在Bean实例化之前就对他们做处理，如果你对一些beanDefinition感到深恶痛绝，可以考虑实现一个BeanFactoryPostProcessor来将讨厌的bean扼杀在摇篮里！interesting~

不打岔了，还是继续回到正事，来看看下面的方法吧：

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
}
```

PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors())，最重要的就是这句了，这句代码执行了beanfactory中所有BeanFactoryPostProcessor的postProcessBeanFactory方法。



## 五、总结

这篇在我看来重要的还是对BeanPostProcessor、BeanFactoryPostProseccor及Aware的理解，整体难度其实还是不大的，一些不影响理解的代码也没有详细介绍，因此花的时间其实不多，可能会有一些遗漏的地方，还望指正。下一篇会讲讲refresh方法剩下的操作，说多也不多，说少也不少，希望能看到一些有趣的东西。