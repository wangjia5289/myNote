---
title: 笔记：Spring Data Redis
date: 2025-05-17
categories:
  - 数据股那里
  - 非关系型数据库
  - 键值型
  - Redis
  - Spring Data Redis
tags: 
author: 霸天
layout: post
---




# 实操

### 1. 基本使用

#### 1.1. 创建 Spring Web 项目，添加 Redis 相关依赖

1. ==Web==
	1. Spring Web
2. ==NoSQL==
	1. Spring Data Redis

---


#### 1.2. 进行 Redis 相关配置

##### 1.2.1. 通用配置

```
spring:  
  data:  
    redis:  
      host: 192.168.136.8  
      port: 6379  
      password:  
      database: 0
"""
1. database：
    1. Redis 的数据库编号，默认是 0。
    2. Redis 默认有 16 个逻辑库，设计初衷是用于区分不同业务，比如库 1 存 Session，库 2 存验证码，避免数据混淆。
    3. 但在实际生产中，大多数项目只用 DB 0，然后通过 Key 前缀来区分业务，例如：
        1. user:session:123
        2. order:lock:456
        3. sms:code:789
    4. 原因主要包括：
        1. 多个库并不是真正隔离，仅是命令作用的划分；
        2. Redis Cluster 模式只支持 DB 0；
        3. 很多客户端或中间件默认只能使用 DB 0；
        4. 多库之间资源共享，比如内存和性能指标不会隔离。
2. password：
	1. 连接密码，默认为空
"""
```

---


##### 1.2.2. 配置类配置

在这个配置类中，我们完成了以下配置：
1. ==启用了缓存功能==：
	1. 支持注解式缓存（如 `@Cacheable`）
2. ==RedisTemplate 的序列化配置==：
	1. 用于使用 ResitTemplate 编程式操作 Redis 时的行为
	2. 所有 Redis key 都以字符串形式存储
	3. 所有 value 通过 Jackson 转成 JSON 格式存储，保证复杂对象安全序列化；
	4. 虽然 Spring Data Redis 有默认行为，但在处理复杂类型时，默认行为可能导致反序列化错误；
3. ==注解式缓存行为定制==：
	1. 缓存值同样采用 Jackson JSON 序列化，保持统一；
	2. 默认缓存过期时间设置为 1 天（TTL）；
	3. Redis key 自动加上前缀，如 `mall:user::123`，便于分类管理。
```
// 用缓存功能
@EnableCaching
@Configuration
public class RedisConfig {

    // Redis 数据库自定义 key 前缀，结果可能为 mall:user::123
    public static final String REDIS_KEY_DATABASE = "mall";

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisSerializer<Object> serializer = redisSerializer();
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(serializer);
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(serializer);
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }

    @Bean
    public RedisSerializer<Object> redisSerializer() {
        // 创建 ObjectMapper
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        // 必须设置，否则无法将JSON转化为对象，会转化成Map类型
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL);
        
        // 使用带 ObjectMapper 的构造函数
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);
        return serializer;
    }

    @Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheWriter redisCacheWriter = RedisCacheWriter.nonLockingRedisCacheWriter(redisConnectionFactory);
        // 设置Redis缓存有效期为1天，并使用 REDIS_KEY_DATABASE 作为前缀
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(redisSerializer()))
                .entryTtl(Duration.ofDays(1))
                .prefixCacheNameWith(REDIS_KEY_DATABASE + ":");
        return new RedisCacheManager(redisCacheWriter, redisCacheConfiguration);
    }
}
```

---


##### 1.2.3. 连接池配置

在 Spring Boot 1.5.x 版本中，默认使用 Jedis 客户端。Jedis 采用直连方式，默认一个 Jedis 实例对应一条 TCP 连接（成员变量），当多个线程同时操作同一个 Jedis 实例（即共享 Jedis Bean）会导致线程安全问题，因此通常通过连接池（如 `JedisPool`）来为每个线程分配独立的连接实例，从而实现线程安全。

