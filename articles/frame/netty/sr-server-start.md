## Netty服务端启动流程分析
### 事例代码
> 源码阅读过程中，我们使用下面这个简单的示例代码做参考；
```
    EventLoopGroup parentGroup = new NioEventLoopGroup(1);
    EventLoopGroup childGroup = new NioEventLoopGroup();
    try {
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap
                .group(parentGroup, childGroup)
                .channel(NioServerSocketChannel.class)
                .handler(new LoggingHandler(LogLevel.INFO))
                .childOption(ChannelOption.TCP_NODELAY, true)
                .childAttr(AttributeKey.newInstance("childAttr"), "childAttrVal")
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast("httpServerCodec", new HttpServerCodec());
                        pipeline.addLast("testHttpServerHandler", new TestHttpServerHandler());
                        pipeline.addLast(new LoggingHandler(LogLevel.INFO));
                    }
                });
        ChannelFuture channelFuture = serverBootstrap.bind(8080).sync();
        channelFuture.channel().closeFuture().sync();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        parentGroup.shutdownGracefully();
        childGroup.shutdownGracefully();
    }
```
### 创建服务端Channel
#### 步骤
- bind()
- initAndRegister()
- newChannel() 反射创建服务端Channel
  - newSocket() 通过jdk来创建底层的Channel
  - NioServerSocketChannelConfig 设置TCP相关参数
  - AbstractNioChannel
    - configureBlocking
    - AbStractChannel() (id unsafe pipeline)
### 分析
我们从bind(port)开始阅读Netty服务端的创建初始化过程
```
serverBootstrap.bind(8080)
```
顺着bind方法我们一直跟到AbstractBootStrap.doBind(final SocketAddress localAddress)
```
    private ChannelFuture doBind(final SocketAddress localAddress) {
            // 初始化和注册
            final ChannelFuture regFuture = initAndRegister();
            ...
    }
```
initAndRegister()
```
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            // 创建NioServerChannel
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            ...
        }
        ...
        return regFuture;
    }
```
创建NioServerChannel的channelFactory是何时被初始化的呢？在事例代码中对启动引导类有如下设置：
```
    serverBootstrap
            .channel(NioServerSocketChannel.class)
```
看下channel(Class<? extends C> channelClass)方法做了什么
```
    public B channel(Class<? extends C> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        // 返回一个创建channel的工厂
        return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
    }
```
我们来看下ReflectiveChannelFactory类的实现
```
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {
    private final Constructor<? extends T> constructor;
    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        ObjectUtil.checkNotNull(clazz, "clazz");
        try {
            // 设置传进来的Channel的构造器
            this.constructor = clazz.getConstructor();
        } catch (NoSuchMethodException e) {
            throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                    " does not have a public non-arg constructor", e);
        }
    }
    @Override
    public T newChannel() {
        try {
            // 通过构造器实例化对象
            return constructor.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
        }
    }
    ...
}
```
跟到这我们发现channelFactory.newChannel()实际上调用的就是NioServerSocketChannel的无参构造方法：
- NioServerSocketChannel无参构造方法创建Channel时包括以下步骤:
  - newSocket创建ServerSocketChannel
  - 调用父类构造器 注册SelectionKey.OP_ACCEPT事件 初始化 id unsafe pipeline
```
public class NioServerSocketChannel extends AbstractNioMessageChannel
                             implements io.netty.channel.socket.ServerSocketChannel {
    ....                             
    private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();
    private final ServerSocketChannelConfig config;
    private static ServerSocketChannel newSocket(SelectorProvider provider) {
        ...
        return provider.openServerSocketChannel();
        ...
    }
    public NioServerSocketChannel() {
        // 1.newSocket创建ServerSocketChannel
        this(newSocket(DEFAULT_SELECTOR_PROVIDER));
    }
    public NioServerSocketChannel(ServerSocketChannel channel) {
        // 2.调用父类构造器 赋值SelectionKey.OP_ACCEPT事件 初始化 id unsafe pipeline
        super(null, channel, SelectionKey.OP_ACCEPT);
        // 3.设置config
        config = new NioServerSocketChannelConfig(this, javaChannel().socket());
    }
    ...
}
```
- 父类AbstractNioChannel的构造方法
```
    // 设置channel和readInterestOp以及把channel设置为非阻塞的读
    protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
        super(parent);
        this.ch = ch; // 以后javaChannel()获取到的serverSocketChannel
        this.readInterestOp = readInterestOp;
        try {
            ch.configureBlocking(false);
        } catch (IOException e) {
            ...
        }
    }
    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = newId();
        // netty内部的对
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
    // 初始化ChannelPipeline
    protected DefaultChannelPipeline(Channel channel) {
        ...
        tail = new TailContext(this);
        head = new HeadContext(this);
        head.next = tail;
        tail.prev = head;
        ...
    }
```
### 初始化服务端Channel
#### 步骤
- set(channelOptions, channelAttrs)
- set(childOption, childAttrs)
- config handler 配置服务端pipeline
- new ServerBootstrapAcceptor
#### 分析
- ServerBootstrap.init()
```
    void init(Channel channel) throws Exception {
        // 1.1 设置ServerSocketChannel的options
        final Map<ChannelOption<?>, Object> options = options0();
        synchronized (options) {
            setChannelOptions(channel, options, logger);
        }
        // 1.2 设置ServerSocketChannel的attrs
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
        // 2.1 设置childoption 如ChannelOption.TCP_NODELAY
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
        }
        // 2.2 设置childAttrs
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
        }
        // 3.初始化acceptor
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```
- new ServerBootstrapAcceptor的核心在于它的channelRead方法，我们先来简单看下：
```
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 获取到SocketChannel，也就是连结接入的客户端Channel
        final Channel child = (Channel) msg;
        ...
        try {
            // 将child注册给其中一个childEventLoop的selector上。
            childGroup.register(child).addListener(new ChannelFutureListener() {
                ...
            });
        } catch (Throwable t) {
            forceClose(child, t);
        }
    }
```
- 
### 注册Selector
#### 步骤
- AbstractChannel.register(channel)入口
  - this.eventLoop = eventLoop
  - register0 实际注册
    - doRegister() 调用jdk底层注册
    - invoke HandlerAddedIfNeeded
    - fireChannelRegistered() 传播事件
