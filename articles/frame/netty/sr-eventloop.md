## NioEventLoop原理解读
### NioEventLoop创建
#### 步骤
- new NioEventLoopGroup() 线程组，默认2*cpu
  - new ThreadPerTaskExecutor() 线程创建器
    - 每次执行任务都会创建一个线程实体
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
#### 分析
- new NioEventLoopGroup() 线程组，默认2*cpu
```
    protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
    }
    static {
        // 可获取线程数*2，目前大部分都是超线程，所以NettyRuntime.availableProcessors()可能获取的核数的2倍
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));
        ...
    }
```
- new ThreadPerTaskExecutor() 线程创建器
```
public final class ThreadPerTaskExecutor implements Executor {

    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        this.threadFactory = threadFactory;
    }

    // 它会给每一个执行任务创建一个新的netty包装过的FastThreadLocalThread去运行
    // 使用了命令设计模式
    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
}
```
-  for(){newChild()} 构造NioEventLoop
```
    children = new EventExecutor[nThreads];
    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
           ...
        } finally {
            ...
        }
    }
```
- 跟着newChild(..)到NioEventLoop的构造方法，依次进入它的父类构造方法
```
   NioEventLoop(..) {
        super(parent, executor, false, DEFAULT_MAX_PENDING_TASKS, rejectedExecutionHandler);
        ...
        provider = selectorProvider;
        // 创建一个selectorTuple，netty对底层的selectionKeySet进行了优化，因为原本的selectionKeySet底层是Map结构，add的时候时间复杂度是O(n)的，netty中直接使用数组的结构来代替它add编程O(1)
        final SelectorTuple selectorTuple = openSelector();
        // 被包装过的selector，它提供了一些基于新的keyset的一些遍历的方法
        selector = selectorTuple.selector; 
        // 没有被包装过的selector 也就nio的selector
        unwrappedSelector = selectorTuple.unwrappedSelector;
        ...
    }
    protected SingleThreadEventExecutor(..) {
        // 省略代码
        ...
        // 保存线程执行器ThreadPerTaskExecutor
        this.executor = ObjectUtil.checkNotNull(executor, "executor");
        ...
    }
    // 它的父类 SingleThreadEventLoop 会初始化个tailTasks
    protected SingleThreadEventLoop(...) {
        super(parent, executor, addTaskWakesUp, maxPendingTasks, rejectedExecutionHandler)
        // 初始化taskQueue
        tailTasks = newTaskQueue(maxPendingTasks);
    }
    // 创建一个MpscQueue
    protected Queue<Runnable> newTaskQueue(int maxPendingTasks) {
        // This event loop never calls takeTask()
        return maxPendingTasks == Integer.MAX_VALUE ? PlatformDependent.<Runnable>newMpscQueue()
                                                    : PlatformDependent.<Runnable>newMpscQueue(maxPendingTasks);
    }
```
- chooserFactory.newChooser() 线程选择器
```
public final class DefaultEventExecutorChooserFactory implements EventExecutorChooserFactory {

    public static final DefaultEventExecutorChooserFactory INSTANCE = new DefaultEventExecutorChooserFactory();

    private DefaultEventExecutorChooserFactory() { }

    @SuppressWarnings("unchecked")
    @Override
    public EventExecutorChooser newChooser(EventExecutor[] executors) {

        /**
        * 这块优化的问题可以简化为：
        *   对于能被2整除的数n，为什么x%n = x&(n-1)
        *   举个例子 int n = 4,8 的二进制存储
        *    0000 0000 0000 0100  -1后   0000 0000 0000 0011
        *    0000 0000 0000 1000  -1后   0000 0000 0000 0111
        *    根据例子可以发现满足了一点就是模一个数n之后最大也就为n-1，当前的数值x我也不用关注大于我的前边的值为1还是0 
        *  只关注按位与上我的后面的结果即可。    
        */
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }
    // 判断是否是2的幂次
    private static boolean isPowerOfTwo(int val) {
        return (val & -val) == val;
    }
    private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }
        @Override
        public EventExecutor next() {
            return executors[idx.getAndIncrement() & executors.length - 1];
        }
    }
    private static final class GenericEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        GenericEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }
        @Override
        public EventExecutor next() {
            return executors[Math.abs(idx.getAndIncrement() % executors.length)];
        }
    }
}
```
### NioEventLoop启动触发器
#### 步骤
- 服务端启动绑定端口
  - bind() -> execute(task) 入口
  - startThread() -> doStartThread() 创建线程
    - ThreadPerTaskExecutor.execute()
      - thread = Thread.currentThread()
      - NioEventLoop.run() 启动
