---
layout:     post
title:      使用JedisPool连接Redis
subtitle:   ——SpringBoot2.1.1+Redis4.0.1
date:       2018-11-08
author:     Zwx
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - SpringBoot
---

---
## 为什么要使用连接池？
　　上一篇安装和简单的使用了Redis，而Redis作为缓存数据库理论上和MySQL一样需要客户端和服务端建立起来连接进行相关操作，
使用MySQL的时候相信大家都会使用一款开源的连接池,例如C3P0.因为直连会消耗大量的数据库资源，每一次新建一个连接之，使用后再断开连接，对于频繁访问的场景，这显然不是高效的。假设Redis服务器与客户端分处在异地，虽然基于内存的Redis数据库有着超高的性能，但是底层的网络通信却占用了一次数据请求的大量时间，因为每次数据交互都需要先建立连接，假设一次数据交互总共用时30ms，超高性能的Redis数据库处理数据所花的时间可能不到1ms，也即是说前期的连接占用了29ms，连接池则可以实现在客户端建立多个链接并且不释放，当需要使用连接的时候通过一定的算法获取已经建立的连接，使用完了以后则还给连接池，这就免去了数据库连接所占用的时间。
                                                                                        
---
## 新建项目
　　之前博客的代码全演示在[→sb_demo](https://github.com/genius6119/sb_demo.git)这个项目中了，写得比较乱，代码质量惨不忍睹，所以写这个例子就开了一个新的项目，源码地址：
[→Jedis_demo](https://github.com/genius6119/Jedis_demo.git)。

　　由于不了解SpringBoot2.0有什么变化，之前的版本用的1.5.18，这几天了解了一下变化，直接上了最新的SpringBoot2.1.1测试版。顺便踩踩坑~

--
## 配置

在application.yml中添加：
```
#redis
redis:
  host: xx.xx.xx.80
  port: 6379
  max-idle: 5
  max-total: 10
  max-wait-millis: 3000
```                    

新建常量类：
```java
package com.zwx.Jedis_demo.constants;

import lombok.Data;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
@Data
public class RedisParameter {
    @Value("${redis.host}")
    private String redisHost;
    @Value("${redis.port}")
    private int redisPort;
    @Value("${redis.max-idle}")
    private int redisMaxTotal;
    @Value("${redis.max-total}")
    private int redisMaxIdle;
    @Value("${redis.max-wait-millis}")
    private int redisMaxWaitMillis;
}

```                

新建JedisPool实例类：
```java
package com.zwx.Jedis_demo.utils;

import com.zwx.Jedis_demo.constants.RedisParameter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;
import javax.annotation.PostConstruct;

@Component
public class JedisPoolWrapper {
    
    private JedisPool jedisPool=null;

    @Autowired
    private RedisParameter parameters;

    @PostConstruct              /**这个注解保证这个方法一定会执行，其实也可以在构造方法中调用这个方法，不过用注解比较帅*/
    public void init() throws Exception{
        try {
            JedisPoolConfig config=new JedisPoolConfig();
            config.setMaxWaitMillis(parameters.getRedisMaxWaitMillis());
            config.setMaxIdle(parameters.getRedisMaxIdle());
            config.setMaxTotal(parameters.getRedisMaxTotal());
            jedisPool=new redis.clients.jedis.JedisPool(config,parameters.getRedisHost(),parameters.getRedisPort(),3000);
        } catch (Exception e) {
            throw new Exception("初始化redis失败");
        }
    }

    public JedisPool getJedisPool() {
        return jedisPool;
    }

}

```   

新建缓存工具类：
```java
package com.zwx.Jedis_demo.utils;


import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;

@Component
public class CacheUtil {

    @Autowired
    JedisPoolWrapper jedisPoolWrapper;
    /**
     * 设置key value 以及过期时间
     * @param key
     * @param value
     * @param expiry
     * @return
     */
    public long cacheNxExpire(String key, String value, int expiry) {
        long result = 0;
        try {
            JedisPool pool = jedisPoolWrapper.getJedisPool();
            if (pool != null) {
                try (Jedis jedis = pool.getResource()) {
                    jedis.select(0);
                    result = jedis.setnx(key, value);
                    jedis.expire(key, expiry);
                }
            }
        } catch (Exception e){
            e.printStackTrace();
        }
        return result;
    }

    /**
     * 获取缓存key
     * @param key
     * @return
     */
    public String getCacheValue(String key) {
        String value = null;
        try {
            JedisPool pool = jedisPoolWrapper.getJedisPool();
            if (pool != null) {
                try (Jedis Jedis = pool.getResource()) {
                    Jedis.select(0);
                    value = Jedis.get(key);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return value;
    }
}

```     
测试类：
```java
package com.zwx.Jedis_demo;

import com.zwx.Jedis_demo.utils.CacheUtil;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class JedisDemoApplicationTests {

	@Autowired
	CacheUtil cacheUtil;

	/**
	*@Auther z
	*@Date 2018-11-08 10:17
	*@Describe 字符串写入缓存，过期时间30秒，每隔3秒取一次
	*/
	@Test
	public void testCache() {
		cacheUtil.cacheNxExpire("1","1",30);
        try {
            int i=0;
            while(i != i + 1){
                Thread.sleep(3000);
                System.out.println(cacheUtil.getCacheValue("1"));
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```        
运行测试类，观察输出结果：
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::  (v2.1.1.BUILD-SNAPSHOT)

2018-11-08 10:19:31.670  INFO 8148 --- [           main] c.z.J.JedisDemoApplicationTests          : Starting JedisDemoApplicationTests on WIN7-1805241357 with PID 8148 (started by Administrator in C:\Project2\Jedis_demo)
2018-11-08 10:19:31.673  INFO 8148 --- [           main] c.z.J.JedisDemoApplicationTests          : No active profile set, falling back to default profiles: default
2018-11-08 10:19:33.815  INFO 8148 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.hateoas.config.HateoasConfiguration' of type [org.springframework.hateoas.config.HateoasConfiguration$$EnhancerBySpringCGLIB$$13a74407] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2018-11-08 10:19:35.011  INFO 8148 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2018-11-08 10:19:35.740  INFO 8148 --- [           main] c.z.J.JedisDemoApplicationTests          : Started JedisDemoApplicationTests in 6.177 seconds (JVM running for 8.697)
1
1
1
1
1
1
1
1
1
null
null
null

```       
- 前30秒每隔3秒输出了缓存中的内容，从第30秒开始，缓存过期，获取到null。成功得操作了Redis缓存，而且不用每操作一次都要重新连接一次。

---
## 后记
　　CacheUtil工具类中的方法可以根据官方的api随意扩展，这里我只写了最简单的String类型的对象的新增和查询。                                     