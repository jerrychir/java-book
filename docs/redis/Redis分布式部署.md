Redis 3.0.0 RC1版本10.9号发布，[Release Note](https://raw.githubusercontent.com/antirez/redis/3.0/00-RELEASENOTES)
这个版本支持**Redis Cluster**，相信很多同学期待已久，不过这个版本只是RC版本，要应用到生产环境，还得等等

Redis Cluster设计要点：

## 架构：无中心

**Redis Cluster采用无中心结构，每个节点都保存数据和整个集群的状态**
每个节点都和其他所有节点连接，这些连接保持活跃
使用gossip协议传播信息以及发现新节点
node不作为client请求的代理，client根据node返回的错误信息重定向请求

## 数据分布：预分桶

预分好16384个桶，根据 CRC16(key) mod 16384的值，决定将一个key放到哪个桶中
每个Redis物理结点负责一部分桶的管理，当发生Redis节点的增减时，调整桶的分布即可
_例如，假设Redis Cluster三个节点A/B/C，则
Node A 包含桶的编号可以为: 0 到 5500.
Node B 包含桶的编号可以为: 5500 到 11000.
Node C包含桶的编号可以为: 11001 到 16384._
当发生Redis节点的增减时，调整桶的分布即可。
**预分桶的方案介于“硬Hash”和“一致性Hash”之间，牺牲了一定的灵活性，但相比“一致性Hash“，数据的管理成本大大降低**

## 可用性：Master-Slave

为了保证服务的可用性，Redis Cluster采取的方案是的Master-Slave
每个Redis Node可以有一个或者多个Slave。当Master挂掉时，选举一个Slave形成新的Master
一个Redis  Node包含一定量的桶，当这些桶对应的Master和Slave都挂掉时，这部分桶对应的数据不可用

## 写

Redis Cluster使用异步复制
一个完整的写操作步骤：
1.client写数据到master
2.master告诉client "ok"
3.master传播更新到slave
**存在数据丢失**的风险：
1\. 上述写步骤1）和2）成功后，master crash，而此时数据还没有传播到slave
2\. 由于分区导致同时存在两个master，client向旧的master写入了数据。
当然，由于Redis Cluster存在超时及故障恢复机制，第2个风险基本上不可能发生

## 数据迁移

Redis Cluster支持在线增/减节点。
基于桶的数据分布方式大大降低了迁移成本，只需将数据桶从一个Redis Node迁移到另一个Redis Node即可完成迁移。
当桶从一个Node A向另一个Node B迁移时，Node A和Node B都会有这个桶，Node A上桶的状态设置为MIGRATING，Node B上桶的状态被设置为IMPORTING
当客户端请求时：
所有在Node A上的请求都将由A来处理，所有不在A上的key都由Node B来处理。同时，Node A上将不会创建新的key

## 多key操作

当系统从单节点向多节点扩展时，多key的操作总是一个非常难解决的问题，Redis Cluster方案如下：
1\. 不支持多key操作
2\. 如果一定要使用多key操作，请确保所有的key都在一个node上，具体方法是使用“hash tag”方案
hash tag方案是一种数据分布的例外情况