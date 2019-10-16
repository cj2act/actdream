> 之前十一假期，基于SpringBoot实现集成公司业务和通用封装的starter，有点类似支付宝的Sofa-Boot。在实现过程中，不断优化的过程发现对源码理解不好，starter很容易写的不那么“聪明”。所以就趁着假期一点点跟着源码阅读了起来，今天来分享一篇总结简单Bean的生命周期。
### 我的技术学习方法
> 我阅读源码的过程中，通常分两步走：跳出来，看全景；钻进去，看本质。先了解Bean的生命周期图对其有大体了解，该技术在大脑中有了一个“简单”的构图后。再阅读相关源码去确认自己在看全景时的大脑构图是否合理。这个过程其实也适用于面对新环境，先熟悉自己所处项目的业务，再过技术栈与代码相关细节。钻进去看本质的过程往往是“煎熬的”，需要坚持与不断的思考。
### Bean的生命周期图
![](https://user-gold-cdn.xitu.io/2019/10/13/16dc4bf61972735b?w=1692&h=1198&f=png&s=263265)
### Bean创建过程的源码阅读索引
> Spring源码版本: 5.0.9.RELEASE
#### 源码阅读索引
```
               DefaultListableBeanFactory
                            | preInstantiateSingletons(726)  -->  getBean(759);
                AbstractBeanFactory
                            | getBean(198)  -->  doGetBean(199)
                            | doGetBean(239)  -->  getSingleton(315)
                                                    DefaultSingletonBeanRegistry
                                                                | getSingleton(202)  -->  singletonFactory.getObject(222)
                            | getSingleton(315)  -->  createBean(317)
                AbstractAutowireCapableBeanFactory
                            | createBean(456)  -->  doCreateBean(495)
                            | doCreateBean(526)  -->  createBeanInstance(535)  return BeanWrapper
                            | doCreateBean(526)  -->  addSingletonFactory(566) 加入到三级缓存SingletonFactories
                            | doCreateBean(526)  -->  populateBean(572)
                            | doCreateBean(526)  -->  initializeBean(573)
                            | initializeBean(1678)  -->  invokeAwareMethods(1686)
                            | invokeAwareMethods(1709)  -->  ((BeanNameAware) bean).setBeanName(1712)
                            | invokeAwareMethods(1709)  -->  ((BeanClassLoaderAware) bean).setBeanClassLoader(1717)
                            | invokeAwareMethods(1709)  -->  ((BeanFactoryAware) bean).setBeanFactory(1721)
                            | initializeBean(1678)  -->  applyBeanPostProcessorsBeforeInitialization(1691)   这里面通常调用的是AwareProcessor
                            | initializeBean(1678)  -->  invokeInitMethods(1695)
                            | invokeInitMethods(1695)  -->  ((InitializingBean) bean).afterPropertiesSet(1758)
                            | invokeInitMethods(1695)  -->  invokeCustomInitMethod(1767)
                            | initializeBean(1678)  -->  applyBeanPostProcessorsAfterInitialization(1703)
```
#### 上面三个类之间的关系
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc914478861926?w=1788&h=772&f=png&s=66364)
> 上面的类继承图是我简化过的，其实真正的Spring的BeanFacotory类泛化体系远比此复杂多。
### Bean的创建过程源码分析
#### 示例代码
```
@Slf4j
@Component
public class ComponentA implements InitializingBean, DisposableBean, BeanFactoryAware, ApplicationContextAware {

    @Autowired
    private ComponentB componentB;

    @Value("ca.test.val:default")
    private String valA;

    @Override
    public void destroy() {
        log.info("$destroy()...");
    }

    @Override
    public void afterPropertiesSet() {
        log.info("$afterPropertiesSet()...");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        log.info("$setApplicationContext() applicationContext:{}", applicationContext);
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        log.info("$setBeanFactory() beanFactory:{}", beanFactory);
    }

}
```
```
@Component
public class ComponentB {
    
    @Autowired
    private ComponentA componentA;

}
```
```
@SpringBootApplication
public class EasyUcApplication {
    public static void main(String[] args) {
         /**
         * 使用SpringBoot-2.0.5.RELEASE去启动Spring容器，对应就是需要的Spring-5.0.9.RELEASE。
         */
        SpringApplication.run(EasyUcApplication.class, args);
    }
}
```
#### 开始源码阅读
- 按照源码阅读索引的入口，我们查看下源码
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc8f7ea4e260ca?w=1844&h=986&f=png&s=295557)
> 因为这块是循环遍历所有`BeanDefinitionNames`，`F9`找到上面我们想要看到初始化过程的Bean。下文提到的所有`F7`、`F8`、`F9`分别代表进入方法内部、跳到下一行、跳到下一个断点处。
- `F8`按照索引目录跟到759行
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc8fd951436df1?w=1792&h=558&f=png&s=103534)
- `F7`进入此方法，进入到抽象类`AbstractBeanFactory`
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc8fe94c9c2bf8?w=1840&h=366&f=png&s=84568)
- `F7`进入doGetBean方法
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc90310a1dafd9?w=1802&h=546&f=png&s=183460)
进入到这个方法后，看下246行，它最终会进入到`DefaultSigletonBeanRegistry`中的`getSingleton`
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc909a31371c1b)
到这先认识下这三个缓存`Map`：

    - singletonObjects: singleton object cache
    - earlySingletonObjects: Cache of singleton objects exposed in advance
    - singletonFactories: singleton object factory cache

    这三级缓存具体作用我们在`populate`阶段再分析
