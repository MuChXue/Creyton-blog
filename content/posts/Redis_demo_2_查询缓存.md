---
date: '2025-03-04T20:26:04+08:00'
draft: false
description: 本文章旨在整理学习Redis的相关知识.
featured_image: "https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/Redis_photo.png"
tags: [Redis,Java]
title: 'Redis_demo_2_查询缓存'
categories: Redis
---



## 1. 什么是缓存？

缓存就是数据交换的缓冲区（Cache），是存贮数据的临时地方，一般读写性能较高。

在实际开发中，系统需要避震器，防止过高的数据量猛冲系统，导致其操作线程无法及时处理信息而瘫痪。

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/c3f7e9b2-bf66-46d9-a701-dd84fee3b88a.png)



## 2. 添加商户缓存

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/1653322097736.png)

## 3. 添加商户类型缓存

与商户数据缓存类似，尝试完成商户类型数据缓存

在ShopTypeController中

```
@RestController
@RequestMapping("/shop-type")
public class ShopTypeController {
    @Resource
    private IShopTypeService typeService;

    @GetMapping("list")
    public Result queryTypeList() {
        return typeService.queryByShopType();
    }
}
```

在ShopTypeServiceImpl中

```
@Service
public class ShopTypeServiceImpl extends ServiceImpl<ShopTypeMapper, ShopType> implements IShopTypeService {

    @Resource
    StringRedisTemplate stringRedisTemplate;

    @Override
    public Result queryByShopType() {
        String key = CACHE_SHOP_TYPE_KEY;
        // 1.从redis中查询商铺类型缓存
        String shopTypeJson = stringRedisTemplate.opsForValue().get(key);
        // 2.判断缓存是否命中
        if (StrUtil.isNotBlank(shopTypeJson)) {
            // 3.命中，返回商铺类型信息
            List<ShopType> shopTypeList = JSONUtil.toList(shopTypeJson, ShopType.class);
            return Result.ok(shopTypeList);
        }
        // 4.未命中，查询数据库
        List<ShopType> shopTypeList = query().orderByAsc("sort").list();
        // 5.判断数据库中是否存在
        if (shopTypeList.isEmpty()) {
            // 6.不存在，返回错误
            return Result.fail("商铺类型不存在！");
        }
        // 7.存在，将商铺类型写入redis
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shopTypeList));
        // 8.返回商铺类型信息
        return Result.ok(shopTypeList);
    }

}
```

## 4. 缓存更新

缓存更新的目的是为了节约Redis中的内存，为避免缓存过多，Redis会对部分数据进行更新。

### 4.1 业务场景：

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/1653322491532.png)

### 4.2 数据库和缓存不一致的解决方案：

由于缓存数据源于数据库，而数据库中的数据会发生变化，因此当数据库中数据发生变化，但是缓存没有同步时，就会有一致性问题存在，导致用户使用缓存中的过时数据。

**解决这个问题主要通过主动更新，有三种方式：**

- Cache Aside Pattern 人工编码方式：缓存调用者在更新完数据库之后再去更新缓存，也称为双写方案。
- Read/Write Through Pattern：缓存与数据库整合为一个服务，由服务来维护一致性。调用者调用该服务，无需关心缓存一致性问题，但是维护这个服务很复杂，市面上不容易找到现成的服务，开发成本高。
- Write Behind Caching Pattern：调用者只操作缓存，其他线程去异步处理数据库，最终实现一致性。但维护这样的一个异步任务很复杂，需要实时监控缓存中的数据更新，其他线程去异步更新数据库不太及时，而缓存服务器如果宕机，那么缓存的数据也丢失了，不能保证数据的一致性。

在企业的实际应用中，方案一最靠谱。若采用方案一，假设每次操作完数据库之后，都更新一下缓存，但是如果中间并没有人查询数据，中间n次更新缓存的操作都是无效的。所以我们可以把缓存直接删除，等待有人再次查询时，再把缓存中的数据加载出来。

- 更细缓存：每次更新数据库时都需要更新缓存，无效写操作较多
- 删除缓存：更新数据库时让缓存失效，再次查询时更新缓存

**如何保证缓存与数据库的操作同时成功/失败**

- 单体系统：将缓存与数据库操作放在同一个事务
- 分布式系统：利用TTC等分布式事务方案

**先操作缓存还是先操作数据库？**

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/1653323595206.png)

由于查询缓存、数据库再写入缓存的操作时间比删除缓存再更新数据库要快很多，并且缓存失效的概率很小，所以认为后者出现线程安全问题的概率相对较低，因此采用后者。

