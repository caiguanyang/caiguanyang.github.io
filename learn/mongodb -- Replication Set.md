# mongodb 副本集学习

[TOC]

## 副本集组成--节点说明

参考：https://docs.mongodb.com/manual/core/replica-set-members/

由一组 mongod实例组成，提供了**数据冗余度和高可用**。

### primary

一个副本集只有一个主节点，**接收所有的写操作**，并将操作信息记录到 oplog 中，供从节点同步数据使用。

​       读操作默认也是分发到主节点的，但可以通过副本集读选项修改此行为，集群中的 secondary 节点也是可以用于读操作的。

**读选项：**

| 副本集读选项             | 说明                                       |
| ------------------ | ---------------------------------------- |
| primary            | 默认模式，所有读操作都在 primary节点                   |
| primaryPreferred   | 大多情况在 primary 节点，当 primary 不可用时，使用 secondary节点 |
| secondary          | 所有读操作都在secondary 节点                      |
| secondaryPreferred | 大多情况在 secondary 节点，secondary 不可用时，切换到 primary 节点 |
| nearest            | 会选择网络延迟最小的节点进行读，与节点类型无关                  |

### secondary

​     根据主节点的 oplog，备份集群数据。

​    当前 primary 节点不可用时，副本集会自动触发选主过程，secondary 节点可以上升为 primary 节点。

**节点类型：**

| 类别    | 说明                                       | 意义                                       |
| ----- | ---------------------------------------- | ---------------------------------------- |
| 优先级为0 | 不会触发选主，参与投票，**但不会升级为 primary**；能接收读请求    | 在特殊部署环境下，**可以优先保证更符号条件的节点上升为主节点**        |
| 隐藏节点  | **不可读**；参与投票，但不会升级为 primary；要求 **priority 必须设置为0** | 可作为备份节点，执行 db.fsyncLock 的情况下不会影响应用程序完成备份；参与投票 |
| 延时节点  | priority=0；隐藏节点；可以控制是否参与投票；相比主节点的数据延后指定时间 | 用于帮助在人为误操作或其他意外情况下恢复数据                   |



### arbiter

唯一作用就是参与选举，避免集群出现平票。不保存数据，也不会为客户端提供服务。

由于仲裁节点不能加快选举过程，也不能加提供更高的数据安全性（不包含数据），因此副本集中尽可能采用奇数个数据成员，而不使用仲裁者。



## 选主过程

参考：http://www.mongoing.com/docs/core/replica-set-elections.html

参考：http://www.ywnds.com/?p=3366

参考：http://www.lanceyan.com/tag/bully%E7%AE%97%E6%B3%95

心跳检测—>维护主节点备选列表（静态信息检查，如优先级，节点属性等）—>选举准备—>投票

（一票否决）

**什么情况触发选主？**

* 主节点故障
* 主节点网络不可达（心跳检测，从节点会发起申请）
* 人工干预（rs.stepDown(600)）
* 初始化一个副本集

​        由于选主过程中，集群没有主节点，不接收写请求，实际是不可用的，mongodb会尽量不进行选举的。部署集群时也需要考虑这个因素，避免由于部署不合理导致容易切主。



参考：http://blog.nosqlfan.com/html/3621.html

  当 primary 降级为 secondary 后，会关闭所有的链接，这样下次客户端发起写操作时，谁出现 socket 错误，就会重新向集群获取primary 地址。（客户端采用异步写时，如果往一个旧的 primary 节点写时，他不能立马感知到 primary 已经损坏）



**详细的选举算法，待补充**



### 细节问题

* 如果副本集中只有少数节点可用，**无主**。集群还能对外提供正常的读服务吗？
* mongodb 默认的写模式是： fire and forget
* 每个成员都需要等待30s 才能开始下一次选举。如果有太多错误发生的话，会导致选主过程耗时过长。


期间整个集群不可写，对可用性要求极高的系统，是个致命缺陷。




## 副本集备份

http://docs.mongoing.com/core/backups.html

`replication is for availability and backups are for disaster recovery`



### mongodump && mongorestore

备份期间对集群的性能影响比较大，对于备份和恢复比较小的数据库是个不错的选择。

通过指定—oplog选项，用于捕获备份期间 DB 发生的更改；恢复时指定—oplogReplay。



https://www.mongodb.com/blog/post/backup-vs-replication-why-do-you-need-both?jmp=docs&_ga=2.54810667.1414718787.1509536024-242632772.1508841151









## 主从同步原理



## oplog

​        数据同步过程中是先复制数据，再写入 oplog，因此备份节点可能出现在已经同步过的数据上再次执行复制操作。mongodb 在设计之处就考虑：将 oplog 中的同一个操作执行多次，于只执行一次的效果是一样的（但需要保证执行的数序不能乱掉）



## 参考资料

http://www.lanceyan.com/tech/mongodb_repset2.html







