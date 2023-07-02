# 简介
Netty是一个异步事件驱动的网络应用框架，用于快速开发可维护的高性能服务器和客户端。 其封装了 JDK 的 NIO，降低了用户的使用难度。社区活跃成熟稳定，经历了大型项目的使用和考验，很多开源项目比如我们常用的 Dubbo、RocketMQ、Elasticsearch、gRPC 等等都用到了 Netty。
并且还有以下优点：

- 统一的 API，支持多种传输类型，阻塞和非阻塞的
- 自带编解码器解决 TCP 粘包/拆包问题
- 自带各种协议栈
- 真正的无连接数据包套接字支持
- 比直接使用 Java 核心 API 有更高的吞吐量、更低的延迟、更低的资源消耗和更少的内存复制
- 解决了 JDK 的很多包括空轮询在内的 Bug
# 一个简单的 demo
## 引入 maven 依赖
使用 idea 的 maven 工程， 本系列 Netty 的分析都是基于 4.1.6.Final 版本的
```java
      <dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-all</artifactId>
        <version>4.1.6.Final</version>
    </dependency>
```

## 服务端代码
```java
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.StandardCharsets;
import java.util.Date;

public class ServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf byteBuf = (ByteBuf) msg;

        System.out.println(new Date() + ": 服务端读到数据 -> " + byteBuf.toString(StandardCharsets.UTF_8));

        // 回复数据到客户端
        ByteBuf out = getByteBuf(ctx);
        ctx.channel().writeAndFlush(out);
    }

    private ByteBuf getByteBuf(ChannelHandlerContext ctx) {
        byte[] bytes = "服务端已收到消息～".getBytes(StandardCharsets.UTF_8);

        ByteBuf buffer = ctx.alloc().buffer();

        buffer.writeBytes(bytes);
        System.out.println("服务端发出数据: 服务端已收到消息～");
        return buffer;
    }
}
```

```java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelOption;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class NettyServer {

    private static final int PORT = 8000;

    public static void main(String[] args) {
        NioEventLoopGroup boosGroup = new NioEventLoopGroup();
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();

        final ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap
                .group(boosGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .childOption(ChannelOption.TCP_NODELAY, true)
                .childHandler(new ServerHandler());


        serverBootstrap.bind(PORT);
    }
}
```

## 客户端代码
```java
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.StandardCharsets;
import java.util.Date;

public class ClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) {

        // 1.获取数据
        ByteBuf buffer = getByteBuf(ctx);

        // 2.写数据
        ctx.channel().writeAndFlush(buffer);
    }

    private ByteBuf getByteBuf(ChannelHandlerContext ctx) {
        byte[] bytes = "你好! 服务端～".getBytes(StandardCharsets.UTF_8);

        ByteBuf buffer = ctx.alloc().buffer();

        buffer.writeBytes(bytes);
        System.out.println("客户端发出数据: 你好! 服务端～");
        return buffer;
    }


    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf byteBuf = (ByteBuf) msg;

        System.out.println(new Date() + ": 客户端读到数据 -> " + byteBuf.toString(StandardCharsets.UTF_8));
    }
}
```

```java
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

public class NettyClient {
    private static final String HOST = "127.0.0.1";
    private static final int PORT = 8000;


    public static void main(String[] args) {
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();

        Bootstrap bootstrap = new Bootstrap();
        bootstrap
                .group(workerGroup)
                .channel(NioSocketChannel.class)
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .option(ChannelOption.TCP_NODELAY, true)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    public void initChannel(SocketChannel ch) {
                        ch.pipeline().addLast(new ClientHandler());
                    }
                });

        bootstrap.connect(HOST, PORT);
    }
}
```

分别运行 NettyServer 和 NettyClient 可以看到如下结果
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1688288268729-2362a866-154d-405f-a317-081265ce7b34.png#averageHue=%233c3536&clientId=u787d0ba1-aa2c-4&from=paste&height=268&id=ub7640c2e&originHeight=536&originWidth=1364&originalType=binary&ratio=2&rotation=0&showTitle=false&size=142406&status=done&style=none&taskId=u51b4588d-18c5-487f-9baa-a5f456481e9&title=&width=682)![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1688288277735-1421e487-1d2d-4910-8dc8-bb650672c774.png#averageHue=%23ebb901&clientId=u787d0ba1-aa2c-4&from=paste&height=291&id=u374720bb&originHeight=582&originWidth=1264&originalType=binary&ratio=2&rotation=0&showTitle=false&size=143129&status=done&style=none&taskId=u6d1fe1ec-ec1d-42a9-925f-837901c6247&title=&width=632)


