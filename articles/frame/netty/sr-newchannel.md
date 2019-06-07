## Netty新连接接入
检测新链接->创建NioSocketChannel->分配线程及注册selector->向selector注册读事件
### 检测新连接
#### 步骤
- processSelectedKey(key,channel)入口
  - NioMessageUnsafe.read()
    - doReadMessages() while循环
      - javaChannel().accept()  
### 创建NioSocketChannel
#### 步骤
- new NioSocketChannel(parent,ch) 入口
  - AbstractNioByteChannel(p,ch,op_read)
    - configureBlocking(false)&save op
    - create id,unsafe,pipeline
  - new NioSocketChannelConfig()
    - setTcpNoDelay(true)禁止Nagle算法 
### Netty中的Channel的分类
- NioServerSocketChannel
- NioSocketChannel
- Unsafe
Channel和Unsafe两张类图理解
### 新连接NioEventLoop分配和selector注册

服务端Channel的pipeline构成
Head --> ServerBootstrapAcceptor --> tail

- ServerBootstrapAcceptor
  - 添加childHandler
  - 设置options和attrs
  - Chooser选择NioEventLopp并注册selector
### NioSocketChannel向selector读事件的注册