从 Spring Boot 2.x 开始，默认客户端切换为 Lettuce。Lettuce 是一个基于 Netty 的高性能 Redis 客户端，具备良好的可伸缩性与线程安全特性。它允许多个线程共享同一个连接实例，内部通过事件循环机制（Event Loop）串行化请求，既避免线程冲突又能让少量线程处理大量请求。Lettuce 同时支持同步、异步以及响应式编程模型，适用于构建高并发、非阻塞的应用场景。

> [!NOTE] 注意事项
> 1. Lettuce 虽然使用 Netty 实现底层通信，但它自己维护一套 NIO 的线程池，其线程池（EventLoopGroup）完全独立于 Tomcat 的线程池。因此，即使使用 Lettuce，也完全可以继续使用 Tomcat 作为 Web 服务容器，不会因此“被迫迁移到 Netty”。

```
spring:  
  data:  
    redis:  
      lettuce:  
        pool:  
          max-active: 8  
          max-idle: 8  
          min-idle: 0  
          max-wait: -1ms
"""
1. max-active：
	1. 连接池最大连接数
2. max-idle：
	1. 连接池最大空闲连接数
3. min-idle：
	1. 连接池最小空闲连接数
4. max-wait：
	1. 连接池最大阻塞等待时间，负值表示没有限制
"""
```

---


#### 1.3. 标注相关注解

##### 1.3.1. @EnableCaching

在启动类或配置类上添加` @EnableCaching` 注解，用于启动缓存功能
```
@EnableCaching
@SpringBootApplication
public class MallTinyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MallTinyApplication.class, args);
    }
}
```

---


##### 1.3.2. @Cacheable

`@Cacheable` 一般用在查询方法上。如果缓存里有数据，就直接取缓存，不会执行方法；如果缓存没有，就执行方法，并把返回结果放进缓存（有 `condition`、`unless` 不算）
```
# 1. 语法结构
@Cacheable(
    value     = "user",
    key       = "#id",
    condition = "#id > 100",
    unless    = "#result == null"
)
"""
1. value:
	1. 缓存的命名空间（必填）
	2. 和 key 一起组成 Redis 中的键，如：user::123，可以简单理解为是前缀
	3. 如果我们在配置类中配置了前缀，最后可能的结果为：mall:user::123
2. key：
	1. 设置在命名空间中的缓存 key 值，可以使用 SpEL 表达式定义；
3. condition：
	1. 条件符合则缓存。
4. unless：
	1. 条件符合则不缓存；


# 2. 示例一
@Cacheable(
    value     = "user",
    key       = "#id",
    condition = "#id > 100",
    unless    = "#result == null"
)
public User getUserById(Long id) {
    System.out.println("方法执行，id = " + id);
    return userRepository.findById(id).orElse(null);
}
```

注意：虽然正常情况下是按照上述流程执行的，但一旦使用了 `condition` 或 `unless`，这个顺序会被打乱。
==1.只有 condition 的情况==
```
flowchart LR
  A[开始调用] --> B{是否符合 condition}
  B -- 否 --> C[执行方法（跳过读写缓存）]
  C --> D[返回结果]
  B -- 是 --> E{是否缓存命中}
  E -- 是 --> F[返回缓存结果]
  E -- 否 --> G[执行方法]
  G --> H[写入缓存]
  H --> I[返回结果]
```
![](image-20250517211323498.png)


==2.只有 unless 的情况==
```
flowchart LR
  A[开始调用] --> B{是否缓存命中}
  B -- 是 --> C[返回缓存结果]
  B -- 否 --> D[执行方法]
  D --> E{是否符合 unless}
  E -- 是 --> F[返回结果（不写缓存）]
  E -- 否 --> G[写入缓存]
  G --> H[返回结果]
```
![](image-20250517211409902.png)


==3.conditon 和 unless 都有的情况==
```
flowchart LR
  A[开始调用] --> B{是否符合 condition}
  B -- 否 --> C[执行方法（跳过读写缓存）]
  C --> D[返回结果]
  B -- 是 --> E{是否缓存命中}
  E -- 是 --> F[返回缓存结果]
  E -- 否 --> G[执行方法]
  G --> H{是否符合 unless}
  H -- 是 --> I[返回结果]
  H -- 否 --> J[写入缓存]
  J --> I
```
![](image-20250517210734445.png)

