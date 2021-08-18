## 1.回顾

上一节讲到了DispatcherServlet的doDispatch方法分发请求的内容，主要涉及了HandlerExecutionChain、HandlerMapping、HandlerAdapter、Handler等类，在请求成功转发到Handler之后，HandlerAdapter就会对其进行处理，在一切开始之前，老规矩，放一下springmvc处理请求的流程图：

![微信图片_20210818135407](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/%25E5%25BE%25AE%25E4%25BF%25A1%25E5%259B%25BE%25E7%2589%2587_20210818135407.jpg)

然后再来看看本节会继续介绍的内容

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ……
    try {
        try {
            ……
            try {
                ……
                //上一节讲的内容就不复述了
                //HandlerAdapter就是在这里处理Handler的
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                if (asyncManager.isConcurrentHandlingStarted()) {
                    return;
                }
                //设置视图
                this.applyDefaultViewName(processedRequest, mv);
                //interceptor的后置处理
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            } catch (Exception var20) {
                dispatchException = var20;
            } catch (Throwable var21) {
                dispatchException = new NestedServletException("Handler dispatch failed", var21);
            }
            //处理请求的结果，处理最终的视图渲染，跳转到标题3
            this.processDispatchResult(processedRequest, response, mappedHandler, mv, (Exception)dispatchException);
        } catch (Exception var22) {
            this.triggerAfterCompletion(processedRequest, response, mappedHandler, var22);
        } catch (Throwable var23) {
            this.triggerAfterCompletion(processedRequest, response, mappedHandler, new NestedServletException("Handler processing failed", var23));
        }
    } finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        } else if (multipartRequestParsed) {
            this.cleanupMultipart(processedRequest);
        }
    }
}
```



## 2.请求的处理

### 2.1 从HandlerAdapter开始

关注上面的代码里的这一行：

```
ha.handle(processedRequest, response, mappedHandler.getHandler());
```

这行代码里HandlerAdapter通过传入request、response、handler进行对请求的处理，HandlerAdapter有许多不同的实现，这里看下对应springmvc的controller相关的实现来看一下：



#### **AbstractHandlerMethodAdapter**#handle

```java
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
    throws Exception {

    return handleInternal(request, response, (HandlerMethod) handler);
}
```

发现最终调用了RequestMappingHandlerAdapter的handleInternal方法，并且Handler被转换成了HandlerMethod实例，这里需要对之前的文章进行**勘误**，前面讲到HandlerExecutionChain中存放的Handler被等价于Controller，其实是不对的，这个Handler应该对应了Controller中的具体的一个接口方法，即这里的HandlerMethod。



### 2.2 RequestMappingHandlerAdapter#handleInternal

```java
protected ModelAndView handleInternal(HttpServletRequest request,
                                      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ModelAndView mav;
    checkRequest(request);

    // 如果需要同步执行，则进入以下代码块
    if (this.synchronizeOnSession) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized (mutex) {
                mav = invokeHandlerMethod(request, response, handlerMethod);
            }
        }
        else {
            mav = invokeHandlerMethod(request, response, handlerMethod);
        }
    }
    else {
        //最终都调用了invokeHandlerMethod方法
        mav = invokeHandlerMethod(request, response, handlerMethod);
    }
	//对响应体做一些处理，这里不作详细介绍
    if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
        if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
            applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
        }
        else {
            prepareResponse(response);
        }
    }
    return mav;
}
```



### 2.3 invokeHandlerMethod

invokeHandlerMethod是一个很长的方法，对于这个方法，关键在于理解**WebDataBinderFactory**、**ModelFactory**、**ServletInvocableHandlerMethod**、**ModelAndViewContainer**这几个类：

#### WebDataBinderFactory

翻译过来就是web数据绑定工厂，顾名思义，其实就是用来做方法上参数的动态绑定的，了解至此即可。



#### ModelFactory

ModelFactory是用来维护Model的(就是ModelAndView的那个Model)，一个ModelFactory包含了**一个ModelMethod的集合**，每个ModelMethod用来存储一个HandlerMethod以及其参数名；**一个SessionAttributeHandler**，用来向session存储属性；**一个BinderFactory**，用于数据绑定



#### ServletInvocableHandlerMethod

ServletInvocableHandlerMethod继承了InvocableHandlerMethod，实际上就是一个HandlerMethod，但是通过能通过invokeAndHandle方法执行方法，因此为请求执行提供了可能。

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201021110631345.png" alt="image-20201021110631345" style="zoom:67%;" />





#### ModelAndViewContainer

比较好理解了，用于存放ModelAndView。



**接下来简单看一下代码**，由于篇幅原因，只注释重要的步骤：

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
                                           HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
	//封装一个ServletWebRequest对象
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        //创建数据绑定工厂
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        //创建model工厂，这里传入了handlerMethod和binderFactory
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
		//将Handler转换成ServletInvocableHandlerMethod
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
           invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
		//创建ModelAndView容器
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        //Model设置属性
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        //将modelFactory中的modelMethods对应的参数进行初始化，并存入mavContainer
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

        AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);

        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.setTaskExecutor(this.taskExecutor);
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        asyncManager.registerCallableInterceptors(this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

        if (asyncManager.hasConcurrentResult()) {
            Object result = asyncManager.getConcurrentResult();
            mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
            asyncManager.clearConcurrentResult();
            LogFormatUtils.traceDebug(logger, traceOn -> {
                String formatted = LogFormatUtils.formatValue(result, !traceOn);
                return "Resume with async result [" + formatted + "]";
            });
            invocableMethod = invocableMethod.wrapConcurrentResult(result);
        }
		//关键在于这句话，执行请求，并处理ModelAndView
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }
		//将请求处理完毕的ModelAndView返回作为结果
        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}
```

关于invokeHandlerMethod里涉及到的一些方法我都没有贴出来，因为调用的方法链路太多了，贴出来反而容易影响思路。而且在我看来上面的代码逻辑已经比较清晰了，处理请求参数，封装ModelAndView，然后通过反射执行特定的方法，将返回结果封装为ModelAndView。现在可以回到doDispatch方法继续往下了~



## 3.结果处理及渲染

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
                                   @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
                                   @Nullable Exception exception) throws Exception {

    boolean errorView = false;
	
    //检查异常，若异常存在，则将其转换为ModelAndView
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        }
        else {
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
            errorView = (mv != null);
        }
    }
    if (mv != null && !mv.wasCleared()) {
        //render用于将ModelAndView渲染到页面上
        render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    }
    else {
        if (logger.isTraceEnabled()) {
            logger.trace("No view rendering, null ModelAndView returned.");
        }
    }
    //如果启动了异步处理则返回
    if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        return;
    }
    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
}
```



## 4.小结

SpringMVC的源码分析大致就到这里，由于时间和精力的关系，可能无法做到面面俱到，一些补充和改进希望能在后续有机会安排，就酱。