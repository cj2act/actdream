## ByteBuf
### 内存与内存管理器的抽象
#### 步骤
- ByteBuf的结构
- mark和reset方法 
- ByteBuf分类
  - Pooled和Unpooled  
  - Unsafe和非Unsafe 
  - Head和Direct 
#### 解析
- ByteBuf的结构


### 不同规格大小和不同类别的内存的分配策略
- 内存分配器ByteBufAllcator
- 两大子类PooledByteBufAllocator与UnPooledByteBufAllocator
- UnpooledByteBufAllocator分析
  - heap内存的分配
  - direct内存的分配
- PooledByteBufAllocator分析
  - 拿到线程局部缓存PoolThreadCache
  - 在线程局部缓存的Arena上进行内存分配 
  - directArena分配direct内存的流程
    - 从内存池中拿到PooledByteBuf对象进行复用
    - 命中缓存从缓存上进行内存分配
    - 如果没命中缓存则进行实际内存分配
- 内存规格的介绍
- 命中缓存的分配逻辑
  - 找到对应size的MemoryRegionCache
  - 从queue中取出一个entry给ByteBuf初始化
  - 将取出的entry扔到对象池中进行复用
- arene、chunk、page、subpage概念
- page级别的内存分配：allocateNormail()
  - 尝试在现有的chunk上分配
  - 创建一个chunk进行分配
  - 初始化PooledByteBuf
-  subpage级别的内存分配：allocateTiny()
   -  定位一个Subpage对象
   -  初始化Subpage
   -  初始化PooledByteBuf
### 内存的回收的过程
- 连续的内存区段加到缓存
- 标记连续的内存区段为未使用
- ByteBuf加入到对象池

本文学习过程中核心问题：
- ByteBuf分类：
  - Unsafe和非Unsafe 直接获取内存地址，然后直接对内存地址进行操作
  - Head和Direct 
  - Pooled和Unpooled  前者会获取内存中已经分配好的区域，后者直接去分配
- PooledByteBufAllocator分析
- 最复杂的PooledByteBufAllocator分析，按照获取内存过程进行分析
  - 从对象池中获取
  - 从缓存中获取
  - 实际内存分配

