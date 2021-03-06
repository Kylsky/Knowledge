# MySQL调优(二)——数据类型优化

## 一、数据类型优化

### 1.数据长度更小的通常更好

### 2.数据类型尽量简单

### 3.避免使用null

### **4.细则**

**整数类型**

如TINYINT、SMALLINT、MEDIUMINT、INT、BIGINT，分别使用8、16、24、32、64位存储空间尽量使用满足需求的最小数据类型

**字符和字符串类型**

varchar根据实际内容长度保存数据，由于每次修改可能需要重新计算存储空间，因此适用于字符串很少更新的场景。char为固定长度，且会自动删除字符串末尾的空格，适用于经常更新的字符串

**BLOB和TEXT**

较少使用

**时间类型**

datetime占8个字节，与时区无关，可保存到毫秒，保存时间范围较大

timestamp占4个字节，时间范围：1970-01-01到2038-01-19，精确到秒，采用整形存储，与数据库时区相关，再添加或修改时能自动更新timestamp的值

date占3个字节，使用字符串存储，还可以利用日期时间函数进行日期之间的计算，date类型用于保存1000-01-01到9999-12-31

**使用枚举代替字符串**

create table enum_test(e enum('fish','apple','dog')not null);

insert into enum_test(e) values('fish',('dog'),('apple'));

select e+0 from enum_test;

**特殊类型数据**

经常用varchar(15)来存储IP地址，然而它的本质是32位的无符号证书二不是字符串，可以用INET_ATON()和INNET_NTOA()函数再这两种表示方法之间转换



## 二、合理使用范式和反范式

**第一范式**

列不可分

**第二范式**

不能存在传递依赖

**第三范式**

必须直接关联主键

**使用范式**

优点：更新比反范式块，重复数据较少，范式化的数据小，可以放在内存中

缺点：通常需要进行关联

**使用反范式**

优点：所有数据都在同一张表，可以避免关联，可以有效地设计索引

缺点：表内冗余数据太多，删除数据会造成表信息丢失



## 三、主键选择

### 代理主键

与业务无关，无意义的数字序列，不与业务耦合，更容易维护，推荐。

### 自然主键

十五属性中的自然唯一标识



## 四、字符集选择

若无需中文或其他特殊字符，使用latin1，若有中文等其他字符情况，使用utf-mb4



## 五、存储引擎选择

MySQL默认引擎位InnoDB

![img](http://kylescloud.top/site/pic/MySQLEgine.jpg)



## 六、适当的数据冗余

被频繁引用且只能通过Join2张表或者更多大表的方式才能得到的独立小字段，会造成大量不必要的IO，完全可以通过空间换取时间的方式来优化。不过，荣誉的同时需要确保数据的一致性不会遭到破坏，确保更新的同时冗余字段也被更新。