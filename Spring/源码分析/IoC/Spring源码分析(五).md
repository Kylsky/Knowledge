# Spring源码分析(五)

## 一、回顾

上回讲到spring在容器启动时需要创建一些单例bean，创建单例bean的过程实际上就是spring创建bean实例的过程，来回顾一下上次的代码,不用全部看完，找到这次需要讲到的**createBeanInstance(beanName, mbd, args)**方法即可

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
            //createBeanInstance，创建bean实例封装到wrapper中
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
 				 //简单来说就是对mbd做一些加工	
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
		// 注册bean为使用后即销毁
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



## 二、createBeanInstance(beanName, mbd, args)

这个方法一看就很好理解，创建bean实例，发现传入了beanName，bean定义以及相关的参数用于执行构造函数，跳转到这个方法看一下

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// 检查class被load
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
		//检查beanClass
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
		//由supplier提供bean实例
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
		//由工厂方法创建bean实例
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// 这里表示创建已创建过的bean，判断是通过无参构造函数，还是有参构造函数依赖注入来完成实例化
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				return instantiateBean(beanName, mbd);
			}
		}
	
		// 下面是第一次实例化bean的情况，首先判断是否采用有参构造函数
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// 有参构造函数依赖注入并实例化bean
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// 无参构造函数实例化bean
		return instantiateBean(beanName, mbd);
}
```



## 三、bean的实例化

其实可以发现，bean最终的实例化无非就是通过无参构造方法或者有参构造方法创建的即通过autowireConstructor方法和instantiateBean方法，由于前者的代码很长，以后有机会会专门写一篇来讲，这次先来看看后者instantiateBean

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						getInstantiationStrategy().instantiate(mbd, beanName, parent),
						getAccessControlContext());
			}
			else {
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
}
```

不难发现bean是由getInstantiationStrategy().instantiate这一行关键代码实现的，先来看看InstantiationStrategy是何方神圣

```java
public interface InstantiationStrategy {
/**
 * 默认的构造方法
 */
Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner)throws BeansException;

/**
 * 指定构造方法
 */

Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
        Constructor<?> ctor, Object... args) throws BeansException;
        
/**
 * 通过指定的工厂方法
 */
Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
        @Nullable Object factoryBean, Method factoryMethod, Object... args)
        throws BeansException;
```

简单来说，InitiationStrategy提供了一系列通过构造器来初始化bean实例的策略。

挑选默认的构造方法来看看spring是如何实现的把

```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
    // Don't override the class with CGLIB if no overrides.
    //如果没有方法被覆盖,通过BeanUtils#instantiateClass来初始化(实质是利用反射来创建)
    if (!bd.hasMethodOverrides()) {
        Constructor<?> constructorToUse;
        //加锁
        synchronized (bd.constructorArgumentLock) {
            //获取构造方法constructorToUse
            constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse == null) {
                //获取Bean的class的对象
                final Class<?> clazz = bd.getBeanClass();
                //如果clazz是接口类型的,直接抛BeanInstantiationException异常
                if (clazz.isInterface()) {
                    throw new BeanInstantiationException(clazz, "Specified class is an interface");
                }
                try {
                    //在当前系统安全模式的情况下
                    if (System.getSecurityManager() != null) {
                        constructorToUse = AccessController.doPrivileged(
                                (PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
                    }
                    //获取构造方法
                    else {
                        constructorToUse = clazz.getDeclaredConstructor();
                    }
                    //将constructorToUse赋给resolvedConstructorOrFactoryMethod属性
                    bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                }
                catch (Throwable ex) {
                    throw new BeanInstantiationException(clazz, "No default constructor found", ex);
                }
            }
        }
        //通过BeanUtils直接使用构造器对象实例化Bean对象
        return BeanUtils.instantiateClass(constructorToUse);
    }
    else {
        //生成CGLIB创建的子类对象
        return instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```