---


##### 1.3.3. @CachePut

`@CachePut` 一般用在新增方法上，每次执行时都会把返回结果存入缓存（有 `condition`、`unless` 不算），使用方法和 `@Cacheable` 一样

---


##### 1.3.4. @CacheEvict

`@CacheEvict` 一般用在更新或删除方法上，每次执行时会清空缓存（有 `condition` 不算），使用方法和 `@Cacheable` 类似，不过，该注解没有 `unless` 属性，只有 `value`、`key` 和 `condition` 三个属性。并且 `condition` 的判断方式相对简单，只要条件满足就执行清空缓存操作，否则不清空。

---


### 2. 业务处理

#### 2.1. 自由操作 Redis

##### 2.1.1. 自由操作 Redis 概述

在缓存场景下，我们更常使用注解，因为它几乎是“无脑式”的操作方式，无需关心底层细节即可轻松完成缓存管理。但当遇到复杂场景时，比如不仅是缓存，还涉及消息队列、排行榜、实时统计等功能，注解就显得力不从心了，这时候就必须大量使用 `RedisTemplate`，因为注解无法满足这些复杂操作需求。因为其存在以下问题：
1. 无法为不同缓存项设置各自独立的过期时间，只能使用我们在配置类中配置的TTL（一天）
2. 只能缓存方法的返回值，无法缓存中间计算结果。
3. 虽然可以将缓存值以 JSON 形式存储，但这实际上是以字符串形式存在 Redis 中，而 Redis 本身支持多种丰富的数据结构，如 List、Set、ZSet、Hash，我们当然需要利用这些优势。
4. 无法进行细粒度操作，无法直接修改 JSON 内部某个字段，只能整体读取、修改后再整体写回，效率低且容易出错，同时频繁序列化与反序列化带来较大开销，尤其是数据量大时更明显。

因此，面对这些复杂需求时，我们需要使用 `RedisTemplate`，手动进行底层操作，来实现更灵活、高效的缓存管理。

---


##### 2.1.2. 封装 RedisService

###### 2.1.2.1. RedisService 接口

`RedisTemplate` 是功能强大的底层工具类，但用起来其实挺“麻烦”的，我们可以封装成 RedisService，该接口中定义了大多数常用操作方法，业务暂时用不到 ZSet，就没有加
```
public interface RedisService {

    // 保存属性
    void set(String key, Object value, long time);

    // 保存属性
    void set(String key, Object value);

    // 获取属性
    Object get(String key);

    // 删除属性
    Boolean del(String key);

    // 批量删除属性
    Long del(List<String> keys);

    // 设置过期时间
    Boolean expire(String key, long time);

    // 获取过期时间
    Long getExpire(String key);

    // 判断是否有该属性
    Boolean hasKey(String key);

    // 按 delta 递增
    Long incr(String key, long delta);

    // 按 delta 递减
    Long decr(String key, long delta);

    // 获取 Hash 结构中的属性
    Object hGet(String key, String hashKey);

    // 向 Hash 结构中放入一个属性
    Boolean hSet(String key, String hashKey, Object value, long time);

    // 向 Hash 结构中放入一个属性
    void hSet(String key, String hashKey, Object value);

    // 直接获取整个 Hash 结构
    Map<Object, Object> hGetAll(String key);

    // 直接设置整个 Hash 结构
    Boolean hSetAll(String key, Map<String, Object> map, long time);

    // 直接设置整个 Hash 结构
    void hSetAll(String key, Map<String, Object> map);

    // 删除 Hash 结构中的属性
    void hDel(String key, Object... hashKey);

    // 判断 Hash 结构中是否有该属性
    Boolean hHasKey(String key, String hashKey);

    // Hash 结构中属性递增
    Long hIncr(String key, String hashKey, Long delta);

    // Hash 结构中属性递减
    Long hDecr(String key, String hashKey, Long delta);

    // 获取 Set 结构
    Set<Object> sMembers(String key);

    // 向 Set 结构中添加属性
    Long sAdd(String key, Object... values);

    // 向 Set 结构中添加属性
    Long sAdd(String key, long time, Object... values);

    // 是否为 Set 中的属性
    Boolean sIsMember(String key, Object value);

    // 获取 Set 结构的长度
    Long sSize(String key);

    // 删除 Set 结构中的属性
    Long sRemove(String key, Object... values);

    // 获取 List 结构中的属性
    List<Object> lRange(String key, long start, long end);

    // 获取 List 结构的长度
    Long lSize(String key);

    // 根据索引获取 List 中的属性
    Object lIndex(String key, long index);

    // 向 List 结构中添加属性
    Long lPush(String key, Object value);

    // 向 List 结构中添加属性
    Long lPush(String key, Object value, long time);

    // 向 List 结构中批量添加属性
    Long lPushAll(String key, Object... values);

    // 向 List 结构中批量添加属性
    Long lPushAll(String key, Long time, Object... values);

    // 从 List 结构中移除属性
    Long lRemove(String key, long count, Object value);
}
```

