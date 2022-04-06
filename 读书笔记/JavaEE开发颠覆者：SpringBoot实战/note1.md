# 1.SpringEL

```java
    @Value("#{systemProperties['os.name']}") //2      
    private String osName;      
    
    @Value("#{ T(java.lang.Math).random() * 100.0 }") //3      
    private double randomNumber;   
    
    @Value("#{demoService.another}") //4      
    private String fromAnother;         
    
    @Value("classpath:com/wisely/highlight_spring4/ch2/el/test.txt") //5      
    private Resource testFile;         
    
    @Value("http://www.baidu.com") //6      
    private Resource testUrl;         
    
    @Value("${book.name}") //7      
    private String bookName; 
```



# 2.自定义事件

## 1.1 自定义事件

```java
public class DemoEvent extends ApplicationEvent{      
    private static final long serialVersionUID = 1L;      
    private String msg;         
    public DemoEvent(Object source,String msg) {          
        super(source);          
        this.msg = msg;      
    }         
    public String getMsg() {          
    	return msg;      
    } 
}
```

## 1.2 事件监听器

```java
@Component  
public class DemoListener implements ApplicationListener<DemoEvent> {
  		//1         
  public void onApplicationEvent(DemoEvent event) {
    	//2 
    	String msg = event.getMsg();                   
    	System.out.println("我(bean-demoListener)接收到了bean-demoPublisher发布的消息:"+ msg);         
	}     
} 
```

## 1.3 事件发布

```java
@Component  
public class DemoPublisher {      
    @Autowired      
    ApplicationContext applicationContext; //1  
    
    public void publish(String msg){          
    	applicationContext.publishEvent(new DemoEvent(this, msg)); //2      
    }     
} 
```

![image-20210203163558648](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210203163558648.png)

# 3.Spring Aware

spring aware使程序能够获取spring容器的支持，比如对于一些依赖容器中的bean实例的工具类，他们无法将自身注册到spring容器中，因此可以通过spring aware的方式获取容器，参考：

```
https://zhuanlan.zhihu.com/p/68877265
```

若使用了SpringAware，则自定义的Bean会和Spring框架耦合。

使用方法：

```java
@Service  
public class AwareService implements BeanNameAware,ResourceLoaderAware{//1           
    private String beanName;      
    private ResourceLoader loader;           
    @Override      
    public void setResourceLoader(ResourceLoader resourceLoader) {//2          
        this.loader = resourceLoader;      
    }         
    @Override      
    public void setBeanName(String name) {//3          
        this.beanName = name;      
    } 
}
```



# 4.多线程

Spring通过任务执行器（TaskExecutor）来实现多线程和并发编程。使用ThreadPoolTaskExecutor可实现一个基于线程池的TaskExecutor。而实际开发中任务一般是非阻碍的，即异步的，所以我们要在配置类中通过@**EnableAsync**开启对异步任务的支持，并通过在实际执行的Bean的方法中使用@**Async**注解来声明其是一个异步任务。

## 配置类：

```java
@Configuration
@EnableAsync
public class TaskExecutorConfig implements AsyncConfigurer{
    @Override
    public Executor getAsyncExecutor() {//2           
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();              
        taskExecutor.setCorePoolSize(5);              
        taskExecutor.setMaxPoolSize(10);              
        taskExecutor.setQueueCapacity(25);              
        taskExecutor.initialize();              
        return taskExecutor;      
    }
    
    @Override  
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() { 
			return null;
    }
}
```



## 任务执行类：

```java
@Service  
public class AsyncTaskService {           
    @Async //1      
    public void executeAsyncTask(Integer i){          
        System.out.println("执行异步任务: "+i); 
    }
    
    @Async
    public void executeAsyncTaskPlus(Integer i){          
        System.out.println("执行异步任务+1: "+(i+1));      
    }     
} 

```



# 5.定时任务

通过@**EnableScheduling**开启对计划任务的支持，然后在计划任务的方法上注解@**Scheduled**声明定时任务

```java
@Service  
@Configuration
@EnableScheduling
public class ScheduledTaskService {             
    private static final SimpleDateFormat dateFormat = new SimpleDateFormat("HH:mm:ss");           
    @Scheduled(fixedRate = 5000) //1	每隔5秒执行一次
    public void reportCurrentTime() {             
        System.out.println("每隔五秒执行一次 " + dateFormat.format(new Date()));         
    }           
    @Scheduled(cron = "0 28 11 ? * *"  ) //2	每天11.28分执行     
    public void fixTimeExecution(){            
        System.out.println("在指定时间 " + dateFormat.format(new Date())+"执行");        
    }     
} 
```



# 6.条件注解@Conditional

@Conditional根据满足某一个特定条件创建一个特定的Bean。比方说，当某一个jar包在一个类路径下的时候，自动配置一个或多个Bean；或者只有某个Bean被创建才会创建另外一个Bean。总的来说，就是根据特定条件来控制Bean的创建行为，这样我们可以利用这个特性进行一些自动的配置。

