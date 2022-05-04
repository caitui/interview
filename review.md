## 拍卖项目
### 提高读qps
#### 方案
目前采用的方式是write through模式，也就是在写操作时更新缓存。采取的策略是先加redis分布式锁，防止更新缓存时并发读，执行完业务逻辑后先删除缓存，再更新mysql，最后重建redis缓存。
把缓存操作都收敛在写请求，并且写请求加分布式锁，有效防止读写并发导致的写入脏缓存数据问题。
#### 其他方案对比
##### mysql读写分离
一主多从的设计，读走从库，写走主库。对于拍卖这种特殊的场景，可以读写都走主库。
###### 主从延迟原因：
（1）主库正在进行大事务操作，这时应该将大事务拆分成小事务，分批次提交。  
（2）主库正在进行DDL或DML操作，比如更改索引。这种操作应该放在负载低的时间段  
### MySQL高可用  
#### 主从复制的模式
（1）异步复制  
（2）半同步复制：master更新操作写入binlog之后会主动通知slave，slave接收到之后写入relay log 即可应答，master只要收到至少一半ack应答，则会提交事务  
（3）同步复制：全同步复制必须收到所有从库的ack，才会提交事务  
#### 高可用方案
(1) 主从或主主+Keepalived:  在主实例的 Keepalived 中，增加监测本机 MySQL 是否存活的脚本，如果监测 MySQL 挂了，就会重启 Keepalived，从而使 VIP 飘到从实例。  
(2) MHA方案：MHA在监测到主实例无响应后，可以自动将同步最靠前的 Slave 提升为 Master，然后将其他所有的 Slave 重新指向新的 Master。  
### redis高可用 
#### redis哨兵模式
redis使用一组哨兵（sentinel）节点来监控主从redis服务的可用性。一旦发现redis主节点失效，将选举出一个哨兵节点作为领导者（leader）。哨兵领导者再从剩余的从redis节点中选出一个redis节点作为新的主redis节点对外服务。  
#### redis cluster  
Redis Cluster则采用的是虚拟槽分区算法，在Redis中将存储空间分成了16384个槽，也就是说 Redis Cluster槽的范围是 0 -16383。在存储信息的时候，集群会对 Key 进行 CRC16 校验并对 16384 取模（slot = CRC16(key)%16383）。
##### 查询数据  
（1）未在做数据迁移, MOVED重定向请求：  
![image](https://user-images.githubusercontent.com/35059921/166395602-cdcd4b89-1549-4d6f-97e8-c58728c571aa.png)  
（2）正在做节点数据迁移，ASK重定向请求：  
![image](https://user-images.githubusercontent.com/35059921/166395695-44f47957-dd6a-495c-814b-87fb16f388f9.png)  
##### 缓存节点的扩展和收缩  
cluster meet 命令让新节点加入进来。  
![image](https://user-images.githubusercontent.com/35059921/166395956-f568c54d-6693-4c9b-836b-92aecbd604e9.png)  
## 标签系统
### 提高导入性能
#### 调度服务Worker设计
wokrker是将大的execl等文件拆分成小的文件并将这些小文件导入到mongo生成元数据，并将每条记录生成一个任务发送到消息队列。
#### 消息队列设计
消息队列采用redis stream，每个业务单独一个stream通道，一个stream设置一个消费者组，组内可设置多个消费者进行消费。
#### 导入服务设计
导入服务会通过一个协程池将任务再次分解，每个goroutine进行标签的生成的提取，最后讲结果汇聚到一个channel里面，同时写入mongo和redis。
##### 为什么要双写
mongo用于数据归档，redis用于数据查询。
##### 双写如何保证数据一致
只要mongo和redis一个写失败，就将数据放在redis stream里面进行重试。
##### 多次写入如何保证数据不重复
做幂等性保证，跟标签的唯一名称（提前生成mongo的唯一id，作为标签的id）
#### mongo和redis高可用
##### mongo高可用
##### redis高可用
（1）同一个区域   
同一个区域，采用redis cluster,




