## 1.概论

Spring IoC 是 一 个 独 立 的 模 块， 它 并 不 是 直 接 在 Web 容 器 中 发 挥 作 用 的， 如 果 要 在 Web 环 境 中 使 用 IoC 容 器， 需 要 Spring 为 IoC 设 计 一 个 启 动 过 程， 把 IoC 容 器 导 入， 并 在 Web 容 器 中 建 立 起 来。 具 体 说 来， 这 个 启 动 过 程 是 和 Web 容 器 的 启 动 过 程 集 成 在 一 起 的。 在 这 个 过 程 中， 一 方 面 处 理 Web 容 器 的 启 动， 另 一 方 面 通 过 设 计 特 定 的 Web 容 器 拦 截 器， 将 IoC 容 器 载 入 到 到 Web 环 境 中 来， 并 将 其 初 始 化。 在 这 个 过 程 建 立 完 成 以 后， IoC 容 器 才 能 正 常 工 作， 而 Spring MVC 是 建 立 在 IoC 容 器 的 基 础 上 的， 这 样 才 能 建 立 起 MVC 框 架 的 运 行 机 制， 从 而 响 应 从 Web 容 器 传 递 的 HTTP 请 求。

![image-20201013153236709](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201013153236709.png)

上图为tomcat的web.xml对应内容，关注点主要放在**DispatcherServlet**和**ContextLoaderListener**上，DispathcerServlet作为SpringMVC的请求分派器已经熟知。 ContextLoaderListener 被 定 义 为 一 个 监 听 器，其实是SpringMVC的启动类， 这 个 监 听 器 是 与 Web 服 务 器 的 生 命 周 期 相 关 联 的， 由 ContextLoaderListener 监 听 器 负 责 完 成 IoC 容 器 在 Web 环 境 中 的 启 动 工 作。

DispatchServlet 和 ContextLoaderListener 提 供 了 在 Web 容 器 中 对 Spring 的 接 口， 也 就 是 说， 这 些 接 口 与 Web 容 器 耦 合 是 通 过 ServletContext 来 实 现 的。 这 个 ServletContext 为 Spring 的 IoC 容 器 提 供 了 一 个 宿 主 环 境， 在 宿 主 环 境 中， Spring MVC 建 立 起 一 个 IoC 容 器 的 的 体 系。 这 个 IoC 容 器 体 系 是 通 过 ContextLoaderListener 的 初 始 化 来 建 立 的， 在 建 立 IoC 容 器 体 系 后， 把 DispatchServlet 作 为 Spring MVC 处 理 Web 请 求 的 转 发 器 建 立 起 来， 从 而 完 成 响 应 HTTP 请 求 的 准 备。 有 了 这 些 基 本 配 置， 建 立 在 IoC 容 器 基 础 上 的 Spring MVC 就 可 以 正 常 地 发 挥 作 用 了。 了 解 Spring MVC 在 Web 容 器 中 的 配 置 以 后， 下 面 就 来 看 看 IoC 容 器 在 Web 容 器 中 的 启 动 过 程 是 怎 样 实 现 的。



## 2.启动过程

### 2.1 IoC容器的创建

IoC 容 器 的 启 动 过 程 就 是 建 立 上 下 文 的 过 程， 该 上 下 文 是 与 ServletContext 相 伴 而 生 的， 同 时 也 是 IoC 容 器 在 Web 应 用 环 境 中 的 具 体 表 现 之 一。 由 ContextLoaderListener 启 动 的 上 下 文 为 根 上 下 文。 在 根 上 下 文 的 基 础 上， 还 有 一 个 与 Web MVC 相 关 的 上 下 文 用 来 保 存 控 制 器（ DispatcherServlet） 需 要 的 MVC 对 象， 作 为 根 上 下 文 的 子 上 下 文， 构 成 一 个 层 次 化 的 上 下 文 体 系。 在 Web 容 器 中 启 动 Spring 应 用 程 序 时， 首 先 建 立 根 上 下 文， 然 后 建 立 这 个 上 下 文 体 系 的， 这 个 上 下 文 体 系 的 建 立 是 由 ContextLoder 来 完 成 的。

ContextLoaderListener 是 Spring 提 供 的 类， 是 为 在 Web 容 器 中 建 立 IoC 容 器 服 务 的， 它 实 现 了 ServletContextListener 接 口。 这 个 接 口 是 在 Servlet API 中 定 义 的， 提 供 了 与 Servlet 生 命 周 期 结 合 的 回 调， 比 如 contextInitialized 方 法 和 contextDestroyed 方 法。在 web 容 器 中， **建 立 WebApplicationContext 的 过 程**， 是 在 contextInitialized 的 接 口 实 现 中 完 成 的。 具 体 的 载 入 IoC 容 器 的 过 程 是 由 ContextLoaderListener 交 由 ContextLoader 来 完 成 的， 而 ContextLoader 本 身 就 是 ContextLoaderListener 的 基 类。

![image-20201013154718898](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201013154718898.png)

```java
package org.springframework.web.context;

import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;

public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    public ContextLoaderListener() {
    }

    public ContextLoaderListener(WebApplicationContext context) {
        super(context);
    }

    //看这里
    public void contextInitialized(ServletContextEvent event) {
        this.initWebApplicationContext(event.getServletContext());
    }

    public void contextDestroyed(ServletContextEvent event) {
        this.closeWebApplicationContext(event.getServletContext());
        ContextCleanupListener.cleanupAttributes(event.getServletContext());
    }
}
```

