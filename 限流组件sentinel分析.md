## 1、Sentinel介绍
* 面向云原生，提供熔断降级、流量管控功能
* 可独⽴部署Dashboard/控制台组件，支持在控制台上添加熔断降级、流量管控规则
* 动态修改规则，不用重新部署
* 支持配置持久化，但是要整合nacos、zookeeper等
* 对Dubbo和Spring Cloud都有很好的支持
## 2、sentinel原理分析
### 2.1 资源和功能插槽
* sentinel将需要进行规则控制的对象抽象为资源，每个资源都有自己的功能插槽，常见的功能插槽有例如指标统计slot、流控slot、降级slot等，方便扩展
```java
// NodeSelectorSlot 负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；
// ClusterBuilderSlot 则用于存储资源的统计信息以及调用者信息，例如该资源的 RT, QPS, thread count 等等，这些信息将用作为多维度限流，降级的依据；
// StatisticSlot 则用于记录、统计不同纬度的 runtime 指标监控信息；
// FlowSlot 则用于根据预设的限流规则以及前面 slot 统计的状态，来进行流量控制；
// AuthoritySlot 则根据配置的黑白名单和调用来源信息，来做黑白名单控制；
// DegradeSlot 则通过统计信息以及预设的规则，来做熔断降级；
// SystemSlot 则通过系统的状态，例如 load1 等，来控制总的入口流量；
public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();
        chain.addLast(new NodeSelectorSlot());
        chain.addLast(new ClusterBuilderSlot());
        chain.addLast(new LogSlot());
        chain.addLast(new StatisticSlot());
        chain.addLast(new SystemSlot());
        chain.addLast(new AuthoritySlot());
        chain.addLast(new FlowSlot());
        chain.addLast(new DegradeSlot());

        return chain;
}
```
* 资源和插槽的对应关系，统一存储在全局的一个静态Map中，如何对该静态Map进行高效的线程安全的读写，显然是对其性能好坏的一个关键因素
* sentinel使用Copy-On-Write思想，对该Map进行维护
```java
// 使用 volatile 保证 synchronized关键字之外的内存可见性
private static volatile Map<ResourceWrapper, ProcessorSlotChain> chainMap
        = new HashMap<ResourceWrapper, ProcessorSlotChain>();
 
	// 根据资源获取对应的SlotChain
 ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) { 
        ProcessorSlotChain chain = chainMap.get(resourceWrapper);
        if (chain == null) {
            synchronized (LOCK) {
                chain = chainMap.get(resourceWrapper);
                if (chain == null) {
                    // Entry size limit.
                    if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                        return null;
                    }

                    chain = SlotChainProvider.newSlotChain();
                  // 创建一个新的Map，对老Map完全不加锁，则除了第一次的资源请求，其他资源请求都不受影响
                    Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(
                        chainMap.size() + 1);
                    newMap.putAll(chainMap);
                    newMap.put(resourceWrapper, chain);
                    chainMap = newMap;
                }
            }
        }
        return chain;
}
```
* 则会产生阻塞的只是增加元素的操作，**时间花费只与资源的数量成正比，正常应用资源个数一般在数千以内**，并且有最大值限制，默认为6000，所以实际不会耗费太多时间
  * 应用中一般rest api或者rpc会成为资源，但是现在的rest往往使用restful风格，会导致sentinel资源数量的迅速上升，并因为资源数量限制，导致无法进入规则控制，所以通常需要重写sentinel的UrlCleaner，以对url进行清洗
### 2.2 高效的指标统计
* sentinel的指标统计是基于时间窗口的，过期的统计数据要定期删除。则如何进行高效的基于时间窗的指标统计及定期清理，则是性能好坏的另一个关键因素
* sentinel使用了环形时间窗数组，维护统计数据，则过期的时间窗会被直接复用，没有了清理问题
```java
public abstract class LeapArray<T> {

    protected int windowLengthInMs;
    protected int sampleCount;
    protected int intervalInMs;
    // 使用了AtomicReferenceArray，以CAS方式设置数组元素
    protected final AtomicReferenceArray<WindowWrap<T>> array;
}
```
* 获取当前时间窗的具体实现
```java
// 获取当前时间窗
public WindowWrap<T> currentWindow(long timeMillis) {
    if (timeMillis < 0) {
        return null;
    }

    int idx = calculateTimeIdx(timeMillis);
    // Calculate current bucket start time.
  // 当前时间戳，应该除于哪个时间窗口内，返回这个时间窗口的起始时间
    long windowStart = calculateWindowStart(timeMillis);

    while (true) {
        WindowWrap<T> old = array.get(idx);
        if (old == null) {
           
          // 当前时间戳，还没有创建时间窗口
            WindowWrap<T> window = new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
          
          // CAS创建之
            if (array.compareAndSet(idx, null, window)) {
                // Successfully updated, return the created bucket.
                return window;
            } else {
                // Contention failed, the thread will yield its time slice to wait for bucket available.
              // 失败就直接等待一会儿，其他线程应该是已经成功创建了这个时间窗口，之后做该时间窗口的统计指标更新即可
                Thread.yield();
            }
        } else if (windowStart == old.windowStart()) {
            
          // 时间窗口已经存在，返回即可
            return old;
        } else if (windowStart > old.windowStart()) {
           
          //数组是环形的，要重复利用，这里已经重新回到前面的bucket了，则该bucket已经废弃了，需要重置
          // 这里也是唯一需要加锁的地方，因为重置操作，子类可能会有些其他的处理
        // 并且是tryLock，一个线程重置即可，其他的线程不用阻塞等待
            if (updateLock.tryLock()) {
                try {
                    // Successfully get the update lock, now we reset the bucket.
                    return resetWindowTo(old, windowStart);
                } finally {
                    updateLock.unlock();
                }
            } else {
                Thread.yield();
            }
        } else if (windowStart < old.windowStart()) {
            // Should not go through here, as the provided time is already behind.
            return new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
        }
    }
}	
```
