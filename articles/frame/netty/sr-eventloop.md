## NioEventLoop原理解读
### NioEventLoop创建
#### 步骤
- new NioEventLoopGroup() 线程组，默认2*cpu
  - new ThreadPerTaskExecutor() 线程创建器
    - 每次执行任务都会创建一个线程实体
    - NioEventLoop线程命名规则nioEventLoop-1-xx
  - for(){newChild()} 构造NioEventLoop
    - 保存线程执行器ThreadPerTaskExecutor
    - 创建一个MpscQueue
    - 创建一个selector
  - chooserFactory.newChooser() 线程选择器
    - isPowerOfTwo() 是否等于2的幂次
      - PowerOfTwoEventExecutorChooser
        - index++&(length-1)
      - GenericEventExecutorChooser
        - abs(index++%length)
### NioEventLoop启动触发器
#### 步骤
- 服务端启动绑定端口
  - bind() -> execute(task) 入口
  - startThread() -> doStartThread() 创建线程
    - ThreadPerTaskExecutor.execute()
      - thread = Thread.currentThread()
      - NioEventLoop.run() 启动
- 新连结接入通过chooser选择个EventLoop

### NioEventLoop执行逻辑
####  步骤
-  NioEventLoop.run()
   -  run() -> for(;;)
      -  select() 检查是否有io事件
         -  deadline以及任务穿插逻辑处理
         -  阻塞式select
         -  避免jdk空轮询bug
      -  processSelectedKeys() 处理io事件
         -  selected keySet优化 实际上是数组 add事件复杂度为O(1)
         -  proccessSelectedKeysOptimized()
      -  runAllTasks() 处理异步任务队列
         -  task的分类和添加
            -  普通taskQueue
            -  scheduleQueue创建的添加
         -  任务的聚合
         -  任务的执行


