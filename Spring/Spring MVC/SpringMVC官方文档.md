## 1. Spring Web MVC

Spring Web MVC是在Servlet API上构建的原始Web框架，从一开始就包含在Spring框架中。其正式名称“Spring Web MVC”来自其源模块(Spring -webmvc)的名称，但它更常见的名称是“Spring MVC”。

###  1.1. DispatcherServlet

与许多其他web框架一样，Spring MVC是围绕前端控制器模式设计的，其中一个中央Servlet DispatcherServlet提供了一个用于请求处理的共享算法，而实际工作是由可配置的委托组件执行的。该模型灵活，支持多种工作流程。

DispatcherServlet使用Spring配置来发现delegate并使之处理请求映射、视图解析、异常处理等方面。