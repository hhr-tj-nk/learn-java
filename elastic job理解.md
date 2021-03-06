

## 1、Elastic-Job介绍

ElasticJob 是一个分布式调度解决方案，由 2 个相互独立的子项目 ElasticJob-Lite 和 ElasticJob-Cloud 组成

* ElasticJob-Lite 定位为轻量级无中心化解决方案，使用jar的形式提供分布式任务的协调服务；
* ElasticJob-Cloud 使用 Mesos 的解决方案，额外提供资源治理、应用分发以及进程隔离等服务

## 2、Elastic-Job-Lite

### 2.1 分布式调度应该包含的功能

* 支持集群批量跑批，一个大的定时作业拆分给集群中多个实例，每个实例跑一部分任务
* 重分片
* 作业高可用，故障转移
* 幂等控制


### 2.2 elastic-job-lite对分布式调度的支持

#### 2.2.1 elastic-job-lite整合条件

* 项目引入jar包 + zookeeper服务即可

#### 2.2.2 弹性调度

* 每台任务服务器会被分配一个或多个分片。
  * 例如一个作业被分为4片，如果存在两台服务器，则每台服务器会被分配2个分片。
  * 服务器自己根据分片编号，自行进行业务数据分片。分片编号从0开始。
  * 个性化分片，例如0=北京,1=上海,2=广州。则可以更简单的建立分片与业务数据的对应关系。
  * 从而实现了集群批量跑批和单机单线程执行任务的支持。


* 重分片
  * 近乎实时的感知服务器数量的变更，从而重新为分布式的任务服务器分配更加合理的任务分片
* 作业高可用（失效转移）
  * 当作业服务器在运行中宕机时，注册中心同样会通过临时节点感知，并将在下次运行时将分片转移至仍存活的服务器，以达到作业高可用的效果。（依赖于开启幂等控制）
  * 开启失效转移功能效果更好，可以保证在本次作业在执行时宕机的情况下，备机立即启动替补执行。




#### 2.2.3 失效转移

* 开启失效转移功能+监听作业运行状态功能
* 开启失效转移功能之后，ElasticJob 的其他服务器能够在感知到宕机的作业服务器之后，补偿执行该分片作业。
* 如果不开启失效转移功能，不会在本次执行过程中进行重新分片，而是等待下次调度之前才开启重新分片流程。该分片作业可能丢失执行。
* 不适合执行频率很短的任务，适合于耗时长，执行频率低的任务。

#### 2.2.4 幂等控制

* 开启misfire + monitorExecution（属于作业配置，默认值都为true）
* 开启后不允许作业在同一时间内叠加执行。 当作业的执行时长超过其运行间隔，错过任务重执行（开启misfire）能够保证作业在完成上次的任务后继续执行逾期的作业。
* 错过重新执行，如果错过了三次，会重新执行几次？
  * 只会重新执行一次，elastic-job只会记录错过执行状态，不会记录错过几次，也没有这个必要。
* 不适合执行频率很短的任务，适合于耗时长，执行频率低的任务。

### 3、缺点

* 选举master时，所有任务阻塞

## 3、Elastic-Job-Lite基本原理

### 3.1 分布式调度原理

* 通过zookeeper进行master选举，
  * 第一台服务器上线触发主服务器选举
  * 主服务器一旦下线，则重新触发选举
  * 选举过程中阻塞，只有主服务器选举完成，才会执行其他任务。
* 某作业服务器上线时会自动将服务器信息注册到注册中心，下线时会自动更新服务器状态。
  * 服务器下线，主服务器进行重新分片
* 定时任务执行时，如果触发重新分片，则任务阻塞，先进行重分片，再执行任务。
  * 任务执行中，只会标记分片状态，不会重新分片，分片仅会在下次任务触发前执行
* 注册中心数据结构

![image.png](https://upload-images.jianshu.io/upload_images/23653832-dcc9d5eb4acdb00f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 3.2 失效转移

* 开启失效转移功能，ElasticJob 会监控作业每一分片的执行状态，并将其写入注册中心，供其他节点感知。
* 当作业执行节点宕机时，会触发失效转移流程。ElasticJob 根据触发时的分布式作业执行的不同状况来决定失效转移的执行时机。
* 在某台服务器执行完毕后主动抓取未分配的分片，并且在某台服务器下线后主动寻找可用的服务器执行任务。
* 作业服务在本次任务执行结束后，会向注册中心问询待执行的失效转移分片项，如果有，则开始补偿执行。 

![image.png](https://upload-images.jianshu.io/upload_images/23653832-df98a78429f6d8d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.3 幂等控制

* 开启幂等控制后提供的特性
  * 确保本轮定时任务未执行结束的情况下，下轮任务不会执行。
  * 确保重新分片在本轮任务结束之后才进行。从而不影响本轮任务的执行。更精确的说，重新分片不会在有分片正在执行任务时触发。
  * 开启幂等控制后，ElasticJob能感知正在执行处理的分片
* monitorExecution + misfire配置
  * 开启misfire机制，任务执行前会在内存中设置任务为running状态
  * 开启了monitor机制，同时会在zookeeper中创建/sharding/{item}/running临时节点，在下次任务调度到来时，会查看是否存在分片正在执行中running临时节点，如果前面有分片任务在执行，这个时候就会设置任务分片错过misfire，创建/sharding/{item}/misfire节点，待上次作业执行完成的时候，查看是否有错过执行的分片任务，重新补偿执行分片任务。
  * 同时重新分片在本轮任务结束之后进行原理，也是通过判断/sharding/{item}/running节点是否存在即可
  * 但是因为/sharding/{item}/running是临时节点，当节点执行任务宕机，则临时节点会自动删除，会影响幂等控制。





