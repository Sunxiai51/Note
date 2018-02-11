# Redis缓存中间件的集成、配置与使用

本文档描述了**Redis缓存中间件**在项目中的集成与配置过程，并以示例说明其在实际项目中的应用场景。

## 1. 相关项目及依赖

Reids缓存中间件直接依赖的jar包括：

- org.springframework.data: **spring-data-redis** (1.8.9.RELEASE)
- org.springframework.integration: **spring-integration-redis** (4.3.13.RELEASE)
- redis.clients: **jedis** (2.9.0)
- com.fasterxml.jackson.core: **jackson-databind** (2.9.3)

## 2. Spring集成redis-replication

Redis中间件可以采用replication形式部署并提供服务。Replication为redis主从复制(master-slave)模式，对于客户端来说，仅仅需要配置到master的连接，部署好的slave会自动完成备份。

**Replication模式仅提供不完全安全的数据备份**，redis服务器的配置与部署可参考以下文档：

> redis-replication相关文档： [Replication - Redis](https://redis.io/topics/replication)

本文档使用spring-data-redis来集成redis，并使用spring-integration-redis来提供分布式环境下的锁服务，后文将列出主要配置，插件详细特性及更多配置可参考以下文档：

> spring-data-redis文档：[Spring Data Redis](https://docs.spring.io/spring-data/redis/docs/1.8.9.RELEASE/reference/html/)
>
> spring注解配置缓存文档：[Cache Abstraction](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#cache)
>
> spring-integration-redis文档：[Redis Support](https://docs.spring.io/spring-integration/reference/html/redis.html)

Spring集成redis需要配置相应的bean，包括：

- redis.clients.jedis.**JedisPoolConfig**
- org.springframework.data.redis.connection.jedis.**JedisConnectionFactory**
- org.springframework.data.redis.core.**RedisTemplate**
- org.springframework.data.redis.cache.**RedisCacheManager**
- org.springframework.integration.redis.util.**RedisLockRegistry**

如果需要使用 `@Cacheable` 等注解来自动缓存内容，还需要配置 `<cache:annotation-driven />` 以开启缓存注解驱动，该配置项缺省使用了一个bean名为 `cacheManager` 的缓存管理器。

示例 `Spring-redis-replication.xml` 如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:cache="http://www.springframework.org/schema/cache"
	xsi:schemaLocation="
		http://www.springframework.org/schema/beans 
		http://www.springframework.org/schema/beans/spring-beans.xsd 
		http://www.springframework.org/schema/cache 
		http://www.springframework.org/schema/cache/spring-cache.xsd">

	<description>Spring集成redis主从配置</description>

	<!--配置 jedis pool -->
	<bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
		<property name="maxTotal" value="${redis.replication.jedis.pool.maxTotal}" />
		<property name="maxIdle" value="${redis.replication.jedis.pool.maxIdle}" />
		<property name="minIdle" value="${redis.replication.jedis.pool.minIdle}" />
		<property name="numTestsPerEvictionRun"
			value="${redis.replication.jedis.pool.numTestsPerEvictionRun}" />
		<property name="timeBetweenEvictionRunsMillis"
			value="${redis.replication.jedis.pool.timeBetweenEvictionRunsMillis}" />
		<property name="minEvictableIdleTimeMillis"
			value="${redis.replication.jedis.pool.minEvictableIdleTimeMillis}" />
		<property name="softMinEvictableIdleTimeMillis"
			value="${redis.replication.jedis.pool.softMinEvictableIdleTimeMillis}" />
		<property name="maxWaitMillis"
			value="${redis.replication.jedis.pool.maxWaitMillis}" />
		<property name="testOnBorrow" value="${redis.replication.jedis.pool.testOnBorrow}" />
		<property name="testWhileIdle"
			value="${redis.replication.jedis.pool.testWhileIdle}" />
		<property name="blockWhenExhausted"
			value="${redis.replication.jedis.pool.blockWhenExhausted}" />
	</bean>

	<!-- 配置JedisConnectionFactory -->
	<bean id="jedisConnectionFactory"
		class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
		<property name="hostName" value="${redis.replication.host}" />
		<property name="port" value="${redis.replication.port}" />
		<property name="password" value="${redis.replication.password}" />
		<property name="poolConfig" ref="jedisPoolConfig" />
	</bean>

	<!-- redis template definition -->
	<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
		<property name="connectionFactory" ref="jedisConnectionFactory" />
		<!-- 支持事务 -->
		<property name="enableTransactionSupport" value="true" />
		<property name="keySerializer">
			<bean class="org.springframework.data.redis.serializer.StringRedisSerializer" />
		</property>
		<property name="valueSerializer">
			<bean class="org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer" />
		</property>
	</bean>

	<!-- declare Redis Cache Manager -->
	<bean id="redisCacheManager" class="org.springframework.data.redis.cache.RedisCacheManager">
		<constructor-arg name="redisOperations" ref="redisTemplate" />
		<!-- 默认有效期,单位:秒 -->
		<property name="defaultExpiration" value="3600" />
		<!-- 多个缓存有效期,单位:秒 -->
		<property name="expires">
			<map>
				<entry key="redis-users" value="10" />
			</map>
		</property>
	</bean>

	<!-- 配置RedisLockRegistry -->
	<bean id="redisLockRegistry"
		class="org.springframework.integration.redis.util.RedisLockRegistry">
		<constructor-arg type="org.springframework.data.redis.connection.RedisConnectionFactory"  ref="jedisConnectionFactory" />
		<constructor-arg type="java.lang.String" value="${redis.replication.registry.key}" />
	</bean>

</beans>
```

其对应的 `redis-replication.properties` 如下：

```properties
# redis主从复制配置
# 为防止混淆，该配置文件所有配置项前缀均应为"redis.replication."

### ---JedisPoolConfig参数配置---
# 最大连接数(默认8)
redis.replication.jedis.pool.maxTotal=30
# 最大空闲时间(默认8)
redis.replication.jedis.pool.maxIdle=10
# 最小空闲时间(默认8)
redis.replication.jedis.pool.minIdle=10
# 每次最大连接数(默认-1)
redis.replication.jedis.pool.numTestsPerEvictionRun=1024
# 释放扫描的扫描间隔(默认30000)
redis.replication.jedis.pool.timeBetweenEvictionRunsMillis=30000
# 连接的最小空闲时间(默认60000)
redis.replication.jedis.pool.minEvictableIdleTimeMillis=60000
# 连接空闲时间多久后释放，当空闲时间>该值且空闲连接>最大空闲连接数时直接释放(默认-1)
redis.replication.jedis.pool.softMinEvictableIdleTimeMillis=10000
# 获得链接时的最大等待毫秒数，小于0：阻塞(默认-1)
redis.replication.jedis.pool.maxWaitMillis=1500
# 在获得链接的时候检查有效性(默认false)
redis.replication.jedis.pool.testOnBorrow=true
# 在空闲时检查有效性(默认true)
redis.replication.jedis.pool.testWhileIdle=true
# 连接耗尽时是否阻塞，false报异常，true阻塞(默认true)
redis.replication.jedis.pool.blockWhenExhausted=false

### ---主机信息---
# 主机IP
redis.replication.host=192.168.51.51
# 指定Redis监听端口，默认端口为6379
redis.replication.port=6379
# 授权密码
redis.replication.password=123456

### ---分布式锁配置---
# 头寸
redis.replication.registry.key=registryKey
```

## 3. Spring集成redis-sentinel

在replication模式的基础上，为了保证redis的高可用性，可以使用redis的sentinel模式来完成redis的故障自动迁移。Sentinel为redis岗哨模式，它能通过设置岗哨（这些岗哨是通过特殊方式启动的redis进程）来监测指定的主从复制中的master，当该master宕机后，岗哨会通过投票机制选择它的其中一个slave作为新的master来提供服务，其余slave转换成新master的slave。Sentinel模式的更多详细特性与配置可参考以下文档：

> redis-sentinel相关文档：[Redis Sentinel Documentation](https://redis.io/topics/sentinel)

Sentinel模式是基于replication模式的，即只有存在主从复制时进行故障自动迁移才有意义。

Spring集成redis-sentinel与集成redis-replication过程基本一致，唯一的区别是它的JedisConnectionFactory的bean配置方式：

```xml
<!-- 配置sentinel下的JedisConnectionFactory -->
<bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
  <property name="password" value="${redis.sentinel.password}" />
  <constructor-arg ref="jedisPoolConfig" />
  <constructor-arg ref="redisSentinelConfiguration" />
</bean>
```

与redis-replication不同的是，sentinel模式去掉了IP、端口的配置，引入一个名为redisSentinelConfiguration的bean，该bean类型为 `org.springframework.data.redis.connection.RedisSentinelConfiguration` ，配置了redis岗哨的相关信息。示例 `Spring-redis-sentinel.xml` 配置文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:cache="http://www.springframework.org/schema/cache"
	xsi:schemaLocation="
		http://www.springframework.org/schema/beans 
		http://www.springframework.org/schema/beans/spring-beans.xsd 
		http://www.springframework.org/schema/cache 
		http://www.springframework.org/schema/cache/spring-cache.xsd">

	<description>Spring集成redis岗哨配置</description>

	<!--配置 jedis pool -->
	<bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
		<property name="maxTotal" value="${redis.sentinel.jedis.pool.maxTotal}" />
		<property name="maxIdle" value="${redis.sentinel.jedis.pool.maxIdle}" />
		<property name="minIdle" value="${redis.sentinel.jedis.pool.minIdle}" />
		<property name="numTestsPerEvictionRun"
			value="${redis.sentinel.jedis.pool.numTestsPerEvictionRun}" />
		<property name="timeBetweenEvictionRunsMillis"
			value="${redis.sentinel.jedis.pool.timeBetweenEvictionRunsMillis}" />
		<property name="minEvictableIdleTimeMillis"
			value="${redis.sentinel.jedis.pool.minEvictableIdleTimeMillis}" />
		<property name="softMinEvictableIdleTimeMillis"
			value="${redis.sentinel.jedis.pool.softMinEvictableIdleTimeMillis}" />
		<property name="maxWaitMillis" value="${redis.sentinel.jedis.pool.maxWaitMillis}" />
		<property name="testOnBorrow" value="${redis.sentinel.jedis.pool.testOnBorrow}" />
		<property name="testWhileIdle" value="${redis.sentinel.jedis.pool.testWhileIdle}" />
		<property name="blockWhenExhausted"
			value="${redis.sentinel.jedis.pool.blockWhenExhausted}" />
	</bean>

	<!-- 配置redisSentinelConfiguration -->
	<bean id="redisSentinelConfiguration" class="org.springframework.data.redis.connection.RedisSentinelConfiguration">
		<property name="master">
			<bean class="org.springframework.data.redis.connection.RedisNode">
				<property name="name" value="mymaster" />
			</bean>
		</property>
		<property name="sentinels">
			<set>
				<bean class="org.springframework.data.redis.connection.RedisNode">
					<constructor-arg name="host" value="${redis.sentinel.node1.host}" />
					<constructor-arg name="port" value="${redis.sentinel.node1.port}" />
				</bean>
				<bean class="org.springframework.data.redis.connection.RedisNode">
					<constructor-arg name="host" value="${redis.sentinel.node2.host}" />
					<constructor-arg name="port" value="${redis.sentinel.node2.port}" />
				</bean>
				<bean class="org.springframework.data.redis.connection.RedisNode">
					<constructor-arg name="host" value="${redis.sentinel.node3.host}" />
					<constructor-arg name="port" value="${redis.sentinel.node3.port}" />
				</bean>
			</set>
		</property>
	</bean>

	<!-- 配置JedisConnectionFactory -->
	<bean id="jedisConnectionFactory"
		class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
		<property name="password" value="${redis.sentinel.password}" />
		<constructor-arg ref="jedisPoolConfig" />
		<constructor-arg ref="redisSentinelConfiguration" />
	</bean>

	<!-- 配置redisTemplate -->
	<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
		<property name="connectionFactory" ref="jedisConnectionFactory" />
		<!-- 支持事务 -->
		<property name="enableTransactionSupport" value="true" />
		<property name="keySerializer">
			<bean class="org.springframework.data.redis.serializer.StringRedisSerializer" />
		</property>
		<property name="valueSerializer">
			<bean class="org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer" />
		</property>
	</bean>

	<!-- 配置redisCacheManager -->
	<bean id="redisCacheManager" class="org.springframework.data.redis.cache.RedisCacheManager">
		<constructor-arg name="redisOperations" ref="redisTemplate" />
		<!-- 默认有效期,单位:秒 -->
		<property name="defaultExpiration" value="3600" />
		<!-- 多个缓存有效期,单位:秒 -->
		<property name="expires">
			<map>
				<entry key="redis-users" value="10" />
			</map>
		</property>
	</bean>

	<!-- 配置RedisLockRegistry -->
	<bean id="redisLockRegistry"
		class="org.springframework.integration.redis.util.RedisLockRegistry">
		<constructor-arg type="org.springframework.data.redis.connection.RedisConnectionFactory"
			ref="jedisConnectionFactory" />
		<constructor-arg type="java.lang.String" value="${redis.sentinel.registry.key}" />
	</bean>

</beans>
```

其对应的 `redis-sentinel.properties` 如下：

```properties
# redis岗哨模式配置
# 为防止混淆，该配置文件所有配置项前缀均应为"redis.sentinel."

### ---JedisPoolConfig参数配置---
# 最大连接数(默认8)
redis.sentinel.jedis.pool.maxTotal=30
# 最大空闲时间(默认8)
redis.sentinel.jedis.pool.maxIdle=10
# 最小空闲时间(默认8)
redis.sentinel.jedis.pool.minIdle=10
# 每次最大连接数(默认-1)
redis.sentinel.jedis.pool.numTestsPerEvictionRun=1024
# 释放扫描的扫描间隔(默认30000)
redis.sentinel.jedis.pool.timeBetweenEvictionRunsMillis=30000
# 连接的最小空闲时间(默认60000)
redis.sentinel.jedis.pool.minEvictableIdleTimeMillis=60000
# 连接空闲时间多久后释放，当空闲时间>该值且空闲连接>最大空闲连接数时直接释放(默认-1)
redis.sentinel.jedis.pool.softMinEvictableIdleTimeMillis=10000
# 获得链接时的最大等待毫秒数，小于0：阻塞(默认-1)
redis.sentinel.jedis.pool.maxWaitMillis=1500
# 在获得链接的时候检查有效性(默认false)
redis.sentinel.jedis.pool.testOnBorrow=true
# 在空闲时检查有效性(默认true)
redis.sentinel.jedis.pool.testWhileIdle=true
# 连接耗尽时是否阻塞，false报异常，true阻塞(默认true)
redis.sentinel.jedis.pool.blockWhenExhausted=false

### ---岗哨信息---
redis.sentinel.node1.host=192.168.51.51
redis.sentinel.node1.port=26379
redis.sentinel.node2.host=192.168.51.52
redis.sentinel.node2.port=26380
redis.sentinel.node3.host=192.168.51.53
redis.sentinel.node3.port=26381

# 授权密码
redis.sentinel.password=123456

### ---分布式锁配置---
# 头寸
redis.sentinel.registry.key=registryKey
```

从配置文件中可以看出，客户端不再关心提供服务的master或slave的实际IP与端口，只需要连接到岗哨，岗哨会指定master来提供服务。

需要注意的是，如果在sentinel模式中设置了jedisConnectionFactory的密码，则该岗哨监测的master及其所有slave均需要设置该密码，否则客户端有可能访问到不需要密码的redis实例，这会在客户端抛出redis连接的异常。

## 4.  Redis常用操作封装

`RedisUtil` 封装了基于RedisTemplate的一些对redis服务器的基本操作，例如存缓存、取缓存、删除缓存、查看是否存在于缓存中等，调用时需要注意分布式环境下的一致性问题。如下：

```java
import java.util.Arrays;
import java.util.concurrent.TimeUnit;

import org.springframework.data.redis.core.RedisTemplate;

/**
 * 缓存工具类，提供直接操作Redis的接口<p>
 * 调用方式：通过spring-data-redis提供的{@link RedisTemplate}操作
 * 
 * @author 51
 * @version $Id: RedisUtil.java, v 0.1 2018年1月13日 下午4:04:11 51 Exp $
 */
public class RedisUtil {

    /** 
     * 默认缓存失效时间<p>
     * 仅供{@link RedisUtil#setWithDefaultExpire(RedisTemplate, String, Object)}使用
     */
    private static final long DEFAULT_EXPIRE_TIME = 10L;

    /**
     * 保存数据至Redis<p>
     * 
     * @param redisTemplate must not be null.
     * @param key must not be null.
     * @param value
     * @param timeout
     * @param unit must not be null.
     * @see <a href="http://redis.io/commands/setex">Redis Documentation: SETEX</a>
     */
    public static void set(RedisTemplate<String, Object> redisTemplate, String key, Object value, Long timeout, TimeUnit unit) {
        redisTemplate.opsForValue().set(key, value, timeout, unit);
    }

    /**
     * 以默认过期时间保存数据至Redis<p>
     * 默认过期时间为{@link RedisUtil#DEFAULT_EXPIRE_TIME}，默认的时间单位为：分
     * 
     * @param redisTemplate must not be null.
     * @param key
     * @param value
     * @see RedisUtil#set(RedisTemplate, String, Object, Long, TimeUnit)
     */
    public static void setWithDefaultExpire(RedisTemplate<String, Object> redisTemplate, String key, Object value) {
        set(redisTemplate, key, value, DEFAULT_EXPIRE_TIME, TimeUnit.MINUTES);
    }

    /**
     * 通过key从Redis中取缓存数据<p>
     * 该方法会将结果根据clazz强转
     * 
     * @param redisTemplate must not be null.
     * @param key
     * @param clazz
     * @return
     * @throws ClassCastException 强制类型转换不成功时会抛出ClassCastException异常
     */
    @SuppressWarnings("unchecked")
    public static <T> T get(RedisTemplate<String, Object> redisTemplate, String key, Class<T> clazz) throws ClassCastException {
        return (T) redisTemplate.boundValueOps(key).get();
    }

    /**
     * 通过key从Redis中取缓存数据
     * 
     * @param redisTemplate must not be null.
     * @param key
     * @return
     */
    public static Object get(RedisTemplate<String, Object> redisTemplate, String key) {
        return redisTemplate.boundValueOps(key).get();
    }

    /**
     * 从Redis中删除keys
     * 
     * @param redisTemplate must not be null.
     * @param keys must not be null.
     */
    public static void delete(RedisTemplate<String, Object> redisTemplate, String... keys) {
        redisTemplate.delete(Arrays.asList(keys));
    }

    /**
     * key是否存在Redis中
     * 
     * @param redisTemplate must not be null.
     * @param key must not be null.
     * @return
     */
    public static boolean exists(RedisTemplate<String, Object> redisTemplate, String key) {
        return redisTemplate.hasKey(key);
    }

}

```

## 5. 缓存服务和分布式锁服务

### 5.1 查询时缓存

每次执行查询操作时，先查询缓存，如果缓存中存在该条记录则直接作为查询结果返回（不执行方法），如果缓存中不存在该条记录则执行方法获取查询结果返回，并将结果存入缓存。

该场景可使用 `@Cacheable` 配置，示例如下：

```java
@Cacheable(value = "redis-users", key = "#userId")
public User getUserByID(String userId) {
  return geneUserById(userId);
}
```

`@Cacheable` 可以标记在一个方法上，也可以标记在一个类上。标记在一个方法上时表示该方法支持缓存，标记在一个类上时则表示该类所有的方法都支持缓存。

`@Cacheable` 常用的有三个属性：value、key、condition。

- **value**：cache名称，必填，可以指定一至多个cache。

  ```java
  @Cacheable(value = "cache1") // 一个cache

  @Cacheable(value = {"cache1", "cache2"}) // 多个cache
  ```

- **key**：指定缓存的key，该属性支持SpringEL表达式。未指定该属性时，将使用默认策略生成key。

  ```java
  @Cacheable(value="redis-users", key="#id")
  public User find(Integer id)

  @Cacheable(value="redis-users", key="#p0")
  public User find(Integer id)

  @Cacheable(value="redis-users", key="#user.id")
  public User find(User user)

  @Cacheable(value="redis-users", key="#p0.id")
  public User find(User user)
  ```

- **condition**：指定发生的条件，默认为空，表示将缓存所有的调用情形。其值通过SpringEL表达式指定，当为 `true` 时表示进行缓存处理，当为 `false` 时表示不进行缓存处理。如下示例表示，只有当user的name长度小于32时才会进行缓存：

  ```java
  @Cacheable(value = "redis-users", key="#user.id", condition="#user.name.length() < 32")
  public User find(User user)
  ```

有关于 `@Cacheable` 的更多配置项及说明可以通过第二章中的参考文档了解。

### 5.2 缓存服务场景 - 更新

每次执行更新操作时，执行更新操作后，更新结果将存入缓存（覆盖之前可能存在的值）。

该场景使用 `@CacheEvict` 配置，示例如下：

```java
@CacheEvict(value = "redis-users", key = "#userId")
public User updateUserByID(String userId)
```

`@CacheEvict` 配置项说明见下节。

### 5.3 缓存服务场景 - 删除

每次执行删除操作时，将对应的缓存也删除。

该场景使用 `@CacheEvict` 配置，示例如下：

```java
@CacheEvict(value = "redis-users", key = "#userId")
public int deleteUserByID(String userId)
```

`@CacheEvict` 配置项与 `@Cacheable` 也基本相同，值得一提的是其allEntries和beforeInvocation属性。

- **allEntries**：是否要清除整个缓存，默认false，当指定为true时，将会清空value属性对应的整个缓存，此时框架将忽略 `@CacheEvict` 所配置的key属性。
- **beforeInvocation**：是否在方法执行之前删除缓存，默认false，表示方法成功完成后删除缓存（如果该方法不执行或发生异常，则不会删除缓存），当指定为true时，在执行方法前就删除缓存，与方法执行结果无关。

### 5.4 分布式环境下访问缓存

以获取头寸和修改头寸来模拟分布式环境下访问缓存的场景。该场景中，要求在分布式环境下操作缓存并保证数据一致性，采用分布式锁的方式来实现。

例如，在修改头寸的方法中，不是直接修改缓存的值，而是首先尝试获取锁，获取到锁之后才进行修改缓存的操作，代码如下：

```java
public void changePosition(String positionId, int change) {
  final Lock lock = redisLockRegistry.obtain(positionId);
  try {
    if (lock.tryLock(RETRY_TIMEOUT, TimeUnit.SECONDS)) { // 尝试获取锁
      Integer oldPosition = RedisUtil.get(redisTemplate, positionId, Integer.class); // 查找头寸

      // 设置头寸开始
      if (null == oldPosition) {
        // 如果缓存中不存在头寸，则将change视为初始头寸，存入缓存
        RedisUtil.setWithDefaultExpire(redisTemplate, positionId, change);
      } else {
        // 如果缓存中存在头寸，则计算修改后的头寸，存入缓存
        RedisUtil.setWithDefaultExpire(redisTemplate, positionId, oldPosition + change);
      }
      // 设置头寸结束
    } else {
      LogUtil.warn(logger, "头寸修改获取锁失败,id={0},change={1}", positionId, change);
    }
  } catch (Exception e) {
    LogUtil.error(e, logger, "头寸修改失败,id={0},change={1}", positionId, change);
  } finally {
    try {
      lock.unlock();
    } catch (IllegalStateException e) {
      LogUtil.error(e, logger, "头寸修改尝试解锁异常");
    }
  }
}

/**
     * 测试分布式锁服务
     * <p>每隔0.5秒创建并启动一个线程，共启动10个线程；</p>
     * <p>对于每个线程：每隔0.5秒对头寸增加1，共增加5次；</p>
     * <p>主线程创建完最后一个线程后，阻塞10秒以等待其余线程执行完毕，然后取头寸值，预期为50。</p>
     * 
     * @throws InterruptedException
     */
    @Test
    public void testChangePosition() throws InterruptedException {
        Assert.assertNull(RedisUtil.get(redisTemplate, positionId)); // 初始缓存中对应头寸数据为空
        for (int i = 0; i < 10; i++) {
            // 每隔0.5秒新建并启动一个线程，共启动10个线程
            new Thread("thread_" + i) {
                public void run() {
                    // 每个线程每隔0.5秒头寸+1，执行5次
                    for (int i = 0; i < 5; i++) {
                        positionService4RedisCacheTest.changePosition(positionId, 1);
                        try {
                            Thread.sleep(500);
                        } catch (InterruptedException e) {
                            LogUtil.error(e, logger, "InterruptedException");
                        }
                    }
                }
            }.start();
            Thread.sleep(500);
        }

        try {
            Thread.sleep(10 * 1000); // 确保上述线程执行结束
        } catch (InterruptedException e) {
            LogUtil.error(e, logger, "InterruptedException");
        }

        Assert.assertEquals(Integer.valueOf(50), positionService4RedisCacheTest.getPosition(positionId)); // 预期头寸值为50

    }
```

