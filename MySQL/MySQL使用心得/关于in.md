# 关于in

今天在写公司接口的时候，碰到了需要经过双层遍历的形式访问数据库，大致流程就是一个楼宇中存在多个房间，一个房间存在多个设备，因此要通过遍历每个房间，再遍历每个房间的所有设备，现在数据库中模拟的测试房间有323个，那么就要执行323条sql语句，这是绝对非常影响效率的。

果不其然，在进行测试时响应时间真是够我吃一碗面了，于是我想，可以通过将所有房间集合映射成一个索引集合，大致操作如下

```
List idList = rooms.map(room->room.getId()).collect(Collector.toList());
```

由于ORM使用的是MybatisPlus，因此使用如下代码进行in操作：

```
selectList(new EntityWrapper<RoomDevice>()
	.in("room_id",idList)
);
```

这样一改，极大程度降低了sql的执行频率，提高了效率。