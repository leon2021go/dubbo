ExtensionLoader

三个静态字段

![diagram](D:\dubboumlpng\diagram.png)



```java

// LoadingStrategy 接口有三个实现（通过 JDK SPI 方式加载的），如上图所示，分别对应前面介绍的三个 Dubbo SPI 配置文件所在的目录，且都继承了 Prioritized 这个优先级接口，默认优先级是 
// DubboInternalLoadingStrategy > DubboExternalLoadingStrategy > DubboLoadingStrategy > ServicesLoadingStrategy
private static volatile LoadingStrategy[] strategies = loadLoadingStrategies();

private static LoadingStrategy[] loadLoadingStrategies() {
        return stream(load(LoadingStrategy.class).spliterator(), false)
                .sorted()
                .toArray(LoadingStrategy[]::new);
}

public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
}

// Key 为扩展接口，Value 为加载其扩展实现的 ExtensionLoader 实例
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>(64);
// 缓存了扩展实现类与其实例对象的映射关系。在前文示例中，Key 为 Class，Value 为 DubboProtocol 对象
private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<>(64);
```

```java
// 当前 ExtensionLoader 实例负责加载扩展接口
private final Class<?> type;
// type 这个扩展接口上 @SPI 注解的 value 值, 默认扩展名
private String cachedDefaultName;
// 缓存了该 ExtensionLoader 加载的扩展实现类与扩展名之间的映射关系
private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<>();
// 该 ExtensionLoader 加载的扩展名与扩展实现类之间的映射关系。cachedNames 集合的反向关系缓存
private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();
// 缓存了该 ExtensionLoader 加载的扩展名与扩展实现对象之间的映射关系
private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>();
```



//Dubbo SPI核心逻辑使用示例：

Protocol protocol = ExtensionLoader 
   .getExtensionLoader(Protocol.class).getExtension("dubbo");



```java
// 从 EXTENSION_LOADERS 缓存中查找相应的 ExtensionLoader 实例 得到接口对应的 ExtensionLoader 对象
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }

    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}


public T getExtension(String name) {
        return getExtension(name, true);
    }

    public T getExtension(String name, boolean wrap) {
        if (StringUtils.isEmpty(name)) {
            throw new IllegalArgumentException("Extension name == null");
        }
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        final Holder<Object> holder = getOrCreateHolder(name);
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    instance = createExtension(name, wrap);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }


	private Holder<Object> getOrCreateHolder(String name) {
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<>());
            holder = cachedInstances.get(name);
        }
        return holder;
    }


	private T createExtension(String name, boolean wrap) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);


            if (wrap) {

                List<Class<?>> wrapperClassesList = new ArrayList<>();
                if (cachedWrapperClasses != null) {
                    wrapperClassesList.addAll(cachedWrapperClasses);
                    wrapperClassesList.sort(WrapperComparator.COMPARATOR);
                    Collections.reverse(wrapperClassesList);
                }

                if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
                    for (Class<?> wrapperClass : wrapperClassesList) {
                        Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
                        if (wrapper == null
                                || (ArrayUtils.contains(wrapper.matches(), name) && !ArrayUtils.contains(wrapper.mismatches(), 					name))) {
                            instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                        }
                    }
                }
            }

            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }

	//先从cachedClasses里获取 如果 cachedClasses 未初始化，则会扫描前面介绍的三个 SPI 目录获取查找相应的 SPI 配置文件，然后加载其中的扩展实现类，最		后将扩展名和扩展实现类的映射关系记录到 cachedClasses 缓存中。这部分逻辑在 loadExtensionClasses() 和 loadDirectory() 方法中
	private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```



@Adaptive 注解

一  

AdaptiveExtensionFactory 不实现任何具体的功能，而是用来适配 ExtensionFactory 的 SpiExtensionFactory 和 SpringExtensionFactory 这两种实现。AdaptiveExtensionFactory 会根据运行时的一些状态来选择具体调用 ExtensionFactory 的哪个实现



![AdaptiveExtensionFactory](D:\dubboumlpng\AdaptiveExtensionFactory.png)



二

加到接口方法之上，Dubbo 会动态生成适配器类



