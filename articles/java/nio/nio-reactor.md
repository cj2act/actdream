> 趁着三天假期，把`Java NIO`和`Reactor`模式整理总结了下，文章特别细节的知识点没有写，如一些`API`的具体实现。类似数据读到`Buffer`后再写出时，为什么需要复位操作，这些都属于`NIO`基础知识，是学习`Reactor`模式的前置条件。
### 1. 原始Ractor模式
![](https://user-gold-cdn.xitu.io/2019/4/4/169e68d9038948e3?w=701&h=259&f=png&s=74385)

#### 相关组件的解释
1. **Handle（句柄或是描述符）**：本质上表示一种资源，是操作系统提供的；该资源用于表示一个个`事件`,比如文件描述符，或者是针对于网络编程中的`Socket`描述符。事件既可以来自于外部，也可以来自内部；外部事件比如说客户端的连接请求，客户端发送过来数据等；内部事件比如说操纵系统产生的定时器事件等。它本质上就是一个文件描述符。`Handle`是事件产生的发源地。
2. **Synchronous Event Demultiplexer（同步事件分离器）**：它本身是一个系统调用，用于等待事件的发生（事件可能是一个，也可能是多个）。调用方在调用它的时候会被阻塞，一直阻塞到同步事件分离器上有事件产生为止。对于`Linux`来说，同步事件分离器指的就是常用的`I/O`多路复用机制，比如说`select`、`poll`、`epoll`等。在`Java NIO`中，同步事件分离器对应的组件就是`Selector`；对应的阻塞方法就是`select`方法。
3. **Event Handler（事件处理器）** 本身由多个回调方法构成，这些回调构成了与应用相关的对于某个事件的反馈机制。`Netty`相比于`Java NIO`来说，在事件处理器这个角色上进行一个升级，它为我们开发者提供了大量的回调方法，供我们在待定事件产生时实现相应的回调方法进行业务逻辑的处理。
4. **Concrete Event Handler（具体事件处理器）**：它本身实现了事件处理所提供的各个回调方法，从而实现了特定于业务的逻辑。它本质上就是我们所编写的一个个的处理器实现。
5. **Initiation Dispatcher（初始分发器）**：实际上就是`Reactor`角色。它本身定义了一些规范，这些规范用于控制事件的调度方式，同时又提供了应用进行事件处理器的注册、删除等。`Initiation Dispatcher`会通过同步事件分离器来等待事件的发生，一旦事件发生，`Initiation Dispatcher`首先会分离出每一个事件，然后调用事件处理器，最后调用相关的回调方法来处理事件。

#### 执行流程分析
1. 当应用像`Initiation Dispatcher`注册具体的事件处理器时，应用会标识出事件处理器希望`Initiation Dispatcher`在某个事件发生时向其通知该事件，该事件与`Handle`关联。
2. `Initiation Dispatcher`会要求每个事件向其传递内部的`Handle`。该`Handle`向操作系统标识了事件处理器。
3. 当所有事件处理器注册完毕后，应用会调用`handle_events`方法来启动`Initiation Dispatcher`的事件循环。这时，`Initiation Dispatcher`会将每个注册的事件管理器的`Handle`合并起来，并使用同步事件分离器等待这些事件的发生。比如说，`TCP`协议层使用`select`同步事件分离器操作来等待客户端发送的数据到达连接的`socker handle`上。
4. 当与某个事件源对应的`Handle`变为`ready`状态时（比如说，`TCP socker`变为等待读状态时），同步事件分离器就会通知`Initiation Dispatcher`。
5. `Initiation Dispatcher`会触发事件处理器的回调方法，从而响应这个处于`ready`状态的`Handle`。`Initiation Dispatcher`会回调事件处理器的`handle_events`回调方法来执行特定于应用的功能（开发者自己所编写的功能），从而响应这个事件。所发生的事件类型可以作为该方法参数并被该方法内部使用来执行额外的特定于服务的功能。
> 以上描述的内容似乎和本文的标题不大，其实不然，它正是下面介绍的内容的开端。

### 2. 通过一个例子拉近与Java NIO的距离
```
/**
 * @Author CoderJiA
 * @Description NIOServer
 * @Date 13/2/19 下午4:59
 **/
public class NIOServer {

    public static void main(String[] args) throws Exception{

        // 1.创建ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        ServerSocket serverSocket = serverSocketChannel.socket();
        serverSocket.bind(new InetSocketAddress(8899));

        // 2.创建Selector，并ServerSocketChannel注册OP_ACCEPT事件，接收连接。
        Selector selector = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        // 3.开启轮询
        while (selector.select() > 0) {
            // 从selector所有事件就绪的key，并遍历处理。
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            selectionKeys.forEach(selectionKey -> {
                SocketChannel client;
                try {
                    if (selectionKey.isAcceptable()) {  // 接受事件就绪
                        // 获取serverSocketChannel
                        ServerSocketChannel server = (ServerSocketChannel)selectionKey.channel();
                        // 接收连接
                        client = server.accept();
                        client.configureBlocking(false);
                        client.register(selector, SelectionKey.OP_READ);
                    } else if (selectionKey.isReadable()) {  // 读事件就绪
                        // 获取socketChannel
                        client = (SocketChannel) selectionKey.channel();
                        // 创建buffer,并将获取socketChannel中的数据读入到buffer中
                        ByteBuffer readBuf = ByteBuffer.allocate(1024);
                        int readCount = client.read(readBuf);
                        if (readCount <= 0) {
                            return;
                        }
                        Charset charset = Charset.forName(StandardCharsets.UTF_8.name());
                        readBuf.flip();
                        System.out.println(String.valueOf(charset.decode(readBuf).array()));
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
                selectionKeys.remove(selectionKey);
            });
        }

    }
```
> 通过这个例子，与原始`Reactor`模式相对应的理解，比如同步事件分离器对应着`Selector`的`select()`方法，再比如`ServerSocketChannel`注册给`Selector`的`OP_ACCEPT`，还有`SocketChannel`的`OP_READ`与`OP_WRITE`，这些事件保存在操作系统上，其实就是原始`Reactor`中的`Handle`。
#### 四个重要api
1. `Channel`：Connections to files,sockets etc that support non-blocking reads.
2. `Buffer`：Array-like objects that can be directly read or written by Channels.
3. `Selector`：Tell which of a set of Channels have IO events.
4. `SelectionKeys`：Maintain IO event status and bingdings.
### 3.用Java NIO对Reactor模式的应用。
![](https://user-gold-cdn.xitu.io/2019/4/5/169ec2a5292faeeb?w=816&h=516&f=png&s=135652)
#### 3.1 Single threaded version
```
/**
 * @Author CoderJiA
 * @Description Reactor
 * @Date 5/4/19 下午2:25
 **/
public abstract class Reactor implements Runnable{


    protected final Selector selector;
    protected final ServerSocketChannel serverSocket;

    protected final long port;
    protected final long timeout;

    public Reactor(int port, long timeout) throws IOException {
        this.port = port;
        this.timeout = timeout;
        selector = Selector.open();
        serverSocket = ServerSocketChannel.open();
        serverSocket
                .socket()
                .bind(new InetSocketAddress(port));
        serverSocket.configureBlocking(false);
        SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT);
        sk.attach(newAcceptor(selector));
    }

    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                if (selector.select(timeout) > 0) {
                    Set<SelectionKey> selected = selector.selectedKeys();
                    selected.forEach(sk -> {
                        dispatch(sk);
                        selected.remove(sk);
                    });
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void dispatch(SelectionKey sk) {
        Runnable r = (Runnable)(sk.attachment());
        if (Objects.nonNull(r)) {
            r.run();
        }
    }

    public abstract Acceptor newAcceptor(Selector selector);

}

```
```
/**
 * @Author CoderJiA
 * @Description Acceptor
 * @Date 5/4/19 下午2:58
 **/
public class Acceptor implements Runnable {

    private final Selector selector;
    private final ServerSocketChannel serverSocket;

    public Acceptor(Selector selector, ServerSocketChannel serverSocket) {
        this.selector = selector;
        this.serverSocket = serverSocket;
    }

    @Override
    public void run() {
        try {
            SocketChannel socket = serverSocket.accept();
            if (Objects.nonNull(socket)) {
                new Handler(selector, socket);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```
```
/**
 * @Author CoderJiA
 * @Description Handler
 * @Date 5/4/19 下午4:25
 **/
public class Handler implements Runnable {

    private static final int MB = 1024 * 1024;

    protected final SocketChannel socket;
    protected final SelectionKey sk;
    protected final ByteBuffer input = ByteBuffer.allocate(MB);
    protected final ByteBuffer output = ByteBuffer.allocate(MB);

    private static final int READING = 0, SENDING = 1;
    private int state = READING;

    public Handler(Selector selector, SocketChannel socket) throws IOException {
        this.socket = socket;
        socket.configureBlocking(false);
        sk = socket.register(selector, SelectionKey.OP_READ);
        sk.attach(this);
    }

    @Override
    public void run() {
        try {
            if (state == READING) read();
            else if (state == SENDING) send();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void read() throws IOException {
        socket.read(input);
        if (inputIsComplete()) {
            state = SENDING;
            sk.interestOps(SelectionKey.OP_WRITE);
        }
        input.clear();
    }

    private void send() throws IOException {
        socket.write(output);
        if (outputIsComplete()) {
            sk.cancel();
        }
    }

    private boolean inputIsComplete() {
        return input.position() > 0;
    }

    private boolean outputIsComplete() {
        return !output.hasRemaining();
    }

}
```
```
/**
 * @Author CoderJiA
 * @Description EchoReactor
 * @Date 5/4/19 下午5:01
 **/
public class EchoReactor extends Reactor {

    private static final int PORT = 9999;
    private static final long TIME_OUT = TimeUnit.MILLISECONDS.toMillis(10);

    public EchoReactor(int port, long timeout) throws IOException {
        super(port, timeout);
    }

    @Override
    public Acceptor newAcceptor(Selector selector) {
        return new Acceptor(selector, this.serverSocket);
    }

    public static void main(String[] args) throws IOException {
        new EchoReactor(PORT, TIME_OUT).run();
    }

}
```
##### 核心组件组件分析
1. `Reactor`等同于`原始Reactor模式`的`Initiation Dispatcher`，它负责所有就绪事件统一分发到事件处理器，如`Acceptor`和`Hanlder`。
2. `Acceptor`用于将接收到的`SocketChannel`交给Handler处理。
3. `Handler`处理读写操作。
> 这是`Reactor`的单线程版本，这个版本一个线程处理`客户端的接收`和`数据处理`以及`读写操作`，数据处理往往就是我们实际开发中的业务处理，是比较耗时的。如果一个处理过程处于`阻塞`，那么这个模型所表现出的就处于`阻塞`，所以一个数据处理的阻塞会导致不能处理客户端连接的接收。因此衍生出来下面的多工作线程版本来优化`Handler`。
#### 3.2 Worker Threads version

![](https://user-gold-cdn.xitu.io/2019/4/6/169f08273c125d28?w=1288&h=866&f=png&s=274243)

调整下Handler
```
package cn.coderjia.nio.douglea.reactor2;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @Author CoderJiA
 * @Description Handler
 * @Date 5/4/19 下午4:25
 **/
public class Handler implements Runnable {

    private static final int MB = 1024 * 1024;

    protected final SocketChannel socket;
    protected final SelectionKey sk;
    protected final ByteBuffer input = ByteBuffer.allocate(MB);
    protected final ByteBuffer output = ByteBuffer.allocate(MB);

    private static final int READING = 0, SENDING = 1, PROCESSING = 3;
    private int state = READING;

    private static final ExecutorService EXECUTOR_SERVICE = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    public Handler(Selector selector, SocketChannel socket) throws IOException {
        this.socket = socket;
        socket.configureBlocking(false);
        sk = socket.register(selector, SelectionKey.OP_READ);
        sk.attach(this);
    }

    @Override
    public void run() {
        try {
            if (state == READING) read();
            else if (state == SENDING) send();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void read() throws IOException {
        socket.read(input);
        if (inputIsComplete()) {
            state = PROCESSING;
            EXECUTOR_SERVICE.execute(new Processer());
        }
        input.clear();
    }

    private void send() throws IOException {
        socket.write(output);
        if (outputIsComplete()) {
            sk.cancel();
        }
    }

    private void process() {
        System.out.println("Handler.process()...");
    }

    private boolean inputIsComplete() {
        return input.position() > 0;
    }

    private boolean outputIsComplete() {
        return !output.hasRemaining();
    }

    class Processer implements Runnable {
        public void run() {
            processAndHandOff();
        }
    }

    synchronized void processAndHandOff() {
        process();
        state = SENDING;
        sk.interestOps(SelectionKey.OP_WRITE);
    }

}
```
> `Handler`多工作线程版本将耗时的`process()`，创建线程去处理。这个版本`Reactor`既负责客户端的接收事件，又负责读写事件，因为对于高并发场景连接数巨大，`Reactor`可能有时候会力不从心。因此衍生出下面的`主从Reactor`模型。
#### 3.3 Multiple Reactors Version
![](https://user-gold-cdn.xitu.io/2019/4/6/169f09f330eaf24b?w=1262&h=876&f=png&s=289890)
调整Acceptor

```
/**
 * @Author CoderJiA
 * @Description Acceptor3
 * @Date 6/4/19 下午6:51
 **/
public class Acceptor3 implements Runnable {

    private final ServerSocketChannel serverSocket;

    public Acceptor3(ServerSocketChannel serverSocket) {
        this.serverSocket = serverSocket;
    }

    @Override
    public void run() {
        try {
            SocketChannel socket = serverSocket.accept();
            if (Objects.nonNull(socket)) {
                new Handler(EchoReactor.nextSubReactor().selector, socket);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
}
```
调整Reactor
```
/**
 * @Author CoderJiA
 * @Description Reactor3
 * @Date 6/4/19 下午6:51
 **/
public abstract class Reactor3 implements Runnable {


    protected Selector selector;
    protected ServerSocketChannel serverSocket;

    protected final int port;
    protected final long timeout;
    protected final boolean isMainReactor;

    public Reactor3(int port, long timeout, boolean isMainReactor) {
        this.port = port;
        this.timeout = timeout;
        this.isMainReactor = isMainReactor;
    }

    @Override
    public void run() {
        try {
            init();
            while (!Thread.interrupted()) {
                if (selector.select(timeout) > 0) {
                    System.out.println("isMainReactor:" + isMainReactor);
                    Set<SelectionKey> selected = selector.selectedKeys();
                    selected.forEach(sk -> {
                        dispatch(sk);
                        selected.remove(sk);
                    });
                    selected.clear();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void init() throws IOException {
        selector = Selector.open();
        if (isMainReactor) {
            serverSocket = ServerSocketChannel.open();
            serverSocket
                    .socket()
                    .bind(new InetSocketAddress(port));
            serverSocket.configureBlocking(false);
            SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT);
            sk.attach(newAcceptor());
        }
    }

    private void dispatch(SelectionKey sk) {
        Runnable r = (Runnable)(sk.attachment());
        if (Objects.nonNull(r)) {
            r.run();
        }
    }

    public abstract Acceptor3 newAcceptor();

}

```
```
/**
 * @Author CoderJiA
 * @Description EchoReactor
 * @Date 6/4/19 下午5:35
 **/
public class EchoReactor extends Reactor3 {

    private static final int PORT = 9999;
    private static final long TIME_OUT = TimeUnit.MILLISECONDS.toMillis(10);

    private static final int SUB_REACTORS_SIZE = 2;
    private static final Reactor3[] SUB_REACTORS = new Reactor3[SUB_REACTORS_SIZE];
    private static final AtomicInteger NEXT_INDEX = new AtomicInteger(0);

    static {
        // 初始化子Reactor
        IntStream.range(0, SUB_REACTORS_SIZE).forEach(i -> SUB_REACTORS[i] = new EchoReactor(PORT, TIME_OUT, false));
    }

    public static Reactor3 nextSubReactor(){

        int curIdx = NEXT_INDEX.getAndIncrement();

        if(curIdx >= SUB_REACTORS_SIZE){
            NEXT_INDEX.set(0);
            curIdx = 0;
        }
        return SUB_REACTORS[(curIdx % SUB_REACTORS_SIZE)];
    }

    public EchoReactor(int port, long timeout, boolean isMainReactor) {
        super(port, timeout, isMainReactor);
    }

    @Override
    public Acceptor3 newAcceptor() {
        return new Acceptor3(this.serverSocket);
    }

    public static void main(String[] args) {

        Reactor3 mainReactor = new EchoReactor(PORT, TIME_OUT, true);

        // 启动主Reactor
        new Thread(mainReactor).start();

        // 启动子Reactor
        IntStream.range(0, SUB_REACTORS_SIZE).forEach(i -> new Thread(SUB_REACTORS[i]).start());
    }

}
```
> 主从`Reactor`模型，`主Reactor`用于处理客户端连接的接收转发给`Acceptor`处理，`子Reactor`处理读写事件的接收转发给`Handler`处理。
### 参考文章
> Scalable IO in Java 
### 源码地址
> https://github.com/coderjia0618/basic-study

