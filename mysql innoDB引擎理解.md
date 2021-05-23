## 概览
* InnoDB引擎
  * 内存部分
  * 磁盘部分
  * 锁和死锁
* 日志文件
  * undo log
  * redo log
  * bin log
  * 一次update的执行流程
* 索引
  * 主键索引、辅助索引
  * 覆盖索引
  * exlpain


## 1、InnoDB引擎
![image.png](https://upload-images.jianshu.io/upload_images/23653832-d4465e1d10739deb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.1 内存部分
* 自适应hash索引：用于优化缓冲池查询，InnoDB会自动根据访问的频率和模式来为缓冲池中的某些页建立哈希索引
* Buffer Pool：缓存池，一块连续的内存，以页为单位
* Change Buffer：写缓冲
* Log Buffer：日志缓冲区
#### Buffer Pool
* Innodb 中，数据的访问是按照页/块（默认为16KB）的方式从数据文件（磁盘）中读取到 buffer pool中，然后在内存中用同样大小的内存空间做一个映射
* 三种page

  * free page：空闲的，可以被使用的
  * clean page：被占用的page，数据未被修改过
  * dirty page：脏页，被占用的page，数据被修改过，和磁盘不一样，所以叫脏页（显然和redo log是一样的）
* 使用了改进版的lru算法
  * 普通lru：单独维护一个链表，最近使用过的数据添加到链表头，淘汰数据时，从尾部进行淘汰

    * 缺点，当有某个查询返回大量结果时（例如全表扫描，超大pageSize等）会直接将普通lru头部打满，则热数据会被移到尾部，甚至直接移除出链表。
  * 改进版lru：从midpoint插入最近使用过的数据（new list尾部，old list头部）。如果数据很快被访问，则使其向new list方向移动，否则自然而然的向old list方式移动（等待被淘汰）。

    * 则变年轻，需要至少两次访问。防止了不常查询的操作反而占据头部，从而淘汰热点数据
    * 每当有新数据进入 buffer pool，判断是否有 free page。有的话，将free page从free list中删除，添加到 lru list，没有的话，从lru list 的old list尾部进行淘汰
#### Change Buffer 
* 写缓冲，降低磁盘IO，提升数据库写性能的一种机制。
* 默认占 Buffer Pool的25%，可调整的最大值为50%，写业务较少的话，可以适当调小该值
* 和Buffer Pool的协作
   * 写操作时，如果页已经在Buffer Pool中，则直接写Buffer Pool，否则先记录到Change Buffer，然后写入redo log，一次磁盘顺序写操作
  * 读操作时，合并Change Buffer的修改到Buffer Pool，形成脏页
* 为什么需要Change Buffer
  * 因为Buffer Pool中，都是完整的数据页，进行写操作时，如果Buffer Pool中没有该数据页，则必须从磁盘读取。则Change Buffer帮助节省了一次磁盘IO
* 仅适用于非唯一普通索引
  * 对具有唯一属性的数据进行新增时，需要校验唯一性，必须从磁盘读取数据，干脆就直接放到Buffer Pool中就好了，不用再走Change Buffer了

#### Log Buffer
* 刷盘时机
  * 缓冲区满
  * 配置innodb_flush_log_at_trx_commit
    *  0 ：表示每次事务提交时都只是把 redo log 留在 redo log buffer 中 ;
    *  1 ：默认值，表示每次事务提交时都将 redo log 直接持久化到磁盘；
    *  2：表示每次事务提交时都只是把 redo log 写到 page cache。（配置为2的话，mysql挂不会丢失数据）
* 经常使用的是redo log和bin log（sync_binlog）的双1配置

### 1.2 磁盘部分
* frm：表结构文件
* idb：索引文件
* undo log
* redo log
* bin log
#### undo log
* undo log
  * 对行进行修改时，MySQL是修改一行先上行锁，然后写undo记录到undo数据区，然后修改数据页，然后写日志到log buffer
  * 事务回滚时或者数据库崩溃时，可以利用 Undo 日志，撤销未提交事务对数据库产生的影响。
  * 不会在事务提交就立刻删除undo log，通过后台线程删除（purge thread）
* 逻辑日志，记录变化过程，例如
  * 例如删除操作，undo log里面记录一个插入
  * 插入操作，记录一个删除
  * 更新操作，记录一个反向的更新逻辑
* 功能
  * 事务回滚
  * 多版本并发控制
    *  一致性视图提供当前事务对于可以看到的事务id
    * 对于某个数据行，其回滚指针关联了undo log，undo log中包含事务id，从而可以不断的向前滚动，找到对应的undo log，和数据行当前状态一起计算得出之前的版本。
#### redo log
  * 重做日志，用于容灾，进行崩溃恢复
  * 顺序写，循环覆盖写
  * buffer pool中的脏页刷盘后，redo log的空间可以释放
#### bin log
  * 归档日志，redo log是大小有限，记录的内容有限，时间跨度长恢复使用bin log
  * 主从复制日志
  * 三种格式
    * row：清楚记录每一个行数据的修改细节，能完全实现主从数据同步和数据的恢复。
      * 缺点是产生大量日志
    * statement：只记录sql语句，占用空间小
      * 缺点是不准确，例如now()函数，在从库上面执行，肯定结果是不对的
     * mixed：混合型

#### 一次update的执行流程
* 记录Change Buffer 或者 Buffer Pool
* 进行redo log和bin log的两阶段提交
  * 写redo log ，状态是prepare
  * 写bin log
  * 提交事务，redo log变为commit状态
  * 则崩溃恢复时：commit状态的redo log可以使用，prepare状态的但是bin log写入完整的也可以恢复（可能已经传递到从库，redo log和bin log通过txid关联），只有prepare状态的redo log不能用于恢复
  * 两阶段提交，主要保证redo log和bin log都写成功，才算作一次完整的事务提交

### 1.3 锁和死锁
#### 行锁和表锁
* 行锁
  * 记录锁，锁定单个行记录的锁
  * 间隙锁，锁定索引记录间隙，确保索引记录的间隙不变。（范围锁，RR隔离级别支持）
  * Next-key Lock 锁：记录锁和间隙锁组合，同时锁住数据，并且锁住数据前后范围。（记录锁+范 围锁，RR隔离级别支持）
  * 需要借助索引，没有索引时只能使用表锁
#### 加锁行为
* 普通的select语句：没有锁
* select ... from lock in share mode语句：共享锁
* select ... from for update、update ... where、delete ... where：使用Next-Key Lock锁进行处理，如果扫描发现唯一索引，可以 降级为RecordLock锁
* insert语句：InnoDB会在将要插入的那一行设置一个排他的RecordLock锁。
#### 常见死锁场景
* 两个更新行为，共同持有对方需要的表锁
* 两个更新行为，共同持有对方需要的行锁
* 间隙锁：
  * 间隙锁可重入，只阻塞插入行为
  * 两个事务，调用select ... from for update
  * 如果获取的间隙锁范围存在交集，则接下来两个事务向交集范围内的插入操作可能会形成死锁
* 多数据源应用
  * 连接池最大连接数是有限制的，则极端场景，可能会出现短暂的死锁，毕竟数据库连接在不断释放
  * Sharding JDBC以同步方式，解决了这个问题
## 2、索引
### 2.1 主键索引、辅助索引
* 主键索引：叶子节点存储完整数据
* 辅助索引：叶子节点存储主键。所以借助普通索引读取数据时，要回到主键索引读取完整数据
### 2.2 覆盖索引
* 普通索引读取数据时，要回到主键索引读取完整数据，但是如果需要查询的字段，都在查询用到的普通索引的字段内，就不用会主键索引读取数据了
* explain，extra列为using index
### 2.3 exlpain
#### keys列、key_length列
* keys列表示查询时真正使用到的索引，显示的是索引名称
* 表示查询使用了索引的字节数量，配合keys列可以判断是否全部使用了组合索引。
#### rows列
* 扫描行数，直接体现索引的过滤性好坏
#### extra
* Using where 表示查询需要通过索引回表查询数据。
* Using index 表示查询需要通过索引，索引（覆盖索引）就可以满足所需数据，不会回表查询。
* Using filesort 表示查询出来的结果需要额外排序，数据量小在内存，大的话在磁盘，因此有Using filesort 建议优化。
* Using temprorary 查询使用到了临时表，一般出现于去重、分组、大数据量排序等操作。
* Using index Condition， where查询，如果有索引字段，先从索引过滤，然后剩下的字段再去主键索引过滤
#### type列
*  ALL：表示全表扫描，不一定性能最差，用不到索引排序时，mysql会在index和all之间会选择all，然后进行文件排序
*  index：表示基于索引的全表扫描，先扫描索引再扫描全表数据。
*  range：表示使用索引范围查询。使用>、>=、<、<=、in等等。
*  ref：表示使用非唯一索引进行单值查询
*  eq_ref：一般情况下出现在多表join查询，表示前面表的每一个记录，都只能匹配后面表的一行结果。 
*  const：表示使用主键或唯一索引做等值查询，常量查询。
*  NULL：表示不用访问表，速度最快。
## 3、Mysql集群
### 3.1 从库并行复制
#### 基于锁的并行复制
* 事务之间如果持有锁的时间没有有重叠，并且都能提交成功，显然这些事务之间没有冲突，可以并行提交
* 记录一个事务的GTID信息： L时间点（lock interval开始时间点、last_committed）和C时间点（事务提交，lock interval结束时间点、sequence_number），则两个事务：

  * 如果事务A的C时间点大于等于事务B的L时间点，则不能并行执行（锁持有时间没有重叠）
  * 如果事务A的C时间点小于事务B的L时间点，则可以并行执行（锁持有时间存在重叠）
#### 8.0基于影响行的并行复制
* 记录每个事务修改的行的主键值，主键值之间没有重合的事务可以并行重放

### 3.2 常见部署架构 
* 主从 todo
* 主主 todo
* 水平拆分 todo
