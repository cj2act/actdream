> 前前后后利用工作之余的闲暇时间阅读Netty源码有一个月了，现在俯瞰Netty各组件之间的联系，已经有了一个初步的认识。此文为Netty系列的第一篇，主要描述一个客户端请求在Netty中流转的整个生命周期，它经过了哪些组件之间相互配合。后面的文章会对每一个组件进行源码分析，以及此文没有体现出来的组件它们的作用。如`ChannelFuture`、`ByteBuf`等。在阅读之前，推荐先阅读下之前的一篇文章([浅谈NIO中的Reactor设计模式应用](./articles/java/nio/nio-reactor.md))。
### 1. 回顾Ractor模型的Multiple Reactors Version
`Ractor`模型的`Multiple Reactors Version`是[浅谈NIO中的Reactor设计模式应用](./articles/java/nio/nio-reactor.md)中的3.3，DougLea在`Scalable IO in Java`画的模型图如下所示：
![](https://user-gold-cdn.xitu.io/2019/6/3/16b1b1684145dc7c?w=1262&h=876&f=png&s=554666)
> `mainReactor`的`acceptor`只负责接收客户端的连接事件，`subReactor`只负责`网络`读写`I/O`事件，其实`subReactor`可以有多个，这样减轻每个`subReactor`负载。对于耗时compute业务代码，可以交给线程池处理，处理后注册写事件返回给客户端。`acceptor`和`subReactor`都轮询自己的`selector.select()`，处理着自己"感兴趣"的事件。其实Netty底层模型应用便是这种，只不过各组件的名字和作用都有些差异。
```
/**
 * 举个例子：现在我想一个accptor和多个subReactor。
 **/
// 一个Acceptor
EventLoopGroup parentEventLoopGroup = new NioEventLoopGroup(1);
// 当前CPU核数*2的subReactor个数，对于现在超线程CPU那就再*n
EventLoopGroup childEventLoopGroup = new NioEventLoopGroup();
```
### 2. Netty中的各组件联系
#### 2.1 请求在各组件间的流转图
![](https://user-gold-cdn.xitu.io/2019/6/3/16b1b462fc42fa68?w=2350&h=1058&f=png&s=364235)
#### 2.2 图中Netty的各组件的作用
> 对于selector、channel等组件的作用可以在[浅谈NIO中的Reactor设计模式应用](./articles/java/nio/nio-reactor.md)，本文只介绍Netty相关的组件用途，以下内容根据图自顶向下
1. `NioEventLoopGroup`：直译这个类名就是`nio`的事件轮询组，其实轮询的就是`Selector.select()`。文档中的解释是"it is used for NIO Selector based Channel."。对于接收客户端的连接事件与selectionKey绑定的是ServerSocketChannel；对于读写网络`I/O`事件与selectionKey绑定的便是SocketChannel。
2. `NioEventLoop`：通过名称就可以看出来，它便是事件轮询组中的一个轮询，也就是一个`subReactor`或者`acceptor`（通常做法在创建接收连接的轮询组时都是创建一个）。
3. `ChannelPipeline`：文档中的解释是"it handles or intercepts inbound events and outbound operations of a implements an advanced form of the <a href="http://www.oracle.com/technetwork/java/interceptingfilter-142169.html">Intercepting Filter</a> pattern"，翻译过来就是它处理或者是拦截入栈事件和出栈的一些操作，实现了一种更加高级的拦截过滤器模式。过滤器模式是入出走同一个过滤器，而对于`ChannelPipeline`入栈处理和出栈处理分离开了，`ChannelPipeline`
底层维护着一个`ChannelHandlerContext`双向链表。`ChannelPipeline`的处理流程如下：
![](https://user-gold-cdn.xitu.io/2019/6/3/16b1b7aa0b6042d3)
从`Channel`里读出来的数据，需要经过与`Channel`绑定的`ChannelPipeline`中的`InboundHandler`处理，再注册写事件。写出的时候需要经`ChannelPipeline`中的`OutboundHandler`处理在写回给客户端。
4. `ChannelHandlerContext`：`ChannelPipeline`中是`ChannelHandlerContext`双向链表，它起到一个承上启下的作用，意思是能够向上获取到`ChannelPipeline`和`Channel`，又能向下获取到`ChannelHandler`。
5. `ChannelInboundHandlerAdapter`/`ChannelOutboundHandlerAdapter`：入栈出栈的处理器对象，它应用适配器设计模式，后续文章中我们在从细节分析。
6. `ServerBootStrapAcceptor`：`Netty`中`acceptor`对象，用于将`EventLoop`中`Selector`接收到的`Channel`（这里面其实与`selectionKey`绑定的是`serverChannel`然后再通过`serverChannel.accept()`得到的），将`Channel`注册给`ChildEventLoop`（这块`Netty`是通过`round-robin`去获取`EventLoop`）中个其中一个`Selector`上。



