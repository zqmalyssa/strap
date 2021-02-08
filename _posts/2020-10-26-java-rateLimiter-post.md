---
layout: post
title: java限流
tags: [java, code]
author-id: zqmalyssa
---

Java中限流的使用

#### 概况

比较常见的有阿里的s，不过比较重，比较常见的有google的GUAVA，而信号量也可以做简单的限流


guava的限流方式，采用令牌桶算法（另一种是漏桶算法，严格控制量），令牌桶算法是控制每秒的请求数，应对突发

两种的限流方式是两种相反的行为，一个是放水，一个是加水

```java
public void guavaRateLimiterTest() throws Exception {
    final RateLimiter rateLimiter = RateLimiter.create(10);
    ExecutorService executorService = Executors.newFixedThreadPool(50);
    for (int i = 0; i < 50; i++) {
      int finalI = i;
      executorService.execute(new Runnable() {
        @Override public void run() {
          rateLimiter.acquire();
          System.out.println(String.format("------------------action: %s -------------------", finalI));
          List<Integer> time = RandomNum.randomNumbers(3, 6, 1);
          try {
            //            Thread.sleep(2000);
            Thread.sleep(time.get(0) * 1000);
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
          System.out.println(String.format("------------------finish: %s -------------------", finalI));
        }
      });
    }
    try {
      executorService.awaitTermination(10, TimeUnit.MINUTES);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

```

而令牌桶比较适合的场景是每秒钟 有多少请求能进来，剩余的处理大多是丢弃，而实际后端处理的话需要等待，是一种量的限制

rateLimiter是单机版的，分布式情况下需要改造，这个时候基本需要中间件了，如redis的支持


#### 分布式

1、用成熟的框架，如redisson，里面有一个`RPermitExpirableSemaphore`，基于信号量的可过期式限流机制，但是目前看下来有点不稳点，底层也是redis+lua脚本实现，在第一次执行的时候会有倍增情况

```java
RPermitExpirableSemaphore semaphore = redissonFactory.getSemaphore("fan-test", 6);
id = semaphore.tryAcquire(3600, 120, TimeUnit.SECONDS);
semaphore.release(id);
```

2、自己写一个lua，然后eval，跟redis集成起来

配置文件

```java

@Component
public class RedisConfiguration {

    /**
     * 读取限流脚本
     *
     * @return
     */
    @Bean
    public DefaultRedisScript<Number> redisluaScript() {
        DefaultRedisScript<Number> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("rateLimit.lua")));
        redisScript.setResultType(Number.class);
        return redisScript;
    }

    /**
     * 读取限流Drop脚本
     *
     * @return
     */
    @Bean
    public DefaultRedisScript<Number> redisluaDropScript() {
        DefaultRedisScript<Number> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("rateLimitDrop.lua")));
        redisScript.setResultType(Number.class);
        return redisScript;
    }

    /**
     * RedisTemplate
     *
     * @return
     */
    @Bean
    public RedisTemplate<String, Serializable> limitRedisTemplate(LettuceConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Serializable> template = new RedisTemplate<String, Serializable>();
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

}


@Configuration("utilRestTemplateConfig")
public class RestTemplateConfig {
  @Bean("utilRestTemplate")
  RestTemplate restTemplate() {
    return RestClient.custom().setConnectTimeout(15).setSocketTimeout(60).build();
  }
}

```

lua脚本

```html

增

-- 限流KEY
local key = "rate.limit:" .. KEYS[1]
-- 限流大小
local limit = tonumber(ARGV[1])
local current = tonumber(redis.call('get', key))
if current == nil then
 current = 0
end
-- 如果超出限流大小
if current + 1 > limit then
  return 0
-- 请求数+1，并设置2秒过期，有的场景有需求
else
  redis.call("INCRBY", key, "1")
--  redis.call("expire", key, "2")
  return current + 1
end


-- 解

-- 限流KEY
local key = "rate.limit:" .. KEYS[1]
-- 限流大小
local limit = tonumber(ARGV[1])
local current = tonumber(redis.call('get', key))
redis.call("DECRBY", key, "1")
return current - 1

```


大致的用法

```java

@Autowired private RedisTemplate<String, Serializable> limitRedisTemplate;

@Autowired private DefaultRedisScript<Number> redisluaScript;

@Autowired private DefaultRedisScript<Number> redisluaDropScript;

public void redisLuaTest() throws Exception {
    ExecutorService executorService = Executors.newFixedThreadPool(20);
    for (int i = 0; i < 50; i++) {
      int finalI = i;
      executorService.execute(new Runnable() {
        @Override
        public void run() {
          try {
            // 保持需要做完
            while(true) {
              Number number = limitRedisTemplate.execute(redisluaScript, Collections.singletonList("redis-lua"), 5);
              if (number != null && number.intValue() != 0 && number.intValue() <= 5) {
                System.out.println(String.format("------------------action: %s -------------------", finalI));
                break;
              }
            }
          } catch (Exception e) {
            System.out.println(String.format("------------------error: %s -------------------", finalI));
            e.printStackTrace();
          }
          List<Integer> time = RandomNum.randomNumbers(3, 6, 1);
          try {
            Thread.sleep(time.get(0) * 1000);
          } catch (InterruptedException e) {
            e.printStackTrace();
          }

          Number number = limitRedisTemplate.execute(redisluaDropScript, Collections.singletonList("redis-lua"), 5);
          System.out.println(String.format("------------------recover: %s -------------------", finalI));

        }
      });
    }
    try {
      executorService.awaitTermination(10, TimeUnit.MINUTES);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }
```
