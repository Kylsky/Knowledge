## C程序编译系统

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201224150119940.png" alt="image-20201224150119940" style="zoom: 50%;" />



## 系统硬件组成

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201224160451082.png" alt="image-20201224160451082" style="zoom: 33%;" />



## 虚拟地址空间

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201225094040595.png" alt="image-20201225094040595" style="zoom: 33%;" />



## 虚拟地址空间与物理地址的转换

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201225095019363.png" alt="image-20201225095019363" style="zoom:50%;" />



## 并发与并行

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201225101045120.png" alt="image-20201225101045120" style="zoom: 67%;" />



## 核与高速缓存

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201225101619028.png" alt="image-20201225101619028" style="zoom: 80%;" />



## 超线程

![image-20201225101734326](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201225101734326.png)



## RAM

![image-20201229161158676](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201229161158676.png)

![image-20201229161430508](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201229161430508.png)



## ROM、PROM、EEPROM、闪存

![image-20201229162958414](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201229162958414.png)



## 磁盘

![image-20201229164444891](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201229164444891.png)

![image-20201229164538949](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201229164538949.png)



## 磁盘容量

![image-20201229164710687](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201229164710687.png)

![image-20201229164729455](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201229164729455.png)



## 磁盘读写操作

![image-20201229165018267](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201229165018267.png)



## 缓存命中

![image-20201231085212521](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231085212521.png)

![image-20201231085233657](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231085233657.png)

![image-20201231085314191](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231085314191.png)



## 缓存行

![image-20201231090855221](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231090855221.png)

![image-20201231090904358](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231090904358.png)

![image-20201231090922679](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231090922679.png)

![image-20201231091326091](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231091326091.png)



## 直接映射高速缓存与读操作

![image-20201231091909325](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231091909325.png)

![image-20201231091923436](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231091923436.png)



## 组相联高速缓存

![image-20201231094858175](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231094858175.png)



## 全相联高速缓存

![image-20201231095218726](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231095218726.png)



## 链接

![image-20201231100357790](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231100357790.png)



## 链接器与动态链接器

![image-20201231102957297](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231102957297.png)



## ECF(Exceptional Control Flow)

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201231103918804.png" alt="image-20201231103918804" style="zoom: 33%;" />



## 地址空间

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210113112737363.png" alt="image-20210113112737363" style="zoom:67%;" />



## 虚拟内存

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210113144918004.png" alt="image-20210113144918004" style="zoom:67%;" />

![image-20210113144936653](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210113144936653.png)



## 内存映射

内存映射可以减少拷贝，另外不会产生缓存不命中产生的缺页问题，引发磁盘的低性能IO寻址

```
https://blog.csdn.net/qq_33369979/article/details/109074372
```

![image-20210113150942270](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210113150942270.png)



## 共享对象

![image-20210113154952563](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210113154952563.png)



## 写时复制

举例：fork函数

<img src="http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210113171450167.png" alt="image-20210113171450167" style="zoom:67%;" />



