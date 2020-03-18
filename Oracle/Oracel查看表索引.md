# Oracle查看表索引

oracle存储索引名的方式与mysql似乎不太一样，在mysql中查看索引只需要show index from #{table_name}即可。

oracle中表的索引信息存在 user_indexes 和 user_ind_columns 两张表里面，其中，**user_indexes** 系统视图存放的是索引的名称以及该索引是否是唯一索引等信息，**user_ind_columns** 系统视图存放的是索引名称，对应的表和列等。

###  查看oracle单个数据表包含的索引

```
select * from user_indexes where table_name=upper('table_name');
```

### 根据索引名查看索引包含的字段

```
select * from user_ind_columns where index_name = 'INDEXS_NAME';
```