- 新连结接入通过chooser选择个EventLoop
#### 分析
- bind() -> execute(task) 入口
```
    private static void doBind0(..) {
        // 省略代码...
        channel.eventLoop().execute(task);
    }
```
- startThread() -> doStartThread() 创建线程
```
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        addTask(task);
        if (!inEventLoop) {
            startThread();
            ...
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
    private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
                // thread = Thread.currentThread()
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }
                boolean success = false;
                updateLastExecutionTime();
                try {
                    // 这个时候NioEventLoop的run方法才执行
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    ...
                }
            }
        });
    }
```
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
         -  task的添加
            -  普通taskQueue
            -  scheduleQueue创建的添加
         -  任务的聚合
         -  任务的执行
#### 分析
-  run() -> for(;;)
```
    @Override
    protected void run() {
        for (;;) {
            try {
                try {
                    switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // ...

                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));
                        // 省略代码...
                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                        // fall through
                    default:
                    }
                } catch (IOException e) {
                    ...
                    continue;
                }
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;

                if (ioRatio == 100) {
                    try {
                        // 处理select出来的事件
                        processSelectedKeys();
                    } finally {
                        ...
                        runAllTasks();
                    }
                } else {   // 默认是50
                    final long ioStartTime = System.nanoTime();
                    try {
                        // 处理select出来的事件
                        processSelectedKeys();
                    } finally {
                        ...
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // ...省略代码
        }
    }
```
-  select() 检查是否有io事件
```
    private void select(boolean oldWakenUp) throws IOException {
        Selector selector = this.selector;
        try {
            int selectCnt = 0;
            long currentTimeNanos = System.nanoTime();
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

            for (;;) {

                // 如果到了第一个定时任务的截止事件还没有select，那么就进行一次非阻塞的selectNow，这样会执行堆积的task
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
                if (timeoutMillis <= 0) {
                    if (selectCnt == 0) {
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    break;
                }

                // 如果我们的任务队列中有任务，也进行一次selectNow处理任务
                if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                // 如果上面情况都不满足，就进行一次阻塞的select
                int selectedKeys = selector.select(timeoutMillis);
                selectCnt ++;

                // 下面这段代码是解决jdk空轮询的bug
                long time = System.nanoTime();
                // 执行到这的时间time - 在selector.select(timeoutMillis)方法之前获取的时间 currentTimeNanos 都大于等于 轮询的时间timeoutMillis 就说明轮询是空轮询
                if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                    // timeoutMillis elapsed without anything selected.
                    selectCnt = 1;
                } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) { // 空轮询次数大于重新构建selector阈值512
                    // 避免jdk空轮询bug，解决方式就是重新构建selector
                    selector = selectRebuildSelector(selectCnt);
                    selectCnt = 1;  // 轮询次数归1
                    break;
                }
                currentTimeNanos = time;
            }
        } catch (CancelledKeyException e) {
            //省略代码...
        }
    }
```
-  processSelectedKeys() 处理io事件
  - selected keySet优化 实际上是数组 add事件复杂度为O(1) 在上面newChild有提到过