### 4.3 实现商铺缓存与数据库双写一致

核心思路如下：

修改ShopController中的业务逻辑，满足以下要求

- 根据id查询店铺时，如果缓存未命中，则查询数据库，并将数据库结果写入缓存，并设置TTL

- 根据id修改店铺时，先修改数据库，再删除缓存

## 5. 缓存穿透

缓存穿透是指客户端请求的数据在缓存中和数据库中都不存在，这样的缓存永远不会生效，这些请求都会打到数据库。

空查询本身确实性能开销不高，但在高并发场景下，如果没有对无效请求做缓存空值或预过滤，那么这些重复的空查询就会导致数据库连接资源浪费、吞吐下降，甚至造成数据库崩溃。

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/024e3c91-aa92-42f6-9a87-beac7633d356.png)

### 5.1 解决方法

- 缓存空对象

  优点：实现简单，维护方便

  缺点：额外的内存消耗（可设置TTL）

  ​	   可能造成短期的不一致  

  ![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/b831a31f-402c-4cd8-be66-e234d52cc165.png)

- 布隆过滤

​	优点：内存占用少，没有多余的key

​	缺点：实现复杂

​		   存在误判可能

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/1653326156516.png)

缓存空对象分析：当客户端访问不存在的数据时，会先请求Redis，但是此时Redis中也没有该数据，随后就会访问数据库，但是数据库中也没有该数据，那么这个数据就穿透缓存，直击数据库，但是数据库能承载的并发不如Redis那么高，所以大量请求同时都来访问这个不存在的数据，那么这些请求都会访问到数据库。简单的解决办法就是哪怕这个数据在数据库中不存在，我们也把这个数据存入Redis中，这就是为什么会有**额外内存消耗**，这样下次用户来访问数据库时，redis缓存也能找到这个数据。可能造成的**短期不一致**是指在空对象的存活期间，我们更新了数据库，把这个空对象变成了正常的可以访问的数据，但由于Redis中存储的空对象的TTL还没过，当用户来查询时，查询到的还是空对象，等待TTL过了之后，才能访问到正确数据，但这种情况比较少见

布隆过滤分析：布隆过滤器其实采用的是哈希思想解决这个问题，通过一个庞大的二进制数组，根据哈希思想判断当前这个要查询的数据是否存在，存在则放行，不存在则直接返回。但是布隆过滤器使用的哈希思想会存在哈希冲突，所以存在**误判可能**

上述两种解决方案只是被动解决缓存穿透，我们还可以采用一些主动手段：

- 增加id复杂度，避免被猜测id规律
- 做好数据的基础格式校验
- 加强用户权限校验
- 做好热点参数限流

### 5.2 解决商铺的缓存穿透问题

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/1653327124561.png)

## 6. 缓存雪崩

缓存雪崩是指在同一时间段，大量缓存的key同时失效，或者Redis服务宕机，导致大量请求到达数据库，带来巨大压力

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/1653327884526.png)

### 6.1 解决方案

- 给不同的Key的TTL添加随机值，让其在不同时间段分批失效
- 利用Redis集群提高服务的可用性（使用一个或者多个哨兵(Sentinel)实例组成的系统，对redis节点进行监控，在主节点出现故障的情况下，能将从节点中的一个升级为主节点，进行故障转义，保证系统的可用性）
- 给缓存业务添加降级限流策略
- 给业务添加多级缓存（浏览器访问静态资源时，优先读取浏览器本地缓存；访问非静态资源（ajax查询数据）时，访问服务端；请求到达Nginx后，优先读取Nginx本地缓存；如果Nginx本地缓存未命中，则去直接查询Redis（不经过Tomcat）；如果Redis查询未命中，则查询Tomcat；请求进入Tomcat后，优先查询JVM进程缓存；如果JVM进程缓存未命中，则查询数据库）
  

## 7. 缓存击穿

缓存击穿也叫热点Key问题，就是一个被高并发访问并且缓存重建业务比较复杂的key突然失效了，那么无数请求访问就会在瞬间给数据库带来巨大的冲击

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/1653328022622.png)

### 7.1 解决方案一：互斥锁

缺点：性能较差

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/1653328288627.png)

### 7.2 解决方案一：逻辑过期

该方法巧妙在异步构建缓存数据，缺点是在重建完缓存数据之前，返回的都是脏数据

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/1653328663897.png)

互斥锁与逻辑过期的对比

![](https://raw.githubusercontent.com/MuChXue/Creyton-blog/main/images/1653357522914.png)
