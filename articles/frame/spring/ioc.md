### 1.IOC的理论背景
在面向对象设计的软件系统中，它的底层都是由N个对象构成的，各个对象之间通过相互合作，最终实现系统地业务逻辑。<br>
如果我们打开机械式手表的后盖，就会看到与上面类似的情形，各个齿轮分别带动时针、分针和秒针顺时针旋转，从而在表盘上产生正确的时间。上图中描述的就是这样的一个齿轮组，它拥有多个独立的齿轮，这些齿轮相互啮合在一起，协同工作，共同完成某项任务。<br>
齿轮组中齿轮之间的啮合关系,与软件系统中对象之间的耦合关系非常相似。对象之间的耦合关系是无法避免的，也是必要的，这是协同工作的基础。现在，伴随着工业级应用的规模越来越庞大，对象之间的依赖关系也越来越复杂，经常会出现对象之间的多重依赖性关系，因此，架构师和设计师对于系统的分析和设计，将面临更大的挑战。对象之间耦合度过高的系统，必然会出现牵一发而动全身的情形。<br>
耦合关系不仅会出现在对象与对象之间，也会出现在软件系统的各模块之间，以及软件系统和硬件系统之间。如何降低系统之间、模块之间和对象之间的耦合度，是软件工程永远追求的目标之一。为了解决对象之间的耦合度过高的问题，软件专家Michael Mattson 1996年提出了IOC理论，用来实现对象之间的“解耦”，目前这个理论已经被成功地应用到实践当中。
### 2.什么是IOC
`IOC`是Inversion of Control的缩写，多数书籍翻译成“控制反转”。<br>
1996年，Michael Mattson在一篇有关探讨面向对象框架的文章中，首先提出了`IOC`这个概念。对于面向对象设计及编程的基本思想，前面我们已经讲了很多了，不再赘述，简单来说就是把复杂系统分解成相互合作的对象，这些对象类通过封装以后，内部实现对外部是透明的，从而降低了解决问题的复杂度，而且可以灵活地被重用和扩展。<br>
`IOC`理论提出的观点大体是这样的：借助于“第三方”实现具有依赖关系的对象之间的解耦。如下图：
### 3.IOC也叫依赖注入(DI)
2004年，Martin Fowler探讨了同一个问题，既然`IOC`是控制反转，那么到底是“哪些方面的控制被反转了呢？”，经过详细地分析和论证后，他得出了答案：“获得依赖对象的过程被反转了”。控制被反转之后，获得依赖对象的过程由自身管理变为了由`IOC`容器主动注入。于是，他给“控制反转”取了一个更合适的名字叫做“依赖注入（Dependency Injection）”。他的这个答案，实际上给出了实现`IOC`的方法：注入。所谓依赖注入，就是由`IOC`容器在运行期间，动态地将某种依赖关系注入到对象之中。
### 4.Spring IOC原理
#### 4.1 概念
Spring 通过一个`XML`配置文件或者`@Bean`描述`Bean`及`Bean`之间的依赖关系,利用`Java`语言的反射功能实例化`Bean`并建立`Bean`之间的依赖关系。`Spring`的`IoC`容器在完成这些底层工作的基础上,还提供了`Bean`实例缓存、生命周期管理、`Bean`实例代理、事件发布、资源装载等高级服务。
#### 4.2 Spring容器视图
`Spring`启动时读取应用程序提供的`Bean`配置信息,并在`Spring` 容器中生成一份相应的`Bean`配置注册表,然后根据这张注册表实例化`Bean`,装配好`Bean`之间的依赖关系,为上层应用提供准 备就绪的运行环境。其中`Bean`缓存池为`HashMap`实现。
#### 4.3 相关源码阅读
```
public class DefaultSingletonBeanRegistry {

	/** Cache of singleton objects: bean name --> bean instance */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name --> ObjectFactory */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name --> bean instance */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
}

public class DefaultListableBeanFactory {
    /** Map of bean definition objects, keyed by bean name */
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
}
/**
* Return the (raw) singleton object registered under the given name.
* <p>Checks already instantiated singletons and also allows for an early
* reference to a currently created singleton (resolving a circular reference).
* @param beanName the name of the bean to look for
* @param allowEarlyReference whether early references should be created or not
* @return the registered singleton object, or {@code null} if none found
*/
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```