#### 分析
- 还是看AbstactBootstrap.initAndRegister()
```
    final ChannelFuture initAndRegister() {
        ...
        ChannelFuture regFuture = config().group().register(channel);
        ...
        return regFuture;
    }
```
- 跟着register方法一直到AbstractUnsafe.register0(..)方法
```
    private void register0(ChannelPromise promise) {
        try {
            ...
            // 1.doRegister() 调用jdk底层注册
            doRegister();
            // 2.invoke HandlerAddedIfNeeded
            pipeline.invokeHandlerAddedIfNeeded();
            // 3.调用ServerSocketChannle的fireChannelRegistered
            pipeline.fireChannelRegistered();
            ...
        } catch (Throwable t) {
            ...
        }
    }
```
- AbstractNioChannel.doRegister() 调用jdk底层注册
```
    protected void doRegister() throws Exception {
        ...
        for (;;) {
            try {
                // 将ServerSocketChannel注册给selector，这块0代表不关注任何事件，因为会在绑定的时候监听OP_ACCEPT
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                ...
            }
        }
    }
```
- javaChannel(),在之前步骤AbstractNioChannel初始化的时候，javaChannel()其实在当前情况下获取到的就是ServerSocketChannnel
- 这个注册其实你可以发现是服务端ServelSocketChannel的注册，而我们上边发现，对于接入的客户端调用acceptor的channelRead方法时，其实对应的过程是SocketChannel的注册，对于客户端读写事件的selector我们通常起多个EventLoop，所以这块会有个简单的轮训算法找到EventLoop，然后在将channel注册到它的selector上，后面的文章会更加详细的说着点。

### 完成端口绑定
#### 步骤
- AbstractUnsafe.bind()
  - doBind()
    - javaChannel().bind() jdk底层绑定
  - pipeline.fireChannelActive()
    - HeadContext.readIfIsAutoRead() 将selector绑定事件为OP_ACCEPT事件
#### 介绍
- AbstractUnsafe.bind()
```
    public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
        ...
        // 这个时候channel并不是active所以为false
        boolean wasActive = isActive();
        try {
            // 1.jdk ServerSocketServer绑定端口
            doBind(localAddress);
        } catch (Throwable t) {
            ...
            return;
        }
        // 因为上边已经完成端口绑定所以 isActive()此时为true
        if (!wasActive && isActive()) {
            invokeLater(new Runnable() {
                @Override
                public void run() {
                    // 2. 一直跟踪最后发现调用的是 HeadContext.readIfIsAutoRead()
                    pipeline.fireChannelActive();
                }
            });
        }
        ...
    }
```
- isActive()其实调用的与ServerSocketChannel绑定的ServerSocket的isBound方法
```
    public boolean isActive() {
        return javaChannel().socket().isBound();
    }
    public boolean isBound() {
        // Before 1.3 ServerSockets were always bound during creation
        return bound || oldImpl;
    }
```
- HeadContext.readIfIsAutoRead()再一直跟踪 AbstractNioChannel.doBeginRead()完成的OP_ACCEPT事件绑定
```
    protected void doBeginRead() throws Exception {
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }
        readPending = true;
        // interestOps 完成OP_ACCEPT事件绑定
        final int interestOps = selectionKey.interestOps();
        if ((interestOps & readInterestOp) == 0) {
            selectionKey.interestOps(interestOps | readInterestOp);
        }
    }
```
### 总结
写到这，通过本文了解到了：
- 服务端channel的创建过程
- 服务端channel的初始化过程
- 服务端channel向selector的注册
- 完成端口绑定，和底层的serverSocket激活
