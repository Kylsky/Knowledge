## 1.回顾

![springmvcarch](http://kylescloud.top/site/pic/springmvcarch.jpg)

上一次讲到了请求进入DispatcherServlet的doService方法中执行一系列操作，最终会进入**doDispatch(request, response)**方法开始执行请求的转发，这一节就从这里开始继续。

先简单看一下上图，有一定开发经验的人都了解，DispatcherServlet会将请求转发到Controller中进行处理，按照途中流程来看，其实就是第二步到第五步，这几步中涉及到了一些十分关键的类，如HandlerMapper、HandlerExcutorChain等，会在后面一步步分析与展开。



## 2.doDispatch之前

在上节提到的doDispatch方法，其实就是整个springmvc的关键所在，在看代码之前，有些类必须要来回顾或认实一下

### 2.1 HandlerMapping

这个类在源码分析(一)介绍过——HandlerMapping 的 作 用 是， 为 HTTP 请 求 找 到 相 应 的 Controller 控 制 器， 从 而 利 用 这 些 控 制 器 Controller 去 完 成 设 计 好 的 数 据 处 理 工 作。 HandlerMappings 完 成 对 MVC 中 Controller 的 定 义 和 配 置。也就是说，如果想让一个请求走向正确的Controller，那么HandlerMapping一定是分发请求所必须的。



### 2.2 HandlerAdapter

看类名，顾名思义，通过适配器模式对Handler进行适配服务，简单来说，就是处理Handler的，Handler其实指的就是Controller了，因此当handlerMapping获取到执行请求的handler(controller)时，DispatcherServlet会根据handler对应的类型来调用相应的HandlerAdapter来进行处理。



### 2.3 Handler

这里暂时不做过多的介绍，理解成Controller即可。



### 2.4 HandlerExcutionChain

这是一个比较有意思的类，简单贴一下它里面的属性

```java
private static final Log logger = LogFactory.getLog(HandlerExecutionChain.class);
private final Object handler;
@Nullable
private HandlerInterceptor[] interceptors;
@Nullable
private List<HandlerInterceptor> interceptorList;
private int interceptorIndex;
```

不难发现，一个HandlerExcutionChain包含了一个Handler，以及一系列的HandlerInterceptor，其实可以理解成每一个Handler都被封装在一个Chain中，当这个handler被执行时，会有一系列的Interceptor拦截器来对其进行一些附加操作与处理。



## 3.doDispatch

有了上面的一些类的认识，进来看doDispatch的代码就不会太懵了。

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    //预定义一个HandlerExecutionChain
    HandlerExecutionChain mappedHandler = null;
    //用来判断是否为文件上传类的请求
    boolean multipartRequestParsed = false;
    //异步管理器
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        try {
            //预定义请求的返回值，这个也挺重要，会在后续讲
            ModelAndView mv = null;
            //预定义异常
            Object dispatchException = null;

            try {
                //检查请求类型
                processedRequest = this.checkMultipart(request);
                multipartRequestParsed = processedRequest != request;
                //这一步很重要，点进去看一下，代码分析见3.1
                mappedHandler = this.getHandler(processedRequest);
                if (mappedHandler == null) {
                    this.noHandlerFound(processedRequest, response);
                    return;
                }
                //到此，HandlerExecutionChain已经获取成功了
				//接下来是获取HandlerAdapter，它是真正处理Handler的家伙，代码分析见3.2
                HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());
                String method = request.getMethod();
                //get方法的last-modified请求头判断，不重要，就略过了
                ……

                //使用interceptors进行preHandle预处理
                if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }
                //哈哈哈，看到没，HandlerAdapter就是在这里处理Handler的
                //由于这个方法以及下一部分的代码涉及到请求的处理和返回值的封装
                //就留在下一节再详细展开了
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
            //处理请求分发的结果
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



### 3.1 getHandler(processedRequest)

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    //一般来说，HandlerMapping在容器启动时已经预初始化
    if (this.handlerMappings != null) {
        Iterator var2 = this.handlerMappings.iterator();
        while(var2.hasNext()) {
            HandlerMapping hm = (HandlerMapping)var2.next();
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Testing handler map [" + hm + "] in DispatcherServlet with name '" + this.getServletName() + "'");
            }
			//这里通过HandlerMapping获取Handler，如果不为空，则封装为HandlerExecutionChain，这行代码很重要，点进去看下
            HandlerExecutionChain handler = hm.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
```

#### AbstractHandlerMapping::getHandler(HttpServletRequest request)

为了保证文章可读性，一些不重要的代码就略过了

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    	//这个不做过多介绍，可以看一下AbstractUrlHandlerMapping的方法实现，理解起来应该不困难
        Object handler = this.getHandlerInternal(request);
        ……
        if (handler == null) {
            return null;
        } else {
            if (handler instanceof String) {
                String handlerName = (String)handler;
                //handler为string类型，则通过容器获取bean
                handler = this.obtainApplicationContext().getBean(handlerName);
            }
			//关键的代码，通过handler和request封装int点进去看下CHainhain,点进去看下
            HandlerExecutionChain executionChain = this.getHandlerExecutionChain(handler, request);
            ……
            return executionChain;
        }
    }
```

#### AbstractHandlerMapping::getHandlerExecutionChain()

```java
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
        HandlerExecutionChain chain = handler instanceof HandlerExecutionChain ? (HandlerExecutionChain)handler : new HandlerExecutionChain(handler);
        String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
    	//获取当前HandlerMapping下的所有interceptor
        Iterator var5 = this.adaptedInterceptors.iterator();
		//遍历interceptor
        while(var5.hasNext()) {
            HandlerInterceptor interceptor = (HandlerInterceptor)var5.next();
            if (interceptor instanceof MappedInterceptor) {
                MappedInterceptor mappedInterceptor = (MappedInterceptor)interceptor;
                //若interceptor满足pathMatcher，则将其添加到HandlerExecutionChain中
                if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                    chain.addInterceptor(mappedInterceptor.getInterceptor());
                }
            } else {
                chain.addInterceptor(interceptor);
            }
        }
		//返回HandlerExecutionChain
        return chain;
    }
```

到这里HandlerExecutionChain就能顺利返回了，回到上面的doDispatch方法本体吧~



### 3.2 getHandlerAdapter(mappedHandler.getHandler())

getHandlerAdapter就相对好理解了，由于HandlerAdapters也是在DispatcherServlet初始化上下文时就实例化了，以此通过supports方法判断是否为需要的HandlerAdapter。

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        Iterator var2 = this.handlerAdapters.iterator();

        while(var2.hasNext()) {
            HandlerAdapter ha = (HandlerAdapter)var2.next();
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Testing handler adapter [" + ha + "]");
            }
            //看这里
            if (ha.supports(handler)) {
                return ha;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler + "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

HandlerAdapter的实现类不多，看看下面这个：

![image-20201020100121440](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201020100121440.png)

#### SimpleControllerHandlerAdapter::supports(Object handler)

```java
public boolean supports(Object handler) {
    return handler instanceof Controller;
}
```

so easy，继续回到doDispatch方法吧



## 4.WebAsyncManager

在结束doDispatch之前，还有一个点需要介绍一下——WebAsyncManager。从名字上来看，大致可以理解为网络异步管理器，来看下这个类在doDispatch方法中起到的作用

### isConcurrentHandlingStarted()

```
Whether the selected handler for the current request chose to handle the request asynchronously. A return value of "true" indicates concurrent handling is under way and the response will remain open. A return value of "false" means concurrent handling was either not started or possibly that it has completed and the request was dispatched for further processing of the concurrent result.
```

翻译过来就是：

为当前请求选择的处理程序是否为异步。返回值“true”表示正在进行并发处理，并且响应将保持打开状态。返回值为“false”意味着并发处理没有启动，或者可能已经完成，请求被分派以进一步处理并发结果。

也就是说，该方法用来判断当前请求的异步处理是否已经启动，如果返回true，则说明请求已经在处理了，因此在doDispatch方法中检测到true会将request直接return。



## 5.总结

本节主要分析了doDispatch方法的请求分发相关的内容，由于代码的跳转比较多，所以可能会比较繁琐，但是总体来说逻辑还是比较清晰的。下一节再分析请求的处理及后续的内容吧~