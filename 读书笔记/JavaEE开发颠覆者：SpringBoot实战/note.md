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
@Component  public class DemoListener implements ApplicationListener<DemoEvent> {//1         public void onApplicationEvent(DemoEvent event) {//2 
    	String msg = event.getMsg();                   
    	System.out.println("我(bean-demoListener)接收到了bean-demoPublisher发布的消息:"+ msg);         
	}     
} 
```

## 1.3 事件发布

```java
@Component  public class DemoPublisher {      
    @Autowired      
    ApplicationContext applicationContext; //1  
    
    public void publish(String msg){          
    	applicationContext.publishEvent(new DemoEvent(this, msg)); //2      
    }     
} 
```

![image-20210203163558648](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210203163558648.png)

# 3.Spring Aware