# Netty 服务端启动流程详解
以下是 Netty 服务端启动的详细代码和流程讲解
```java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelOption;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class NettyServer {

    private static final int PORT = 8000;

    public static void main(String[] args) {
        NioEventLoopGroup boosGroup = new NioEventLoopGroup();
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();

        final ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap
                .group(boosGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .childOption(ChannelOption.TCP_NODELAY, true)
                .childHandler(new ServerHandler());


        serverBootstrap.bind(PORT);
    }
}
```

1. 首先创建了两个NioEventLoopGroup，bossGroup和 workerGroup
   1. bossGroup主要负责接收客户端的连接请求
      1. 一般使用一个单独的线程或线程池来监听指定的端口，并接受客户端的连接。
      2. 当有新的客户端连接请求到达时，bossGroup 将接收该连接，并将其注册到 workerGroup 中的某个 worker 线程上。
      3. bossGroup 的线程数量通常为 1
   2. workerGroup 是用于处理客户端连接 I/O 操作的事件循环组。
      1. 它包含多个线程或线程池，每个线程都会被分配一个客户端连接
      2. workerGroup 负责处理客户端连接的读写操作、协议解析、业务处理等
      3. 当客户端连接建立后，workerGroup 中的线程将负责处理该连接的所有 I/O 事件
      4. workerGroup 的线程数量通常根据系统的 CPU 核心数来配置
   3. 通过将 bossGroup 和 workerGroup 分开处理，Netty 实现了一种多线程的事件驱动模型。bossGroup 负责接收连接请求，并将请求分发给 workerGroup 中的线程来处理实际的 I/O 操作。这种模型可以提高并发处理能力和系统的吞吐量。
   4. NioEventLoopGroup 本质上是一个 EventLoop 数组，EventLoop 的职责是处理所有注册到本线程多路复用器 Selector 上的 Channel，Selector 的轮询操作由绑定的 EventLoop 线程 的 run()方法 驱动，在一个循环体内循环执行。
2. 接下来，我们创建了一个引导类 ServerBootstrap，这个类的主要作用是帮助我们配置和启动 Netty 服务器。具体而言，ServerBootstrap 类提供了一组方法，用于设置服务器的各种参数、协议和处理器。以下是 ServerBootstrap 类的一些关键方法和其作用的简要说明：
   1. group()：设置服务器的事件循环组，包括接收连接的 boss 线程组和处理连接的 worker 线程组。
   2. channel()：指定服务器所使用的通道类型，例如 NIO 通道或者 Epoll 通道
   3. option()：设置服务器的选项，例如 SO_KEEPALIVE、SO_BACKLOG 等
   4. childOption()：设置连接的子通道的选项
   5. handler()：为 boss 线程组设置处理器，用于处理接收连接的事件
   6. childHandler()：为 worker 线程组设置处理器，用于处理连接的 I/O 事件
   7. bind()：绑定服务器的监听端口，并启动服务器

一般来说， 要启动一个Netty服务端，必须要指定三类属性，分别是group()线程模型、channel()IO 模型、childHandler()连接读写处理逻辑。
# Netty 服务端创建源码分析
## NioEventLoopGroup事件循环组源码解析
NioEventLoopGroup 是 Netty 框架中用于处理网络事件的事件循环组，它利用 NIO 模型和事件驱动机制，提供了高性能和并发处理能力，是构建高性能网络应用的重要组件之一。
首先查看 NioEventLoopGroup 的整体继承关系图，有个整体了解
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1688263007748-b8c1814b-a877-4b07-8bdd-6489ab90b2cb.png#averageHue=%2332342f&clientId=u787d0ba1-aa2c-4&from=paste&height=622&id=ubcf9a891&originHeight=1244&originWidth=956&originalType=binary&ratio=2&rotation=0&showTitle=false&size=304282&status=done&style=none&taskId=u9014798a-67d3-4451-91d2-955d878be92&title=&width=478)
可以看到，NioEventLoopGroup继承自抽象类多线程事件循环组MultithreadEventLoopGroup，并且实现了 EventLoopGroup，EventExecutorGroup 等多个接口。

