# Mybatis返回新增主键

假设sql如下：

```
<insert id="addUser" parameterType="com.hans.entity.User" >
    	INSERT INTO USER (
			username,
			password
		)
		VALUES
			(#{name},#{password})
</insert>
```

只需要在<insert>标签中加入以下内容即可：

```
useGeneratedKeys="true" keyProperty="id"
```



