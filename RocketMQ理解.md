# 概览
* RocketMQ集群组成
* 高可用
* 消息推送模式
* 顺序消息
* 消费者流控
* 消息存取机制
## 1、RocketMQ集群组成
![image.png](https://upload-images.jianshu.io/upload_images/23653832-3f2a0006b3630544.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### Producer集群

* Producer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从 NameServer获取Topic路由信息
* 并且和Topic所在的Master Broker建立长连接，且定时心跳
* 完全无状态，可集群部署

#### Consumer集群

* Consumer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从 NameServer获取Topic路由信息
* 并且和Topic所在的Master Broker，Slave Broker建立长连接（即可以向Master Broker获取消息，也可以向Slave Broker获取消息），且定时心跳
* 消费者在向Master Broker拉取消息时，Master Broker服务器会根据拉取偏移量与最大偏移量的距离 （判断是否读老消息，产生读I/O），以及从服务器是否可读等因素建议下一次是从Master还 是Slave拉取
* 完全无状态，可集群部署

#### Broker集群

* 消息暂存
* 支持主从架构，**brokerName相同的，即为一组broker（broker组），组内不同的broker实例通过broker-id区分**
  * 通常brokerId为0的，会被选为master。其他的，作为slave
  * 提供消息消费的高可用
  * 支持读写分离，主服务器支持读写操作，但是只有 BrokerId=1的从服务器才会参与消息的读负载
* 每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有 NameServer

#### NameServer集群

* Broker管理，管理Broker各种元数据，例如主题、分区等
* 无状态，可以轻易进行横向扩展，集群中节点也不需要同步信息
* 功能：
  * Broker管理，主从切换？
  * 向生产者、消费者提供主题的物理分区信息，避免生产者、消费者轮询成百上千的Broker，以获取物理分区信息（相当于把原本broker的功能，给抽出来了）
#### Topic

* 区分消息种类，一个逻辑概念
* 一个发送者可以发送消息给一个或者多个Topic
* 一个消息的接收者 可以订阅一个或者多个Topic消息

#### Message Queue

* 相当于是Topic的实际物理分区；分布在不同Broker上面
* 提供消息处理能力的横向扩展
* 默认  readQueueNums=8, writeQueueNums=8

## 2、高可用
### 2.1 消息可靠性存储
* RocketMQ提供了两组配置，提供不同的可靠性级别
* 主从同步复制、主从异步复制：`brokerRole=SYNC_MASTER`
* 消息同步刷盘、异步刷盘：`SYNC_FLUSH`、`ASYNC_FLUSH`
* 建议配置：主从同步复制+异步刷盘
### 2.2 生产高可用
* 生产者默认策略为轮询，则Topic分布的broker组中，有至少一个master存活，即可发送消息
### 2.3 消费高可用
* Master Broker高负载或宕机时，消费者可以取从机消费数据

## 3、消息推送模式
### 3.1 传统消息推送模式
* push模式
  * 服务器主动向客户端推送消息
  * 优点：实时性高
  * 缺点：占用服务器资源、难以按消费者消费能力进行推送
* pull模式
  * 客户端可以量力而行的拉取消息
  * 缺点：实时性、当服务器没有消息时的处理
### 3.2 RocketMQ
* 默认实现：基于拉模式实现的推模式
* 基于拉模式，则客户端可以量力而行
* 如果服务端没有消息，会暂时存储这个请求，不返回客户端，超时或者有消息时再返回
* 缺点：服务端暂存这个请求，会占用服务端的连接和资源
* 似乎和Apollo拉取配置的原理类似

## 4、顺序消息
### 4.1 全局有序
* 实现全局有序，需要全局只有一个生产者、消费者以单线程运行，并且对应的Topic只能有一个Message Queue
### 4.2 局部有序
* 例如订单业务，只要求每个订单自己的消息是有序的即可
* 实现思路：每个订单都只发送到固定的消息队列，消息队列内部有序，则配置RocketMQ的顺序消费者进行消费即可
#### 4.2.1 生产者改造思路
* 基于hash改造：
  * 对订单和Message Queue进行哈希取余，得到要发送的目标Message Queue
  * 缺点：如果Broker宕机，则Message Queue的数量会发生变化，可能会破坏顺序性
* 基于配置改造
  * 例如写死配置，订单id奇数结尾的发送到Message Queue1，偶数结尾的发送到Message Queue2
  * 缺点：保证了一致性，但是牺牲了可用性
* 基于hash改造并且消费端对业务进行版本控制
#### 4.2.2 顺序消费者
* 默认使用的是RocketMQ提供的并发消费者
* 顺序消费者特点
  * 单线程消费
  * 本地负载均衡时，先获取Message Queue的锁，再进行消费
  * 并发消费者重试的消息，进入重试队列，不影响后续消息消费。顺序消费者重试会阻塞后面的消息
* 因为是单线程并且要先获取Message Queue锁，再进行消费，保证了全局同时不会有两个消费者线程同时对一个Message Queue进行消费
  * 并发消费者即使是单线程，也不能保证这一点，因为消费者本身的重新负载均衡

## 5、消费者流控
### 基于ProcessQueue
* 消费者拉取到消息后，将消息存入ProcessQueue，并传递给消费者回调进行消费，每消费完一个消息，将它从ProcessQueue中移除
* 则ProcessQueue可以看做消费者拉取到本地，但是未完成消费的消息，则用它来做流控再合适不过了
* 基本上是拉消息前，判断ProcessQueue中消息数量、大小等判断是否跳过本次的消息拉取

## 6、消息存取机制
### 6.1 消息存储
![image.png](https://upload-images.jianshu.io/upload_images/23653832-4ecbf51b66947270.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### CommitLog
* 顺序写，单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量
* 单一文件存储所有主题消息
#### ConsumeQueue
* 相当于CommitLog的索引，消费者基于ConsumeQueue获取消息，所以消息写到ConsumeQueue后，才能被消费者消费
* 写完CommitLog，异步写ConsumeQueue，因为是异步的，所以CommitLog冗余了恢复ConsumeQueue需要的信息，从而异常宕机后，可以重做ConsumeQueue
* 使用ConsumeQueue的原因
  * 文件是定长的，并且文件内每个条目也都是定长的：可以像访问数组一样，访问其中的内容
  * 单个文件不会太大，几M作用，往往可以全部加载到页缓存
  * CommitLog因为消息不定长，不能像数组一样访问，只能直接遍历整个CommitLog查找消息
* ConsumeQueue组成
![image.png](https://upload-images.jianshu.io/upload_images/23653832-30b4180e73ae2bed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  *  tag hashcode：消息tag的hash值，服务端对hash进行过滤，只能基于hash进行比较，客户端需要再次进行字符串比对。因为Consume Queue内容定长，所以只能存储hash值
### 6.2 消息获取
#### 基于偏移量获取
* 因为ConsumeQueue文件定长，并且文件中每个条目也是定长的，所以可以像访问数组一样直接定位到具体文件，再定位到具体文件的偏移量
* 每条记录都是20字节，则消息偏移量*20，可以获取到ConsumeQueue的总体物理偏移量
* 物理偏移量/20 - 当前的第一个ConsumeQueue的起始物理偏移量/20，即可以获取到现在偏移量所在的ConsumeQueue，是第几个ConsumeQueue文件
* 拿到ConsumeQueue之后，使用物理偏移量和单个文件大小取模即可定位到具体条目
* 然后根据CommitLog偏移量去CommitLog获取长度为消息size大小的内容即可
#### 基于时间戳查找消息
* 根据时间戳和ConsumeQueue文件的修改时间，定位ConsumeQueue
* 然后对定位到的ConsumeQueue进行二分查找

### 6.3 RocketMQ与零拷贝
* ConsumeQueue基于`FileChannel#map`，映射到操作系统缓存
* 临时存储池，`TransientStorePool`
  * RocketMQ对操作系统缓存的优化，因为内存交换机制，操作系统缓存可能随时被交换到磁盘，导致缺页中断
  * 开启`TransientStorePool`配置后，会基于native函数，锁定一块直接内存，表示这块内存不允许被交换，提高性能

### 6.4 总结
* 顺序写+随机读
* 但是ConsumeQueue因为内容小，一般都可以全部加载到操作系统缓存，以提供较好的性能，并且RockeMQ提供了预热配置，启动时完成ConsumeQueue的映射
