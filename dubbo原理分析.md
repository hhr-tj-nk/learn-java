- [1、Dubbo SPI]
    - [1.1 扩展类加载器]
    - [1.2 普通扩展]
    - [1.3 Adaptive扩展]
    - [1.4 Activate扩展]
- [2、Dubbo调用流程]
    - [2.1 调用流程概览]
    - [2.2 服务端启动及调用流程总结]
      [启动流程]
    - [2.3 消费端]
- [3、dubbo负载均衡总结]
   - [3.1 带权重的随机负载均衡]
   - [3.2 最小活跃数调用]
   - [3.3 一致性hash+虚拟节点]
   - [3.4 带权重的轮询]

   



## 1、Dubbo SPI
[^xr1]:1

* 声明了@SPI注解的接口，可以通过扩展加载器，获取普通扩展、自适应扩展和Activate扩展，并且支持缓存、依赖注入（setter注入）、包装类自动嵌套（构造器注入）


### 1.1 扩展类加载器
[^xr2]:2
```java
public class ExtensionLoader<T> {
  // 静态方法，获取SPI接口的扩展类加载器实例
   public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
        }
        // 是否声明了@SPI注解
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type (" + type +
                    ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
        }

        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            // 缓存
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }


    // 构造器
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        //  默认实现，AdaptiveExtensionFactory
        //  获取自适应扩展或者Spring bean
        objectFactory = (type == ExtensionFactory.class ?
                null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
}
```
* 每个SPI接口，都有自己的扩展类加载器，通过静态方法获取
* 创建好的扩展类加载器会被缓存

### 1.2 普通扩展
[^xr15]:15
* 和普通策略模式类似，但是Dubbo的普通扩展增加了缓存、setter注入、包装类嵌套特性
```java
// ExtensionLoader
private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                // 缓存
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }

            // 寻找setter，注入依赖的对象
            // 依赖的对象可能是Spring上下文的bean，或者是Dubbo SPI的自适应扩展 
            // 见 SpringExtensionFactory、AdaptiveExtensionFactory
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            // wrapperClasses，即包装类
            // 当前SPI接口的扩展中，构造器只有一个参数，并且类型就是当前SPI接口类型
            // 用于包装类，实现增强逻辑。典型的例如：
            //                  MockClusterWrapper增强普通Cluster的join
            //                  ProtocolFilterWrapper增强普通Protocol的export、refer
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    // 包装类再进行setter注入
                    instance = injectExtension(
                            // 包装类层层嵌套
                            (T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
           // Lifecycle#initialize()
            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
}
```

### 1.3 Adaptive扩展
[^xr3]:3
* 自适应代理类通常基于一个SPI接口进行创建，该接口声明了@SPI注解，接口中有一些声明了@Adative注解的方法
* 使用示例
```java
Protocol PROTOCOL =   
          ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()
```
* 功能
  * 运行时动态扩展，运行时根据调用入参，动态决定具体的扩展类
   * 例：