---


###### 2.1.2.2. RedisServiceImpl 实现类

```
@Service
public class RedisServiceImpl implements RedisService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Override
    public void set(String key, Object value, long time) {
        redisTemplate.opsForValue().set(key, value, time, TimeUnit.SECONDS);
    }

    @Override
    public void set(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }

    @Override
    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    @Override
    public Boolean del(String key) {
        return redisTemplate.delete(key);
    }

    @Override
    public Long del(List<String> keys) {
        return redisTemplate.delete(keys);
    }

    @Override
    public Boolean expire(String key, long time) {
        return redisTemplate.expire(key, time, TimeUnit.SECONDS);
    }

    @Override
    public Long getExpire(String key) {
        return redisTemplate.getExpire(key, TimeUnit.SECONDS);
    }

    @Override
    public Boolean hasKey(String key) {
        return redisTemplate.hasKey(key);
    }

    @Override
    public Long incr(String key, long delta) {
        return redisTemplate.opsForValue().increment(key, delta);
    }

    @Override
    public Long decr(String key, long delta) {
        return redisTemplate.opsForValue().increment(key, -delta);
    }

    @Override
    public Object hGet(String key, String hashKey) {
        return redisTemplate.opsForHash().get(key, hashKey);
    }

    @Override
    public Boolean hSet(String key, String hashKey, Object value, long time) {
        redisTemplate.opsForHash().put(key, hashKey, value);
        return expire(key, time);
    }

    @Override
    public void hSet(String key, String hashKey, Object value) {
        redisTemplate.opsForHash().put(key, hashKey, value);
    }

    @Override
    public Map<Object, Object> hGetAll(String key) {
        return redisTemplate.opsForHash().entries(key);
    }

    @Override
    public Boolean hSetAll(String key, Map<String, Object> map, long time) {
        redisTemplate.opsForHash().putAll(key, map);
        return expire(key, time);
    }

    @Override
    public void hSetAll(String key, Map<String, Object> map) {
        redisTemplate.opsForHash().putAll(key, map);
    }

    @Override
    public void hDel(String key, Object... hashKey) {
        redisTemplate.opsForHash().delete(key, hashKey);
    }

    @Override
    public Boolean hHasKey(String key, String hashKey) {
        return redisTemplate.opsForHash().hasKey(key, hashKey);
    }

    @Override
    public Long hIncr(String key, String hashKey, Long delta) {
        return redisTemplate.opsForHash().increment(key, hashKey, delta);
    }

    @Override
    public Long hDecr(String key, String hashKey, Long delta) {
        return redisTemplate.opsForHash().increment(key, hashKey, -delta);
    }

    @Override
    public Set<Object> sMembers(String key) {
        return redisTemplate.opsForSet().members(key);
    }

    @Override
    public Long sAdd(String key, Object... values) {
        return redisTemplate.opsForSet().add(key, values);
    }

    @Override
    public Long sAdd(String key, long time, Object... values) {
        Long count = redisTemplate.opsForSet().add(key, values);
        expire(key, time);
        return count;
    }

    @Override
    public Boolean sIsMember(String key, Object value) {
        return redisTemplate.opsForSet().isMember(key, value);
    }

    @Override
    public Long sSize(String key) {
        return redisTemplate.opsForSet().size(key);
    }

    @Override
    public Long sRemove(String key, Object... values) {
        return redisTemplate.opsForSet().remove(key, values);
    }

    @Override
    public List<Object> lRange(String key, long start, long end) {
        return redisTemplate.opsForList().range(key, start, end);
    }

    @Override
    public Long lSize(String key) {
        return redisTemplate.opsForList().size(key);
    }

    @Override
    public Object lIndex(String key, long index) {
        return redisTemplate.opsForList().index(key, index);
    }

    @Override
    public Long lPush(String key, Object value) {
        return redisTemplate.opsForList().rightPush(key, value);
    }

    @Override
    public Long lPush(String key, Object value, long time) {
        Long index = redisTemplate.opsForList().rightPush(key, value);
        expire(key, time);
        return index;
    }

    @Override
    public Long lPushAll(String key, Object... values) {
        return redisTemplate.opsForList().rightPushAll(key, values);
    }

    @Override
    public Long lPushAll(String key, Long time, Object... values) {
        Long count = redisTemplate.opsForList().rightPushAll(key, values);
        expire(key, time);
        return count;
    }

    @Override
    public Long lRemove(String key, long count, Object value) {
        return redisTemplate.opsForList().remove(key, count, value);
    }
}
```

