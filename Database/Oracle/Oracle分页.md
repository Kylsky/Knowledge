# Oracle分页

在Oracle中实现分页的方法大致分为两种，用ROWNUM关键字和用ROWID关键字，下面来详细介绍一下：

### 1.ROWNUM

```
SELECT *
  FROM (SELECT ROW_.*, ROWNUM ROWNUM_
          FROM (SELECT *
                  FROM TABLE1
                 WHERE TABLE1_ID = XX
                 ORDER BY GMT_CREATE DESC) ROW_
         WHERE ROWNUM <= 20)
 WHERE ROWNUM_ >= 10;
```

oracle中ROWNUM用于表示行号，因此当客户端传入页数index和每页的数据量number时就可以判断ROWNUM的值，从效率上看，上面的SQL语句在大多数情况拥有较高的效率，主要体现在WHERE ROWNUM <= 20这句上，这样就控制了查询过程中的最大记录数，而在查询的最外层控制最小值。但最大值意味着如果查到了很大的范围（如百万级别的数据），查询就会从很大范围内往里减少，效率就会很低。



### 2.ROW_ID

```
SELECT *
  FROM (SELECT RID
          FROM (SELECT R.RID, ROWNUM LINENUM
                  FROM (SELECT ROWID RID
                          FROM TABLE1
                         WHERE TABLE1_ID = XX
                         ORDER BY order_date DESC) R
                 WHERE ROWNUM <= 20)
         WHERE LINENUM >= 10) T1,
       TABLE1 T2
WHERE T1.RID = T2.ROWID;
```

这种情况还没在实践中遇到过，所以先不讨论了