```java
@SPI
public interface MySpi {

    @Adaptive
    void say(URL url);
}

// 实现1，打印bye
public class SayByeExtension implements MySpi {
    @Override
    public void say(URL url) {
        System.out.println("bye");
    }
}

// 实现2，打印hello
public class SayHelloExtension implements MySpi {
    @Override
    public void say(URL url) {
        System.out.println("hello");
    }
}

public class MySPIDemo {
    // 测试，新建文件 org.apache.dubbo.demo.spi.MySPI，内容为：
    //  sayHello=org.apache.dubbo.demo.spi.SayHelloExtension
    //  sayBye=org.apache.dubbo.demo.spi.SayByeExtension
    public static void main(String[] args) {
        URL url = new URL("dubbo", "127.0.0.1", 8080);
        MySpi adaptiveExtension = ExtensionLoader.getExtensionLoader(MySpi.class).getAdaptiveExtension();

        // StringUtils.camelToSplitName(MySPI.class.getSimpleName(), ".")
        // 输出  hello
        URL sayHelloUrl = url.addParameter("my.spi", "sayHello");
        adaptiveExtension.say(sayHelloUrl);

        // 输出 bye
        URL sayByeUrl = url.addParameter("my.spi", "sayBye");
        adaptiveExtension.say(sayByeUrl);
    }
}
``` 
* 通过对@Adaptive方法中Url类型增加参数，指定运行时具体调用的普通扩展名，从而决定真正调用的扩展
* 如果入参包含返回值类型为Url的getter，也会自动生成调用这个入参Url Getter的代码，以获取Url
* 相关源码
```java
// AdaptiveClassCodeGenerator#generateMethodContent
private String generateMethodContent(Method method) {
        Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
        StringBuilder code = new StringBuilder(512);
        if (adaptiveAnnotation == null) {
            return generateUnsupported(method);
        } else {
            // 查找Url类型入参
            int urlTypeIndex = getUrlTypeIndex(method);

            // found parameter in URL type
            if (urlTypeIndex != -1) {
                // Null Point check
                code.append(generateUrlNullCheck(urlTypeIndex));
            } else {
                // 未找到Url类型，遍历入参中是否有Url类型的getter
                // 找到的话，生成相应的代码 ，类似于 Url url = argi.getUrl();
                code.append(generateUrlAssignmentIndirectly(method));
            }

            // 方法的@Adaptive注解的value，未配置，使用类名小写+.
            String[] value = getMethodAdaptiveValue(adaptiveAnnotation);

            // 入参是否有 Invocation 类型，有的话，获取Url参数时，调用 getMethodParameter()
            boolean hasInvocation = hasInvocationArgument(method);
            code.append(generateInvocationArgumentNullCheck(method));

            // 生成根据Url获取扩展名的方法，
            // 例：String extName = url.getParameter
            code.append(generateExtNameAssignment(value, hasInvocation));
            // check extName == null?
            code.append(generateExtNameNullCheck(value));
            // 生成根据扩展名获取普通扩展的代码
            code.append(generateExtensionAssignment());

            // return statement
            code.append(generateReturnAndInvocation(method));
        }

        return code.toString();
}
```

### 1.4 Activate扩展
[^xr4]:4
* 获取一组普通扩展，并根据分组、Url中的key进行过滤、排序等，典型应用为Dubbo获取Filter时
```java
List<Filter> filters = ExtensionLoader
              .getExtensionLoader(Filter.class)
              .getActivateExtension(invoker.getUrl(), key, group);
```
* 相关源码
```java
/**
* @param values extension point names，url中通过key配置的过滤器名，例如service.filter
* @param group  group，当前需要获取哪个分组，例如CommonConstants.PROVIDER
*/
public List<T> getActivateExtension(URL url, String[] values, String group) {
        List<T> activateExtensions = new ArrayList<>();
        List<String> names = values == null ? new ArrayList<>(0) : asList(values);
        // -default 代表不过滤
        if (!names.contains(REMOVE_VALUE_PREFIX + DEFAULT_KEY)) {
            // 当前类加载器如果没有加载过，加载并缓存
            getExtensionClasses();
            for (Map.Entry<String, Object> entry : cachedActivates.entrySet()) {
                String name = entry.getKey();
                Object activate = entry.getValue();

                String[] activateGroup, activateValue;

                if (activate instanceof Activate) {
                    activateGroup = ((Activate) activate).group();
                    activateValue = ((Activate) activate).value();
                } else {
                    continue;
                }
                // 按照分组过滤
                if (isMatchGroup(group, activateGroup)
                        && !names.contains(name)
                        // -filer名 表示不要这个过滤器
                        && !names.contains(REMOVE_VALUE_PREFIX + name)
                        // @Active(value="key1:value1, key2:value2")，url参数中key存在，并且value相等
                        // 从代码看，多个key value应该是或的关系
                        && isActive(activateValue, url)) {
                    // 获取普通扩展
                    activateExtensions.add(getExtension(name));
                }
            }
            activateExtensions.sort(ActivateComparator.COMPARATOR);
        }
        List<T> loadedExtensions = new ArrayList<>();
        for (int i = 0; i < names.size(); i++) {
            String name = names.get(i);
            if (!name.startsWith(REMOVE_VALUE_PREFIX)
                    && !names.contains(REMOVE_VALUE_PREFIX + name)) {
                if (DEFAULT_KEY.equals(name)) {
                    if (!loadedExtensions.isEmpty()) {
                        activateExtensions.addAll(0, loadedExtensions);
                        loadedExtensions.clear();
                    }
                } else {
                    loadedExtensions.add(getExtension(name));
                }
            }
        }
        if (!loadedExtensions.isEmpty()) {
            activateExtensions.addAll(loadedExtensions);
        }
        return activateExtensions;
}
```

