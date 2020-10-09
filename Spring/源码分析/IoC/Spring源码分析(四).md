# Spring源码分析(四)

## 一、回顾

上回讲到……讲到哪了……

先贴代码回顾一下，找到其中唯一一行注释吧！

```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		prepareRefresh();
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
		prepareBeanFactory(beanFactory);
		try {
			postProcessBeanFactory(beanFactory);
			invokeBeanFactoryPostProcessors(beanFactory);
			initMessageSource();
			initApplicationEventMulticaster();
			onRefresh();
			registerListeners();
            //恭喜你找到了注释，是的，上次讲到这儿了
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
上回讲到，在BeanFactory创建和BeanDefinitions注册完成后，spring对BeanFactory进行了一些相关的后续操作，如执行BeanFactoryPostProcessors的方法，注册一些BeanPostProcessors，设置事件广播器等等。今天来讲一下重头戏——**finishBeanFactoryInitialization(beanFactory)**，一起点进去看看吧。



## 二、finishBeanFactoryInitialization(beanFactory)

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// 初始化 conversion service ，不展开
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}
		// Resolver相关，不展开
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}
		// AspectJ相关，不展开
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}
		// 停止使用暂时的类加载器
		beanFactory.setTempClassLoader(null);
		// 冻结配置，防止在初始化bean过程中出现干扰
		beanFactory.freezeConfiguration();
		// 初始化所有单例bean，最关键的一步，进去看看
		beanFactory.preInstantiateSingletons();
}
```



## 三、preInstantiateSingletons()

由于代码比较长，所以忽略一些无关紧要的代码，在看代码之前，先想一想，为什么BeanFactory需要初始化单例Bean，有哪些单例Bean是需要在一开始就被初始化的？带着这样的问题来看源码，会有更清晰的认知。

```java
public void preInstantiateSingletons() throws BeansException {
   // 这里获取到了工厂里的所有的beanNames
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

   // 要开始从万花丛中找单例的beanDefinition了
   for (String beanName : beanNames) {
   	  //获取BeanDefinition
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
      //找出非抽象非懒加载的单例beanDefinition
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
      	 //这里出现了一个新的概念，FactoryBean，很重要，之后讲
         if (isFactoryBean(beanName)) {
            //对bean name加工，用于区分普通bean和factory bean
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
            if (bean instanceof FactoryBean) {
               final FactoryBean<?> factory = (FactoryBean<?>) bean;
               //判断当前FactoryBean是否是SmartFactoryBean的实现，这里不影响后续的阅读，可以跳过，因此省略了
               boolean isEagerInit;
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                  ......
               }
               else {
                   ......
               }
               //getBean方法，这是重点
               if (isEagerInit) {
                  getBean(beanName);
               }
            }
         }
         //若当前的beanDefinition不是factory bean，则直接调用getBean
         else {
            getBean(beanName);
         }
      }
   }

   // 到这里为止，所有的非懒加载的singleton beans已经完成了初始化
   // 如果我们定义的单例bean是实现了SmartInitializingSingleton接口，那么当他们在初始化后会在这里得到回调，就不展开了
   for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton) {
         final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
         if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
               smartSingleton.afterSingletonsInstantiated();
               return null;
            }, getAccessControlContext());
         }
         else {
            smartSingleton.afterSingletonsInstantiated();
         }
      }
   }
}
```

上面的代码虽然冗长，但其实可以发现其实初始化单例bean的关键就在于getBean方法，其实也可以说，初始化所有bean的关键就在于getBean,因为在这里只不过是对bean的scope做了筛选。

在对getBean做了解之前，先看看先前讲到的FacotryBean是怎么回事



## 四、FactoryBean

将FactoryBean翻译成中文，就是工厂Bean，也就是说，这个Bean在初始化以后会变成一个工厂，这样就很好理解了~那么这个工厂Bean具体是干什么的？工厂用来生产什么东西？来看看这个类的注释

```java
/* Interface to be implemented by objects used within a {@link BeanFactory} which
 * are themselves factories for individual objects. If a bean implements this
 * interface, it is used as a factory for an object to expose, not directly as a
 * bean instance that will be exposed itself.
 *
 * <p><b>NB: A bean that implements this interface cannot be used as a normal bean.</b>
 * A FactoryBean is defined in a bean style, but the object exposed for bean
 * references ({@link #getObject()}) is always the object that it creates.
 */
```

