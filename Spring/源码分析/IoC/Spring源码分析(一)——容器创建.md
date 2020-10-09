# Spring源码分析(一）——容器创建

源码分析基于原文<https://blog.csdn.net/nuomizhende45/article/details/81158383>的基础做一些搬运以及自己的理解，侵删。另外，本文的Spring源码版本有点古老，但是对于spring原理的核心理解基本不会影响。

## 一、概述

Spring的核心理念在于IOC和AOP，IOC，即Inversion Of Control，控制反转，即开发者在开发过程中对服务对象的控制完全依赖于Spring，而不需要自己手动操作。服务对象在Spring中的体现就是Bean，一个Dao可以是Bean，一个Service可以是Bean，一个定时任务可以是Bean，而Spring做的就是将这些Bean的定义（相当于元数据）注册到自己的Bean工厂中，当Bean被需要时，就将其定义从工厂中取出给开发者使用。

## 二、引出关键类

Bean是怎么被注册到工厂中的？工厂又是怎么样的？保留这些问题并从源头开始观察：

![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/1.png)

![img](https://www.javadoop.com/blogimages/spring-context/2.png)

这两张图很复杂，但如果耐心观察可以发现一些有趣的东西：

1.同时出现了**ApplicationContext**；

2.第二张图的顶级接口**BeanFactory**正是之前讨论到的**Bean工厂**。

下面来看一下两个接口的源码，接口嘛，看起来并不复杂的



## 三、BeanFactory&ApplicationContext

### BeanFactory

```java
public interface BeanFactory {
    String FACTORY_BEAN_PREFIX = "&";
	//根据bean name获取bean
    Object getBean(String var1) throws BeansException;
	//根据bean name和class获取bean
    Object getBean(String var1, Class var2) throws BeansException;
	//根据bean name和对象获取bean
    Object getBean(String var1, Object[] var2) throws BeansException;
	//根据bean name判断bean是否存在
    boolean containsBean(String var1);
	//根据bean name 判断bean作用域是否为单例
    boolean isSingleton(String var1) throws NoSuchBeanDefinitionException;
	//根据bean name判断bean是否为prototype，即每次使用都创建新的实例
    boolean isPrototype(String var1) throws NoSuchBeanDefinitionException;
	//判断bean是否满足类型匹配
    boolean isTypeMatch(String var1, Class var2) throws NoSuchBeanDefinitionException;
	//获取bean的类型
    Class getType(String var1) throws NoSuchBeanDefinitionException;
	//获取bean的别名
    String[] getAliases(String var1);
}
```

显而易见，BeanFactory作为Bean的工厂，提供了一些方法来对Bean进行一系列操作，了解Spring的同学应该知道Bean定义通常是由Bean Name或Bean Class Type来进行存储的，因此上面的方法是比较容易理解的，不了解的话也没有关系，可以通过方法名和方法参数来推断，我在代码上做了一些注释，可以帮助理解，由于我对spring的理解也不够深入，因此可能存在一些理解上的偏差，所以不能说完全正确。

另外还有一些细节，如：

```
String FACTORY_BEAN_PREFIX = "&";
```

我认为跳过这个对于当前的可达性分析暂时没有太大的影响，因此以后再介绍吧。



### ApplicationContext

````java
public interface ApplicationContext extends ListableBeanFactory, HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
    ApplicationContext getParent();

    AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;

    String getDisplayName();

    long getStartupDate();
}
````

ApplicationContext可真不是一个单纯的接口，居然继承了这么多其他接口，这里我不会贴代码，但是对于ApplicationContext实现的这些接口，他们一定赋予了其一定的抽象能力，简单概括一下：

**ListableBeanFactory**

为ApplicationContext提供容器中bean迭代的功能,不再需要一个个bean地查找**比如**可以一次获取全部的bean(太暴力了),**或者**根据类型获取一个或多个bean。

**HierarchicalBeanFactory**

是AppllicationContext能够获取到他的父容器，并判断本地工厂是否包含这个Bean（忽略其他所有父工厂）。总的来说，HierarchicalBeanFactory提供了工厂的分层功能

**MessageSource**

一个字，国际化！虽然我不知道国际化到底是什么作用，只知道这能用于支持信息的国际化和包含参数的信息的替换，不过我想了解到这里暂时也够了

**ApplicationEventPublisher**

使ApplicationContext能进行事件的发布

**ResourcePatternResolver**

用于解析资源文件的策略接口，其特殊的地方在于，它应该提供带有*号这种通配符的资源路径。

一下子多了这么多能力，ApplicationContext牛逼！另外，需要注意的是ApplicationContext中还存在一个属性**AutowireCapableBeanFactory**，作为一个BeanFactory，它也足够引起重视——**用来自动装配 Bean**



## 四、Spring！启动！

知道了spring的两大核心类，是时候把这个容器创建出来了，在阅读源码时我的建议是带有目的性的阅读（通关主线任务），在这里即Spring的容器是如何被创建的，因此一些支线任务我会暂时跳过（虽然有时候支线也很重要）

### 1.启动类

```java

public class App {
    public static void main(String[] args) {
        // 用我们的配置文件来启动一个 ApplicationContext
        ApplicationContext context = new ClassPathXmlApplicationContext("classpath:application.xml");
}
```

这段代码又让我想起了初学spring之初的那段时光，现在已经很少接触了，毕竟springboot是真的香……为什么用ClassPathXmlApplicationContext来做解释呢，这是因为看了网上好多文章都是用这个类来讲解的，所以相关的理解思路很多，能帮助自己理解，二来从spring的xml配置文件写过来确实也比较熟悉（但不熟练）了，那么就来看一下ClassPathXmlApplicationContext是如何创建容器的把~