## 2、Dubbo调用流程
[^xr5]:5
### 2.1 调用流程概览
[^xr6]:6
![image.png](https://upload-images.jianshu.io/upload_images/23653832-d80f221e3452efa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.2 服务端启动及调用流程总结
[^xr7]:7
#### 启动流程
[^xr8]:8
* 将服务实例封装为Invoker，因为Dubbo整体的调用流程都是基于Invoker，入参为Invocation，出参为AsyncRpcResult，所以需要这么一层Invoker对服务实例的调用进行适配
```java
   // JdkProxyFactory#getInvoker
   public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        return new AbstractProxyInvoker<T>(proxy, type, url) {
          // 重新AbstractProxyInvoker#doInvoke，AbstractProxyInvoker#invoke方法中，进行入参和出参的适配封装
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                Method method = proxy.getClass().getMethod(methodName, parameterTypes);
              // 服务实例，反射调用
                return method.invoke(proxy, arguments);
            }
        };
    }

    // AbstractProxyInvoker#invoke
    public Result invoke(Invocation invocation) throws RpcException {
        try {
          //  Invocation 转为调用服务实例所需要的信息
            Object value = doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments());
			CompletableFuture<Object> future = wrapWithFuture(value);
            // 对出参和异常进行封装
            CompletableFuture<AppResponse> appResponseFuture = future.handle((obj, t) -> {
                AppResponse result = new AppResponse();
                if (t != null) {
                    if (t instanceof CompletionException) {
                        result.setException(t.getCause());
                    } else {
                        result.setException(t);
                    }
                } else {
                    result.setValue(obj);
                }
                return result;
            });
            return new AsyncRpcResult(appResponseFuture, invocation);
        } catch (InvocationTargetException e) {
            if (RpcContext.getContext().isAsyncStarted() && !RpcContext.getContext().stopAsync()) {
                logger.error("Provider async started, but got an exception from the original method, cannot write the exception back to consumer because an async result may have returned the new thread.", e);
            }
            return AsyncRpcResult.newDefaultAsyncResult(null, e.getTargetException(), invocation);
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
* 服务暴露，首先创建通信服务器，以接收、处理客户端请求
```java
// DubboProtocol#openServer
private void openServer(URL url) {
        // find server.
        String key = url.getAddress();
        //client can export a service which's only for server to invoke
        boolean isServer = url.getParameter(IS_SERVER_KEY, true);
        if (isServer) {
            ProtocolServer server = serverMap.get(key);
            if (server == null) {
                synchronized (this) {
                    server = serverMap.get(key);
                    if (server == null) {
                        // 创建通信服务器
                        serverMap.put(key, createServer(url));
                    }
                }
            } else {
                // server supports reset, use together with override
                server.reset(url);
            }
        }
}
```
* 创建后，将Invoker封装为Exporter，并将服务key和Exporter映射起来，存入map，则服务端处理请求时，通过服务key去找到Exporter，就能找到Invoker进行调用，见：DubboProtocol#requestHandler

* 将通信服务器信息注册到注册中心
```java
        // RegistryProtocol#export
        // url to registry
        final Registry registry = getRegistry(originInvoker);
        final URL registeredProviderUrl = getUrlToRegistry(providerUrl, registryUrl);

        // decide if we need to delay publish
        boolean register = providerUrl.getParameter(REGISTER_KEY, true);
        if (register) {
            register(registryUrl, registeredProviderUrl);
}
```
### 2.3 消费端
[^xr9]:9
* 消费者的几个关键组件
    * Directory
        * 对内：监听注册中心配置，刷新通信客户端列表和路由规则
        * 对外：输出经过路由规则过滤后的Invoker列表（通信客户端列表）
  * LoadBalance：负载均衡组件
  * ClusterInvoker：封装容错调用逻辑，向Directory组件获取Invoker列表，通过LoadBalance组件进行负载均衡，选出一个Invoker进行调用
  ```java
    // AbstractClusterInvoker
    @Override
    public Result invoke(final Invocation invocation) throws RpcException {
        checkWhetherDestroyed();

        // binding attachments into invocation.
        Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
        if (contextAttachments != null && contextAttachments.size() != 0) {
            ((RpcInvocation) invocation).addObjectAttachments(contextAttachments);
        }
      
        // directory.list(invocation)
        List<Invoker<T>> invokers = list(invocation);
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        return doInvoke(invocation, invokers, loadbalance);
    }
  ```
* 创建消费代理
消费者配置是一个工厂Bean，则getObject即其创建入口，见ReferenceBean#getObject

## 3、dubbo负载均衡总结
[^xr10]:10
* 权重预热
  * 在达到预热时间之前，权重的值都是取启动时间/预热时间/原始权重，则权重的值会是1到原始权重之间的一个值，并随着启动时间的增加，满满变大，逐渐恢复到原始权重

### 3.1 带权重的随机负载均衡
[^xr11]:11

* 默认算法，实现简单，基本最后是按权重值均匀调用
* 如果权重都一样，那么随机调用
* 如果权重不一样，则取[0，权重]之间的一个数字，对所有节点权重进行轮询，不断使用这个数字减去节点权重，先减到负数，代表这次调用的就是这个节点



### 3.2 最小活跃数调用
[^xr12]:12

* 使用过滤器，使用一个AtomicInteger，记录方法的调用量统计，调用前自增，调用后自减，则当前值是活跃调用数量
* 调用前，会检查这个活跃调用数量，如果过大，会阻塞，直到小于临界值，再对节点进行调用。
* 负载均衡：从节点中，挑一个最小活跃数量最小的，如果有多个，根据权重选择一个。如果权重也一样，随机选一个
* 服务器之间，性能有较大差异时，可能这个策略有些用



### 3.3 一致性hash+虚拟节点
[^xr13]:13

* TreeMap模拟哈希环，每个节点放一些虚拟节点到TreeMap，Hash值通过节点地址（ip+port）+自增数字计算。
* 负载均衡选择节点，将调用参数拼起来，计算hash，返回Treemap中大于等于key的Entry。如果不存在，返回Treemap中最小的Entry，由此形成了一个环。
* 一些使用了本地缓存的服务端，相同参数会调用相同的节点，提高本地缓存命中率



### 3.4 带权重的轮询
[^xr14]:14
* 平滑加权算法
  * 普通加权算法生成的序列，例如A、B两个节点，A权重3、B权重1，则可能生成的调用序列是A A A  B，这显然对A不友好
  * 而平滑加权生成的序列在保证了权重的前提下，分布会更为均匀
* 基本实现原理
  * 每轮都选出权重最大的，但是每轮被选出的节点减去当前轮的总权重（从而降低了自己的权重，减少了连续被选中的几率）
  * 每轮开始前，节点都加上自己的原始权重。（从而，最终还是原始权重越大的，选中几率也还是越大）
