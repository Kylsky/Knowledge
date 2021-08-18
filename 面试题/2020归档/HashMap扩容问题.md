# HashMap扩容问题

谈到hashmap的扩容问题，可以知道hashmap在向table中填充数据时会对key做取模运算，算法如下：

```
hashCode&(capacity-1)
```

由于操作系统识别的是二进制数据，因此若在扩容时能够通过二进制数的位运算操作来进行扩容，那么**速度将会更快**，且**不会造成较大的空间浪费**，由于造成空间浪费可能会导致出现无效的桶，因此在一定程度上**减少了碰撞的几率**

回到hashmap填充数据，按理说要将指定数据填充到table中，应该使用hashcode%capacity才对，但是却使用了与运算，后来发现，原来sun公司大佬们在研究时发现hashCode%capacity正好与hashCode&(capacity-1)相等，因此这恰到好处的解决了问题。