- 返回到`AbstractBeanFactory`的`doGetBean(246)`,`F8`接着跟到315行
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc923637efc05e?w=1882&h=558&f=png&s=164797)
第二个参数是个函数式接口
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc928e8210d5cf?w=1250&h=368&f=png&s=79089)
- `F7`跟进去，我们看到到第222行调用了`getObject()`
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc92a05ce66ab4?w=1874&h=768&f=png&s=297510)
- `getObject()` `F7`跟进去其实调用的就是`createBean`
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc92d0ee8dd33b?w=1884&h=490&f=png&s=124893)
- `F7`跟进去，到了`AbstractAutowireCapableBeanFacotory`的`createBean(456)`, 在这个方法内找到495行, `F7`进入此方法
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc930154057a47?w=1844&h=512&f=png&s=142213)
接下来才开始正式进入到创建Bean的过程
#### 创建Bean对象，加入第三级缓存中
- 上一步我们进入`AbstractAutowireCapableBeanFacotory`的`doCreateBean(526)`行，`F8`一直到535行,`F7`进入此方法。
```
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    /*省略代码*/
    // 调用有Autowire的构造方式: 就是说构造方法的成员变量是需要注入值的。
    Class<?> beanClass = resolveBeanClass(mbd, beanName);
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null ||
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        return autowireConstructor(beanName, mbd, ctors, args);
    }
    // 简单的无参构造器：我们这里会走到这种方式
    return instantiateBean(beanName, mbd);  //1128
}
```
- 接着`F7`进入到上面的`instantiateBean(1128)`
```
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    /*省略代码*/
    // 1.实例化对象
    Object beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
    // 2.创建该对象的BeanWrapper
    BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    initBeanWrapper(bw);
    return bw;
}
```
- 上面出现个BeanWrapper，它的作用是干什么的呢？

    - 我们实例化对象的时候，其实是调用构造器去反射创建对象。
    - 当我们想对其属性值赋值的时候,就可以使用`setPropertyValue`去进行赋值，`BeanWrapper`提供一种可以直接对属性赋值的能力。如下面所示：