---


##### 2.1.3. Controller 执行操作

```
/**
 * @auther macrozheng
 * @description Redis测试Controller
 * @date 2020/3/3
 * @github https://github.com/macrozheng
 */
@Api(tags = "RedisController", description = "Redis测试")
@Controller
@RequestMapping("/redis")
public class RedisController {
    @Autowired
    private RedisService redisService;
    @Autowired
    private PmsBrandService brandService;

    @ApiOperation("测试简单缓存")
    @RequestMapping(value = "/simpleTest", method = RequestMethod.GET)
    @ResponseBody
    public CommonResult<PmsBrand> simpleTest() {
        List<PmsBrand> brandList = brandService.list(1, 5);
        PmsBrand brand = brandList.get(0);
        String key = "redis:simple:" + brand.getId();
        redisService.set(key, brand);
        PmsBrand cacheBrand = (PmsBrand) redisService.get(key);
        return CommonResult.success(cacheBrand);
    }

    @ApiOperation("测试 Hash 结构的缓存")
    @RequestMapping(value = "/hashTest", method = RequestMethod.GET)
    @ResponseBody
    public CommonResult<PmsBrand> hashTest() {
        List<PmsBrand> brandList = brandService.list(1, 5);
        PmsBrand brand = brandList.get(0);
        String key = "redis:hash:" + brand.getId();
        Map<String, Object> value = BeanUtil.beanToMap(brand);
        redisService.hSetAll(key, value);
        Map<Object, Object> cacheValue = redisService.hGetAll(key);
        PmsBrand cacheBrand = BeanUtil.toBean(cacheValue, PmsBrand.class);
        return CommonResult.success(cacheBrand);
    }

    @ApiOperation("测试 Set 结构的缓存")
    @RequestMapping(value = "/setTest", method = RequestMethod.GET)
    @ResponseBody
    public CommonResult<Set<Object>> setTest() {
        List<PmsBrand> brandList = brandService.list(1, 5);
        String key = "redis:set:all";
        redisService.sAdd(key, (Object[]) ArrayUtil.toArray(brandList, PmsBrand.class));
        redisService.sRemove(key, brandList.get(0));
        Set<Object> cachedBrandList = redisService.sMembers(key);
        return CommonResult.success(cachedBrandList);
    }

    @ApiOperation("测试 List 结构的缓存")
    @RequestMapping(value = "/listTest", method = RequestMethod.GET)
    @ResponseBody
    public CommonResult<List<Object>> listTest() {
        List<PmsBrand> brandList = brandService.list(1, 5);
        String key = "redis:list:all";
        redisService.lPushAll(key, (Object[]) ArrayUtil.toArray(brandList, PmsBrand.class));
        redisService.lRemove(key, 1, brandList.get(0));
        List<Object> cachedBrandList = redisService.lRange(key, 0, 3);
        return CommonResult.success(cachedBrandList);
    }
}
```

