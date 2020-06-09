# Springboot事务处理

## 一、前言

最近在处理客户编辑信息时需要同时更新对应的客户端信息，因此需要在一个接口中同时更新两张表的数据，于是就涉及到了事务的问题，之前网上有看到springboot处理相关问题的解决方案，不过由于一直没碰到使用场景，所以也就没有细看。现在来详细分析一下



## 二、@Transactional

springboot使用@Transactional注解，通过声明式事务提供了比较好的解决方案，这个注解可以使用在类上，也可以使用在方法上，当使用在类上时，表示当前类的所有public方法都支持事务，来仔细看一下位于org.springframework.transaction.annotation包下的该注解

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
    @AliasFor("transactionManager")
    String value() default "";

    @AliasFor("value")
    String transactionManager() default "";

    Propagation propagation() default Propagation.REQUIRED;

    Isolation isolation() default Isolation.DEFAULT;

    int timeout() default -1;

    boolean readOnly() default false;

    Class<? extends Throwable>[] rollbackFor() default {};

    String[] rollbackForClassName() default {};

    Class<? extends Throwable>[] noRollbackFor() default {};

    String[] noRollbackForClassName() default {};
}
```

主要来看看@Transactional注解的这几个参数：

```
rollbackFor：导致事务回滚的异常类数组
rollbackForClassName：导致事务回滚的异常类类名的数组
noRollbackFor：不会导致事务回滚的异常类数组
noRollbackForClassName：不会导致事务回滚的异常类名字数组
```



## 三、编码实现

上面可以发现，当业务出现问题导致需要做事务回滚时，只要在tranasactional中指定异常类型并适时抛出异常，但是抛出异常往往会终断程序运行，因此需要使用TransactionAspectSupport来回滚，最终代码的结果示例如下

```java
@Transactional
public Result update(Organization org,Client client){
	int result1 = orgMapper.update(org);
	int result2 = clientMapper.update(client);
	if(result1 + result2<2){
	TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
	return Result.err();
	}
	return Result.ok();
}
```

