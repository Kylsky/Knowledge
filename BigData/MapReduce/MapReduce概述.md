## 1.map&reduce

Map:

对一条记录做映射吗



Reduce：

对一组记录做运算



![image-20201130143028624](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201130143028624.png)



## 2.split的理解

![image-20201130143056579](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201130143056579.png)

split作为逻辑上的分区，通常与hdfs物理上的block分块呈1：1关系，但是这也不绝对，在不同的任务情况下，一些任务属于IO密集型，一些属于计算密集型，通过加入split层进行逻辑上的划分解耦，增强了灵活性。



## 3.map过程

![image-20201209150934817](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201209150934817.png)

在对数据进行分组计算时，考虑数据会进入不同的分区，因此需要对key进行模运算，或类似生成md5摘要后进行模运算等操作计算分区号。



在map过程中，映射后的记录被写进buffer，并保证buffer中的内容按照分区有序分布，这样在buffer写入磁盘文件时保证外部无序而内部有序，最后通过归并产生整体有序的文件。并通过文件偏移量将不同分区的数据传递给对应的reduce进程进行后续计算。



此时，文件中已经存储的是经过映射后的数据，数据以<k,v,p>格式存储，k为key，v为value，p为分区。且文件中的记录已经根据分区排序。但是当同一个分区抵达的数据中存在key为乱序的情况下，reduce读取数据仍会因为乱序消耗大量的IO，因此，在内存buffer中不仅需要对partition进行排序，也要对key进行排序。