虽然我也看不懂英语，但是不代表我不能翻译，简单来讲，FactoryBean能够生产并暴露独特的Bean，而不是直接将FactoryBean本身暴露出来，这句话有一点抽象，你可以理解成政府（BeanFactory）建立（初始化）了一家棒冰厂家（FactoryBean），用来生产棒冰给员工（开发者）降暑，你得知道员工需要的只是棒冰而并不是厂家本身。

因此，如果一个类实现了FactoryBean，那么你就不能把他当作一个普通Bean来使用，此时打开你的脑洞，想象FactoryBean可以从一个其他的远程项目中获取一个对象来作为Bean实例返回，或者更夸张一些，远程获取的Bean甚至可以是一个服务，只需调用这个服务，便可以让自己的项目功能瞬间完成，这让你又多了一个小时额外的自主学习（摸鱼）时间，岂不快哉？

好了好了，到此为止，是时候看看getBean了，前面提到getBean是初始化Bean的关键



## 五、getBean(beanName)

getBean方法位于AbstractBeanFactory中，该类提供了很多该方法的重载，不过功能其实是一样的——获取bean实例，来看看这次的主角**getBean(String beanName)**

```java
public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
}
```

跳转到doGetBean,做好心理准备……不过也别担心，按着注释一步步来，不会有什么难度

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
		//传来的name可能是带$的FactoryBean或者别名，要转换一下，不展开了
		final String beanName = transformedBeanName(name);
		//该bean用来作为返回值，关注一下	
    	Object bean;

		// 首先检查是否为已经手动创建的单例bean
		Object sharedInstance = getSingleton(beanName);
    	//若是已创建的单例bean，则会判断args，args不为空表示创建一个新的bean并返回，比如让FactoryBean返回一个新的bean
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					......
				}
				else {
					......
				}
			}
            //赋值给bean
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		//若是未创建的，则进入下面的逻辑
		else {
			// 正在创建此 beanName 的 prototype 类型的 bean，那么抛异常，这往往是因为陷入了循环引用
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 检查工厂中是否存在beanDefinition
			BeanFactory parentBeanFactory = getParentBeanFactory();
            //若当前工厂没有，检查父工厂
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				String nameToLookup = originalBeanName(name);
				//父工厂为AbstractBeanFactory实例，让父工厂返回bean
                if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
                //父工厂不是AbstractBeanFactory实例，且args不为空
				else if (args != null) {
					// 传入args让父工厂代理产生bean
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
                //两个条件都不满足，代理到普通的getBean方法
				else {
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}
			//typeCheckOnly是传参为false，这里会进入到if语句中
			if (!typeCheckOnly) {
                //标记当前bean已经被创建
				markBeanAsCreated(beanName);
			}
            
			//如果不需要父工厂处理，当前工厂就自己处理
			try {
                //获取bean定义，由于bean中很可能有依赖项，所以管他叫MergedLocalBeanDefinition
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                //检查是否为abstract，很简单不展开
				checkMergedBeanDefinition(mbd, beanName, args);

				// 获取当前bean所依赖的bean
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							//循环引用抛出异常
                            ...... 
						}
                        //注册依赖关系，后面会展开，可以先往下看
						registerDependentBean(dep, beanName);
						try {
                            //实例化该依赖
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 当bean属于普通单例，则实例化该单例
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
                        //这里真正创建了Bean，很重要，之后展开
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// 删除缓存中的Bean
							// 同时删除所有接收了当前Bean的临时引用的Bean
							destroySingleton(beanName);
                            //抛出异常
							throw ex;
						}
					});
                    //若创建Bean成功，则为bean赋值，前面讲到bean用来作为返回值
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				//如果bean属于prototype
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
                        //在创建bean之前的操作，比较简单，不展开
						beforePrototypeCreation(beanName);
                        //创建bean
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
                        //和boefore操作差不都，也不展开
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
				//bean属于其他域，操作和前面差不多，这块就省略了
				else {
					......
				}
			}
			catch (BeansException ex) {
                //若bean产生失败，则进行清理，并抛出异常，较简单，就是在表示已创建bean的Set集合中将当前beanName移除
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// requiredType是要检索的bean的所需类型，若不为空，且类型匹配，则进行类型转换
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
    //返回Bean
		return (T) bean;
}
```

值得一提的是spring加载bean的方式其实很有趣，当子容器在当前beanDefinitionMap中找不到beanDefinition时就会把任务丢给父容器去做，如果父容器找不到，还会丢给他的父容器去找，实在是有点坑爹的嫌疑，但其实这个递归的调用却又很巧妙，每个容器只要做好自己分内的事即可，将不属于自己的任务向上抛出，层层向上处理，真是妙哉。下面是getBean方法中一些重要的相关操作，可以对着上面的代码找到对应的实现



## 六、registerDependentBean(dep, beanName)

在初始化一个bean时，需要将bean内的依赖与当前bean注册一个依赖关系，来看看spring为什么要注册依赖关系，并且是怎么做的。在下面的代码中找到**dependentBeanMap**和**dependenciesForBeanMap**，这是注册关系的关键。

找到两者在源码中的对应位置，看看他们是什么样的

**dependentBeanMap**

```java
/** Map between dependent bean names: bean name --> Set of dependent bean names */
	private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);