```
    Class<?> clazz = Class.forName("cn.coderjia.easyuc.component.ComponentA");
    Constructor<?> ctor = clazz.getDeclaredConstructor();
    Object bean = ctor.newInstance();
    BeanWrapper beanWrapper = new BeanWrapperImpl(bean);
    beanWrapper.setPropertyValue("valA", "val");
```
- 我们一直`F8`返回`AbstractAutowireCapableBeanFacotory`的`doCreateBean(535)`, 然后`F8`到559行。
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc9890b56dbb5c?w=1906&h=622&f=png&s=165392)
- 接着`F8`跟到566行
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc98c765430425?w=1966&h=588&f=png&s=191421)
- `F7`进入此方法，到了DefaultSingletonBeanRegistry
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc98e69392b628?w=1864&h=614&f=png&s=213732)
![](https://user-gold-cdn.xitu.io/2019/10/14/16dc993431569f2d)
我们可以发现这步是将对象创建的工厂函数缓存到`singletonFactories`，目的为了解决循环引用的问题，那么什么是循环引用的情况与如何解决的呢？且看下文分解。
#### 注入Bean的属性
- 上小节我们提出了疑问什么是循环引用？我们先来看一张图。
![](https://user-gold-cdn.xitu.io/2019/10/14/16dcaab6ae039020?w=1352&h=658&f=png&s=50669)
正如我们的示例代码，`ComponetA`依赖`ComponentB`，`ComponentB`又依赖于`ComponentA`，`Spring`中`Bean`的实例化`Bean`对象后，会对属性值注入（`populate`），这样就会出现如上图的循环调用过程(`instantiate A`->`populate ComponentB`->`instantiateB`->`populate ComponentA`->`instantiate A`)。
- 上面我们分析了什么是循环引用，接下来我们先接着跟源码。
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcd3711805fc22?w=1886&h=754&f=png&s=222067)
我们上回书说到，将实例化后未初始化的`Bean`对象工厂放置到第三级缓存中。
- `F7`进入到getEarlyBeanReference(566)
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcd39e7d12fcab?w=1916&h=658&f=png&s=198825)
理论上讲，两级缓存就够了，对象实例化后放到`earlySingletonObjects`缓存中，在需要提前引用的地方取出就可以解决循环引用的问题。但是第三级缓存存在的意义除了保存`Bean`外，还可以在获取的时候，对`Bean`可以做前置处理(`InstantiationAwareBeanPostProcessor`)，有这样一种拓展的能力。
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcd5da9992152a?w=1424&h=628&f=png&s=78779)
简单的理解就如上面图中的过程: 创建`Component A`后，把`Component A`缓存起来，注入`Component B`属性时，创建`Component B`，`Component B`需要注入`Component A`，从缓存把`Component A`取出来，完成`Component B`的创建全过程，在将`Component B`返回给`Component A`，`Component A`也可完成创建全过程。
- `F8`跟到`populateBean(572)`
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcd66a0efe3913?w=1918&h=756&f=png&s=223913)
按照我们上面的解释，这步是会去注入`Componet B`，然后自然而然需要创建`Componet B`。接下来看下是不是这样。
- `F7`进入此方法，到了`AbstractAutowireCapableBeanFactory`的`populateBean(1277)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcd6e2683e1aef?w=1936&h=698&f=png&s=200875)
- `F8`一直往下跟1339行打个断点，然后一直`F9`跳到`bp`为`AutowiredAnnotationBeanPostProcessor`时候停止，`F8`跟到 1341行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcd7cf12a4a924?w=1956&h=736&f=png&s=204020)
- `F7`进入此方法，来到`CommonAnnotationBeanPostProcessor`的`metadata.inject(318)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcd7d369cfb2fa?w=1858&h=812&f=png&s=218176)
- `F7`进入此方法，来到`InjectionMetadata`的`inject(81)`
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcd813691dfab0?w=1960&h=1066&f=png&s=342520)
写到这我们发现需要注入的属性`componetB`和`valA`已经在`checkoutedElements`中，要对其进行值的注入。可以发现@Autowired和@Value值就是在这阶段对其赋值的。那么`componetB`和`valA`什么时候加到`checkoutedElements`中的呢？
- 先回退下，找到`AbstractAutowireCapableBeanFactory`的`applyMergedBeanDefinitionPostPorcessors(547)`行，打个断点
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce59842a8b4e4?w=1952&h=944&f=png&s=227012)
- 重新启动容器，`F9`一直跳到`ComponetA`
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce5cce81dd9b7?w=1996&h=804&f=png&s=275612)
- `F7`跟进，进入到`AbstractAutowireCapableBeanFactory`的`applyMergedBeanDefinitionPostPorcessors(1009)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce5eea872f65e?w=2044&h=780&f=png&s=330871)
- `1011`行打个断点，然后`F9`一直跳到`bp`为`AutowiredAnnotationBeanPostProcessor`
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce6513e0f3aed?w=1938&h=922&f=png&s=316848)
- `F7`进入`AutowiredAnnotationBeanPostProcessor`的`postProcessMergedBeanDefinition(231)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce68ed5509da1?w=1908&h=538&f=png&s=156817)
- `F8`跟到232行，`F7`进入此方法，来到了`AutowiredAnnotationBeanPostProcessor`的`findAutowiringMetadata(405)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce6aca02f89ef?w=1936&h=758&f=png&s=214365)
- `F8`跟到`AutowiredAnnotationBeanPostProcessor`的`buildAutowiringMetadata(417)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce6dc3a54d550?w=1988&h=716&f=png&s=251172)
- `F7`进入此方法,来到了`AutowiredAnnotationBeanPostProcessor`的`buildAutowiringMetadata(425)`行,然后在433行打个断点
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce72dcc54c8e1?w=1930&h=790&f=png&s=229310)
- `F9`一直跳到`ComponetB`或者`valA`
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce741f3678f4d?w=1992&h=1176&f=png&s=350124)
- `F7`进入此方法,来到了`AutowiredAnnotationBeanPostProcessor`的`findAutowiredAnnotation(480)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce75dc464ba97?w=1930&h=598&f=png&s=173752)
跟到这我们明白了，获取到当前对象的所有`field`，然后找看看有没有我们“感兴趣”的`field`，“感兴趣”的注解是提前定义好的`，放在autowiredAnnotationTypes`中。
- `autowiredAnnotationTypes`中都有哪些注解，我们找到`AutowiredAnnotationBeanPostProcessor`的构造方法。
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce7a1826ac42f?w=1838&h=788&f=png&s=200755)
我们可以看到“感兴趣”的注解有`@Autowired`、`@Value`、`@Inject`，所以`ComponetB`或者`valA`会最终被放置到`checkedElements`中。
- 接下来我们回到`InjectionMetadata`的`inject(81)`，接着分析注入的过程
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce96306ba137f?w=1960&h=800&f=png&s=261180)
通过上面分析我们知道elementsToIterate一共有两个属性需要注入，分别是`ComponetB`或者`valA`
- `F7`进入`InjectionMetadata`的`inject(90)`行，来到`AutowiredAnnotationBeanPostProcessor`的`inject(570)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce99e38c22061?w=1806&h=776&f=png&s=203606)
- `F8`跟到583行，`F7`进入`beanFactory.resolveDependency(583)`行，来到了`DefaultListableBeanFacotory`的`doResolveDependency(1062)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dce9d49fc4dbee?w=1930&h=732&f=png&s=242809)
- `F7`进入`doResolveDependency(1062)`,来到`DefaultListableBeanFacotory`的1069行，接着在1135行打个断点，`F9`跳到此位置
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcea3419995cb2?w=1916&h=796&f=png&s=250402)
我们知道当前需要注入的一个属性是ComponetB是个类，所以会走这进行初始化
- `F7`进入`descriptor.resolveCandidate(1135)`行，来到`DependencyDescriptor`的`resolveCandidate(248)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcea62cab19660?w=1980&h=644&f=png&s=225795)
- `F7`进入`beanFactory.getBean(251)`行，一直跟到`AbstractBeanFacotory`的`doGetBean(239)`行，以下过程都是创建`Componet B`的过程，所以聚集在一起，等`Componet B`中需要注入`Component A`时我们在分开。    
![](https://user-gold-cdn.xitu.io/2019/10/15/16dceab7cb8d6d18?w=1954&h=808&f=png&s=294486)
到246行,其实就是到了上面循环依赖解决方案图中第4步先从cache取出B的实例，显然没有那么就接着按图中走，第5步没有B的示例去instantiate个示例。
![](https://user-gold-cdn.xitu.io/2019/10/15/16dceb7ea069c04e?w=2122&h=952&f=png&s=360023)
然后在317行打个断点，`F9`跳到这
![](https://user-gold-cdn.xitu.io/2019/10/15/16dceb9d302353f0?w=1978&h=996&f=png&s=270262)
`F7`进入，`F8`跟到`doCreateBean(495)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcebe78fc02e3d?w=1776&h=724&f=png&s=199143)
`F7`进入，`F8`跟到`doCreateBean(572)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcec3642261c3e?w=1930&h=808&f=png&s=274750)
我们现在应该很清楚的知道，我们到了解决循环依赖问题的第7步骤`populate ComponentA`，`F7`进入`populateBean`方法。
- `populate ComponentA`时，肯定最终会去试图创建`ComponentA`，也会回到`AbstractBeanFacotry`的`doGetBean(239)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcec7c166ebed2?w=1936&h=778&f=png&s=275507)
- `F7`进入`getSingleton(246)`行，一直跟到`DefaultSingletonBeanRegistry`的`getSingleton(176)`行，`F8`跟到185行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf1cef23c9ef9?w=2098&h=1330&f=png&s=549842)
我们到了解决循环依赖问题的第8步骤，从`cache`中取出`Bean`，而且这个`Bean`正好是我们上文书所说的那样，是实例化后的，未完成属性注入和初始化的。
- `F8`到最后`doGetBean(392)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf22f90c8b5ec?w=1332&h=574&f=png&s=142905)
- 接下来就是一直`F8`跟到`AutowiredAnnotationBeanPostProcessor`的inject(611)行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf272a2bde7b5?w=1994&h=1208&f=png&s=291286)
这个时候就可以发现，`ComponetB`创建完成，注入的是预先缓存好的示例化的`ComponetA`
- 接下仍然一直`F8`跟回到`AbstractAutowireCapableBeanFacotry`的`doCreateBean(573)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf2afeb15c400?w=1962&h=788&f=png&s=255726)
到这可以知道，`ComponetB`对于`ComponetA`已经注入完成，573行是执行`ComponetB`初始化操作，我们主要看`ComponetA`的生命周期，所以可以直接`F9`跳过后面过程回到`ComponetA`的`populateBean`
- `F9`跳回到`ComponetA`的`populateBean`
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf32258b2e9e8?w=2436&h=1454&f=png&s=553150)
至此`Component A`彻底完成初始化和属性注入的过程，而且控制台没有任何输出也符合我们生命周期图的预测。
#### 初始化方法的前置处理
- `F7`进入`AbstractAutowireCapableBeanFacotry`的`doCreateBean(573)`行来到`initializeBean(1678)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf34feb4a04e1?w=1998&h=950&f=png&s=274913)
- `F7`进入`invokeAwareMethods(1686)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf35c271a560a?w=1846&h=634&f=png&s=135974)
我们的示例代码`ComponetA implements BeanFactoryAware`, 所以这块会回调`setBeanFactory`
- `F8`执行完`invokeAwareMethods`
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf3830473e393?w=2464&h=1320&f=png&s=432588)
符合我们生命周期图的预期，打印了`$setBeanFactory()...`
- `F8`跳回到`initializeBean(1691)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf39fdec97ad5?w=1984&h=768&f=png&s=246065)
- `F7`跟进去，到了`AbstractAutowireCapableBeanFacotry`的`applyBeanPostProcessorsBeforeInitialization(411)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf3e520bdb3b5?w=1994&h=876&f=png&s=234697)
跟到416行时候可以发现两个事情，第一个是这里面通常执行的是`*AwareProcessor`且是在`invokeAwareMethods`后面执行的，符合我们生命周期图，第二个是`ApplicationContextAwareProcessor`是默认给我们所有bean都创建的`*AwareProcessor`，我们来看下这个`ApplicationContextAwareProcessor`是干什么的。
- `F7`进入`applyBeanPostProcessorsBeforeInitialization(416)`行，来到了`ApplicationContextAwareProcessor`的`postProcessBeforeInitialization(79)`行，`F8`一直跟到`invokeAwareInterfaces(96)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf426b21d4b6a?w=1976&h=850&f=png&s=275261)
- `F7`进入`invokeAwareInterfaces(96)`行，来到了`invokeAwareInterfaces(102)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf43ce5ce1c14?w=1960&h=836&f=png&s=300553)
原来还会帮我们默认加`ApplicationContextAwareProcessor`，如果我们实现了这些接口，它就会回调。我们的示例代码中` ComponentA implements ApplicationContextAware`，所以应该会执行ComponentA的`setApplicationContext`
- `F8`跟到`invokeAwareInterfaces(102)`这个方法的最后，看下打印结果。
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf486f1335404?w=2312&h=1388&f=png&s=463574)
符合我们直接执行的`*Aware`接口，生命周期要早于`*AwareProcessor`。
#### 所有属性注入结束
- `AbstractAutowireCapableBeanFacotry`的`initializeBean(1678)`中的`invokeInitMethods(1695)`行，`F7`进入此方法，再`F8`跟到`invokeInitMethods(1758)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf4dfcf1a5873?w=1838&h=860&f=png&s=193120)
- `F8`执行`afterPropertiesSet`，因为`ComponentA implements InitializingBean`，所以控制台肯定会打印`$afterPropertiesSet()...`，如下图所示：
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf4f4334839d8?w=2090&h=1336&f=png&s=403836)
#### 开始init
- `F8`下面的1767行就是执行，目标init方法
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf516a863510e?w=2016&h=952&f=png&s=262192)
#### 创建结束
- `F8`跟回到`AbstractAutowireCapableBeanFacotry`的`applyBeanPostProcessorsAfterInitialization(1703)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf5350d5b27f7?w=1944&h=852&f=png&s=263541)
这个过程就是调用初始化的后置处理器和前置处理器过程类似。
- `F8`跟到`AbstractAutowireCaplableBeanFacotry`的`doCreateBean(614)`行
![](https://user-gold-cdn.xitu.io/2019/10/15/16dcf6e5ef7f107a?w=1960&h=864&f=png&s=230738)
`Bean`注册为`DisposableBean`会随着`容器`销毁而销毁。
- `F8`一直跟回到`DefaultSingletonBeanRegistry`的`getSingleton(248)`行
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd1b1e1fdb49d3?w=1912&h=806&f=png&s=235788)
- `F7`进入到此方法，看看创建Bean的最后一步
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd1b292c9bafde?w=1816&h=640&f=png&s=219828)
将bean从二级缓存`earlySingletonObjects`转存到以及缓存`singletonObjects`，以上过程我们完成了`Bean`的整个创建过程，`Spirng`之所以处理的这么“松散”(低耦合)，其实就是在`Bean`整个生命周期过程中，提供给用户更好的拓展能力。结合自己的业务场景，灵活定制。
### Bean的销毁过程源码分析
- 先来了解下`Runtime.getRuntime().addShutdownHook(Thread hook)`，写段测试代码。
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd25bd19fbbcf5?w=1266&h=332&f=png&s=62032)
运行结果：
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd25b81ccb9813?w=1266&h=96&f=png&s=9140)
当main方法执行结束，相当于`System.exit(0)`退出，就会回调是添加的`Hook Thread`，`Spring`也是这样做的。
- 理解`Spring`的钩子线程，找到`SpringApplication`的`run(333)`行，进入`refreshContext(333)`行，来到`SpringApplication`的415行，`F7`进入此方法，来到`AbstractApplicationContext`的`registerShutdownHook(930)`行
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd26514f1c835c?w=1766&h=524&f=png&s=115334)
这段代码与我们的示例代码类似，这个`钩子`当系统退出时调用`doClose()`方法，这段小内容还要碎碎念个事儿，`SpringBoot`我们知道他帮我们自动装配了很多配置，其实本质是在`spring.factories`定义好需要自动装配的类，把定义好的类包装成`Spring`需要的`BeanDefinition`，剩下的事儿就交给`Spring`就好了。当然`SpringBoot`还有自己完整的事件发布体系（`观察者模式`），让整个`SpringBoot`的代码结构看起来非常清晰。这些话虽然有些跑题，但是这就是我为什么搞`SpringBoot starter`却进入了`Spring`的世界。
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd26ba5695e36d?w=2408&h=804&f=png&s=451751)
- 一个不得不说的点
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd271f7419b3fd?w=1272&h=204&f=png&s=44283)
不知道你有没有想过一个问题，上面的main方法为什么执行完不退出。我相信你看完这篇文章（[JVM退出问题](http://dubbo.apache.org/zh-cn/blog/spring-boot-dubbo-start-stop-analysis.html)）,就会找到答案。
- 回到正题，你知道有非`后台线程`存在，只能使用`System.exit(0)`或者`Runtime.getRuntime().exit(0)`，那么我们稍做修改代码。
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd2776b6499b94?w=1266&h=238&f=png&s=53686)
- 在重新启动之前，我们在`AbstractApplicationContext`的`doClose(1017)`行打个断点
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd27aac4d2788c?w=1964&h=1256&f=png&s=322787)
- 一直`F7`跟进到`DefaultSingletonBeanRegistry`的`destroySingletons(491)`行，在504行打个断点。我们上文说到`ComponentA`注册为`DisposableBean`，`F9`跳到`ComponetA`。`F7`进入此方法，`F8`跟到543行，
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd33474c00942e?w=1970&h=854&f=png&s=254817)
- `F7`进入此方法，`F8`跟到`destroyBean(571)`行并`F8`执行
![](https://user-gold-cdn.xitu.io/2019/10/16/16dd33ae03029e83?w=1890&h=1066&f=png&s=276265)
### 总结
> 整篇文章不仅是想从表面了解`Bean`的世界，而是要走进它的世界。我在写这篇文章的时候，经常问自己这地方你懂了吗？答案模棱两可就是不懂，就需要去仔细品味源码。读源码的作用到底是什么？它会让你见识到一个顶级项目整体的架构，应用了哪些设计模式，一些问题的处理方式等等。这样在日常工作中就可以“借鉴”，让自己的代码是充满着“温度”的好代码。
### 相关资料
- 官方文档：https://docs.spring.io/spring/docs/5.0.15.RELEASE/spring-framework-reference/core.html#beans-child-bean-definitions
- 循环依赖：http://www.programmersought.com/article/9703154930/
- JVM退出问题：http://dubbo.apache.org/zh-cn/blog/spring-boot-dubbo-start-stop-analysis.html