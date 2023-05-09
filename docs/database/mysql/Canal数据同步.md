# Canal数据同步

## Client-Adapter基本能力

目前Adapter具备以下基本能力：

- 对接上游消息，包括kafka、rocketmq、canal-server
- 实现mysql数据的增量同步
- 实现mysql数据的全量同步
- 下游写入支持mysql、es、hbase

### 迁移数据库

看到说，可以通过使用 adapter 的 REST 接口手动触发 ETL 任务，实现全量同步。同时看到说，在全量同步时，在同一个 destination 的增量同步任务会被 阻塞，待全量同步完成被阻塞的增量同步会被 重新唤醒。那就有个问题：在业务不停的情况下，即MySQL数据既有大量存量数据（部分binlog已经删除），又在不停增加的情况下。是先手工触发全量同步，完成后再启动adapter？这个好像比较符合逻辑，只是启动adapter后，如何保证增量是刚好接着全量最后一条记录开始的？还是首先启动adapter，再手工触发全量同步？这个顺序的话，adapter同步了一部分增量，再手工触发全量的话，以前增量同步的会不会重复同步？而且同样有全量结束后，adapte重新开始后，是如何接上的问题？还是说，在全量同步时，业务要暂停，等全量同步结束后，在启动adapter和业务？那这样，也有增量同步如何接上的问题？谢谢！

全量和增量都是在adapter完成的，所以这个不管如何都是需要启动的，先启动 deployer，再启动adapter，启动后就会自动增量更新，然后随时可以全量更新，全量更新时会暂停增量更新，然后重复的主键，数据会覆盖(反正都是最新的)，全量完成后，在全量过程中发生的增量会消费回来，不用担心。有个问题，比如ES，假如有一条id为1001的数据，然后数据库中并没有这条数据，那么全量迁移的时候并不会删除1001这条数据的。

前置知识：[基于canal的client-adapter数据同步必读指南](https://cloud.tencent.com/developer/beta/article/1809823)

迁移方案见：

[使用canal同步](https://help.aliyun.com/document_detail/307064.html?spm=a2c4g.124392.0.0.7e0852c9AgUX2b)

https://github.com/alibaba/canal/issues/3474

https://github.com/alibaba/canal/wiki/ClientAdapter

### 迁移到es

https://juejin.cn/post/7092320318353571848#heading-11

https://blog.51cto.com/gblfy/5670294#demo_423

两篇文章结合使用



### 同步到kafka

[使用Canal将MySQL的数据同步至消息队列Kafka版](https://help.aliyun.com/document_detail/273086.html)









