### 序
> 掘金是自己刚发现不久的平台，原本一些学习笔记都是记录在有道，因为正好两边都支持markdown，现在打算把一些整理后的笔记分享出来。这篇主要来简单的聊聊网络请求中的内存拷贝。
### 网络请求中数据传输过程图
#### 数据传输类型一（read）

![](https://user-gold-cdn.xitu.io/2019/2/23/169190d79631bdd5?w=1338&h=910&f=png&s=108391)
> 该数据传输模型正是传统的IO进行网络通讯时所采用的方式，数据在用户空间（JVM内存）与内核空间进行多次拷贝和上下文切换，对于没有对数据进行业务处理的时候，这样拷贝显得很没有必要。
    
#### 数据传输类型二（sendFile）

![](https://user-gold-cdn.xitu.io/2019/2/23/169191ec50e200d5?w=1366&h=944&f=png&s=97145)
> 该数据传输模型是的NIO进行网络通讯时所采用的方式，它依赖于操作系统是否支持这种对于内核的操作（图中第4个过程），这个模型对比第一种减少了两次不必要的用户空间和内核之间的数据拷贝过程。
#### 数据传输类型三（支持聚集的sendFile）


![](https://user-gold-cdn.xitu.io/2019/2/23/169191fb314d6ff8?w=1390&h=762&f=png&s=81514)
> 从中我们可以发现这种真正实现了**零拷贝**，这种传输模型它依赖于操作系统是否支持这种对于内核的操作（图中4过程），图中4过程看着很难理解，下面把四过程里面的奥秘分解下。


![](https://user-gold-cdn.xitu.io/2019/2/23/1691976f97598eaa?w=746&h=196&f=png&s=135314)
> 四过程其实是将内核中文件信息（文件地址、大小等信息）appendStr到Sokcet Buffer中，这样Sokcet Buffer中存有很少的信息，然后在协议引擎传输之前使用Gather将两个Buffer聚集。
#### 数据传输类型四（mmap，本文先不介绍）
### 代码实现
#### 模型一（BIO）
```
/**
 * @Author CoderJiA
 * @Description TransferModel1Client
 * @Date 23/2/19 下午3:01
 **/
public class TransferModel1Client {

    private static final String HOST = "localhost";
    private static final int PORT = 8899;
    private static final String FILE_PATH = "/Users/coderjia/Documents/gradle-5.2.1-all.zip";
    private static final int MB = 1024 * 1024;

    public static void main(String[] args) throws Exception{
        Socket socket = new Socket(HOST, PORT);
        InputStream input = new FileInputStream(FILE_PATH);
        DataOutputStream output = new DataOutputStream(socket.getOutputStream());
        byte[] bytes = new byte[MB];
        long start = System.currentTimeMillis();
        int len;
        while ((len = input.read(bytes)) != -1) {
            output.write(bytes, 0, len);
        }
        long end = System.currentTimeMillis();
        System.out.println("耗时:" + (end - start) + "ms");
        output.close();
        input.close();
        socket.close();
    }
}
```
```
/**
 * @Author CoderJiA
 * @Description TransferModel1Server
 * @Date 23/2/19 下午3:01
 **/
public class TransferModel1Server {

    private static final int PORT = 8899;
    private static final int MB = 1024 * 1024;

    public static void main(String[] args) throws Exception{
        ServerSocket serverSocket = new ServerSocket(PORT);
        for (;;) {
            Socket socket = serverSocket.accept();
            DataInputStream input = new DataInputStream(socket.getInputStream());
            byte[] bytes = new byte[MB];
            for (;;) {
                int readSize = input.read(bytes, 0, MB);
                if (-1 == readSize) {
                    break;
                }
            }
        }
    }
}
```
#### 模型二（NIO）
```
/**
 * @Author CoderJiA
 * @Description TransferModel2Client
 * @Date 23/2/19 下午3:36
 **/
public class TransferModel2Client {

    private static final String HOST = "localhost";
    private static final int PORT = 8899;
    private static final String FILE_PATH = "/Users/coderjia/Documents/gradle-5.2.1-all.zip";

    public static void main(String[] args) throws Exception {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.connect(new InetSocketAddress(HOST, PORT));
        socketChannel.configureBlocking(true);
        FileChannel fileChannel = new FileInputStream(FILE_PATH).getChannel();
        long start = System.currentTimeMillis();
        fileChannel.transferTo(0, fileChannel.size(), socketChannel);
        long end = System.currentTimeMillis();
        System.out.println("耗时:" + (end - start) + "ms");
        fileChannel.close();
    }

}
```
```
/**
 * @Author CoderJiA
 * @Description TransferModel2Server
 * @Date 23/2/19 下午3:36
 **/
public class TransferModel2Server {

    private static final int PORT = 8899;
    private static final int MB = 1024 * 1024;

    public static void main(String[] args) throws Exception {

        InetSocketAddress address = new InetSocketAddress(PORT);

        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        ServerSocket serverSocket = serverSocketChannel.socket();
        serverSocket.setReuseAddress(true);
        serverSocket.bind(address);

        ByteBuffer byteBuffer = ByteBuffer.allocate(MB);
        for (;;) {
            SocketChannel socketChannel = serverSocketChannel.accept();
            socketChannel.configureBlocking(true);
            int readSize = 0;
            while (-1 != readSize) {
                readSize = socketChannel.read(byteBuffer);
                byteBuffer.rewind();
            }

        }

    }
}
```
> fileChannel.transferTo(0, fileChannel.size(), socketChannel)

transferto方法的文档注释:This method is potentially much more efficient than a simple loop that reads from this channel and writes to the target channel.Many operating systems can transfer bytes directly from the filesystem cache to the target channel without actually copying them.

这句话简单的理解就是：该方法比传统的简单轮询（指的就是IO中的拷贝过程）更加高效，tranferto的拷贝方式依赖于底层操作系统，目前很多操作系统支持像模型二拷贝过程。在内核版本2.4中，修改了套接字缓冲区描述符以适应这些要求——在Linux下称为零拷贝。 这种方法不仅减少了多个上下文切换，还完全消除了处理器的数据复制操作。
### 性能对比
> 在同一台机器上，相同环境下测试结果如图。

![](https://user-gold-cdn.xitu.io/2019/2/23/169196c28d08da3f?w=1278&h=734&f=png&s=60408)

### 参考文章地址 
> https://www.jianshu.com/p/e9f422586749
### 源码地址
> https://github.com/coderjia0618/basic-study