---


#### 2.2. 实现验证码逻辑

##### 2.2.1. 实现验证码逻辑概述

以短信验证码的存储与校验为例，具体流程如下：系统生成验证码后，将其与自定义的 Redis key 前缀及用户手机号拼接生成完整的 Redis key，并将验证码作为 value 存入 Redis，同时设置指定的过期时间（例如 120 秒）。在验证码校验环节，根据手机号构造出 Redis key，获取存储的验证码值，与用户提供的验证码进行比对判断。

---


##### 2.2.2. UmsMemberService 接口

```
public interface UmsMemberService {

    // 生成验证码
    CommonResult generateAuthCode(String telephone);

	// 判断验证码和手机号码是否匹配
    CommonResult verifyAuthCode(String telephone, String authCode);
}
```

---


##### 2.2.3. UmsMemberServiceImpl 实现类

```
@Service
public class UmsMemberServiceImpl implements UmsMemberService {

	// 注入 RedServiceImpl
    @Autowired
    private RedisService redisService;
    
    // 获取 Redis 验证码的 key 前缀
    @Value("${redis.key.prefix.authCode}")
    private String REDIS_KEY_PREFIX_AUTH_CODE;
    
	// 获取验证码的有效期
    @Value("${redis.key.expire.authCode}")
    private Long AUTH_CODE_EXPIRE_SECONDS;

    @Override
    public CommonResult generateAuthCode(String telephone) {
        StringBuilder sb = new StringBuilder();
        Random random = new Random();
        for (int i = 0; i < 6; i++) {
            sb.append(random.nextInt(10));
        }
        //验证码绑定手机号并存储到redis
		redisService.set(
			REDIS_KEY_PREFIX_AUTH_CODE + telephone, 
			sb.toString(), 
			AUTH_CODE_EXPIRE_SECONDS
		);

        return CommonResult.success(sb.toString(), "获取验证码成功");
    }


    //对输入的验证码进行校验
    @Override
    public CommonResult verifyAuthCode(String telephone, String authCode) {
        if (StrUtil.isEmpty(authCode)) {
            return CommonResult.failed("请输入验证码");
        }
        String realAuthCode = (String) redisService.get(REDIS_KEY_PREFIX_AUTH_CODE + telephone);
        boolean result = authCode.equals(realAuthCode);
        if (result) {
            return CommonResult.success(null, "验证码校验成功");
        } else {
            return CommonResult.failed("验证码不正确");
        }
    }
}
```

---


##### 2.2.4. Controller 执行操作

```
@Controller
@Api(tags = "UmsMemberController")
@Tag(name = "UmsMemberController", description = "会员登录注册管理")
@RequestMapping("/sso")
public class UmsMemberController {
    @Autowired
    private UmsMemberService memberService;

    @ApiOperation("获取验证码")
    @RequestMapping(value = "/getAuthCode", method = RequestMethod.GET)
    @ResponseBody
    public CommonResult getAuthCode(@RequestParam String telephone) {
        return memberService.generateAuthCode(telephone);
    }

    @ApiOperation("判断验证码是否正确")
    @RequestMapping(value = "/verifyAuthCode", method = RequestMethod.POST)
    @ResponseBody
    public CommonResult updatePassword(@RequestParam String telephone,
                                       @RequestParam String authCode) {
        return memberService.verifyAuthCode(telephone,authCode);
    }
}
```

---








