# Mysql表连接

本文参考：https://blog.csdn.net/ly294687451/article/details/88251761



Mysql的表连接分为内连接、外连接(左外连接、右外连接、全外连接)、自然连接和交叉连接。这些分类都是在连接的基础上，从两个表的笛卡尔积中选取满足连接的记录

### 内连接

形如：

```
SELECT *
FROM table_1 AS t1,
table_2 AS t2
WHERE t1.id = t2.id
```

或

```
SELECT *
FROM table_1 AS t1
(INNER) JOIN table_2 AS t2
ON t1.id = t2.id
```

或

```
SELECT *
FROM table_1 AS t1
(INNER) JOIN table_2 AS t2
USING(id)
```

内连接基本与自然连接相同，不同之处在于自然连接要求是同名属性列的比较，而内连接则不要求两属性列同名，可以用using或on来指定某两列字段相同的连接条件。使用USING会消除重复的属性列，而其他两种会将同名的列重命名。



### 左外连接

形如：

```
SELECT *
FROM table_1 AS t1
LEFT JOIN table_2 AS t2
ON t1.id = t2.id
```

对两表进行连接，若左表存在与右表不匹配的项，则该项仍将会被保留，而涉及到右表的不匹配的字段都将会被置为NULL



### 右外连接

右连接的语法与语义与左连接相似

形如：

```
SELECT *
FROM table_1 AS t1
RIGHT JOIN table_2 AS t2
ON t1.id = t2.id
```

对两表进行连接，若右表存在与左表不匹配的项，则该项仍将会被保留，而涉及到左表的不匹配的字段都将会被置为NULL



### 全外连接

形如：

```
SELECT *
FROM table_1 AS t1
FULL JOIN table_2 AS t2
```

只要其中某个表存在匹配，FULL JOIN 关键字就会返回行。(*返回JOIN 两端表的所有数据，无论其与另一张表有没有匹配。显示左连接、右连接和内连接的并集*)

全外连接与交叉连接的区别在于，交叉连接可以通过where子句筛选出笛卡尔积中符合条件的项，而全外连接无法使用where子句，它更像是对两张表并集的一个硬性的指定集



### 自然连接

形如：

```
SELECT *
FROM table_1 
NATURAL JOIN table_2
```

自然连接是一种特殊的等值连接，他要求两个关系表中进行比较的必须是**相同的属性列**，无须添加连接条件，并且在结果中**消除重复的属性列**。



### 交叉连接

形如：

```
SELECT *
FROM table_1 AS t1
CROSS JOIN table_2 AS t2
WHERE(或者on) t1.id = t2.id
```

交叉连接(cross join)即笛卡尔积，即左表的每一行与右表的每一行进行组合并按照条件选择。