```

这里存储的是被依赖者与依赖者的关系，比如User中有一项依赖叫Car，那么dependentBeanMap存储的就是car:user



**dependenciesForBeanMap**

```java
/** Map between depending bean names: bean name --> Set of bean names for the bean's dependencies */
private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);
```

这里就是和上面的反一反，user:car

现在就来看看代码把~

```java
/**
	 * Register a dependent bean for the given bean,
	 * to be destroyed before the given bean is destroyed.
	 * @param beanName the name of the bean
	 * @param dependentBeanName the name of the dependent bean
	 */
public void registerDependentBean(String beanName, String dependentBeanName) {
		String canonicalName = canonicalName(beanName);

		synchronized (this.dependentBeanMap) {
            //若被依赖项的依赖关系不存在，则创建一个
			Set<String> dependentBeans =
		this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
            //添加依赖关系，若beanName：dependentBeanName已存在，则return
			if (!dependentBeans.add(dependentBeanName)) {
				return;
			}
		}

		synchronized (this.dependenciesForBeanMap) {
            //若被依赖项的依赖关系不存在，则创建一个
			Set<String> dependenciesForBean =
this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
            //添加依赖关系
			dependenciesForBean.add(canonicalName);
		}
}
```

其实可以从方法上的注释即可知道：“to be destroyed before the given bean is destroyed”，即“被依赖项会在给定的bean被销毁之前被销毁”，spring这么做的目的就很清晰了——为了GC。

再来讨论一下里面的这一句：

```java
//添加依赖关系，若beanName：dependentBeanName已存在，则return
if (!dependentBeans.add(dependentBeanName)) {
    return;
}
```

为什么如果被依赖项与依赖项的关系已建立，就不用再建立依赖项与被依赖项的关系？……这听起来很拗口，但确实是这么回事儿。想知道答案可以去同一个类中registerContainedBean方法去看看就能明白啦~好了，现在回到**五**继续看后面的代码把



## 七、createBean(beanName, mbd, args)

第三个参数 args 数组代表创建实例需要的参数，就是给构造方法用的参数，或者是工厂 Bean 的参数。不过要注意的是在初始化阶段，args 是 null。createBean的实现在AbstractAutowireCapableBeanFactory类中

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		......
		RootBeanDefinition mbdToUse = mbd;

		// 确保 BeanDefinition 中的 Class 被加载
		// 同时在动态解析类的情况下克隆bean定义提供使用，这个bean定义不能被存储在mbd中
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// 准备当前bean的lookupMethod和replacedMathoed等，可以参考https://www.cnblogs.com/ViviChan/p/4981619.html
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			//抛出异常
            ......
		}
		try {
			// 使BeanPostProcessors有机会返回一个代理bean
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			//抛出异常
            ......
		}
		try {
            //doCreateBean才是关键，后面会展开
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			//抛出异常
            ......
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			//抛出异常
            ......
		}
		catch (Throwable ex) {
			//抛出异常
            ......
		}
}
```



## 八、doCreateBean(beanName, mbdToUse, args)

