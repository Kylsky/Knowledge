## 1.date_format

date_format(time,'%y-%m-%d')



## 2.case when

```
case a when '1' then 'tom' else 'jerry' end


case when a>100 then 10 else 1000 end 
```



## 3.concat

字符拼接

```
concat(year,'-',month,'-',day,' 00:00:00')
```



## 4.date_sub

日期的减法

```
#7天前
date_sub(curdate(),INTERVAL 7 day)

#7天后
date_sub(curdate(),INTERVAL -7 day)
```



## 5.week()

获取当前是第几周,需要注意：

```
如果星期包含1月1日，并且在新的一年中有4天或更多天，那么这周是第1周。
否则，这一周的数字是前一年的最后一周，下周是第1周。
```

例如： 此时2021年1月28日为第4周

![image-20210128163345446](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210128163345446.png)



## 6.