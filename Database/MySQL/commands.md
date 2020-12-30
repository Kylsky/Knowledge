## 1.登录mysql

mysql -u root -p



## 2.创建数据库

create database <database_name>



## 3.删除数据库

drop database <database_name>



## 4.创建表

create table (column_name column_type)

```
CREATE TABLE IF NOT EXISTS `runoob_tbl`(
   `runoob_id` INT UNSIGNED AUTO_INCREMENT,
   `runoob_title` VARCHAR(100) NOT NULL,
   `runoob_author` VARCHAR(40) NOT NULL,
   `submission_date` DATE,
   PRIMARY KEY ( `runoob_id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8;
```



## 5.删除表

drop table <table_name>



## 6.新增表记录

insert into <table_name> (field1, field2 ...) value (value1, value2 ...)

insert into <table_name> (field1, field2 ...) value (valueA1, valueA2 ...),(valueB1, valueB2)



## 7.查询表记录

select column_name,column_name from <table_name> [WHERE Clause] [LIMIT N] [ OFFSET M]



## 8.更新表记录

update<table_name> set field1=new-value1, field2=new-value2 [WHERE Clause]



## 9.删除表记录

delete from <table_name> [WHERE Clause]



## 10.合并结果集

```
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions]
UNION [ALL | DISTINCT]
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions];
```

UNION会删除重复得数据，若需要重复，则使用UNION ALL



## 11.事务

begin

commit

rollback



## 12.添加表字段

alter table <table_name> add <field> int;



## 13.修改表字段

alter table <table_name> modify <field> char;



## 14.删除表字段

alter table <table_name> drop <field>



## 15.修改字段默认值

alter table <table_name> alter <field> set default 0;



## 16.删除字段默认值

alter table <table_name> alter <field> drop default;



## 17.修改数据表类型

alter table <table_name> engine=MYISAM;



## 18.修改表名

alter table <table_name> rename to <new_table_name>;



## 19.创建索引

直接创建：

create [UNIQUE] index <index_name> on <table_name> (cloumn_name)

修改表结构：

alter table tbl_name add PRIMARY KEY (column_list)

alter table <table_name> add [UNIQUE | INDEX | FULL TEXT] index_name(column_name)

建表时指定：

```
CREATE TABLE mytable(  
 
ID INT NOT NULL,   
 
username VARCHAR(16) NOT NULL,  
 
[INDEX|UNIQUE] [indexName] (username(length))  
 
);  
```



## 20.删除索引

drop index <index_name> on <table_name>;



## 21.查看索引

　show index from <table_name>;