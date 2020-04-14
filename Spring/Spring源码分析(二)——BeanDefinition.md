# Spring源码分析(二）

## 一、上文回顾

上节基于可达性分析讲了Spring的BeanFactory的初始化，现在来回顾一下BeanFactory创建过程中关键的方法refresh,另外已经从当前文章开始后续的文章会把源码更新到spring5.x版本，这对于之前的文章不会涉及很大的影响。

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

可以发现其实代码很简单，找到上面的createBeanFactory()方法，它创建了ConfigurableListableBeanFactory，到此BeanFactory就正式创建了

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

所以拍脑袋一想就不难发现，如果BeanFactory中存放的只是一个Bean的定义，那么上述的问题都可以得到解决，BeanFactory只需要在用户需要使用Bean的时候为其创建（当然有些Bean可能需要应用运行时即创建），且能够通过修改Bean定义对其进行增强，这里我不禁要感叹这是多棒的设计啊

这里所说的Bean定义，在Spring对应的即为**BeanDefinition**，在上面的refreshBeanFactory方法中，可以发现，当BeanFactory被创建之后，他会在第一时间做**loadBeanDefinitions**操作，即将Bean定义加载到Bean工厂中，精彩内容要开始了！



## 三、loadBeanDefinitions(DefaultListableBeanFactory beanFactory)

AbstractRefreshableApplicationContext下的loadBeanDefinitions是一个抽象方法，这也很好理解，一个Spring应用可以有很多定义Bean的方法，如xml、注解方式，甚至还可以有远程获取Bean的方法，因此也有很多获取Bean定义的方法，简单看一下重写了**loadBeanDefinitions**方法的类有哪些

![img](http://kylescloud.top/site/pic/loadDefinition.png)

这些类都重写了loadBeanDefinitions方法，不难通过类名判断各个类的意思，本节会先通过AbstractXmlApplicaitonContext来将如何加载bean定义，虽然现在普遍流行的是springboot通过注解形式来加载bean，但是AbstractXmlApplicaitonContext的loadBeanDefinitions方法下有很多值得关注的内容，且相信很多人在新手入门时接触的可能更多的是XML，因此也更容易理解。下面贴源码~在阅读源码之前，不难想到，如果需要从XML文件中读取配置，那么就需要通过I/O来获取文件数据流，带着这个概念看下面的代码就不难理解

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

发现调用了XmlBeanDefinitionReader的构造器的父类构造方法，这里令我感到奇怪的是，为什么参数从ConfigurableListableBeanFactory变成了BeanDefinitionRegistry……后来仔细一看，发现ConfigurableListableBeanFactory原来实现了BeanDefinitionRegistry接口，那么BeanDefinitionRegistry接口为其提供了什么能力？看一下代码吧

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

BeanDefinitionReader通过传入的参数为自己的Bean注册器赋值，这样在读取XML文件后就有了注册Bean的工具，这比较好理解。然后是该方法中也比较重要的两个操作，初始化ResourceLoader&Environment。

首先大致了解什么是ResourceLoader和Environment。其实在两个接口的源码中可以非常清晰的看到他们的功能：

**ResourceLoader：**

用于加载classpath或文件系统下的资源

```
Strategy interface for loading resources (e.. class path or file system resources)
```

**Environment：**

代表了当前应用运行时所处环境，这也不难理解，开发流程中我们难免会在开发环境、测试环境、生产环境之间切换，不同的环境使用不同的profile和不同的properties。当然profile和properties暂时不深究，理解到这里就够了

```
Interface representing the environment in which the current application is running
```



## 四、loadBeanDefinitions(XmlBeanDefinitionReader reader)

在**第三个标题loadBeanDefinitions(DefaultListableBeanFactory beanFactory)**方法中，主要讲解了以下操作

```
XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
```

由于后面的几句都比较好理解，因此不做介绍了，重点关注其调用的重载的loadBeanDefinitions方法

，也就是本标题讨论的内容，点进来看看吧~

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
        Resource[] configResources = this.getConfigResources();
        if (configResources != null) {
            reader.loadBeanDefinitions(configResources);
        }

        String[] configLocations = this.getConfigLocations();
        if (configLocations != null) {
            reader.loadBeanDefinitions(configLocations);
        }
}
```

显而易见，loadBeanDefinitions(XmlBeanDefinitionReader reader)方法通过重载的**loadBeanDefinitions(Resource... resources)**和**loadBeanDefinitions(String... locations)**方法加载了Bean定义，即从配置资源和配置路径下加载Bean定义，分别点进去发现，前者最终跳转到了BeanDefinitionReader的实现类XmlBeanDefinitionReader的以下方法：

```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
}
```

后者则调用了**AbstractBeanDefinitionReader**的**loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources)**，不过这个分支很快通过解析路径转换为 Resource 以后也会进到loadBeanDefinitions(EncodedResource resource)方法中。

由于之后的方法调用链比较长，因此以方法注解的形式呈现，同时省略了不重要的代码



### 1.loadBeanDefinitions(EncodedResource encodedResource)

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		……
		//这里使用了ThreadLocal来存储资源
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
                //这里是关键的步骤，看方法名知道其用来加载Bean定义
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			……
		}
}
```