发现进入了**ContextLoader**的initWebApplicationContext方法：

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    //如果根上下文已存在applicationContext容器，则抛出异常    
    if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
        throw new IllegalStateException("Cannot initialize context because there is already a root application context present - check whether you have multiple ContextLoader* definitions in your web.xml!");
    } 
    else {
        servletContext.log("Initializing Spring root WebApplicationContext");
        Log logger = LogFactory.getLog(ContextLoader.class);
        if (logger.isInfoEnabled()) {
            logger.info("Root WebApplicationContext: initialization started");
        }

        long startTime = System.currentTimeMillis();

        try {
            if (this.context == null) {
                //这里创建了spring上下文并赋值到了ContextLoaderListener的context属性中
                this.context = this.createWebApplicationContext(servletContext);
            }

            if (this.context instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext)this.context;
                if (!cwac.isActive()) {
                    if (cwac.getParent() == null) {
                        ApplicationContext parent = this.loadParentContext(servletContext);
                        cwac.setParent(parent);
                    }
                    //调用applicationContext的refresh方法，生成IoC容器
                    this.configureAndRefreshWebApplicationContext(cwac, servletContext);
                }
            }

            //这里将springIoC容器绑定到ServletContext上
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
            ClassLoader ccl = Thread.currentThread().getContextClassLoader();
            if (ccl == ContextLoader.class.getClassLoader()) {
                currentContext = this.context;
            } else if (ccl != null) {
                currentContextPerThread.put(ccl, this.context);
            }

            if (logger.isInfoEnabled()) {
                long elapsedTime = System.currentTimeMillis() - startTime;
                logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
            }

            return this.context;
        } catch (Error | RuntimeException var8) {
            logger.error("Context initialization failed", var8);
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, var8);
            throw var8;
        }
    }
}
```

定位到这一行代码：

```
this.createWebApplicationContext(servletContext)
```

理解起来并不困难：

```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
    	//默认情况下的ContextClass是XmlWebApplicationContext
		Class<?> contextClass = determineContextClass(sc);
		if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
			throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
					"] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
		}
		return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

到这里，IoC容器就创建成功，并被绑定上ServletContext上了。



### 2.2 DispathcerServlet的初始化

```
在 完 成 对 ContextLoaderListener 的 初 始 化 以 后， Web 容 器 开 始 初 始 化 DispatcherServlet， 这 个 初 始 化 的 启 动 与 在 web.xml 中 对 载 入 次 序 的 定 义 有 关。 DispatcherServlet 会 建 立 自 己 的 上 下 文 来 持 有 Spring MVC 的 Bean 对 象， 在 建 立 这 个 自 己 持 有 的 IoC 容 器 时， 会 从 ServletContext 中 得 到 根 上 下 文 作 为 DispatcherServlet 持 有 上 下 文 的 双 亲 上 下 文。 有 了 这 个 根 上 下 文， 再 对 自 己 持 有 的 上 下 文 进 行 初 始 化， 最 后 把 自 己 持 有 的 这 个 上 下 文 保 存 到 ServletContext 中， 供 以 后 检 索 和 使 用。——《Spring技术内幕》
```

书上的描述比较拗口，简单来说，DispatcherServlet建立属于自己的上下文，并持有springMVC的相关Bean，并将自己的上下文与ServletContext关联父子关系。

在看代码前先了解一下DispatcherServlet的继承关系

![image-20201013164918710](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201013164918710.png)

DispatcherServlet 通 过 继 承 FrameworkServlet 和 HttpServletBean 而 继 承 了 HttpServlet， 通 过 使 用 Servlet API 来 对 HTTP 请 求 进 行 响 应， 成 为 Spring MVC 的 前 端 处 理 器， 同 时 成 为 MVC 模 块 与 Web 容 器 集 成 的 处 理 前 端。

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201013165123277.png" alt="image-20201013165123277" style="zoom:67%;" />

简单了解一下DispatcherServlet的工作流程，下面开始其初始化的代码追踪：

#### 2.2.1 init()

```java
@Override
public final void init() throws ServletException {

    // Set bean properties from init parameters.
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
    if (!pvs.isEmpty()) {
        try {
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            initBeanWrapper(bw);
            bw.setPropertyValues(pvs, true);
        }
        catch (BeansException ex) {
            if (logger.isErrorEnabled()) {
                logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
            }
            throw ex;
        }
    }

    // 调用子类FrameWork的initServletBean进行servlet的初始化
    initServletBean();
}
```

作为一个Servlet， DispatcherServlet 的 启 动 与 Servlet 的 启 动 过 程 是 相 联 系 的。 在 Servlet 的 初 始 化 过 程 中， Servlet 的 init 方 法 会 被 调 用， 以 进 行 初 始 化。 DispatcherServlet 的 init方 法 位 于 基 类 HttpServletBean中。init方法主要由2部分构成，**第一部分**，通过熟悉的BeanWrapper和ResourceLoader进行属性赋值。**第二部分**，调用initServletBean进行DispatcherServlet的上下文的创建，来看一下initServletBean：

```java
@Override
protected final void initServletBean() throws ServletException {
    getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
    if (logger.isInfoEnabled()) {
        logger.info("Initializing Servlet '" + getServletName() + "'");
    }
    long startTime = System.currentTimeMillis();

    try {
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet();
    }
    catch (ServletException | RuntimeException ex) {
        logger.error("Context initialization failed", ex);
        throw ex;
    }

    if (logger.isDebugEnabled()) {
        String value = this.enableLoggingRequestDetails ?
            "shown which may lead to unsafe logging of potentially sensitive data" :
        "masked to prevent unsafe logging of potentially sensitive data";
        logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
                     "': request parameters and headers will be " + value);
    }

    if (logger.isInfoEnabled()) {
        logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
    }
}
```

