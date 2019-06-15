## Netty设计模式应用
### 单例模式
- 应用程序中的Singleton对象都使用Singleton类的同一实例
#### 实现方式
- 实现方式一
```
public class Singleton1 {

    public static final Singleton1 INSTANCE = new Singleton1();

    private Singleton1(){}

}
```
- 实现方式二
```
public class Singleton2_1 {

    private Singleton2_1() {}

    private static Singleton2_1 instance = null;

    public static Singleton2_1 getInstance() {
        if (null == instance) {
            instance = new Singleton2_1();
        }
        return instance;
    }

}
```
```
public class Singleton2_2 {

    private Singleton2_2() {}

    private static Singleton2_2 instance = null;

    public static Singleton2_2 getInstance() {
        if (null == instance) {
            synchronized(Singleton2_2.class) {
                if (null == instance) {
                    instance = new Singleton2_2();
                }
            }
        }
        return instance;
    }

}
```
```
public class Singleton2_3 {

    private Singleton2_3() {}

    // 防止指令重排
    private static volatile Singleton2_3 instance = null;

    public static Singleton2_3 getInstance() {
        if (null == instance) {
            synchronized(Singleton2_3.class) {
                if (null == instance) {
                    instance = new Singleton2_3();
                }
            }
        }
        return instance;
    }

}
```
- 实现方式三
```
public class Singleton3 {

    private Singleton3() {}

    public static Singleton3 getInstance() {
        return SingletonHolder.instance;
    }

    private static class SingletonHolder {
        private static final Singleton3 instance = new Singleton3();
    }

}
```
- 实现方式四
```
public enum Singleton4 {
    INSTANCE;
}
```
#### Netty中的应用
- ReadTimeoutException
```
public final class ReadTimeoutException extends TimeoutException {

    private static final long serialVersionUID = 169287984113283421L;

    public static final ReadTimeoutException INSTANCE = new ReadTimeoutException();

    private ReadTimeoutException() { }
}
```
 - MqttEncoder
```
@ChannelHandler.Sharable
public final class MqttEncoder extends MessageToMessageEncoder<MqttMessage> {

    public static final MqttEncoder INSTANCE = new MqttEncoder();

    private MqttEncoder() { }

    //...省略代码
}
```
### 策略模式
- 对象都具有职责
- 这些职责不同的具体实现是通过多态完成
- 相同的算法具有多个不同的实现，需要进行管理
#### 组成
- 抽象策略角色：通常有一个接口或一个抽象类实现
- 具体策略角色：包装了相关的算法和行为
- 环境角色(也称上下文)：持有一个策略类的应用，最终供客户端调用
#### 实现方式
- 抽象策略角色
```
public interface PubStrategy {
    void pub(String key);
}
```
- 具体策略角色
```
public class RolePubStrategy implements PubStrategy {

    @Override
    public void pub(String key) {
        // get users by role key
        System.out.println("1.Get users by role key...");
        // foreach pub msg to users
        System.out.println("2.Foreach pub msg to users...");
    }

}
```
```
public class UserPubStrategy implements PubStrategy {

    @Override
    public void pub(String key) {
        // find user by key
        System.out.println("1.Get user by key...");
        // pub msg to user
        System.out.println("2.Pub msg to user...");
    }

}
```
```
public class StrategyContext {

    private PubStrategy strategy;

    public StrategyContext(PubStrategy strategy) {
        this.strategy = strategy;
    }

    public void pub(String key) {
        strategy.pub(key);
    }

}
```
#### Netty中的应用
- 一个变种的策略模式应用EventExecutorChooserFactory#newChooser(EventExecutor[])
```
@UnstableApi
interface EventExecutorChooser {

    /**
        * Returns the new {@link EventExecutor} to use.
        */
    EventExecutor next();
}
```
```
public final class DefaultEventExecutorChooserFactory implements EventExecutorChooserFactory {

    public static final DefaultEventExecutorChooserFactory INSTANCE = new DefaultEventExecutorChooserFactory();

    private DefaultEventExecutorChooserFactory() { }

    @SuppressWarnings("unchecked")
    @Override
    public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }

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
### 装饰者模式
- 动态的给对象添加个职责
#### 组成
- 装饰接口
- 装饰者
- 被装饰者
#### 实现方式
> 上面策略模式的StrategyContext修改成StrategyContextDecorator。
```
public class StrategyContextDecorator implements PubStrategy {
    
