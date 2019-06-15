## Netty解码器
### ByteToMessageDecoder解码步骤
> 其实Netty解码过程只有decode步骤是调用具体子类的decode方法
1. 累加字节流
2. 调用子类的decode方法进行解析  
3. 将解析到的ByteBuf向下传播
### 常见解码器分析
- 固定长度解码器：FixedLengthFrameDecoder
![](https://user-gold-cdn.xitu.io/2019/6/10/16b40be7882a9348?w=1214&h=424&f=png&s=24370)
> 根据frameLength去截取等长的字节数。
- 什么是丢弃模式
> 丢弃模式的意义是因为字节不断写到同一个ByteBuf中，然后去decode这个ByteBuf，如果超过最大的长度(maxLength)的时候，会将此ByteBuf进入丢弃模式(discarding为true)，这样在找到下一个结束符的时候就会将此段超过长度的数据丢弃掉，并且此ByteBuf进入非丢弃模式(discarding为false)。
- 行解码器：LineBasedFrameDecoder
> stripDelimiter为true会跳过行分隔符，stripDelimiter为false不会跳过行分隔符。
![](https://user-gold-cdn.xitu.io/2019/6/10/16b417eddb63a7ba?w=1446&h=694&f=png&s=50708)
- 分隔符解码器：DelimiterBasedFrameDecoder
> 分隔符解码器与行解码器类似，根据分隔符分隔，当分隔符为\n时，底层调用的就是行解码器。
- 长度域解码器：LengthFieldBasedFrameDecoder
![](https://user-gold-cdn.xitu.io/2019/6/11/16b43cf535fa17ff?w=1424&h=532&f=png&s=61594)
> lengthFieldOffset=1代表从第一个字节开始，lengthFieldLength=2代表长度域的长度为2，那也即使图中的16进制0x0010，它表示的长度为16，lengthAdjustment=-3代表调整为-3也就是16-3=13个字节长度，initialBytesToStrip=3,代表从开始跳过3个字节。