我们查看 NioEventLoopGroup 的类方法，可以看到大多数都是构造方法。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1688263989846-4c83d523-86fc-4d68-ae07-859c92150fbb.png#averageHue=%23ab8031&clientId=u787d0ba1-aa2c-4&from=paste&height=675&id=u100e52dc&originHeight=1350&originWidth=2744&originalType=binary&ratio=2&rotation=0&showTitle=false&size=533262&status=done&style=none&taskId=u3c88abe4-2199-4e54-b01d-45b6e84d874&title=&width=1372)

可以看到构造方法最终调用到MultithreadEventLoop 的构造方法，其中线程数为 0 的话， 会取静态代码块中设置的 DEFAULT_EVENT_LOOP_THREADS数。也就是说当我们没有指定NioEventLoopGroup的构造器参数的时候，即nThread=0或 io.netty.enentLoopThreads没有设置的时候，那么传入的线程数就变为cpu核心数乘以2.
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1688266982259-03b50860-b701-417e-8aa5-50b5ed1faaff.png#averageHue=%232d2b2b&clientId=u787d0ba1-aa2c-4&from=paste&height=521&id=u0596fb77&originHeight=1042&originWidth=1972&originalType=binary&ratio=2&rotation=0&showTitle=false&size=285989&status=done&style=none&taskId=u0411fadd-be5d-4693-8a63-06ee4c703eb&title=&width=986)

最终调用到的静态函数如下所示：
```java
...
// NioEventLoopGroup所有的变量都在下面， 继承的其他抽象类和接口都没有其他成员变量
private final EventExecutor[] children;
private final Set<EventExecutor> readonlyChildren;
private final AtomicInteger terminatedChildren = new AtomicInteger();
private final Promise<?> terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);
private final EventExecutorChooserFactory.EventExecutorChooser chooser;
...

protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        chooser = chooserFactory.newChooser(children);

        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```
构造函数进行了以下流程操作：

1. 参数验证：首先，构造函数对传入的参数进行验证，并且如果 executor 为null，则创建一个 ThreadPerTaskExecutor 使用默认的线程工厂。
2. 创建子实例：接下来，通过循环创建 nThreads 个子 EventExecutor 实例，并将它们存储在 children 数组中。每个子实例通过调用 newChild() 方法来创建。每个 EventExecutor 实例代表一个可执行任务的执行器，通常是一个线程。它可以是 NioEventLoop、EpollEventLoop 或其他扩展自 EventExecutor 的具体实现。在NioEventLoopGroup 中， 可以看到实现了 newChild()方法， 生成了对应的 NioEventLoop实例。
3. 处理创建失败：如果创建子实例失败，构造函数将关闭已创建的子实例，并等待它们终止。通过循环调用 shutdownGracefully() 方法关闭子实例，然后通过循环调用 awaitTermination() 方法等待它们终止。
4. 创建选择器：使用 chooserFactory 创建一个选择器（chooser），它将负责选择下一个可用的子实例。选择器的实现由具体的 EventExecutorChooserFactory 实现类决定。在 NioEventLoopGroup 中使用的是默认的DefaultEventExecutorChooserFactory
5. 监听子实例终止：为每个子实例的终止未来（terminationFuture）添加一个监听器（terminationListener）。当每个子实例的终止未来完成时，监听器会增加 terminatedChildren 的计数器，并在所有子实例都终止时，将 terminationFuture 设置为成功。
6. 创建只读子实例集合：最后，通过将 children 数组转换为 LinkedHashSet，创建了一个只读的子实例集合（readonlyChildren）。这个集合用于提供对子实例的只读访问，以避免对子实例的修改。

MultithreadEventExecutorGroup 构造函数完成了创建子实例、处理创建失败以及设置选择器和监听器等重要的逻辑处理。

