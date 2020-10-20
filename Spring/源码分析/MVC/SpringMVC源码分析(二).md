# SpringMVC源码分析(二)

之前讲了WEB容器启动时涉及到的Spring容器及MVC相关的初始化，在初始化完成之后，SpringMVC就能开始处理web请求了，其实呢，SpringMVC底层的实现其实是通过对servlet进行了一层封装来处理web请求，在开始分析代码之前，首先了解下springmvc处理请求的整个流程，只需要看个大概即可。

![springmvcarch](http://kylescloud.top/site/pic/springmvcarch.jpg)

先对各模块的名字熟悉一下，在我看来Spring处理请求的总体流程还是比较经典的java web处理方式，要对整个spring mvc的工作原理和工作流程进行具体的了解，其实随便跑一个项目运行一下debug模式就能进行。这里拿手头的一个项目作为例子

```java
@RequestMapping("/findAll")
public List<InterviewLog> findAll(){
    Iterable<InterviewLog> interviewLogs = esService.findAll();
    List<InterviewLog> logs = new ArrayList<>();
    interviewLogs.forEach(interviewLog -> {
        logs.add(interviewLog);
    });
    return logs;
}
```

这是在一个controller里的一个方法实现，如果在浏览器输入localhost:1234/interview/findAll就能进到该方法，在方法第一行打个断点，启动程序，即可观察整个springmvc的运行流程，浏览器发出请求被服务器接收，服务端会跑起一个线程以执行任务，包括建立三次握手、请求过滤等操作，这些都不细说了，由于本次重点分析的是Spring的MVC，所以一切就从有关servlet的处理开始。开到以下红线处，就从这里讲起吧

<img src="C:\Users\Kyle\AppData\Roaming\Typora\typora-user-images\image-20200507141713761.png" alt="image-20200507141713761" style="zoom:67%;" />



## 一、service

service从ApplicationFilterChain中被internalDoFilter方法调用，当请求过滤链完成之后，ApplicationFilterChain调用自身的Servlet执行service方法，debug中发现，该Servlet通过运行时绑定为DispacherServlet类，这想必是Spring IoC在启动时的配置了。因此，在调用service方法时，通过了以下Servlet实现类的逐级调用：Servlet->HttpServlet->FrameWorkServlet,由于DispacherServlet并没有实现service方法，因此调用链到此为止，来看看这三个类的service方法

### **Servlet**

简洁明了，Servlet作为顶级接口，提供了方法声明

```java
void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;
```

### **HttpServlet**

HttpServlet实际处理逻辑由重载方法service(HttpServletRequest request，HttpServletResponse response)实现

```java
public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
    HttpServletRequest request;
    HttpServletResponse response;
    try {
        request = (HttpServletRequest)req;
        response = (HttpServletResponse)res;
    } catch (ClassCastException var6) {
        throw new ServletException(lStrings.getString("http.non_http"));
    }
    this.service(request, response);
}
```

### **FrameworkServlet**

这里很好理解，FrameworkServlet主要用来处理PATCH方法，其余http方法由上面的HttpServlet重载的service实现

```java
@Override
protected void service(HttpServletRequest request, HttpServletResponse response)
      throws ServletException, IOException {

   HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
   if (httpMethod == HttpMethod.PATCH || httpMethod == null) {
      processRequest(request, response);
   }
   else {
      super.service(request, response);
   }
}
```

下面来看看主要的重载的service(HttpServletRequest request，HttpServletResponse response)如何实现吧：

```java
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    //获取方法类型名称
    String method = req.getMethod();
    //用于判断是否缓存了该请求
    long lastModified;
    //GET进到这里
    if (method.equals("GET")) {
        lastModified = this.getLastModified(req);
        //没有缓存过，直接处理
        if (lastModified == -1L) {
            this.doGet(req, resp);
        } else {
            //缓存过，则需判断请求头中的If-Modified-Since，若该信息不存在，则
            long ifModifiedSince;
            try {
                ifModifiedSince = req.getDateHeader("If-Modified-Since");
            } catch (IllegalArgumentException var9) {
                ifModifiedSince = -1L;
            }
			//把ifModifiedSince与服务器上实际文件的最后修改时间进行比较。
            if (ifModifiedSince < lastModified / 1000L * 1000L) {
                //说明被修改过，则需要执行请求，同时尝试更新Last-Modified请求头到respons
                this.maybeSetLastModified(resp, lastModified);
                this.doGet(req, resp);
            } else {
                //说明没有被修改，返回304，让浏览器使用缓存填充视图
                resp.setStatus(304);
            }
        }
    } 
    //后面的http方法由于都不是幂等操作，因此不会考虑使用浏览器缓存
    else if (method.equals("HEAD")) {
        lastModified = this.getLastModified(req);
        this.maybeSetLastModified(resp, lastModified);
        this.doHead(req, resp);
    } else if (method.equals("POST")) {
        this.doPost(req, resp);
    } else if (method.equals("PUT")) {
        this.doPut(req, resp);
    } else if (method.equals("DELETE")) {
        this.doDelete(req, resp);
    } else if (method.equals("OPTIONS")) {
        this.doOptions(req, resp);
    } else if (method.equals("TRACE")) {
        this.doTrace(req, resp);
    } else {
        //处理异常http方法，返回501
        String errMsg = lStrings.getString("http.method_not_implemented");
        Object[] errArgs = new Object[]{method};
        errMsg = MessageFormat.format(errMsg, errArgs);
        resp.sendError(501, errMsg);
    }
}
```



## 二、doGet&processRequest

由于在项目中调用的接口使用了Get方法，所以这里就挑doGet方法进去看看

```java
@Override
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
      throws ServletException, IOException {

   processRequest(request, response);
}
```

比较有趣，只有一行代码，进入到了同在FrameworkServlet的processRequest方法中，来看一下：

```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
      throws ServletException, IOException {
   //记录开始时间
   long startTime = System.currentTimeMillis();
   //定义并初始化错误报告
   Throwable failureCause = null;
   //下面的Locale是指一些本地信息，不是很重要，与后面的request相同，两者都被存储在ThreadLocal中，由于filter可能对request做相关操作，所以这里分为previous和非previous两部分
   //拿到LocaleContext上下文
   LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
   //用request创建一个Local的上下文
   LocaleContext localeContext = buildLocaleContext(request);
   //获取request
   RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
   ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);
   //获取异步管理器，关于异步管理器其实我有点一知半解
   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
   //为异步管理器注册回调的拦截器
   asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());
   //传入三个参数，初始化上下文，说白了就是把locale和request放到Holder里
   initContextHolders(request, localeContext, requestAttributes);
   try {
      //这里是关键的一步
      doService(request, response);
   }
   catch (ServletException | IOException ex) {
      failureCause = ex;
      throw ex;
   }
   catch (Throwable ex) {
      failureCause = ex;
      throw new NestedServletException("Request processing failed", ex);
   }
   finally {
      //清空ContextHolders
      resetContextHolders(request, previousLocaleContext, previousAttributes);
      if (requestAttributes != null) {
         requestAttributes.requestCompleted();
      }
      logResult(request, response, failureCause, asyncManager);
      //无论请求是否正确处理，都会产生一个请求处理事件
      publishRequestHandledEvent(request, response, startTime, failureCause);
   }
}
```

processRequest在我看来其实主要的工作分为三部分：

1.处理locale和request，将他们初始化到ContextHolder中以便后续对Locale和Request进行获取

2.执行请求，这步交给了doService去做

3.请求执行完毕后，清空ContextHolder，并发布事件

接下来主要看看doService是怎么处理请求的：



## 3.doService

进入到FrameworkServlet的doService方法，发现其实是一个模板方法

```java
protected abstract void doService(HttpServletRequest request, HttpServletResponse response)
      throws Exception;
```

不难想到，是时候进入dispatcherServlet的工作时间了：

```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
   // 记录日志
   logRequest(request);
   //如果请求uri属于include类型的，则把请求参数存储在一个Map中，当作一个快照 
   Map<String, Object> attributesSnapshot = null;
   if (WebUtils.isIncludeRequest(request)) {
      attributesSnapshot = new HashMap<>();
      Enumeration<?> attrNames = request.getAttributeNames();
      while (attrNames.hasMoreElements()) {
         String attrName = (String) attrNames.nextElement();
         if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
            attributesSnapshot.put(attrName, request.getAttribute(attrName));
         }
      }
   }

   // 设置一些有用的属性，方便后续开发
   request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
   request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
   request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
   request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
   //flashMapManager主要处理的是重定向的逻辑，如果是重定向的请求，则加入以下的属性
   if (this.flashMapManager != null) {
      FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
      if (inputFlashMap != null) {
         request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
      }
      request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
      request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
   }

   try {
      doDispatch(request, response);
   }
   finally {
      //判断请求是否为异步起动
      if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
         // 若请求为include，则在请求后重新存储快照
         if (attributesSnapshot != null) {
            restoreAttributesAfterInclude(request, attributesSnapshot);
         }
      }
   }
}
```

可以看到doService方法主要是在request上添加了一些属性，随后使用doDispatch方法将请求分发，请求分发就留在下次写了~