---
date: '2025-03-04T20:26:04+08:00'
draft: false
description: 本文章旨在整理学习Redis的相关知识.
featured_image: "/images/Redis_photo.png"
tags: [Redis,Java]
title: 'Redis_demo_2_查询缓存'
categories: Redis
---



## 1. 什么是缓存？

缓存就是数据交换的缓冲区（Cache），是存贮数据的临时地方，一般读写性能较高。

在实际开发中，系统需要避震器，防止过高的数据量猛冲系统，导致其操作线程无法及时处理信息而瘫痪。

![](/images/c3f7e9b2-bf66-46d9-a701-dd84fee3b88a.png)



## 2. 添加商户缓存

![](/images/7147641a-c534-4ddf-9b4f-5496fd4cb27b.png)

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

![](/images/f1d50e2c-6442-4d1e-afe7-4d50a09a4ca5.png)

### 4.1 业务场景：

- 低一致性需求（即数据长久不发生变化）：使用默认的内存淘汰记住，例如店铺类型的查询缓存。
- 高一致性需求：主动更新，并以超时剔除作为兜底方案，例如店铺详情查询的存储。

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

![](/images/db935104-89a7-4c90-9e09-a420df7dd64b.png)

由于查询缓存、数据库再写入缓存的操作时间比删除缓存再更新数据库要快很多，并且缓存失效的概率很小，所以认为后者出现线程安全问题的概率相对较低，因此采用后者。
