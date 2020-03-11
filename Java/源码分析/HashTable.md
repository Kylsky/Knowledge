# HashTable

在理解hashTable的源码前，先说出以下结论，将其与HashMap做一下比较，这样在阅读源码时更有针对性

```
1.HashTable无论是键值对都不能存放null，而HashMap可以
2.HashTable初始容量为11，而HashMap为16
3.HashTable的扩容制度为capacity*2+1，而HashMap为capacity*2
4.HashTable的index方法为(hash&0x7FFFFFFF)%table.length，而HashMap的index方法为hash&(table.length-1)
5.HashTable线程安全，而HashMap线程不安全
```