```
// 优化原本的set，底层直接用数组存储。因为这里不会轮到重复的key
final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {

    SelectionKey[] keys;
    int size;

    SelectedSelectionKeySet() {
        keys = new SelectionKey[1024];
    }

    @Override
    public boolean add(SelectionKey o) {
        if (o == null) {
            return false;
        }

        keys[size++] = o;
        if (size == keys.length) {
            increaseCapacity();
        }

        return true;
    }

    @Override
    public boolean remove(Object o) {
        return false;
    }

    @Override
    public boolean contains(Object o) {
        return false;
    }

    @Override
    public int size() {
        return size;
    }

    @Override
    public Iterator<SelectionKey> iterator() {
        return new Iterator<SelectionKey>() {
            private int idx;

            @Override
            public boolean hasNext() {
                return idx < size;
            }

            @Override
            public SelectionKey next() {
                if (!hasNext()) {
                    throw new NoSuchElementException();
                }
                return keys[idx++];
            }

            @Override
            public void remove() {
                throw new UnsupportedOperationException();
            }
        };
    }

    void reset() {
        reset(0);
    }

    void reset(int start) {
        Arrays.fill(keys, start, size, null);
        size = 0;
    }

    private void increaseCapacity() {
        SelectionKey[] newKeys = new SelectionKey[keys.length << 1];
        System.arraycopy(keys, 0, newKeys, 0, size);
        keys = newKeys;
    }
}
```
// 通过反射的方式对原先的selectedKeysField进行重新赋值
```
 if (!(maybeSelectorImplClass instanceof Class) ||
            // ensure the current selector implementation is what we can instrument.
            !((Class<?>) maybeSelectorImplClass).isAssignableFrom(unwrappedSelector.getClass())) {
            if (maybeSelectorImplClass instanceof Throwable) {
                Throwable t = (Throwable) maybeSelectorImplClass;
                logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, t);
            }
            return new SelectorTuple(unwrappedSelector);
        }

        final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;
        final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

        Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
                    Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

                    if (PlatformDependent.javaVersion() >= 9 && PlatformDependent.hasUnsafe()) {
                        long selectedKeysFieldOffset = PlatformDependent.objectFieldOffset(selectedKeysField);
                        long publicSelectedKeysFieldOffset =
                                PlatformDependent.objectFieldOffset(publicSelectedKeysField);

                        if (selectedKeysFieldOffset != -1 && publicSelectedKeysFieldOffset != -1) {
                            PlatformDependent.putObject(
                                    unwrappedSelector, selectedKeysFieldOffset, selectedKeySet);
                            PlatformDependent.putObject(
                                    unwrappedSelector, publicSelectedKeysFieldOffset, selectedKeySet);
                            return null;
                        }
                        // We could not retrieve the offset, lets try reflection as last-resort.
                    }

                    Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField, true);
                    if (cause != null) {
                        return cause;
                    }
                    cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField, true);
                    if (cause != null) {
                        return cause;
                    }

                    selectedKeysField.set(unwrappedSelector, selectedKeySet);
                    publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
                    return null;
                } catch (NoSuchFieldException e) {
                    return e;
                } catch (IllegalAccessException e) {
                    return e;
                }
            }
        });
```
-  处理事件proccessSelectedKeysOptimized()
```
      int readyOps = k.readyOps();
      // 处理连接事件
      if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
          // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
          // See https://github.com/netty/netty/issues/924
          int ops = k.interestOps();
          ops &= ~SelectionKey.OP_CONNECT;
          k.interestOps(ops);

          unsafe.finishConnect();
      }
      // 处理写事件
      // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
      if ((readyOps & SelectionKey.OP_WRITE) != 0) {
          // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
          ch.unsafe().forceFlush();
      }
      // 处理读事件
      // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
      // to a spin loop
      if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
          unsafe.read();
      }
```
-  runAllTasks() 处理异步任务队列
-  普通任务队列taskQueue的新增
```
   public void execute(Runnable task) {
        ...
        addTask(task);
        ...
    }
   protected void addTask(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (!offerTask(task)) {
            reject(task);
        }
    }

    final boolean offerTask(Runnable task) {
        if (isShutdown()) {
            reject();
        }
        return taskQueue.offer(task);
    }
```
-  任务的聚合与执行 在NioEventLoop的run方法中，runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
```
    protected boolean runAllTasks(long timeoutNanos) {
        // ...省略代码
        // 定时任务聚合到普通任务队列
        fetchFromScheduledTaskQueue();
        // 运行所有普通任务队列的任务
        afterRunningAllTasks();
        return true;
    }
    // 把大于当前执行事件的定时任务都加入到普通队列中
    private boolean fetchFromScheduledTaskQueue() {
        long nanoTime = AbstractScheduledEventExecutor.nanoTime();
        Runnable scheduledTask  = pollScheduledTask(nanoTime);
        while (scheduledTask != null) {
            if (!taskQueue.offer(scheduledTask)) {
                // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
                scheduledTaskQueue().add((ScheduledFutureTask<?>) scheduledTask);
                return false;
            }
            scheduledTask  = pollScheduledTask(nanoTime);
        }
        return true;
    }
    // 所有任务
    protected final boolean runAllTasksFrom(Queue<Runnable> taskQueue) {
        Runnable task = pollTaskFrom(taskQueue);
        if (task == null) {
            return false;
        }
        for (;;) {
            // 执行task的run方法
            safeExecute(task);
            task = pollTaskFrom(taskQueue);
            if (task == null) {
                return true;
            }
        }
    }
```