    private PubStrategy pubStrategy;
    
    public StrategyContextDecorator(PubStrategy pubStrategy) {
        this.pubStrategy = pubStrategy;
    }
    
    @Override
    public void pub(String key) {
        pubStrategy.pub(key);
    }
    
}
```
#### Netty中的应用
```
class WrappedByteBuf extends ByteBuf {

    protected final ByteBuf buf;

    protected WrappedByteBuf(ByteBuf buf) {
        if (buf == null) {
            throw new NullPointerException("buf");
        }
        this.buf = buf;
    }

    @Override
    public final boolean hasMemoryAddress() {
        return buf.hasMemoryAddress();
    }

    // ...省略代码
}
```
### 观察者模式
#### 组成
- 抽象主题（被观察的对象）：
  - 主题是观察者观察的对象，一个主题必须具备下面三个特征：
    - 持有监听的观察者的引用
    - 支持增加和删除观察者
    - 主题状态改变，通知观察者
- 观察者：
  - 当主题发生变化，收到通知进行具体的处理是观察者必须具备的特征。
- 具体的抽象主题
- 具体的观察者
#### 实现方式
```
public abstract class Observerable {

    protected List<Observer> observers;
    protected String message;

    /**
     * 添加一个观察者
     * @param observer 观察者对象
     */
    abstract public void registerObserver(Observer observer);

    /**
     * 移除一个观察者
     * @param observer 观察者对象
     */
    abstract public void removeObserver(Observer observer);

    /**
     * 通知所有观察者
     */
    abstract public void notifyObserver(String message);

}
```
```
public class ConcreteObserverable extends Observerable {

    public ConcreteObserverable() {
        observers = Lists.newArrayList();
    }

    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObserver(String message) {
        this.message = message;
        for (Observer observer: observers) {
            observer.update(message);
        }
    }

}
```
```
public interface Observer {
    public void update(String message);
}
```
```
public class ConcreteObserver implements Observer {

    @Override
    public void update(String message) {
        System.out.println("message up: " + message);
    }

}
```
#### Netty中的应用
```
// 被观察对象
public interface ChannelFuture extends Future<Void> {

    Channel channel();

    // ...省略代码
    @Override
    ChannelFuture addListener(GenericFutureListener<? extends Future<? super Void>> listener);

    @Override
    ChannelFuture removeListener(GenericFutureListener<? extends Future<? super Void>> listener);
   
}
// 具体被观察对象
public class DefaultPromise<V> extends AbstractFuture<V> implements Promise<V> {

    private Object listeners;
  
    @Override
    public Promise<V> setSuccess(V result) {
        if (setSuccess0(result)) {
            notifyListeners();
            return this;
        }
        throw new IllegalStateException("complete already: " + this);
    }
}
// notifyListeners 实现
private void notifyListeners0(DefaultFutureListeners listeners) {
    GenericFutureListener<?>[] a = listeners.listeners();
    int size = listeners.size();
    for (int i = 0; i < size; i ++) {
        notifyListener0(this, a[i]);
    }
}
// 观察者
public interface ChannelFutureListener extends GenericFutureListener<ChannelFuture> {
    ChannelFutureListener FIRE_EXCEPTION_ON_FAILURE = new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) {
            if (!future.isSuccess()) {
                future.channel().pipeline().fireExceptionCaught(future.cause());
            }
        }
    };

}
```
### 责任链模式
> 避免请求发送者与接收者耦合在一起，让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止。
#### 组成
```
public abstract class AbstractHandler {

    private AbstractHandler next;

    abstract protected void eventSpread();

    public void setNext(AbstractHandler next) {
        this.next = next;
    }

    public void spread() {
        eventSpread();
        // 向下传播
        if (null != next) {
            next.eventSpread();
        }
    }

}
public class Handle1 extends AbstractHandler {

    @Override
    public void eventSpread() {
        System.out.println("Handle1 event spread...");
    }

}
public class Handle2 extends AbstractHandler {

    @Override
    public void eventSpread() {
        System.out.println("Handle2 event spread...");
    }

}
public class TestChain {

    public static void main(String[] args) {

        AbstractHandler h1 = new Handle1();
        AbstractHandler h2 = new Handle2();

        h1.setNext(h2);

        h1.spread();
    }

}
```
#### Netty中的应用
> ChannelPipeline是责任链模式的变种，详见ChannelPipeline原理解读。