```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.dubbo.remoting;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.Adaptive;
import org.apache.dubbo.common.extension.SPI;

/**
 * Transporter. (SPI, Singleton, ThreadSafe)
 * <p>
 * <a href="http://en.wikipedia.org/wiki/Transport_Layer">Transport Layer</a>
 * <a href="http://en.wikipedia.org/wiki/Client%E2%80%93server_model">Client/Server</a>
 *
 * @see org.apache.dubbo.remoting.Transporters
 */
@SPI("netty")
public interface Transporter {

    /**
     * Bind a server.
     *
     * @param url     server url
     * @param handler
     * @return server
     * @throws RemotingException
     * @see org.apache.dubbo.remoting.Transporters#bind(URL, ChannelHandler...)
     */
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    RemotingServer bind(URL url, ChannelHandler handler) throws RemotingException;

    /**
     * Connect to a server.
     *
     * @param url     server url
     * @param handler
     * @return client
     * @throws RemotingException
     * @see org.apache.dubbo.remoting.Transporters#connect(URL, ChannelHandler...)
     */
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;

}
```

Dubbo 会生成一个 Transporter$Adaptive 适配器实现类，该类继承了 Transporter 接口



public class Transporter$Adaptive implements Transporter { 
    public org.apache.dubbo.remoting.Client connect(URL arg0, ChannelHandler arg1) throws RemotingException { 
        // 必须传递URL参数 
        if (arg0 == null) throw new IllegalArgumentException("url == null"); 
        URL url = arg0; 
        // 确定扩展名，优先从URL中的client参数获取，其次是transporter参数 
        // 这两个参数名称由@Adaptive注解指定，最后是@SPI注解中的默认值 
        String extName = url.getParameter("client",
            url.getParameter("transporter", "netty")); 
        if (extName == null) 
            throw new IllegalStateException("..."); 
        // 通过ExtensionLoader加载Transporter接口的指定扩展实现 
        Transporter extension = (Transporter) ExtensionLoader 
              .getExtensionLoader(Transporter.class) 
                    .getExtension(extName); 
        return extension.connect(arg0, arg1); 
    } 
    ... // 省略bind()方法 
}



生成 Transporter$Adaptive 这个类的逻辑位于 ExtensionLoader.createAdaptiveExtensionClass() 方法



```java
private Class<?> createAdaptiveExtensionClass() {
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    ClassLoader classLoader = findClassLoader();
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```



@Activate 还需继续研究 多看几遍







定时任务

时间轮调度模型：一般会实现成一个环形结构，类似一个时钟，分为很多槽，一个槽代表一个时间间隔，每个槽使用双向链表存储定时任务；指针周期性地跳动，跳动到一个槽位，就执行该槽位的定时任务

Dubbo 的时间轮实现位于 dubbo-common 模块的 org.apache.dubbo.common.timer 包

几个关键的类：

TimerTask： 在 Dubbo 中，所有的定时任务都要继承 TimerTask 接口

```java
/**
 * A task which is executed after the delay specified with
 * {@link Timer#newTimeout(TimerTask, long, TimeUnit)} (TimerTask, long, TimeUnit)}.
 */
public interface TimerTask {

    /**
     * Executed after the delay specified with
     * {@link Timer#newTimeout(TimerTask, long, TimeUnit)}.
     *
     * @param timeout a handle which is associated with this task
     */
    void run(Timeout timeout) throws Exception;
}
```

TimeOut: TimeTask的run方法的入参，通过 Timeout 对象，我们不仅可以查看定时任务的状态，还可以操作定时任务（例如取消关联的定时任务）



Timer: Timer 接口定义了定时器的基本行为