### 2.doLoadBeanDefinitions(inputSource, encodedResource.getResource())

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
            //将xml文件转换成了Document
			Document doc = doLoadDocument(inputSource, resource);
            //根据document执行bean的注册，重要方法，点进去看看
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			……
		}
}
```



### 3.registerBeanDefinitions(doc, resource)

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    	//创建DocumentReader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    	//获取之前已经注册的bean定义数量
		int countBefore = getRegistry().getBeanDefinitionCount();
    	//根据document注册bean定义，重要，点进去看看
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    	//返回值为新注册的bean定义数量
		return getRegistry().getBeanDefinitionCount() - countBefore;
}
```



### 4.registerBeanDefinitions(doc, createReaderContext(resource))

```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
    	//获取xml的document的树状根节点
		Element root = doc.getDocumentElement();
    	//root及子节点即为需要注册的bean定义，重要，点进去看看
		doRegisterBeanDefinitions(root);
}
```



### 5.doRegisterBeanDefinitions(root)

```java
protected void doRegisterBeanDefinitions(Element root) {
    	/* 我们看名字就知道，BeanDefinitionParserDelegate 必定是一个重要的类，它负责解析 Bean 定义
   		 * 这里为什么要定义一个 parent? 看到后面就知道了，是递归问题，
   		 * 因为 <beans /> 内部是可以定义 <beans /> 的，所以这个方法的 root 其实不一定就是 xml 的根节点，也可以是嵌套在里面的 <beans /> 节点，从源码分析的角度，我们当做根节点就好了
   		 */
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
            // 这块说的是根节点 <beans ... profile="dev" /> 中的 profile 是否是当前环境需要的，
      		// 如果当前环境配置的 profile 不包含此 profile，那就直接 return
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
		preProcessXml(root);
    	//这里的调用很重要，点进去看看
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
}
```



### 6.parseBeanDefinitions(root, this.delegate)

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
            //从这里开始解析节点
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
                        // 解析 default namespace 下面的几个元素
						parseDefaultElement(ele, delegate);
					}
					else {
                        // 解析其他 namespace 的元素
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
}
```

这里要插一句，有关与namespace这里不做过多的介绍，其实不难理解，可以去百度，这里也不影响源码的分析，直接点进parseDefaultElement(ele, delegate)看看



### 7.parseDefaultElement(ele, delegate);

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
            //解析import标签
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
            //解析别名标签
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            //解析bean标签，这是阅读的重点，点进去看看
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// 解析嵌套的bean标签
			doRegisterBeanDefinitions(ele);
		}
}
```



