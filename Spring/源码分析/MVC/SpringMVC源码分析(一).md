## 1.回顾

上一回讲到SpringMVC的关键类DispatcherServlet的初始化及相关容器父子关系的绑定，下面放一下关键代码，关注点在唯一的一行注释下。

```java
protected WebApplicationContext initWebApplicationContext() {
    WebApplicationContext rootContext =
        WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    WebApplicationContext wac = null;

    if (this.webApplicationContext != null) {
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
            if (!cwac.isActive()) {
                if (cwac.getParent() == null) {
                    cwac.setParent(rootContext);
                }
                configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }
    if (wac == null) {
        wac = findWebApplicationContext();
    }
    if (wac == null) {
        wac = createWebApplicationContext(rootContext);
    }

    if (!this.refreshEventReceived) {
        //对IoC容器进行初始化，是十分关键的一步
        synchronized (this.onRefreshMonitor) {
            onRefresh(wac);
        }
    }

    if (this.publishContext) {
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
    }
    return wac;
}
```

对于SpringMVC的关键类DispatcherServlet来说，它的应用上下文与基本的spring容器上下文肯定是不一样的了，由于上篇文章只从流程方面介绍了springmvc启动的流程，没有涉及到mvc容器创建的细节，因此这篇文章来具体分析下



## 2.onRefresh

onRefresh方法位于DispathcerServlet中，直接调用了initStrategies方法

```java
@Override
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}
```

点进initStratrgies方法看看：

```java
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```

emmm，这样子看就已经一目了然了，对于SpringMVC来说，需要通过DispatcherServlet对应的上下文初始化以上额外的内容。接下来简单介绍下，重要的部分会讲下实现。

### 2.1 MultipartResolver

主要用于解析上传的文件



### 2.2 LocaleResolver

用于国际化支持



### 2.3 ThemeResolver

提供动态样式的业务逻辑,这个用的不多。



### 2.4 HandlerMappings

HandlerMapping 的 作 用 是， 为 HTTP 请 求 找 到 相 应 的 Controller 控 制 器， 从 而 利 用 这 些 控 制 器 Controller 去 完 成 设 计 好 的 数 据 处 理 工 作。 HandlerMappings 完 成 对 MVC 中 Controller 的 定 义 和 配 置， 只 不 过 在 Web 这 个 特 定 的 应 用 环 境 中， 这 些 控 制 器 是 与 具 体 的 HTTP 请 求 相 对 应 的。

```java
private void initHandlerMappings(ApplicationContext context) {
    this.handlerMappings = null;

    if (this.detectAllHandlerMappings) {
        // 关键在这里，找到所有上下文中的handlerMapping
        Map<String, HandlerMapping> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerMappings = new ArrayList<>(matchingBeans.values());
            //对HandlerMappings进行排序
            AnnotationAwareOrderComparator.sort(this.handlerMappings);
        }
    }
    else {
        try {
            HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
            this.handlerMappings = Collections.singletonList(hm);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Ignore, we'll add a default HandlerMapping later.
        }
    }

    // 确保至少存在一个handlerMapping
    if (this.handlerMappings == null) {
        this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerMappings declared for servlet '" + getServletName() +
                         "': using default strategies from DispatcherServlet.properties");
        }
    }
}
```



### 2.5 HandlerAdapters

调用具体的方法对用户发来的请求来进行处理。当handlerMapping获取到执行请求的controller时，DispatcherServlet会根据controller对应的controller类型来调用相应的HandlerAdapter来进行处理。



### 2.6 ExceptionResolvers

比较好理解，异常处理器



### 2.7 RequestToViewNameTranslator

根据request请求获取来组装视图名称	



### 2.8 ViewResolvers

视图解析器



### 2.9 FlashMapManager

用来管理FlashMap，FlashMap用于在页面redirect时传递参数



## 3.总结

这一节主要介绍了一下DispatcherServlet的上下文需要初始化的一些对象，由于之后的分析中都会碰到它们，因此在这里先混个眼熟。