```java
/**
 * Schedules {@link TimerTask}s for one-time future execution in a background
 * thread.
 */
public interface Timer {

    /**
     * Schedules the specified {@link TimerTask} for one-time execution after
     * the specified delay.
     *
     * @return a handle which is associated with the specified task
     * @throws IllegalStateException      if this timer has been {@linkplain #stop() stopped} already
     * @throws RejectedExecutionException if the pending timeouts are too many and creating new timeout
     *                                    can cause instability in the system.
     */
    Timeout newTimeout(TimerTask task, long delay, TimeUnit unit);

    /**
     * Releases all resources acquired by this {@link Timer} and cancels all
     * tasks which were scheduled but not executed yet.
     *
     * @return the handles associated with the tasks which were canceled by
     * this method
     */
    Set<Timeout> stop();

    /**
     * the timer is stop
     *
     * @return true for stop
     */
    boolean isStop();
}
```

HashedWheelTimer:  是 Timer 接口的实现，它通过时间轮算法实现了一个定时器 根据源码 时间轮本体就是HashedWheelTimer类的静态变量：

```java
private final HashedWheelBucket[] wheel;
```

HashedWheelBucket就是时间轮的一个槽，数组wheel就是这个槽组成的时间轮环形队列， 当指定时间轮槽数为 n 时，实际上会取大于且最靠近 n 的 2 的幂次方值，HashedWheelBucket是HashedWheelTimer的内部类

```java
/**
 * Bucket that stores HashedWheelTimeouts. These are stored in a linked-list like datastructure to allow easy
 * removal of HashedWheelTimeouts in the middle. Also the HashedWheelTimeout act as nodes themself and so no
 * extra object creation is needed.
 */
private static final class HashedWheelBucket {

    /**
     * Used for the linked-list datastructure
     */
    private HashedWheelTimeout head;
    private HashedWheelTimeout tail;

    /**
     * Add {@link HashedWheelTimeout} to this bucket.
     */
    void addTimeout(HashedWheelTimeout timeout) {
        assert timeout.bucket == null;
        timeout.bucket = this;
        if (head == null) {
            head = tail = timeout;
        } else {
            tail.next = timeout;
            timeout.prev = tail;
            tail = timeout;
        }
    }

    /**
     * Expire all {@link HashedWheelTimeout}s for the given {@code deadline}.
     */
    void expireTimeouts(long deadline) {
        HashedWheelTimeout timeout = head;

        // process all timeouts
        while (timeout != null) {
            HashedWheelTimeout next = timeout.next;
            if (timeout.remainingRounds <= 0) {
                next = remove(timeout);
                if (timeout.deadline <= deadline) {
                    timeout.expire();
                } else {
                    // The timeout was placed into a wrong slot. This should never happen.
                    throw new IllegalStateException(String.format(
                            "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
                }
            } else if (timeout.isCancelled()) {
                next = remove(timeout);
            } else {
                timeout.remainingRounds--;
            }
            timeout = next;
        }
    }

    public HashedWheelTimeout remove(HashedWheelTimeout timeout) {
        HashedWheelTimeout next = timeout.next;
        // remove timeout that was either processed or cancelled by updating the linked-list
        if (timeout.prev != null) {
            timeout.prev.next = next;
        }
        if (timeout.next != null) {
            timeout.next.prev = timeout.prev;
        }

        if (timeout == head) {
            // if timeout is also the tail we need to adjust the entry too
            if (timeout == tail) {
                tail = null;
                head = null;
            } else {
                head = next;
            }
        } else if (timeout == tail) {
            // if the timeout is the tail modify the tail to be the prev node.
            tail = timeout.prev;
        }
        // null out prev, next and bucket to allow for GC.
        timeout.prev = null;
        timeout.next = null;
        timeout.bucket = null;
        timeout.timer.pendingTimeouts.decrementAndGet();
        return next;
    }

    /**
     * Clear this bucket and return all not expired / cancelled {@link Timeout}s.
     */
    void clearTimeouts(Set<Timeout> set) {
        for (; ; ) {
            HashedWheelTimeout timeout = pollTimeout();
            if (timeout == null) {
                return;
            }
            if (timeout.isExpired() || timeout.isCancelled()) {
                continue;
            }
            set.add(timeout);
        }
    }

    private HashedWheelTimeout pollTimeout() {
        HashedWheelTimeout head = this.head;
        if (head == null) {
            return null;
        }
        HashedWheelTimeout next = head.next;
        if (next == null) {
            tail = this.head = null;
        } else {
            this.head = next;
            next.prev = null;
        }

        // null out prev and next to allow for GC.
        head.next = null;
        head.prev = null;
        head.bucket = null;
        return head;
    }
}
```