又是一长串代码，真的难顶了，不过我想spring这么做也是有他的苦衷的，给人家一个面子继续看下去吧。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// BeanWrrapper，Bean包装器，简单理解成对bean赋值的，后面展开
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
            //若mbd为单例，需要先判断是否为FactoryBean
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
    	//如果instanceWrapper为空，则创建一个bean实例
		if (instanceWrapper == null) {
            //createBeanInstance，创建bean实例封装到wrapper中，后面展开
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
    	//通过wrapper获取到了bean
		final Object bean = instanceWrapper.getWrappedInstance();
    	//处理bean类型
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}
		// 允许post-processors修改mbd
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
 				 //简单来说就是使用BeanPostProccessors对mbd做一些加工	
                 applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					//抛出异常
                    ......
				}
				mbd.postProcessed = true;
			}
		}
		// 解决循环依赖的问题
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			......
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
		// bean在实例化后，还需要赋值和初始化(init)
		Object exposedObject = bean;
		try {
            //这一步负责属性装配，关键就在于BeanWrapper
			populateBean(beanName, mbd, instanceWrapper);
            //处理 bean 初始化完成后的各种回调
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			//抛出异常
            ......
		}
		//主要用于解决循环引用，这部分先跳过吧
		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						//抛出异常
                        ......
					}
				}
			}
		}
		// 判断注册bean是否需要使用后即销毁
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			//抛出异常
            ......
		}
		return exposedObject;
}
```

doCreateBean方法真是细节拉满，分别来看看以下关键的几步

### **1.BeanWrapper**

可以发现createBeanInstance方法虽然创建了bean实例，但是返回的其实是一个BeanWrapper，他到底是个什么东西？

其实BeanWrapper相当于一个容器，Spring委托BeanWrapper完成Bean属性的填充工作。在Bean实例被创建出来之后，容器主控程序将Bean实例通过BeanWrapper包装起来，这是通过调用BeanWrapper的setWrappedInstance方法完成的。



### **2.createBeanInstance(beanName, mbd, args)**

这里是创建bean实例的关键，由于考虑后续代码依旧有一大把，我的心态实在受到了影响，就放到下一次讨论吧……



### 3.populateBean(beanName, mbd, instanceWrapper)

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   //若beanwrapper为空，则需要判断是否需要为bean赋值
   if (bw == null) {
      //有属性且wrapper为空，则会抛出异常，原因不言而喻
      if (mbd.hasPropertyValues()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      //没有属性，那就不需要赋值啦！直接返回
      else {
         return;
      }
   }

   // 到这步的时候，bean 实例化完成（通过工厂方法或构造方法），但是还没开始属性设值，InstantiationAwareBeanPostProcessor 的实现类可以在这里对 bean进行状态修改
   boolean continueWithPropertyPopulation = true;

   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 如果返回 false，代表不需要进行后续的属性设值，也不需要再经过其他的 BeanPostProcessor 的处理
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               continueWithPropertyPopulation = false;
               break;
            }
         }
      }
   }
   //若不需要再赋值，则return
   if (!continueWithPropertyPopulation) {
      return;
   }
   //获取propertyValues
   PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

   if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
         mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
	  //下面的代码能知道干了啥就行，不是(就是)因为我实力不够，谢谢
      // 通过名字找到所有属性值，如果是 bean 依赖，先初始化依赖的 bean。记录依赖关系
      if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
         autowireByName(beanName, mbd, bw, newPvs);
      }
      // 通过类型装配
      if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
         autowireByType(beanName, mbd, bw, newPvs);
      }
      pvs = newPvs;
   }
   //是否包含InstantiationAwareBeanPostProcessors
   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
   //是否需要深度检查
   boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

   if (hasInstAwareBpps || needsDepCheck) {
      if (pvs == null) {
         pvs = mbd.getPropertyValues();
      }
      PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      if (hasInstAwareBpps) {
         for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
               InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
               //使用postProcessor对propertyValues进行处理
               pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvs == null) {
                  return;
               }
            }
         }
      }
      if (needsDepCheck) {
         //检查依赖
         checkDependencies(beanName, mbd, filteredPds, pvs);
      }
   }

   if (pvs != null) {
      // 设置 bean 实例的属性值，这里就不展开了，到此，bean实例的赋值就完成了
      applyPropertyValues(beanName, mbd, bw, pvs);
   }
}
```



## 九、结论

写了这么多，bean实例的创建还是要被移到后面了，不过这次的收获也不小，从refresh加载单例bean其实可以窥见spring是如何加载所有bean的，其中也包括了一些postProcessor的实际应用等等。由于代码真的很多，所以觉得还是需要花时间消化消化的。

希望后续的代码能善待我吧，阿门！