通过实现Condition接口并重写matches方法实现。

如bean只有在满足操作系统为windows的时候才会被创建

## WindowsCondition

```java
public class WindowsCondition implements Condition {         
    public boolean matches(ConditionContext context,              
                           AnnotatedTypeMetadata metadata) {          
        return context.getEnvironment().getProperty("os.name").contains("Windows"); 
    }
}
```

## ListService

```java
public interface ListService{
	String showListCmd();
}
```

## WindowsListServiceImpl

```java
public class WindowsListServiceImpl implements ListService {         
    @Override 
    public String showListCmd() {          
        return "dir";      
    } 
} 
```

## ConditionConfig

```java
@Configuration  
public class ConditionConifg {      
    @Bean      
    @Conditional(WindowsCondition.class) //1      
    public ListService windowsListService() {          
        return new WindowsListService();      
    } 
    
    @Bean      
    @Conditional(LinuxCondition.class) //2 这里就需要linux的condition了
    public ListService linuxListService() {          
        return new WindowsListService();      
    } 
}
```



# 7.测试

举例：

```java
@RunWith(SpringJUinit4ClassRunner.class)
@ContextConfiguration(classes = {TestConfig.class})
@ActiveProfiles("prod")
public class DemoBeanIntegrationTests{
	@Autowired
	private TestBean testBean;
	
	@Test
	public void prodBeanShouldInject(){          
	String expected = "from production profile";          
	String actual = testBean.getContent();          
	Assert.assertEquals(expected, actual);      
	} 
}
```



# 8.spring mvc基本配置

springmvc的定制配置需要配置类继承一个**WebMvcConfigurerAdapter**，并在此类使用@**EnableWebMvc**注解，来开启对springmvc的支持。

## 8.1 添加静态资源

实现addResourceHandlers方法

```java
@Override      
public void addResourceHandlers(ResourceHandlerRegistry registry) {    
    registry.addResourceHandler("/assets/**").addResourceLocations("classpath:/assets/");//3               
}  
```



## 8.2 添加拦截器

实现addInterceptors方法

```java
@Override      
public void addInterceptors(InterceptorRegistry registry) {//2      
    registry.addInterceptor(demoInterceptor());      
} 
```

拦截器：

```java
public class DemoInterceptor extends HandlerInterceptorAdapter {//1 
    @Override      
    public boolean preHandle(HttpServletRequest request, //2              
                             HttpServletResponse response, Object handler) throws Exception {          
        long startTime = System.currentTimeMillis();          
        request.setAttribute("startTime", startTime);          
        return true;      
    }         
    
    @Override
    public void postHandle(HttpServletRequest request, //3
                           HttpServletResponse response, Object handler,
                           ModelAndView modelAndView) throws Exception {
        long startTime = (Long) request.getAttribute("startTime");
        request.removeAttribute("startTime");
        long endTime = System.currentTimeMillis();
        System.out.println("本次请求处理时间为:" + new Long(endTime - startTime)+"ms");
        request.setAttribute("handlingTime", endTime - startTime);      
    } 
}
```



## 8.3 添加viewController

在无任何业务处理的简单页面转向时，可以使用viewController来简化配置

```java
@Override      
public void addViewControllers(ViewControllerRegistry registry) {          
    registry.addViewController("/index").setViewName("/index");  
} 
```



# 9.ControllerAdvice

通过@ControllerAdvice，我们可以将对于控制器的全局配置放置在同一个位置，注解了@Controller的类的方法可使用@ExceptionHandler、@InitBinder、@ModelAttribute注解到方法上，这对所有注解了@RequestMapping的控制器内的方法有效。

## @ExceptionHandler

用于处理全局处理控制器中的异常

## @InitBinder

用来设置WebDataBinder，WebDataBinder用来自动绑定前台请求参数到model中

```
https://blog.csdn.net/wang0907/article/details/108357696
```

## @ModelAttribute

让全局的@RequestMapping都能获得在此处设置的键值对

```
https://blog.csdn.net/li_xiao_ming/article/details/8349115
```



# 10.文件上传

配置MultipartResolver，使用MultipartFile接收上传的文件，使用FileUtils.writeByteArrayToFile快速写文件到磁盘。

```java
@Bean
public MultipartResolver multipartResolver() {
    CommonsMultipartResolver multipartResolver =  new CommonsMultipartResolver();
    multipartResolver.setMaxUploadSize(1000000);
    return multipartResolver;
} 

```



# 11.服务端推送技术

## 11.1 SSE

```java
@Controller  public class SseController {
    @RequestMapping(value="/push",produces="text/event-stream") //每5秒向浏览器推送随即消息
    public @ResponseBody String push(){
        Random r = new Random();
        try {
            Thread.sleep(5000);
        } 
        catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "data:Testing 1,2,3" + r.nextInt() +"\n\n";      
    }
}
```



## 11.2  servlet3.0+异步方法处理

````
https://www.cnblogs.com/baixianlong/p/10661591.html
````