回到HashedWheelTimer：

```java
// 真正执行定时任务的逻辑封装这个 Runnable 对象中
private final Worker worker = new Worker();
// 时间轮内部真正执行定时任务的线程
private final Thread workerThread;
```



Dubbo 中对时间轮的应用主要体现在如下两个方面：

失败重试， 例如，Provider 向注册中心进行注册失败时的重试操作，或是 Consumer 向注册中心订阅时的失败重试等。

周期性定时任务， 例如，定期发送心跳请求，请求超时的处理，或是网络连接断开后的重连机制。







## 注册中心

Dubbo中的Registry

#### 1 node接口:抽象节点的概念，comsumer、provider还有注册中心节点

三个方法

getUrl() 方法返回表示当前节点的 URL；

isAvailable() 检测当前节点是否可用；

destroy() 方法负责销毁当前节点并释放底层资源。

#### 2 RegistryService 接口抽象了注册服务的基本行为

```java
// 注册一个URL
void register(URL url);
// 取消注册一个URL
void unregister(URL url);
// 订阅一个URL 订阅成功之后，当订阅的数据发生变化时，注册中心会主动通知第二个参数指定的 NotifyListener 对象，NotifyListener 接口中定义的 notify() 方法就是用来接收该通知的
void subscribe(URL url, NotifyListener listener);
// 取消订阅一个URL
void unsubscribe(URL url, NotifyListener listener);
// 查询符合条件的注册数据 subscribe() 方法采用的是 push 模式，lookup() 方法采用的是 pull 模式
List<URL> lookup(URL url);
```



#### 3 Registry接口：继承了 RegistryService 接口和 Node 接口，一个拥有注册中心能力的节点

#### 4 RegistryFactory 接口是 Registry 的工厂接口，负责创建 Registry 对象

```java
@SPI("dubbo")
public interface RegistryFactory {

    /**
     * Connect to the registry
     * <p>
     * Connecting the registry needs to support the contract: <br>
     * 1. When the check=false is set, the connection is not checked, otherwise the exception is thrown when disconnection <br>
     * 2. Support username:password authority authentication on URL.<br>
     * 3. Support the backup=10.20.153.10 candidate registry cluster address.<br>
     * 4. Support file=registry.cache local disk file cache.<br>
     * 5. Support the timeout=1000 request timeout setting.<br>
     * 6. Support session=60000 session timeout or expiration settings.<br>
     *
     * @param url Registry address, is not allowed to be empty
     * @return Registry reference, never return empty value
     */
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);

}
```

#### 5 AbstractRegistryFactory: 提供了规范 URL 的操作以及缓存 Registry 对象的公共能力

```java
// Registry Collection Map<RegistryAddress, Registry> 缓存 Registry 对象是使用 HashMap<String, Registry> 集合实现的（REGISTRIES 静态字段）
protected static final Map<String, Registry> REGISTRIES = new HashMap<>();
```

#### 6 AbstractRegistry: Registry 接口的所有实现类都继承了 AbstractRegistry

AbstractRegistry 实现了 Registry 接口，虽然 AbstractRegistry 本身在内存中实现了注册数据的读写功能，也没有什么抽象方法，但它依然被标记成了抽象类，从前面的Registry 继承关系图中可以看出，Registry 接口的所有实现类都继承了 AbstractRegistry。

为了减轻注册中心组件的压力，AbstractRegistry 会把当前节点订阅的 URL 信息缓存到本地的 Properties 文件中，其核心字段如下：

registryUrl（URL类型）。 该 URL 包含了创建该 Registry 对象的全部配置信息，是 AbstractRegistryFactory 修改后的产物。

