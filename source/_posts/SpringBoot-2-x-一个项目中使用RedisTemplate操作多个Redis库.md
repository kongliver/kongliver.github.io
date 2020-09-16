---
title: SpringBoot 2.x 一个项目中使用RedisTemplate操作多个Redis库
date: 2020-09-15 17:37:13
categories:
  - [Java, Springboot]
  - [技术日常]
tags:
  - Java
  - Springboot
top_image: /assets/blogpicture/valais.jpg
---

背景：我们都知道Redis有16个数据库可以使用，在项目中需要用到redis的多个库，每次使用时再去通过一堆代码切换未免觉得太过麻烦，所以直接通过配置注入多个RedisTemplate，需要用到哪个库时直接使用对应的RedisTemplate即可
<!-- more -->

### 首先是配置文件
**在application.properties中添加redis的相关配置**

```java
#redis多数据配置
redis.database.test1=1
redis.database.test2=2
redis.host=127.0.0.1
redis.port=6379
redis.password=root
##连接超时，此处使用单位秒
redis.timeout=180
##连接池配置
redis.pool.max-active=8
redis.pool.max-idle=8
redis.pool.min-idle=0
redis.pool.max-wait=-1
```
- 数据库的选择可根据业务需求配置多个，此处测试只配置了两个。
- 连接池的配置都是使用的默认值，如有其他需求可自行更改

### 然后通过配置类注入RedisTemplate
- 先看代码
```java
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisPassword;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.connection.lettuce.LettucePoolingClientConfiguration;
import org.springframework.data.redis.core.StringRedisTemplate;

import java.time.Duration;

/**
 * @date 2020/1/21
 */
@Configuration
public class RedisConfiguration {

    @Value("${redis.database.test1}")
    private int test1Database;

    @Value("${redis.database.test2}")
    private int test2Database;

    @Value("${redis.host}")
    private String host;

    @Value("${redis.port}")
    private int port;

    @Value("${redis.password}")
    private String password;

    @Value("${redis.timeout}")
    private int timeout;

    @Value("${redis.pool.max-active}")
    private int maxActive;

    @Value("${redis.pool.max-idle}")
    private int maxIdle;

    @Value("${redis.pool.min-idle}")
    private int minIdle;

    @Value("${redis.pool.max-wait}")
    private int maxWait;

    @Bean
    public GenericObjectPoolConfig getPoolConfig(){
        // 配置redis连接池
        GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig();
        poolConfig.setMaxTotal(maxActive);
        poolConfig.setMaxIdle(maxIdle);
        poolConfig.setMinIdle(minIdle);
        poolConfig.setMaxWaitMillis(maxWait);
        return poolConfig;
    }

    @Bean(name = "test1RedisTemplate")
    public StringRedisTemplate getTest1RedisTemplate(){
        return getStringRedisTemplate(test1Database);
    }

    @Bean(name = "test2RedisTemplate")
    public StringRedisTemplate getTest2RedisTemplate(){
        // 构建工厂对象
        return getStringRedisTemplate(test2Database);
    }

    private StringRedisTemplate getStringRedisTemplate(int database) {
        // 构建工厂对象
        RedisStandaloneConfiguration configuration = new RedisStandaloneConfiguration();
        configuration.setHostName(host);
        configuration.setPort(port);
        configuration.setPassword(RedisPassword.of(password));
        LettucePoolingClientConfiguration clientConfiguration = LettucePoolingClientConfiguration.builder()
                .commandTimeout(Duration.ofSeconds(timeout)).poolConfig(getPoolConfig()).build();
        LettuceConnectionFactory factory = new LettuceConnectionFactory(configuration, clientConfiguration);
        // 设置使用的redis数据库
        factory.setDatabase(database);
        // 重新初始化工厂
        factory.afterPropertiesSet();
        return new StringRedisTemplate(factory);
    }

}
```
#### 解释几个地方：
1. SpringBoot 2.x 之后连接redis驱动默认使用的是**lettuce**，而非之前的**jedis**，这里简要说一下两个驱动的区别：
- jedis采用的是直连redis server，在多个线程之间共用一个jedis实例时，是线程不安全的。如果想避免线程不安全，可以使用连接池pool，这样每个线程单独使用一个jedis实例。由此带来的问题是，如果线程数过多，带来redis server的负载加大。有点类似于BIO的模式。

- lettuce采用netty连接redis server，实例可以在多个线程间共享，不存在线程不安全的情况，这样可以减少线程数量。在特殊情况下，lettuce也可以使用多个实例。有点类似于NIO的模式

2. 翻看源码**lettuce**驱动的连接池是依赖于apache的commons-pool2中的GenericObjectPoolConfig对象实现的，而SpringBoot对Redis的Starter中未引入该依赖
```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
所以如果要配置连接池并且项目中未通过其他组件引入commons-pool2依赖时需要手动引入该依赖
```java
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
    <version>2.8.0</version>
</dependency>
```
### 最后是测试环节
- 直接使用SpringBoot的Junit测试，分别使用两个RedisTemplate操作Redis
```java
@SpringBootTest
class RedisdemoApplicationTests {

    @Resource(name = "test1RedisTemplate")
    private StringRedisTemplate test1RedisTemplate;

    @Resource(name = "test2RedisTemplate")
    private StringRedisTemplate test2RedisTemplate;

    @Test
    public void testRedisTemplate() {
        // 测试用两个模板向redis中存值
        test1RedisTemplate.opsForValue().set("name", "kong");
        test2RedisTemplate.opsForValue().set("age", "20");
    }

}
```
- 查看Redis库看是否操作成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200122172854752.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjIzMzUwMA==,size_16,color_FFFFFF,t_70)
- 可以看到Redis两个库中均已存入相应的值，说明RedisTemplate生效

> 至此项目中就实现了可以使用多个Redis库的操作