### 2.ClassPathXmlApplicationContext和refresh

```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
      throws BeansException {
 
    super(parent);
    // 根据提供的路径，处理成配置文件数组(以分号、逗号、空格、tab、换行符分割)
    setConfigLocations(configLocations);
    if (refresh) {
      // 核心方法
      refresh(); 
    }
}
```

ClassPathXmlApplicationContext有很多的构造方法，但很容易就可以发现它们都是调来调去的，唯独上面这个构造方法不太一样。super(parent)为可能存在的父容器进行了配置，支线任务就不赘述了。重要的是下面的核心方法：**refresh()**，需要注意ClassPathXmlApplicationContext调的是他的父类AbstractApplicationContext的refresh方法，代码稍微长了点，因为里面有点干货

```java
public void refresh() throws BeansException, IllegalStateException {
   // 来个锁，不然 refresh() 还没结束，你又来个启动或销毁容器的操作，那就乱套了
   synchronized (this.startupShutdownMonitor) {
 
      //准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
      prepareRefresh();
      // 这步比较关键，这步完成后，配置文件就会解析成一个个Bean定义，注册到 BeanFactory 中，
      // 当然，这里说的Bean还没有初始化，只是配置信息都提取出来了，
      // 注册也只是将这些信息都保存到了注册中心(说到底核心是一个 beanName-> beanDefinition 的 map)
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
      // 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
      // 这块待会会展开说
      prepareBeanFactory(beanFactory);
      try {
         // 【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
         // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。】
         // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
         // 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
         postProcessBeanFactory(beanFactory);
         // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 方法
         invokeBeanFactoryPostProcessors(beanFactory);
         // 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
         // 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
         // 两个方法分别在 Bean 初始化之前和初始化之后得到执行。注意，到这里 Bean 还没初始化
         registerBeanPostProcessors(beanFactory);
         // 初始化当前 ApplicationContext 的 MessageSource，国际化这里就不展开说了，不然没完没了了
         initMessageSource();
         // 初始化当前 ApplicationContext 的事件广播器，这里也不展开了
         initApplicationEventMulticaster();
         // 从方法名就可以知道，典型的模板方法(钩子方法)，
         // 具体的子类可以在这里初始化一些特殊的 Bean（在初始化 singleton beans 之前）
         onRefresh();
         // 注册事件监听器，监听器需要实现 ApplicationListener 接口。这也不是我们的重点，过
         registerListeners();
         // 重点，重点，重点
         // 初始化所有的 singleton beans
         //（lazy-init 的除外）
         finishBeanFactoryInitialization(beanFactory);
         // 最后，广播事件，ApplicationContext 初始化完成
         finishRefresh();
      }
      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }
         // Destroy already created singletons to avoid dangling resources.
         // 销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
         destroyBeans();
         // Reset 'active' flag.
         cancelRefresh(ex);
         // 把异常往外抛
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

以下是refresh()中的主线任务：

**主线任务1**：从上面的代码找到**ConfigurableListableBeanFactory**类

**主线任务2**：从文章开头的图片中找到**ConfigurableListableBeanFactory**类

找到之后，可以发现ConfigurableListableBeanFactory继承了ListableBeanFactory和AutowireCapableBeanFactory还有ConfigurableBeanFactory……能继承的居然都继承了，看起来这类有点万能啊……值得庆幸的是，目前**ConfigurableListableBeanFactory**还不需要进去读源码，只需要知道“BeanFactory”以及“万能”这两个关键字，不过往后看一点，关注一下obtainFreshBeanFactory()这个方法，这才是重点,因为……容器就要诞生了！

**主线任务3**：找到obtainFreshBeanFactory(),然后……点进去

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        this.refreshBeanFactory();
        ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
        if (this.logger.isInfoEnabled()) {
            this.logger.info("Bean factory for application context [" + ObjectUtils.identityToString(this) + "]: " + ObjectUtils.identityToString(beanFactory));
        }

        if (this.logger.isDebugEnabled()) {
            this.logger.debug(beanFactory.getBeanDefinitionCount() + " beans defined in " + this);
        }

        return beanFactory;
}
```

obtainFreshBeanFactory方法同样位于AbstractApplicationContext类中，重点关注前两句

```java
this.refreshBeanFactory();
ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
```

先看第一句把，点进去发现是个抽象方法，实现类有2个，这可难倒我了，点哪个呢？……两个都点呗，然后你会发现应该点第一个，也就是**AbstractRefreshableApplicationContext**下的refreshBeanFactory方法，来看代码

```java
protected final void refreshBeanFactory() throws BeansException {
    	//既然是refresh了，那么之前的BeanFactory不要也罢
        if (this.hasBeanFactory()) {
            this.destroyBeans();
            this.closeBeanFactory();
        }

        try {
            //创建BeanFactory
            DefaultListableBeanFactory beanFactory = this.createBeanFactory();
            //定制BeanFactory
            this.customizeBeanFactory(beanFactory);
            //装载Bean定义
            this.loadBeanDefinitions(beanFactory);
            //为beanfactory赋值，为保证线程安全加锁
            synchronized(this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        } catch (IOException var5) {
            throw new ApplicationContextException("I/O error parsing XML document for application context [" + this.getDisplayName() + "]", var5);
        }
}
```

wow！我看到了什么？我看到了createBeanFactory()！点进去看吧

```java
protected DefaultListableBeanFactory createBeanFactory() {
        return new DefaultListableBeanFactory(this.getInternalParentBeanFactory());
}
```

简单粗暴，BeanFactory他来了！好了，这次的主线任务(BeanFactory从无到有)到此为止，下一个主线任务是什么？请看下回分解~~

