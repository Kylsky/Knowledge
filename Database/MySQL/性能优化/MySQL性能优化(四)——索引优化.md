# MySQL性能优化(四)——索引优化

## 一、使用索引列查询时尽量不使用表达式

如：

```
select actor_id from actor where actor_id=4;
select actor_id from actor where actor_id+1=5;
```



## 二、尽量使用主键查询

使用主键查询可以减少回表操作，提高效率



## 三、使用前缀索引

参考：https://blog.csdn.net/john1337/article/details/71081827?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task		

有时候需要索引很长的字符串，这会让索引变的大且慢，通常情况下可以使用某个列开始的部分字符串，这样大大的节约索引空间，从而提高索引效率，但这会降低索引的选择性，索引的选择性是指不重复的索引值和数据表记录总数的比值，范围从1/#T到1之间。索引的选择性越高则查询效率越高，因为选择性更高的索引可以让mysql在查找的时候过滤掉更多的行。

一般情况下某个列前缀的选择性也是足够高的，足以满足查询的性能，但是对应BLOB,TEXT,VARCHAR类型的列，必须要使用前缀索引，因为mysql不允许索引这些列的完整长度，使用该方法的诀窍在于要选择足够长的前缀以保证较高的选择性，通常又不能太长。



## 四、排序时使用索引扫描

​	mysql有两种方式可以生成有序的结果：通过排序操作或者按索引顺序扫描，如果explain出来的type列的值为index,则说明mysql使用了索引扫描来做排序

​		扫描索引本身是很快的，因为只需要从一条索引记录移动到紧接着的下一条记录。但如果索引不能覆盖查询所需的全部列，那么就不得不每扫描一条索引记录就得回表查询一次对应的行，这基本都是随机IO，因此按索引顺序读取数据的速度通常要比顺序地全表扫描慢

​		mysql可以使用同一个索引即满足排序，又用于查找行，如果可能的话，设计索引时应该尽可能地同时满足这两种任务。

​		只有当索引的列顺序和order by子句的顺序完全一致，并且所有列的排序方式都一样时，mysql才能够使用索引来对结果进行排序，如果查询需要关联多张表，则只有当orderby子句引用的字段全部为第一张表时，才能使用索引做排序。order by子句和查找型查询的限制是一样的，需要满足索引的最左前缀的要求，否则，mysql都需要执行顺序操作，而无法利用索引排序



## 五、union|union all|in|or

union和union all的区别在于union会做disdinct操作，会产生一定的性能损耗，因此常使用union all。union all、in、or都能够使用索引，但是推荐使用in



## 六、使用索引匹配范围列

范围条件是：<、<=、>、>=、between。范围列可以用到索引，但是范围列后面的列无法用到索引，索引最多用于一个范围列



## 七、强制类型转换会全表扫描

如：

```
explain select * from user where phone=13800001234;//不会触发索引
explain select * from user where phone='13800001234';//会触发索引
```



## 八、不适合建立索引的字段

更新十分频繁，数据区分度不高的字段上不宜建立索引。由于更新会变更B+树，更新频繁的字段建议索引会大大降低数据库性能。

另外，类似于性别这类区分不大的属性，建立索引是没有意义的，不能有效的过滤数据，

一般区分度在80%以上的时候就可以建立索引，区分度可以使用 count(distinct(列名))/count(*) 来计算



## 九、非NULL

创建索引的列值，不允许为null，可能会得到不符合预期的结果



## 十、表连接问题

当需要进行表连接的时候，最好不要超过三张表，因为需要join的字段，数据类型必须一致



## 十一、尽量使用limit



## 十二、单表索引限制

单表索引建议控制在5个以内



## 十三、组合索引限制

单索引字段数不允许超过5个（组合索引）



## 十四、索引监控

使用命令show status like ‘Handler_read%查看’

参数解释：

```
Handler_read_first	读取第一个条目的次数
Handler_read_key	通过索引获取数据的次数
Handler_read_last	读取最后一个条目的次数
Handler_read_next	读取下一条数据的次数
Handler_read_prev	读取上一条数据的次数
Handler_read_rnd	从固定位置读取数据的次数
Handler_read_rnd_next	从数据节点读取下一条数据的次数
```