properties（Properties 类型）、file（File 类型）。 本地的 Properties 文件缓存，properties 是加载到内存的 Properties 对象，file 是磁盘上对应的文件，两者的数据是同步的。在 AbstractRegistry 初始化时，会根据 registryUrl 中的 file.cache 参数值决定是否开启文件缓存。如果开启文件缓存功能，就会立即将 file 文件中的 KV 缓存加载到 properties 字段中。当 properties 中的注册数据发生变化时，会写入本地的 file 文件进行同步。properties 是一个 KV 结构，其中 Key 是当前节点作为 Consumer 的一个 URL，Value 是对应的 Provider 列表，包含了所有 Category（例如，providers、routes、configurators 等） 下的 URL。properties 中有一个特殊的 Key 值为 registies，对应的 Value 是注册中心列表，其他记录的都是 Provider 列表。

syncSaveFile（boolean 类型）。 是否同步保存文件的配置，对应的是 registryUrl 中的 save.file 参数。

registryCacheExecutor（ExecutorService 类型）。 这是一个单线程的线程池，在一个 Provider 的注册数据发生变化的时候，会将该 Provider 的全量数据同步到 properties 字段和缓存文件中，如果 syncSaveFile 配置为 false，就由该线程池异步完成文件写入。

lastCacheChanged（AtomicLong 类型）。 注册数据的版本号，每次写入 file 文件时，都是全覆盖写入，而不是修改文件，所以需要版本控制，防止旧数据覆盖新数据。

registered（Set 类型）。 这个比较简单，它是注册的 URL 集合。

subscribed（ConcurrentMap<URL, Set> 类型）。 表示订阅 URL 的监听器集合，其中 Key 是被监听的 URL， Value 是相应的监听器集合。

notified（ConcurrentMap<URL, Map<String, List>>类型）。 该集合第一层 Key 是当前节点作为 Consumer 的一个 URL，表示的是该节点的某个 Consumer 角色（一个节点可以同时消费多个 Provider 节点）；Value 是一个 Map 集合，该 Map 集合的 Key 是 Provider URL 的分类（Category），例如 providers、routes、configurators 等，Value 就是相应分类下的 URL 集合。

介绍完 AbstractRegistry 的核心字段之后，我们接下来就再看看 AbstractRegistry 依赖这些字段都提供了哪些公共能力。

1. 本地缓存
作为一个 RPC 框架，Dubbo 在微服务架构中解决了各个服务间协作的难题；作为 Provider 和 Consumer 的底层依赖，它会与服务一起打包部署。dubbo-registry 也仅仅是其中一个依赖包，负责完成与 ZooKeeper、etcd、Consul 等服务发现组件的交互。

当 Provider 端暴露的 URL 发生变化时，ZooKeeper 等服务发现组件会通知 Consumer 端的 Registry 组件，Registry 组件会调用 notify() 方法，被通知的 Consumer 能匹配到所有 Provider 的 URL 列表并写入 properties 集合中。

下面我们来看 notify() 方法的核心实现：

复制代码
// 注意入参，第一个URL参数表示的是Consumer，第二个NotifyListener是第一个参数对应的监听器，第三个参数是Provider端暴露的URL的全量数据
protected void notify(URL url, NotifyListener listener,
    List<URL> urls) {
    ... // 省略一系列边界条件的检查
    Map<String, List<URL>> result = new HashMap<>();
    for (URL u : urls) {
        // 需要Consumer URL与Provider URL匹配，具体匹配规则后面详述
        if (UrlUtils.isMatch(url, u)) { 
            // 根据Provider URL中的category参数进行分类
            String category = u.getParameter("category", "providers");
            List<URL> categoryList = result.computeIfAbsent(category, 
                k -> new ArrayList<>());
            categoryList.add(u);
        }
    }
    if (result.size() == 0) {
        return;
    }
    Map<String, List<URL>> categoryNotified = 
      notified.computeIfAbsent(url, u -> new ConcurrentHashMap<>());
    for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
        String category = entry.getKey();
        List<URL> categoryList = entry.getValue();
        categoryNotified.put(category, categoryList); // 更新notified
        listener.notify(categoryList); // 调用NotifyListener
        // 更新properties集合以及底层的文件缓存
        saveProperties(url);
    }
}
在 saveProperties() 方法中会取出 Consumer 订阅的各个分类的 URL 连接起来（中间以空格分隔），然后以 Consumer 的 ServiceKey 为键值写到 properties 中，同时 lastCacheChanged 版本号会自增。完成 properties 字段的更新之后，会根据 syncSaveFile 字段值来决定是在当前线程同步更新 file 文件，还是向 registryCacheExecutor 线程池提交任务，异步完成 file 文件的同步。本地缓存文件的具体路径是：

