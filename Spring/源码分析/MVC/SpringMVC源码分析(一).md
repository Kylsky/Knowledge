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

emmm，这样子看就已经一目了然了，对于SpringMVC来说，需要通过DispatcherServlet对应的上下文初始化以上额外的Bean，这里就不贴上来所有的实现了，因为模式基本相同且很好理解，下面放一下initMultipartResolver的例子

```java
private void initMultipartResolver(ApplicationContext context) {
    try {
        //主要的代码就是这句，从传递过来的上下文中获取bean，其他可以不用看了
        this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Detected " + this.multipartResolver);
        }
        else if (logger.isDebugEnabled()) {
            logger.debug("Detected " + this.multipartResolver.getClass().getSimpleName());
        }
    }
    catch (NoSuchBeanDefinitionException ex) {
        this.multipartResolver = null;
        if (logger.isTraceEnabled()) {
            logger.trace("No MultipartResolver '" + MULTIPART_RESOLVER_BEAN_NAME + "' declared");
        }
    }
}
```

