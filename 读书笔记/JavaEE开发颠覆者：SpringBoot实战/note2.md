# 1.基本配置

## 1.1 关闭特定自动配置

```
@SpringbootApplication(exclude = {DataSourceAutoConfiguration.class})
```



## 1.2 定制banner

```
1.在src/main/resource下创建banner.txt;
通过patorjk.com/software/tagg生成字符，赋值到banner.txt中

2.通过springApplication.setShowBanner设置为false;
```



# 2.springboot运行原理

springboot关于自动配置的源码在spring-boot-auto configure包内。