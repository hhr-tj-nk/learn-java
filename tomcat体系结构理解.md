

# Tomcat体系结构理解


* Tomcat是一个服务器，实现了servlet规范及http服务器功能，以支持客户端请求的处理和响应。

## 1、主要组件

### 1.1 整体组件架构图

![image.png](https://upload-images.jianshu.io/upload_images/23653832-93800d996f4b64cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 2、连接器组件

### 2.1 功能

* 监听客户端建立连接请求，和客户端建立连接
* 提供不同的io模型支持，以处理连接请求、读写事件等。支持NIO、BIO等
* 提供不同的协议支持，以处理不同数据格式
* 将原生数据结构转换为容器可读的数据结构

### 2.2 基本组成

![image.png](https://upload-images.jianshu.io/upload_images/23653832-71566a2264db912b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* EndPoint：终端，通信端点，负责通信相关。支撑传输层。处理tcp协议
* Processor：处理不同协议格式的数据。支撑应用层。将不同协议格式的数据中的具体请求提取出来封装为Tomcat原生Request
* Adapter：将Tomcat原生Request适配为ServletRequest
* ProtocolHandler：ProtocolHandler接口定义了Tomcat IO的基本能力。



### 2.3 基本流程（以NIO模型为例）
![image.png](https://upload-images.jianshu.io/upload_images/23653832-8b3fd11173f0a2fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1. Catalina#start
2. StandardServer#startInternal
3. StandardService#startInternal
4. Connector#startInternal
5. org.apache.coyote.AbstractProtocol#start
6. org.apache.tomcat.util.net.AbstractEndpoint#start
7. NioEndpoint#startInternal
   * 创建Accecptor线程，开始监听客户端连接请求
   * 创建Pollor数组
8. Acceptor#run
   * getPoller0().register(channel)
   * 轮询（自增取余方式）Pollor数组获取Pollor
   * 连接后，客户端连接通道移交Pollor线程
9. Poller#run
   * Pollor线程
   * NioEndpoint.Poller#processKey，处理Nio选择器返回的就绪事件
   * AbstractEndpoint#processSocket
   * NioEndpoint#createSocketProcessor，创建SocketProcessor
   * NioEndpoint.SocketProcessor#doRun，移交Processor组件处理

### 2.4 线程模型
![image.png](https://upload-images.jianshu.io/upload_images/23653832-df079badfb551bd2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* Tomcat服务端Socket通道是阻塞模式，而和客户端建立连接。获取到客户端Socket通道后，设置为非阻塞模式。
* BlockPollor相当于是对java NIO非阻塞模式下，恶劣网络环境或者说是发生网络抖动时的一个处理。
  * 而tomcat的处理是，注册到辅Selector（BlockPollor的Seleclor），当前读写线程阻塞，等待读写重写就绪。
  * 使用辅助Selector的好处是，降低主Selector压力。
  * SelectionKey的attachment持有一个CountdownLatch，BlockPollor获取到就绪事件，使用该CountDownLatch实例唤醒阻塞线程。阻塞线程使用该CountdownLatch阻塞。
  * 注：这个只是是个人的理解

### 2.5 LimitLatch

* 控制连接数，Accecptor的run方法中可以看到建立连接前，有一个countUpOrAwaitConnection，来控制已建立连接数量的上限

```java
countUpOrAwaitConnection();
SocketChannel socket = null;
try {
    // Accept the next incoming connection from the server
    // socket
    socket = serverSock.accept();
} catch (IOException ioe) {
    // We didn't get a socket
    countDownConnection();
    if (running) {
        // Introduce delay if necessary
        errorDelay = handleExceptionWithDelay(errorDelay);
        // re-throw
        throw ioe;
    } else {
        break;
    }
}
```

* countUpOrAwaitConnection方法实现

```java
protected void countUpOrAwaitConnection() throws InterruptedException {
        if (maxConnections==-1) return;
        LimitLatch latch = connectionLimitLatch;
        if (latch!=null) latch.countUpOrAwait();
}
```

使用LimitLatch工具

* LimitLatch
  * 继承AQS组件，复用等待队列和等待逻辑。
  * 因为AQS的state是一个int值，所以额外维护一个AtomicLong作为当前资源数量，和一个final常量limit，作为资源数量最大值。
  * 而LimitLatch能否获取资源，由资源数量自增后，是否超出limit决定，超出后立即自减，复原资源数量。

```java
@Override
protected int tryAcquireShared(int ignored) {
    long newCount = count.incrementAndGet();
    if (!released && newCount > limit) {
        // Limit exceeded
        count.decrementAndGet();
        return -1;
    } else {
        return 1;
    }
}
```



## 3、容器组件

### 3.1 几种容器

![image.png](https://upload-images.jianshu.io/upload_images/23653832-475653f55635e3ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* Catalina下有一个 Server实例（tomcat Server）。Catalina负责解析server.xml（其中的标签和上面的元素一一对应，Catalina、Server、Service......），以此创建一个Server实例。
* 一个Server中可以部署多个服务，即对应多个Service，但是通常只部署一个服务。Server实例负责Service（包含Servlet、Coyote）生命周期管理（启动、初始化、优雅关闭）
* 一个Service由多个Connector和一个Container组成。
* Container：处理Servlet请求，返回相应结果。
  * Engine：servlet引擎。一个Service只有一个Engine，下面可以有多个Host
  * Host：代理主机、域名相关。代表⼀个虚拟主机，或者说⼀个站点（fpg.zcy.com、tyg.zcy.com），可以给Tomcat配置多个虚拟主机地址，⽽⼀个虚拟主机下 可包含多个Context
  * Context：应用程序，一个web应用。下面有多个Servlet
  * Wrapper：Servlet的封装。一个Wrapper对应一个Servlet

### 3.2 容器生命周期管理（初始化，启动等）

* LifeCycle接口

```java
public interface Lifecycle {

    public void init() throws LifecycleException;

    public void start() throws LifecycleException;
    
    public void stop() throws LifecycleException;
    
    public void destroy() throws LifecycleException;
}
```

* 子类LifeCycleBase的start实现，执行顺序

  * init
  * startInternal

  ```java
  public final synchronized void start() throws LifecycleException {
  
          if (state.equals(LifecycleState.NEW)) {
              init();
          } else if (state.equals(LifecycleState.FAILED)) {
              stop();
          } else if (!state.equals(LifecycleState.INITIALIZED) &&
                  !state.equals(LifecycleState.STOPPED)) {
              invalidTransition(Lifecycle.BEFORE_START_EVENT);
          }
  
          try {
              setStateInternal(LifecycleState.STARTING_PREP, null, false);
              startInternal();
              if (state.equals(LifecycleState.FAILED)) {
                  // This is a 'controlled' failure. The component put itself into the
                  // FAILED state so call stop() to complete the clean-up.
                  stop();
              } else if (!state.equals(LifecycleState.STARTING)) {
                  // Shouldn't be necessary but acts as a check that sub-classes are
                  // doing what they are supposed to.
                  invalidTransition(Lifecycle.AFTER_START_EVENT);
              } else {
                  setStateInternal(LifecycleState.STARTED, null, false);
              }
          } catch (Throwable t) {
              // This is an 'uncontrolled' failure so put the component into the
              // FAILED state and throw an exception.
              handleSubClassException(t, "lifecycleBase.startFail", toString());
          }
  }
  ```

  * 大部分容器继承了LifeCycleBase，继承了start模板逻辑

 ![image.png](https://upload-images.jianshu.io/upload_images/23653832-f4abfe14520af999.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* 父容器维护子容器的生命周期

  * 父容器初始化时，触发子容器的初始化

  * 父容器启动时，触发子容器的启动

  * 则从Catalina#start开始，逐级触发子容器的LifeCycleBase#start模板逻辑，先init后start

  * Catalina -> Server -> Service... -> Connector -> ProtocolHandler

    								                      -> Engine -> Host -> Context

### 3.3 容器请求处理

#### 3.3.1 请求路由

* 因为tomcat支持servlet规范和多web应用管理和多域名支持，所以一个请求需要三层路由。

  * 通过请求域名找到对应的Host
  * 一个Host对应多个Context，则通过host寻找到所有相应的Context。通过请求url中context的部分，路由到指定的Context
  * 一个Context对应多个Wrapper（即Servlet），通过url路由到对应的Servlet。

* MappedElement

  * 可以匹配的元素的抽象。例如Host、Context、Wrapper

    ```java
    protected static final class MappedHost extends MapElement<Host> {
    				// 一个Host对应多个Context
            public volatile ContextList contextList;
    }
    
    protected static final class ContextVersion extends MapElement<Context> {
            public final String path;
            public final int slashCount;
            public final WebResourceRoot resources;
            public String[] welcomeResources;
            // Context对应的多个Wrapper
            public MappedWrapper defaultWrapper = null;
            public MappedWrapper[] exactWrappers = new MappedWrapper[0];
            public MappedWrapper[] wildcardWrappers = new MappedWrapper[0];
            public MappedWrapper[] extensionWrappers = new MappedWrapper[0];
            public int nesting = 0;
            private volatile boolean paused;
    
    }
    
    protected static class MappedWrapper extends MapElement<Wrapper> {}
    ```

  * Tomcat定义Mapper类维护所有MappedElement，路由请求时，使用Mapper进行路由

#### 3.3.2 请求处理大致流程

1. CoyoteAdapter#service
   * request、response适配、填充属性
   * Mapper对Request进行匹配并填充匹配结果到Request的属性
   * connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
2. StandardService
   * getContainer()：返回Engine
3. StandardEngine
   * getPipeline ： 返回StandardPipeline（从基类ContainerBase继承的属性）
   * getFirst：StandardEngineValve
   * invoke：request.getHost().getPipeline().getFirst().invoke(request, response)
4. StandardHostValve
   * request.getContext().getPipeline().getFirst().invoke(request, response)
5. StandardContextValve
   * request.getWrapper().getPipeline().getFirst().invoke(request, response)
6. StandardWrapperValve#invoke

```java
// 单例模式创建，并且创建后执行servlet init方法
// 注：还有个SingleThreadModel模式，
//    如果Servlet实现SingleThreadModel接口，则一次请求就创建一次Servlet。
servlet = wrapper.allocate();
// Create the filter chain for this request
// 先执行过滤器链，然后执行servlet.service
ApplicationFilterChain filterChain =
             ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);
filterChain.doFilter(request.getRequest(), response.getResponse());
```

7. 总结：也是父容器子容器不断进行逐级调用

Connector -> Service -> Engine -> Host -> Context -> Wrapper -> FilterChain -> Servlet





