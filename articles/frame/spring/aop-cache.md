### 服务内部方法级别Cache的一种实现
#### 背景
近期工作中，某些场景下，部分暴露的GET Feign接口被频繁调用，由于请求参数相同情况下，只是在不停的通过这个接口拿到想要的信息，比如拥有某部分权限的用户信息接口。权限服务A与用户服务B是拆分开的服务，库并不属于同一个库，所以每次都需要去服务B去获取用户信息，这种情况下对权限服务A的调用函数做缓存优化是一个不错的方案。具体情况如下图: 
![](https://user-gold-cdn.xitu.io/2019/8/22/16cb8da39925d80b?w=1268&h=636&f=png&s=46046)
优化方案，使用Aop代理，在方法调用前，增加预处理。如果命中缓存直接从缓存中取，否则执行目标方法并缓存。优化后的如下图:
![](https://user-gold-cdn.xitu.io/2019/8/22/16cb8eb04ec5ab89?w=1356&h=958&f=png&s=106130)
> 如图需要定义个缓存注解，如果该注解修饰的方法，就根据缓存策略做拦截预处理。
#### 定义命中缓存注解
```
/**
 * 服务内部缓存注解，方法级别注解。
 * @author CoderJiA
 */
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface EasyCache {

    /**
     * 缓存唯一key前缀
     */
    String key() default "";

    /**
     * 数据域表达式
     */
    String fieldExps() default "";

    /**
     * 是否过期
     */
    boolean isExpire() default false;

}
```
- 缓存唯一key前缀规则:
```
缓存唯一key前缀
默认命名规则: ClassSimpleName#MethodName
如: A#method
class A {
     public void method(){}
}
```
- 数据域表达式规则:
```
p/coll/arr 代表支持的参数类型
0 1        代表参数下标
对象与属性用[.]标识 对象多个属性间用[&]
例:
基本数据类型: p
  @EasyCache(fieldExps = "#p0=?")
  public String getCacheByP(String userName);
集合类型: coll
  @EasyCache(fieldExps = "#coll0=?")
  public List<Long> getCacheByColl(List<Long> ids);
集合类型: arr
  @EasyCache(fieldExps = "#coll0=?")
  public List<Long> getCacheByArr(Long[] idArr);
上面三种组合在一起:
  @EasyCache(fieldExps = "#coll0=?#p1=?#arr2=?")   
  public List<Long> getCacheByCom(List<Long> ids, String userName, Long[] idArr);
对象类型:
  @EasyCache(fieldExps = "#coll0.ids=?&p0.userName=?#arr1=?")
  public EasyCacheDemoBean getCacheByBean(EasyCacheDemoBean easyCacheDemoBean, Long[] idArr){
       // 这个方法内部需要使用easyCacheDemoBean的ids与userName。
  }         
```
- 是否过期规则:
```
是否过期，设置过期时间的好处是如果漏掉清楚缓存，会在过期时间外保证缓存正确结果。
开启后，默认过期时间为两分钟(easy.cache.expireTime调整默认过期时间)。
```
#### 命中缓存Aop实现
```
/**
 * 服务内部缓存注解Aop实现
 * @author CoderJiA
 */
@Aspect
@Component
public class EasyCacheAop {

    @Autowired
    private EasyCacheRedisStrategy easyCacheRedisStrategy;

    /**
     * 定义WsCache缓存注解切入点
     */
    @Pointcut("@annotation(cn.coderjia.fp.cache.anno.EasyCache)")
    public void easyCache() { }

    /**
     * 目标方法的环绕处理
     */
    @Around("easyCache()")
    public Object doAround(JoinPoint joinPoint) {
        /**
         * 缓存策略池中Redis命中缓存策略
         */
        return easyCacheRedisStrategy.operate(joinPoint);
    }

}
```
#### 命中缓存策略的定义与实现
针对连接点的缓存策略，可以有很多种实现，本质上就是定义一个抽象策略，然后自己根据抽象策略去做具体实现。也就是策略设计模式的一种变种应用。
- 抽象缓存策略角色的定义
```
/**
 * 抽象缓存策略角色的定义
 * @author CoderJiA
 */
public interface EasyCacheStrategy {

    /**
     * 定义统一缓存操作，在具体的缓存策略角色中实现。
     * @param joinPoint 切入点对象
     */
    Object operate(JoinPoint joinPoint);

}
```
- Redis缓存命中策略的具体实现
```
/**
 * 缓存Redis具体策略角色
 * @author CoderJiA
 */
@Slf4j
@Component
public class EasyCacheRedisStrategy implements EasyCacheStrategy {

    /**
     * 默认过期时间为两分钟
     */
    @Value("${ws.cache.expireTime:120}")
    private Long expireTime;

    @Resource(name = "systemRedis")
    private RedisUtils redisUtils;

    /**
     * 缓存命中Redis具体操作
     */
    @Override
    public Object operate(JoinPoint joinPoint) {
        Object obj = null;
        try {
            // 获取当前注解
            EasyCache easyCache = AopCacheUtil
                    .getMethod(joinPoint)
                    .getAnnotation(EasyCache.class);
            /**
            * easyCache.key() 如果为"" 取ClassSimpleName#MethodName
            * easyCache.fieldExps() 对表达式的一种解析
            * joinPoint 连接点
            */
            String key = UniqueKeyGenerator.gen(easyCache.key(), easyCache.fieldExps(), joinPoint);
            log.info("CacheRedisStrategy#operate key :{}", key);
            // 如果存在key直接返回
            if (redisUtils.exists(key)) {
                log.info("CacheRedisStrategy#operate use redis cache, cur key:{}", key);
                Class<?> methodReturnType = AopCacheUtil.getMethodReturnType(AopCacheUtil.getMethod(joinPoint));
                return JSON.parseObject(redisUtils.get(key), methodReturnType);
            }
            // 执行目标方法
            Method method = AopCacheUtil.getMethod(joinPoint);
            obj = method.invoke(joinPoint.getTarget(), joinPoint.getArgs());
            if (easyCache.isExpire()) {
                // 当前key缓存结果
                redisUtils.save(key, JSON.toJSONString(obj), expireTime);
            } else {
                // 当前key缓存结果
                redisUtils.save(key, JSON.toJSONString(obj));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return obj;
    }
}
```
#### 命中缓存注解的应用与测试
编写个Service Demo来应用@EasyCache
```
/**
 * 缓存使用方式Demo
 * @author CoderJiA
 **/
@Slf4j
@Service
public class EasyCacheDemoService {

    @EasyCache(fieldExps = "#p0=?", isExpire = true)
    public String getCacheByP(String userName) {
        return userName;
    }

    @EasyCache(fieldExps = "#coll0=?")
    public List<Long> getCacheByColl(List<Long> ids) {
        return ids;
    }

    @EasyCache(key = "cj_test_cache_key", fieldExps = "#arr0=?")
    public Long[] getCacheByArr(Long[] ids) {
        return ids;
    }

    @EasyCache(fieldExps = "#coll0=?#p1=?#arr2=?")
    public List<Long> getCacheByCom(List<Long> ids, String userName, Long[] idArr) {
        log.info("ids:{}; userName:{}; idArr:{}", idArr, userName, idArr);
        return ids;
    }

    @EasyCache(fieldExps = "#coll0.ids=?&p0.userName=?#arr1=?")
    public EasyCacheDemoBean getCacheByBean(EasyCacheDemoBean easyCacheDemoBean, Long[] idArr) {
        /**
         * 下面两个说明需要使用的对象属性 然后拼接方式如：
         * #coll0.ids=?&p0.userName=?
         * coll0 代表 ids是 集合属性
         * ids 代表 该easyCacheDemoBean对象的ids属性
         * & 代表他们同属于一个对象 所有p0的0代表着属于第一个参数easyCacheDemoBean
         */
        log.info("easyCacheDemoBean field ids:{}", easyCacheDemoBean.getIds());
        log.info("easyCacheDemoBean field userName:{}", easyCacheDemoBean.getUserName());

        /**
         * 这个同上的普通类型
         */
        log.info("idArr:{}", Arrays.toString(idArr));
        return easyCacheDemoBean;
    }
}
```
```
/**
 * 缓存Demo v0.0.1
 * @Author CoderJiA
 **/
@RestController
@RequestMapping("easy/cache")
public class EasyCacheDemoController extends SimpleController {
    @GetMapping("hitBean")
    public BaseResponse hitBean() {
        EasyCacheDemoBean easyCacheDemoBean = new EasyCacheDemoBean();
        easyCacheDemoBean.setIds(Lists.newArrayList(5L, 6L, 7L, 8L));
        easyCacheDemoBean.setUserName("coder.jia");
        return buildSuccess(easyCacheDemoService.getCacheByBean(easyCacheDemoBean, new Long[]{1L,2L,3L,4L}));
    }
}
```
测试
![](https://user-gold-cdn.xitu.io/2019/8/22/16cb91414c242190?w=1212&h=60&f=png&s=24311)
查看下Redis中是已经缓存
```
127.0.0.1:6379> get "EasyCacheDemoService#getCacheByBean_47378635d529d2511023cf038c4ae03dbd893abc"
"\"{\\\"ids\\\":[5,6,7,8],\\\"userName\\\":\\\"coder.jia\\\"}\""
```
>  写到这，命中缓存注解的基本策略已经满足，这里面有个地方是没有写出来，就是Redis的key生成策略，这部分其实就是ClassSimpleName#MethodName_Sha1("#coll0.ids=[5, 6, 7, 8]&p0.userName=coder.jia#arr1=[1,2,3,4]")，本质是fieldExps中的?占位符替换为参数值加上类名与方法名前缀，当然你定义了key那么前缀就是你定义的。
#### 定义清除缓存注解
> 上面命中缓存已经实现，但是部分业务方法的变更是会影响掉已缓存好的值，比如拥有某个权限的用户列表，如果我把一个权限中的用户变更，那么在读取缓存值是存在无效的值，所以需要定义清除缓存注解。
```
/**
 * 服务内部缓存注解，方法级别注解。
 * @author CoderJiA
 */
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface EasyCacheEvict {

    /**
     * 缓存唯一key前缀列表
     * 如果默认的前缀，请参见{@link cn.coderjia.fp.cache.anno.EasyCache} key的默认生成规则。
     */
    String[] keys();

}
```
#### 清除缓存Aop实现
```
/**
 * 服务内部缓存注解Aop实现
 * @author CoderJiA
 */
@Aspect
@Component
public class EasyCacheEvictAop {

    @Autowired
    private EasyCacheEvictRedisStrategy easyCacheEvictRedisStrategy;

    /**
     * 定义EasyCache缓存注解切入点
     */
    @Pointcut("@annotation(cn.coderjia.fp.cache.anno.EasyCacheEvict)")
    public void easyCacheEvict() { }

    /**
     * 目标方法的后置处理，清除缓存
     */
    @After("easyCacheEvict()")
    public void doAfter(JoinPoint joinPoint) {
        /**
         * 缓存策略池中Redis清除缓存策略
         */
        easyCacheEvictRedisStrategy.operate(joinPoint);
    }

}
```
#### 清除缓存策略的实现
```
/**
 * 缓存清空Redis具体策略角色
 * @author CoderJiA
 */
@Slf4j
@Component
public class EasyCacheEvictRedisStrategy implements EasyCacheStrategy {

    @Resource(name = "systemRedis")
    private RedisUtils redisUtils;

    /**
     * 缓存清空Redis具体操作
     * <p>注解{@link cn.coderjia.fp.cache.anno.EasyCache}的作用是某个方法的执行数据变更，导致对应的缓存失效。
     *     <ul>
     *     例如：
     *         <li> public List<User> query(String name);   // 查询方法</li>
     *         <li> public void update(String name);        // 更新方法，导致查询方法的缓存失效，那么就将redis中包含查询方法定义的key前缀内容全部清空。</li>
     *     </ul>
     * </p>
     * @param joinPoint 切入点对象
     */
    @Override
    public Object operate(JoinPoint joinPoint) {
        /*
         * 获取当前注解参数
         */
        EasyCacheEvict easyCacheEvict = AopCacheUtil
                .getMethod(joinPoint)
                .getAnnotation(EasyCacheEvict.class);
        /*
         * 获取影响到所有key前缀数组，并执行删除操作。
         */
        String[] keys = easyCacheEvict.keys();
        Optional.of(keys).ifPresent(ks -> {
            log.info("WsCacheEvictRedisStrategy#operate keys:{}", Arrays.toString(ks));
            IntStream.range(0, ks.length).forEach(i -> redisUtils.removeByPrefix(ks[i]));
        });
        return null;
    }
```
#### 在Demo中添加清除缓存的测试样例
Service
```
    @EasyCacheEvict(keys = {
            "EasyCacheDemoService#getCacheByCom"
    })
    public void cancelCacheCom() {
        log.info("EasyCacheDemoService#cancelCacheCom...");
    }
```
Controller
```
    @GetMapping("cancel")
    public BaseResponse cancel() {
        easyCacheDemoService.cancelCacheCom();
        return buildSuccess();
    }
```
测试: 先生成缓存
![](https://user-gold-cdn.xitu.io/2019/8/22/16cb91414c242190?w=1212&h=60&f=png&s=24311)
查看下Redis中是已经缓存
```
127.0.0.1:6379> get "EasyCacheDemoService#getCacheByBean_47378635d529d2511023cf038c4ae03dbd893abc"
"\"{\\\"ids\\\":[5,6,7,8],\\\"userName\\\":\\\"coder.jia\\\"}\""
```
调用cancel清除缓存
![](https://user-gold-cdn.xitu.io/2019/8/22/16cb927d7943622e?w=1076&h=60&f=png&s=19705)
```
127.0.0.1:6379> get "WsCacheDemoService#getCacheByBean_47378635d529d2511023cf038c4ae03dbd893abc"
(nil)
```
#### 总结
- 关于方法级别缓存其实Spring已给出默认实现，之所以造这个轮子，是因为想实现更灵活，更可定制化。其实这种缓存的使用场景是受限的，主要三点:
  - 缓存的命中率需要注意，系统并频繁调用的，参数变化度特别高的方法不适合使用，因为缓存命中率太低，失去了缓存的意义。
  - 由于Aop使用Jdk动态代理时本质上的嵌套调用内层的代理失效问题，在@Transactional注解中同样有问题。
  - 代理方法是public修饰。