## 1.前言

最近遇到一种情况，大概的情况描述就是用户下存在一个订单集合，每个订单中包含了一条日志。用户，订单，日志在程序中都使用实体类封装，并且存储在mysql中。由于要查出某用户下的所有订单及下面的商品，因此需要使用mybatis中的collection 和 assosiation来解决。

此前没有使用以上两个属性来操作mybatis，故写此文做个记录。



## 2.情况说明

简单贴一下代码

**User**:

```
@Data
public class User{
	private Integer id;

	private List<Order> orders;
}
```

**Order:**

```
@Data 
public class Order{
	private id;
	
	private Integer userId;

	private Log log;
}
```

**Good:**

```
@Data
public class Log{
	private Integer id;
	private Integer orderId;

	private String content;
}
```



## 3.mybatis部分

```xml
<resultMap id="BaseResultMap" type="top.kylescloud.entity.User">
        <id column="id" property="id"/>
    	<collection property="productVersionClientList" ofType="top.kylescloud.entity.Order">
            <result property="id" column="order_id"
            <association property="oauthClientDetails" javaType="top.kylescloud.entity.Log">
                <result property="id" column="log_id"/>
                <result property="orderId" column="log__order_id"/>
                <result property="content" column="content"/>
            </association>
        </collection>
</resultMap>

<select id="getUserOrderGoods" resultMap="ResResultMap">
        select
        id,
    	order.id as order_id,
		log.id as log_id,
    	log.order_id as log__order_id,
    	log.content as content
        from
        user
    	join order 
    	on user.id = order.user_id
    	join log
    	on order.id = log.order_id
    	order by order.id,log.id
</select>
```



