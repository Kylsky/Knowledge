# MySQL存储过程

今天上班需要在改造好表结构中根据id动态更新数据，组长让我写一下sql脚本，想了想，应该是得创建存储过程才能解决问题了，由于从前从来没有写过，所以赶紧去百度了一下



## 创建存储过程

语法不是很难，下面来记录一下

```
#起始，定义$$为结束符，可以理解为“；”
DELIMITER $$
DROP FUNCTION IF EXISTS genPerson$$
CREATE FUNCTION genPerson(name varchar(20)) RETURNS varchar(50)
BEGIN
#这里跟上函数体
………………
#结束
END $$
DELIMITER ;
```

