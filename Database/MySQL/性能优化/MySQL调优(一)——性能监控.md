# MySQL调优(一)——性能监控

## 一、show profile

**show profiles**

查看所有sql语句的执行时间

**show profile all for query <num>**

显示所有性能信息

**show profile block io for query <num>**

显示块io操作次数

**show profile context switches for query <num>**

显示上下文切换次数，被动和主动

**show profile cpu for query <num>**

显示用户cpu时间，系统cpu时间

**show profile ipc for query <num>**

显示发送和接受的消息数量

**show profile page faults for query <num>**

显示页错误数量

**show profile source for query <num>**

显示源码中的函数名称与位置

**show profile swaps for query <num>**

显示swap次数



## 二、performance schema

比前者更简单高效的性能监控，具体查看performanceSchema.md



## 三、show processlist

查看所有的与数据库的连接