复制代码
/.dubbo/dubbo-registry-[当前应用名]-[当前Registry所在的IP地址].cache
这里首先关注第一个细节：UrlUtils.isMatch() 方法。该方法会完成 Consumer URL 与 Provider URL 的匹配，依次匹配的部分如下所示：

匹配 Consumer 和 Provider 的接口（优先取 interface 参数，其次再取 path）。双方接口相同或者其中一方为“*”，则匹配成功，执行下一步。

匹配 Consumer 和 Provider 的 category。

检测 Consumer URL 和 Provider URL 中的 enable 参数是否符合条件。

检测 Consumer 和 Provider 端的 group、version 以及 classifier 是否符合条件。

第二个细节是：URL.getServiceKey() 方法。该方法返回的 ServiceKey 是 properties 集合以及相应缓存文件中的 Key。ServiceKey 的格式如下：

复制代码
[group]/{interface(或path)}[:version]
AbstractRegistry 的核心是本地文件缓存的功能。 在 AbstractRegistry 的构造方法中，会调用 loadProperties() 方法将上面写入的本地缓存文件，加载到 properties 对象中。

在网络抖动等原因而导致订阅失败时，Consumer 端的 Registry 就可以调用 getCacheUrls() 方法获取本地缓存，从而得到最近注册的 Provider URL。可见，AbstractRegistry 通过本地缓存提供了一种容错机制，保证了服务的可靠性。

2. 注册/订阅
AbstractRegistry 实现了 Registry 接口，它实现的 registry() 方法会将当前节点要注册的 URL 缓存到 registered 集合，而 unregistry() 方法会从 registered 集合删除指定的 URL，例如当前节点下线的时候。

subscribe() 方法会将当前节点作为 Consumer 的 URL 以及相关的 NotifyListener 记录到 subscribed 集合，unsubscribe() 方法会将当前节点的 URL 以及关联的 NotifyListener 从 subscribed 集合删除。

这四个方法都是简单的集合操作，这里我们就不再展示具体代码了。

单看 AbstractRegistry 的实现，上述四个基础的注册、订阅方法都是内存操作，但是 Java 有继承和多态的特性，AbstractRegistry 的子类会覆盖上述四个基础的注册、订阅方法进行增强。



3. 恢复/销毁
AbstractRegistry 中还有另外两个需要关注的方法：recover() 方法和destroy() 方法。

在 Provider 因为网络问题与注册中心断开连接之后，会进行重连，重新连接成功之后，会调用 recover() 方法将 registered 集合中的全部 URL 重新走一遍 register() 方法，恢复注册数据。同样，recover() 方法也会将 subscribed 集合中的 URL 重新走一遍 subscribe() 方法，恢复订阅监听器。recover() 方法的具体实现比较简单，这里就不再展示，你若感兴趣的话，可以参考源码进行学习。

在当前节点下线的时候，会调用 Node.destroy() 方法释放底层资源。AbstractRegistry 实现的 destroy() 方法会调用 unregister() 方法和 unsubscribe() 方法将当前节点注册的 URL 以及订阅的监听全部清理掉，其中不会清理非动态注册的 URL（即 dynamic 参数明确指定为 false）。AbstractRegistry 中 destroy() 方法的实现比较简单，这里我们也不再展示，如果你感兴趣话，同样可以参考源码进行学习。





### dubbo-registry的重试机制

#### FailbackRegistry：覆盖了 AbstractRegistry 中 register()/unregister()、subscribe()/unsubscribe() 以及 notify() 这五个核心方法，结合前面介绍的时间轮，实现失败重试的能力；真正与服务发现组件的交互能力则是放到了 doRegister()/doUnregister()、doSubscribe()/doUnsubscribe() 以及 doNotify() 这五个抽象方法中，由具体子类实现。这是典型的模板方法模式的应用