通过idea debug，我们可以看到具体的构造函数属性值如下：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1688269197320-59036d1e-808f-4f27-bce5-7e41897c30ba.png#averageHue=%23394044&clientId=u787d0ba1-aa2c-4&from=paste&height=900&id=u45ea1399&originHeight=1800&originWidth=2880&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1026772&status=done&style=none&taskId=u9f8b9461-4d64-4fdb-b8e9-370c054df42&title=&width=1440)
其中 nThreads 为 16， 是根据cpu核心数乘以2取的，因为默认nThreads 数为 0 且没设置 io.netty.enentLoopThreads 参数。![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1688269241249-67b82f34-201b-416e-b5f6-bcb636ba1196.png#averageHue=%23508159&clientId=u787d0ba1-aa2c-4&from=paste&height=538&id=u01a2eacb&originHeight=1076&originWidth=1538&originalType=binary&ratio=2&rotation=0&showTitle=false&size=320165&status=done&style=none&taskId=ued5c6404-1093-4b77-9697-19fb7e25944&title=&width=769)

## ServerBootstrap源码解析
### 创建 ServerBootstrap 实例
通过下图， 我们可以看到， ServerBootstrap 的关系图比较简单， 并且无参构造函数并没有做任何的操作。
由于它采用了建造者模式（Builder Pattern）来提供一种流畅的、可读性高的方式来构建和配置服务器。所以可以在无参构造函数不做任何操作，
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1688269956934-a2437f50-71be-4ae7-a3f9-8b3fc47d5f03.png#averageHue=%232e2d2b&clientId=u787d0ba1-aa2c-4&from=paste&height=657&id=u58c54a59&originHeight=1314&originWidth=1922&originalType=binary&ratio=2&rotation=0&showTitle=false&size=506493&status=done&style=none&taskId=u48758907-9908-4d38-b65c-1421a3809a9&title=&width=961)

### 配置两大线程组
在我们通过无参构造函数创建 ServerBootstrap 实例后，通常会创建两个NioEventLoopGroup实例，即bossGroup和 workerGroup。通过 ServerBootstrap 的 group()方法 将两个 EventLoopGroup 实例传入，代码如下。
```java
NioEventLoopGroup boosGroup = new NioEventLoopGroup();
NioEventLoopGroup workerGroup = new NioEventLoopGroup();

final ServerBootstrap serverBootstrap = new ServerBootstrap();
serverBootstrap
        .group(boosGroup, workerGroup);
```
ServerBootstrap的group方法源码如下：
```java
/**
 * Set the {@link EventLoopGroup} for the parent (acceptor) and the child (client). These
 * {@link EventLoopGroup}'s are used to handle all the events and IO for {@link ServerChannel} and
 * {@link Channel}'s.
 */
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    super.group(parentGroup);
    if (childGroup == null) {
        throw new NullPointerException("childGroup");
    }
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    this.childGroup = childGroup;
    return this;
}
```
显然group方法是用于设置父（acceptor）和子（client）的 EventLoopGroup，这些 EventLoopGroup 用于处理所有的事件和 IO 操作。父 EventLoopGroup 负责监听服务器端口和接受连接请求，而子 EventLoopGroup 负责处理每个客户端连接的事件和 IO 操作。分离父子 EventLoopGroup 的设置可以更好地控制服务器的资源分配和并发处理能力。
其中 parentGroup 对象 被设置进了 ServerBootstrap 的父类 AbstractBootstrap 中，代码如下。
```java
/**
 * The {@link EventLoopGroup} which is used to handle all the events for the to-be-created
 * {@link Channel}
 */
@SuppressWarnings("unchecked")
public B group(EventLoopGroup group) {
    if (group == null) {
        throw new NullPointerException("group");
    }
    if (this.group != null) {
        throw new IllegalStateException("group set already");
    }
    this.group = group;
    return (B) this;
}
```
### 设置 IO 模型并绑定服务端 Channel
对于用户而言，并服务端 Channel 的底层实现细节和工作原理，只需要指定具体使用哪种服务端 Channel 即可。因此，Netty 中 ServerBootstrap 的基类提供了 channel()方法，用于指定服务端 Channel 的类型。并通过工厂类，利用反射创建 NioServerSocketChannel 对象。由于服务端监听端口往往只需要在系统启动时才会调用，因此反射对性能的影响并不大。
使用方式如下：
```java
serverBootstrap
    .group(boosGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
```

对应channel 方法源码如下：
```java
/**
 * The {@link Class} which is used to create {@link Channel} instances from.
 * You either use this or {@link #channelFactory(io.netty.channel.ChannelFactory)} if your
 * {@link Channel} implementation has no no-args constructor.
 */
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}

/**
 * {@link io.netty.channel.ChannelFactory} which is used to create {@link Channel} instances from
 * when calling {@link #bind()}. This method is usually only used if {@link #channel(Class)}
 * is not working for you because of some more complex needs. If your {@link Channel} implementation
 * has a no-args constructor, its highly recommend to just use {@link #channel(Class)} for
 * simplify your code.
 */
@SuppressWarnings({ "unchecked", "deprecation" })
public B channelFactory(io.netty.channel.ChannelFactory<? extends C> channelFactory) {
    return channelFactory((ChannelFactory<C>) channelFactory);
}

/**
 * @deprecated Use {@link #channelFactory(io.netty.channel.ChannelFactory)} instead.
 */
@Deprecated
@SuppressWarnings("unchecked")
public B channelFactory(ChannelFactory<? extends C> channelFactory) {
    if (channelFactory == null) {
        throw new NullPointerException("channelFactory");
    }
    if (this.channelFactory != null) {
        throw new IllegalStateException("channelFactory set already");
    }

    this.channelFactory = channelFactory;
    return (B) this;
}
```

```java
package io.netty.channel;

import io.netty.util.internal.StringUtil;

/**
 * A {@link ChannelFactory} that instantiates a new {@link Channel} by invoking its default constructor reflectively.
 */
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Class<? extends T> clazz;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        if (clazz == null) {
            throw new NullPointerException("clazz");
        }
        this.clazz = clazz;
    }

    @Override
    public T newChannel() {
        try {
            return clazz.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }

    @Override
    public String toString() {
        return StringUtil.simpleClassName(clazz) + ".class";
    }
}

```
可以发现Netty 通过 Channel 工厂类 来创建不同类型的 Channel，对于服务端，需要创建 NioServerSocketChannel。所以，通过指定 Channel 类型 的方式创建 Channel 工厂。ReflectiveChannelFactory 可以根据 Channel 的类型 通过反射创建 Channel 的实例。
### 设置底层网络通信相关的参数
有 option和 childOption 两个方法，option() 方法是 Netty 中用于设置特定 Channel 或 ServerChannel 选项的配置方法。通过调用该方法，可以设置底层网络通信相关的参数，以满足应用程序的需求，并优化网络通信的性能和行为。而childOption() 方法则是用于设置子通道（child channel）的选项（options），仅适用于服务器端的子通道，即每个客户端连接的通道。
option() 方法：

- 适用对象：Bootstrap 或 ServerBootstrap 对象
- 作用范围：**影响整个通道，包括服务器端和客户端通道**
- 设置的选项将应用于创建的所有通道，包括服务端的 ServerChannel 和客户端的 Channel
- 例如，可以使用 option() 方法设置连接超时时间、接收和发送缓冲区大小、TCP_NODELAY 参数等

childOption() 方法：

- 适用对象：ServerBootstrap 对象。
- 作用范围：**仅适用于服务器端的子通道，即每个客户端连接的通道。**
- 设置的选项将应用于每个客户端连接的子通道，不会影响服务器端的 ServerChannel。
- 例如，可以使用 childOption() 方法设置子通道的接收缓冲区大小、连接超时时间、TCP_NODELAY 参数等。

option()对应的源码设置如下
```java
private final Map<ChannelOption<?>, Object> options = new LinkedHashMap<ChannelOption<?>, Object>();

/**
 * Allow to specify a {@link ChannelOption} which is used for the {@link Channel} instances once they got
 * created. Use a value of {@code null} to remove a previous set {@link ChannelOption}.
 */
@SuppressWarnings("unchecked")
public <T> B option(ChannelOption<T> option, T value) {
    if (option == null) {
        throw new NullPointerException("option");
    }
    if (value == null) {
        synchronized (options) {
            options.remove(option);
        }
    } else {
        synchronized (options) {
            options.put(option, value);
        }
    }
    return (B) this;
}

```
可以看到，是通过 LinkedHashMap 去存储，并且通过synchronized锁去确保线程安全的。其实可以使用 ConcurrentHashMap 代替 LinkedHashMap，从而避免显式的同步块，以及利用 ConcurrentHashMap 内部的并发机制实现更细粒度的锁控制。

childOption()对应的源码设置如下，也只是通过LinkedHashMap去存储属性值。
```java
private final Map<ChannelOption<?>, Object> childOptions = new LinkedHashMap<ChannelOption<?>, Object>();

/**
 * Allow to specify a {@link ChannelOption} which is used for the {@link Channel} instances once they get created
 * (after the acceptor accepted the {@link Channel}). Use a value of {@code null} to remove a previous set
 * {@link ChannelOption}.
 */
public <T> ServerBootstrap childOption(ChannelOption<T> childOption, T value) {
    if (childOption == null) {
        throw new NullPointerException("childOption");
    }
    if (value == null) {
        synchronized (childOptions) {
            childOptions.remove(childOption);
        }
    } else {
        synchronized (childOptions) {
            childOptions.put(childOption, value);
        }
    }
    return this;
}
```

### 设置服务端处理器
handler() 和 childHandler() 方法是 Netty 中 ServerBootstrap 类的两个方法，用于配置服务器端和客户端的 ChannelHandler。

- handler() 方法用于配置服务器端的 ChannelHandler。它指定了在服务器端的 Channel 的 ChannelPipeline 中添加的处理器。这些处理器将处理传入服务器的请求。通常在 handler() 方法中配置的处理器是针对服务器整体的操作，例如处理服务器的启动和关闭事件、处理服务器级别的业务逻辑等。 
- childHandler() 方法用于配置客户端的 ChannelHandler。它指定了在每个客户端的 Channel 的 ChannelPipeline 中添加的处理器。这些处理器将处理来自客户端的请求。通常在 childHandler() 方法中配置的处理器是针对每个客户端连接的操作，例如处理请求和响应、编解码、业务逻辑等。

这两个方法都接受一个 ChannelHandler 参数，用于指定要添加到对应的 ChannelPipeline 中的处理器。ChannelHandler 是 Netty 中用于处理网络事件的基本组件。通过添加不同的处理器，可以实现数据的编解码、协议解析、业务逻辑处理等功能。
在 handler() 方法中配置的处理器将应用于服务器的整个生命周期，而在 childHandler() 方法中配置的处理器将应用于每个客户端连接的生命周期。这样的设计使得可以针对服务器和每个客户端连接定制不同的处理逻辑。
在使用这两个方法时，可以根据实际需求选择适合的处理器，并按照处理器的顺序将它们添加到 ChannelPipeline 中，以构建完整的处理链。处理器将按照添加的顺序依次处理传入的事件和消息。

handle()方法源码如下
```java
...
private volatile ChannelHandler handler;
...

/**
 * the {@link ChannelHandler} to use for serving the requests.
 */
@SuppressWarnings("unchecked")
public B handler(ChannelHandler handler) {
    if (handler == null) {
        throw new NullPointerException("handler");
    }
    this.handler = handler;
    return (B) this;
}
```

childHandler()源码如下
```java
...
private volatile ChannelHandler childHandler;
...

/**
 * Set the {@link ChannelHandler} which is used to serve the request for the {@link Channel}'s.
 */
public ServerBootstrap childHandler(ChannelHandler childHandler) {
    if (childHandler == null) {
        throw new NullPointerException("childHandler");
    }
    this.childHandler = childHandler;
    return this;
}
```
可以看到， 基本上就只是做一下非空判断， 就给属性设个值就结束了。

### 绑定端口并启动
Netty通过ServerBootstrap 类的bind() 方法将服务器绑定到指定的地址和端口。并且通过 ChannelFuture 对象，可以注册监听器来处理绑定操作完成的事件。可以使用 addListener() 方法添加 GenericFutureListener 监听器，以便在绑定操作完成时执行相应的逻辑。
bind()方法源码如下
```java
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {
    /**
     * Create a new {@link Channel} and bind it.
     */
    public ChannelFuture bind(int inetPort) {
        return bind(new InetSocketAddress(inetPort));
    }
    
    
    /**
     * Create a new {@link Channel} and bind it.
     */
    public ChannelFuture bind(SocketAddress localAddress) {
        validate();
        if (localAddress == null) {
            throw new NullPointerException("localAddress");
        }
        return doBind(localAddress);
    }
    
    /**
     * Create a new {@link Channel} and bind it.
     */
    public ChannelFuture bind(SocketAddress localAddress) {
        validate();
        if (localAddress == null) {
            throw new NullPointerException("localAddress");
        }
        return doBind(localAddress);
    }
    
    
    /**
     * Validate all the parameters. Sub-classes may override this, but should
     * call the super method in that case.
     */
    @SuppressWarnings("unchecked")
    public B validate() {
        if (group == null) {
            throw new IllegalStateException("group not set");
        }
        if (channelFactory == null) {
            throw new IllegalStateException("channel or channelFactory not set");
        }
        return (B) this;
    }
    
    private ChannelFuture doBind(final SocketAddress localAddress) {
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }
    
        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();
    
                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }
}
```
在调用bind()方法后， 最终调用到了doBind()方法，
#### initAndRegister()方法

先看下上述代码调用的 initAndRegister()方法。相关代码如下。
```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // 反射创建channel对象
        channel = channelFactory.newChannel();
        // 初始化channel
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            // channel can be null if newChannel crashed (eg SocketException("too many open files"))
            channel.unsafe().closeForcibly();
        }
        // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
        return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
    }

    // 注册 NioServerSocketChannel 到 Reactor 线程 的多路复用器上，然后轮询客户端连接事件。
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    // If we are here and the promise is not failed, it's one of the following cases:
    // 1) If we attempted registration from the event loop, the registration has been completed at this point.
    //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
    // 2) If we attempted registration from the other thread, the registration request has been successfully
    //    added to the event loop's task queue for later execution.
    //    i.e. It's safe to attempt bind() or connect() now:
    //         because bind() or connect() will be executed *after* the scheduled registration task is executed
    //         because register(), bind(), and connect() are all bound to the same thread.

    return regFuture;
}
```

1. 可以看到它首先通过channelFactory进行反射调用实例化了一个 NioServerSocketChannel 类型 的 Channel 对象。
2. 生成channel后就调用init方法对channel进行初始化，其中init方法在AbstractBootstrap中是一个抽象方法，具体实现交给子类去实现， 这里的实现类是ServerBootstrap。
3. 到此，Netty 服务端监听的相关资源已经初始化完毕，就剩下最后一步，注册 NioServerSocketChannel 到 Reactor 线程 的多路复用器上，然后轮询客户端连接事件。
##### init方法
初始化代码如下：
```java
@Override
void init(Channel channel) throws Exception {
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        channel.config().setOptions(options);
    }

    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
    }

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            // We add this handler via the EventLoop as the user may have used a ChannelInitializer as handler.
            // In this case the initChannel(...) method will only be called after this method returns. Because
            // of this we need to ensure we add our handler in a delayed fashion so all the users handler are
            // placed in front of the ServerBootstrapAcceptor.
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```
可以看到初始化工作完成了选项（options）和属性（attributes），以及设置通道的处理器（handler）和子通道的处理器（childHandler）的配置。

1. 通过调用 options0() 和 attrs0() 方法获取 options 和 attrs 的副本。在对 options 和 attrs 进行操作时，使用 synchronized 关键字来保证线程安全性。 接着通过 channel.config().setOptions(options) 方法将 options 配置到通道的配置对象中。 最后，通过遍历 attrs 集合，将每个属性设置到通道的属性中。
2. 获取通道的管道（pipeline）对象，用于添加通道的处理器。
3. 将 childGroup、childHandler、childOptions 和 childAttrs 的值赋给对应的局部变量，以便在后续的逻辑中使用。
4. 向通道的管道中添加一个 ChannelInitializer，用于初始化通道。在初始化通道时，首先获取用户自定义的处理器（handler），如果存在，则将其添加到管道中。
5. 通过调用 ch.eventLoop().execute() 方法，在事件循环线程中执行一个任务。该任务的作用是向通道的管道中添加一个 ServerBootstrapAcceptor 处理器，该处理器用于接收并处理子通道的连接请求。具体来说，ServerBootstrapAcceptor 实现了 ChannelInboundHandler 接口，它的主要作用是监听 channelRead() 事件。当有新的子通道连接请求到达时，channelRead() 方法会被触发，然后它会获取子通道的相关信息，并根据配置的 childGroup、childHandler、childOptions 和 childAttrs 创建和初始化子通道的管道（pipeline）。然后，将该子通道注册到子事件循环中，以便后续的读写操作可以在子事件循环中处理。
##### register方法

```java

/**
 * A skeletal {@link Channel} implementation.
 */
public abstract class AbstractChannel extends DefaultAttributeMap implements Channel {
  	/**
     * 将完成初始化的 NioServerSocketChannel 注册到 Reactor线程
     * 的多路复用器上，监听新客户端的接入
     */
    @Override
    public final void register(EventLoop eventLoop, final ChannelPromise promise) {
        if (eventLoop == null) {
            throw new NullPointerException("eventLoop");
        }
        if (isRegistered()) {
            promise.setFailure(new IllegalStateException("registered to an event loop already"));
            return;
        }
        if (!isCompatible(eventLoop)) {
            promise.setFailure(
                    new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
            return;
        }
    
        AbstractChannel.this.eventLoop = eventLoop;

        // 首先判断是否是 NioEventLoop 自身发起的操作。如果是，则不存在并发操作，直接
        // 执行 Channel注册；如果由其他线程发起，则封装成一个 Task 放入消息队列中异步执行。
        // 此处，由于是由 ServerBootstrap 所在线程执行的注册操作，所以会将其封装成 Task 投递
        // 到 NioEventLoop 中执行
        if (eventLoop.inEventLoop()) {
            register0(promise);
        } else {
            try {
                eventLoop.execute(new Runnable() {
                    @Override
                    public void run() {
                        register0(promise);
                    }
                });
            } catch (Throwable t) {
                logger.warn(
                        "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                        AbstractChannel.this, t);
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }
    }
    
    private void register0(ChannelPromise promise) {
        try {
            // check if the channel is still open as it could be closed in the mean time when the register
            // call was outside of the eventLoop
            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }
            boolean firstRegistration = neverRegistered;
            // 该方法在本类中是一个空实现，在ServerBootstrap中它的实现类是子类 AbstractNioChannel
            doRegister();
            neverRegistered = false;
            registered = true;
    
            // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
            // user may already fire events through the pipeline in the ChannelFutureListener.
            pipeline.invokeHandlerAddedIfNeeded();
    
            safeSetSuccess(promise);
            pipeline.fireChannelRegistered();
            // Only fire a channelActive if the channel has never been registered. This prevents firing
            // multiple channel actives if the channel is deregistered and re-registered.
            if (isActive()) {
                if (firstRegistration) {
                    pipeline.fireChannelActive();
                } else if (config().isAutoRead()) {
                    // This channel was registered before and autoRead() is set. This means we need to begin read
                    // again so that we process inbound data.
                    //
                    // See https://github.com/netty/netty/issues/4805
                    beginRead();
                }
            }
        } catch (Throwable t) {
            // Close the channel directly to avoid FD leak.
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }

    
    /**
     * Is called after the {@link Channel} is registered with its {@link EventLoop} as part of the register process.
     *
     * Sub-classes may override this method
     */
    protected void doRegister() throws Exception {
        // NOOP
    }
}
```


```java
public abstract class AbstractNioChannel extends AbstractChannel {

	@Override
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                // 将 NioServerSocketChannel 注册到 NioEventLoop 的 多路复用器Selector 上
                // 这里的register方法是调用了jdk的nio相关的注册方法， 这里不做解析。
                selectionKey = javaChannel().register(eventLoop().selector, 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    // Force the Selector to select now as the "canceled" SelectionKey may still be
                    // cached and not removed because no Select.select(..) operation was called yet.
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    // We forced a select operation on the selector before but the SelectionKey is still cached
                    // for whatever reason. JDK bug ?
                    throw e;
                }
            }
        }
    }
}
```
至此， 服务端的源码流程解析完成~
# 参考文章
[source-code-hunter/docs/Netty/基于Netty开发服务端及客户端/基于Netty的服务端开发.md at main · doocs/source-code-hunter](https://github.com/doocs/source-code-hunter/blob/main/docs/Netty/%E5%9F%BA%E4%BA%8ENetty%E5%BC%80%E5%8F%91%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%8F%8A%E5%AE%A2%E6%88%B7%E7%AB%AF/%E5%9F%BA%E4%BA%8ENetty%E7%9A%84%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%BC%80%E5%8F%91.md)
[netty源码分析之服务端启动全解析](https://www.jianshu.com/p/c5068caab217)

