# seata介绍
## 1、介绍
一个分布式事务解决方案
## 2、seata_at模式
对业务代码侵入极小，只需要加注解即可实现对 ACID 事务的关系型数据库的分布式事务管理。

# seata_at原理解析

## 1、基本原理

* 三个角色：TC、TM、RM
* TM：全局事务管理。全局事务发起，提交，回滚。
* TC：事务服务器，维护全局事务及分支事务。（TC有多种存储模式：DB存储，本地文件存储等）
* RM：本地事务代理，创建前后镜像，以生成undoLog用于回滚。上报本地事务状态给TC，对于TC来说则认为是分支事务。


## 2、基本流程图

* 假设全局事务由两个服务组成，本应用的订单服务和一个下游应用的库存服务。
大致的创建订单代码逻辑如下：
```
public void create() {
    // 本地创建订单
    orderMapper.create();
    // 调用库存服务
    Checkers.check(stockService.change());
}
```
则基本流程如下：
![未命名文件 (5).png](https://upload-images.jianshu.io/upload_images/23653832-becbd84103a15c0c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)







* 可以看到TM代理订单服务，进行全局事务管理。订单本身的订单表和库存服务的库存表操作分别被RM代理。由TM决定是否进行全局事务的提交还是全局事务的回滚。由RM上报给TC自己分支事务的状态。

* 全局事务隔离级别为读已提交：可以看到本地事务在全局事务提交前，已经提交。

* 写隔离：对全局事务中涉及到的所有行加锁。事务结束，释放这些锁。
具体实现为（tc为db存储时）：使用专门的表（lock_table，表名可配置）实现锁，数据库类型+表名+涉及数据行主键组成为rowkey，rowKey作为lock_table表主键，具有唯一性。（lock_table表设计：[https://github.com/seata/seata/blob/1.0.0/script/server/db/mysql.sql](https://github.com/seata/seata/blob/1.0.0/script/server/db/mysql.sql)）

* 主键：全局事务涉及到的表最好都有主键。
* XID：XID在全局事务中透传，每个服务感知到XID，则发觉自己处于某个全局事务中。例如dubbo作为rpc框架，则seata通过重写filter，将XID放入dubbo的rpc context。

* 配置 client.report.success.enable：配置为true，则当分支事务提交成功本地事务时，会上报给TC更新该分支事务状态。根据官网介绍，该参数设置为false时可以减少一半rpc。这个成功提交应该确实也不用上报给TC，全局提交时，提交本地事务成功的分支事务可以忽略。全局回滚时，分支事务已经成功注册，通知该分支事务回滚即可。分支事务提交本地事务失败时，一定会上报状态，全局回滚时会直接忽略掉这个分支事务。


# Undo Log
## 1、作用
 undolog由本地事务提交前置镜像和后置镜像组成。全局事务发起分支事务回滚本地事务时，用前置镜像或后置镜像回滚。例如回滚插入语句时，使用后置镜像的主键id，进行删除操作。

## 2、创建前置镜像和后置镜像

| | 前置镜像 | 后置镜像 |
| :-----| :----: | :----: |
| 插入 | 空 | 根据主键id查出插入后的值 |
| 删除 |使用当前读（for update）查询出数据库的值 | 空 |
| 更新 | 使用当前读（for update）查询出数据库中最新值，并只查询更新涉及到的字段的值 | 使用快照读查询 |

* 代理了PreparedStatement的execute系列方法，execute前，创建前置镜像，execute后，创建后置镜像。


## 3、undo log 失效场景

* 前置镜像等于后置镜像，跳过该回滚
* 根据查询当前的数据库值，如果不等于后置镜像，则产生了脏数据，放弃回滚。（有一种可能，该服务有未纳入seata全局事务管理的流程。或者有人手动修改了数据等等）。如果当前值等于前置镜像，则仅打印日志。否则抛出异常。


# 一些问题和思考

## TM扩展
*  实现io.seata.tm.api.FailureHandler处理全局事务运行中发生的已知异常
*  实现io.seata.tm.api.transaction.TransactionHook接口，对全局事务各个阶段实现增强逻辑。

## 2、表信息
at模式会缓存表信息，表名、主键名称等，用于at模式自己查询、删除。所以如果接入了seata at模式，可能表的现有信息要禁止改动。

## 3、集成SpringBoot时的配置方式
实战中发现，集成SpringBoot的话，相关的客户端配置可以写到application.yml中，并且需要加上seata前缀。（加上seata-spring-boot--starter模块时，seata spi获取到的配置类实现为一个适配SpringBoot的配置类SpringBootConfigurationProvider）

例：
```
seata:
  applicationId: ${dubbo.application.name}
  tx-service-group: hhr_at_test_order
  service:
    vgroup-mapping:
      hhr_at_test_order: hhr_at_test
```


## 4、调用的dubbo服务超时会怎么样？
假设这样一个场景。订单服务，调用库存服务。库存服务超时，订单服务察觉到超时，抛出超时异常，发起全局事务回滚。但是同时库存服务执行结束，发起分支事务注册。这种情况会怎么样？

* 从订单服务看，tm发起全局事务回滚，tc通知所有rm进行回滚，但是因为库存服务还未进行分支服务注册，所以不会通知库存服务的rm进行回滚。
* 从库存服务看，发起分支服务注册，可以看到tc的相关代码中会对全局事务上锁（全局事务提交也会上锁，这两个是同一个锁），并判断全局事务状态，如果全局事务状态是已经关闭，则不能注册。注册失败后，库存服务的rm不会执行本地事务的提交。（如果这个锁能锁住的话）

```
GlobalSession globalSession = assertGlobalSessionNotNull(xid, false);
        return globalSession.lockAndExcute(() -> {
            if (!globalSession.isActive()) {
                throw new GlobalTransactionException(GlobalTransactionNotActive, String
                    .format("Could not register branch into global session xid = %s status = %s",
                        globalSession.getXid(), globalSession.getStatus()));
            }
```

* 锁实现
```

private static class GlobalSessionLock {
        private Lock globalSessionLock = new ReentrantLock();

        private static final int GLOBAL_SESSION_LOCK_TIME_OUT_MILLS = 2 * 1000;

        public void lock() throws TransactionException {
            try {
                if (globalSessionLock.tryLock(GLOBAL_SESSION_LOCK_TIME_OUT_MILLS, TimeUnit.MILLISECONDS)) {
                    return;
                }
            } catch (InterruptedException e) {
                LOGGER.error("Interrupted error", e);
            }
            throw new GlobalTransactionException(TransactionExceptionCode.FailedLockGlobalTranscation, "Lock global session failed");
        }

        public void unlock() {
            globalSessionLock.unlock();
        }

    }
```

这个锁似乎就是一个ReentrantLock。如果tc是单机模式+文件存储，应该是可以锁住的并避免上述问题，因为GlobalSession仅创建一次，然后一直存在内存中。如果是集群模式+DB存储，那么应该是锁不住的，并且理论上应该有可能出现上述问题，即分支事务提交本地事务成功，但是全局事务已经回滚。（未经过验证）

* 在git上也找到了类似的issue [https://github.com/seata/seata/issues/2026](https://github.com/seata/seata/issues/2026)，上面最后提到db模式还会不断重构。



