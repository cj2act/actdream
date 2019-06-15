## Netty编码
- writeAndFlush()
  - 从tail节点开始往前传播
  - 逐个调用channelHandler的writer方法
  - 逐个调用channelHandler的flush方法

编码器处理逻辑：MessageToByteEncoder
- 匹配对象，是否能够处理
  - acceptOutboundMessage()   
- 分配内存
- 编码实现，去覆盖encode方法 把对象转存到ByteBuf中
- 释放对象
- 传播数据
- 释放内存

unsafe.write()
- write-写buffer队列
  - direct化ByteBuf
  - 插入写队列
      - ChannelOutboundBuffer flushedEntry --已经写到flush里面--> unflushedEntry --没有写到flush里面--> tailEntry
  - 设置写状态 当超过默认的写队列大小(6K)时候，设置状态为不可写
- flush-刷新buffer队列
  - 添加刷新标志并设置写状态
  - 遍历buffer队列，过滤ByteBuf
    - doWrite(..)
  - 调用jdk底层api进行自旋写 




