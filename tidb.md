## 表热点&索引热点

> TableID + rowID/indexID 这个在TIDB集群内部是唯一的，Region 存储的是一个范围的数据，如果是顺序增长的ID，则会造成数据都在同一个Region写入，导致热点出现，TiDB 中 RowID 默认也按照自增的方式顺序递增，主键不为整数类型时，同样会遇到写入热点的问题。
```
解决该问题：
1、没有设置主键的表，设置SHARD_ROW_ID_BITS ，值为2的N次方，设置多少个分片
2、设置了主键的表，使用 AUTO_RANDOM 处理自增主键热点表，适用于代替自增主键
AUTO_RANDOM(S, R) S 表示分片位的数量，取值范围是 1 到 15。默认为 5。R 表示自动分配值域的总长度，取值范围是 32 到 64。默认为 64。
从V7.1开始，可以设置tidb_load_based_replica_read_threshold 系统变量控制读请求的排队长度。当 leader 节点的预估排队时间超过该阈值时，TiDB 会优先从 follower 节点读取数据。
```

## 异步添加索引
>原理
```
123123
```
>优化设置
```
设置tidb_ddl_reorg_worker_cnt 工作线程默认4，回填批次数据大小tidb_ddl_reorg_batch_size 默认128
```