### 8.processBeanDefinition(ele, delegate)

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    	//将解析出来的bean定义存储到一个BeanDefinitionHolder实例中，核心方法，BeanDefinition正是在这个方法中被创建，点进去看到本文标题8-1
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
            //decorate，修饰的意思，可以简单看一眼源码，是对bean定义加了target和一些interceptor，由于这章重点关注的是bean定义如何被注册到工厂中，因此该步骤以后在进行详述
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 注册最终被增强过的bean定义，点进去，看本文标题8-2
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
}
```



### 8-1.parseBeanDefinitionElement(ele)

```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
    	//获取bean id
		String id = ele.getAttribute(ID_ATTRIBUTE);
    	//获取bean name
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		//获取bean 别名集合
		List<String> aliases = new ArrayList<>();
    	//处理bean name字符串
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}
		//若bean id为空，则从别名中取出第一个作为bean name
		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			……
		}
		//校验beanName唯一性
		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}
		//这里BeanDefinition终于出现了！
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
            // 如果都没有设置 id 和 name，那么此时的 beanName 就会为 null，进入下面这块代码产生
			if (!StringUtils.hasText(beanName)) {
				try {
                    //传参进来时containingBean为null，所以不讨论
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
                    //这里是对beanName和aliases做处理，也不是很重要，简单理解一下就行
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
                        String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					……
				}
				catch (Exception ex) {
					……
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
}
```





### 8-2.registerBeanDefinition(bdHolder, getReaderContext().getRegistry())

```java
public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// 获取bean name
		String beanName = definitionHolder.getBeanName();
    	//将bean definition注册到registry中，之前提到过ConfigurableListableBeanFactory实现了BeanRegistry接口，因此即为将bean definition注册到了bean工厂中
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// 注册bean的别名
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
}
```



### 9.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition())

再坚持一下，马上就要完成了！这串代码很长，和我一起深呼吸放松一下

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		……

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				……
			}
		}

		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    	//处理重复名称的 Bean 定义的情况
		if (existingDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
                //若不允许覆盖，这里会抛出异常
				……
			}
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
				//log...用框架定义的 Bean 覆盖用户自定义的 Bean
				if (logger.isWarnEnabled()) {
					……
				}
			}
            // log...用新的 Bean 覆盖旧的 Bean
			else if (!beanDefinition.equals(existingDefinition)) {
				if (logger.isInfoEnabled()) {
					……
				}
			}
            // log...用同等的 Bean 覆盖旧的 Bean，这里指的是 equals 方法返回 true 的 Bean
			else {
				if (logger.isDebugEnabled()) {
					……
				}
			}
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
    	//不存在重名bean
		else {
            // 判断是否已经有其他的 Bean 开始初始化了.
      		// 注意，"注册Bean" 这个动作结束，Bean 依然还没有初始化，后面会有大篇幅说初始化过程，
      		// 在 Spring 容器启动的最后，会 预初始化 所有的 singleton beans
			if (hasBeanCreationStarted()) {
				// 在启动时集合无法再被修改（为了稳定的迭代）
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
            // 最正常的应该是进到这个分支
			else {
				// 将 BeanDefinition 放到这个 map 中，这个 map 保存了所有的 BeanDefinition
				this.beanDefinitionMap.put(beanName, beanDefinition);
				// 这是个 ArrayList，所以会按照 bean 配置的顺序保存每一个注册的 Bean 的名字
                this.beanDefinitionNames.add(beanName);
				// 这是个 LinkedHashSet，代表的是手动注册的 singleton bean，
         		// 注意这里是 remove 方法，到这里的 Bean 当然不是手动注册的
         		// 手动指的是通过调用以下方法注册的 bean ：
         		// registerSingleton(String beanName, Object singletonObject)
         		// 这不是重点，解释只是为了不让大家疑惑。Spring 会在后面"手动"注册一些 Bean，
         		// 如 "environment"、"systemProperties" 等 bean，我们自己也可以在运行时注册 Bean 到容器中的
                this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
}
```



## 五、总结

到了这里，终于创建了BeanDefinition并将其注册到了BeanFactory中，让我们回顾一下，本章的这些操作是从哪里开始的

**1.refresh()**

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			……
		}
}
```

**2.obtainFreshBeanFactory()**

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

**3.refreshBeanFactory()**

```java
protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			……
			loadBeanDefinitions(beanFactory);
			……
		}
		catch (IOException ex) {
			……
		}
	}
```

不难发现，到此refresh中的obtainFreshBeanFactory()方法已经执行结束并返回了，这一过程中进行了BeanFactory的创建，BeanDefinition从xml中的读取、解析、创建、加强、注册等一系列操作，现在回头想想竟然还有一些意犹未尽，interesting~



## 六、预告

下篇要开始写refresh方法的下一级调用了，真是期待啊~