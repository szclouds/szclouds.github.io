---
title: Redis 分布式锁
layout: default
tags: node
parent: Node
permalink: /docs/node_redis_distributed_lock
---

# Redis 分布式锁

## 描述

使用分布式锁在多实例的 Node.js 应用中查询热点数据，避免瞬时大量的请求到达数据库中。

## 整体流程

1. 查询缓存，若缓存存在则直接响应 
2. 定义加锁 key 和 val，其中 val 作为释放锁时的关键点，避免错误释放锁
3. 进入加锁阶段，使用 set+nx 命令进行加锁
4. 等待 100 ms（可根据业务实际情况进行设置），加锁失败说明出现多个实例并发竞争，此时如果短时间内重试加锁，则达不到重试效果
5. 加锁失败，先查缓存（此时存在其他实例已经新建缓存的可能，避免不必要的重试）
6. 若加锁次数大于指定的最大次数，则抛出异常，此次请求处理失败
7. 加锁成功，使用 try-finally 包裹业务操作，对业务查询结果设置缓存
8. finally 不管是否异常都执行释放锁的过程，释放锁的过程使用 lua 脚本校验 lockVal 是否和加锁过程中的值相同，避免释放错锁，使用 lua 脚本执行可保证原子性操作

## 关键点

1. 对于低延时的系统，加锁失败重试次数不能过多，避免超时
2. 加锁过期时间应当大于业务处理时间，否则存在业务没有处理完成情况下，锁自动释放
3. 使用 try-finally 保证锁的释放，释放时需要校验对应的值是否为加锁时的值，原因：在多个实例并发的情况下，
   A 实例获得锁且处理业务请求时间超过锁的过期时间，此时 B 可加锁成功并处理业务请求，而此时 A 执行完成，
   执行 del 操作，导致 B 的锁被删除，此时其他实例又可获取锁，导致分布式锁成摆设，故需要对锁中的值进行校验

## 编码实现

```javascript
'use strict'
const {ObjectId} = require('mongodb')
const {redisClient} = require('./redis_helper')
const data_key = 'test_data_key'

async function queryData() {
    const cache = getDataFromRedis()
    if (cache){
        return JSON.parse(cache)
    }
    const lockKey = 'test:lock'
    const lockVal = new ObjectId().toString()
    console.log('lockKey:%s  lockVal:%s', lockKey, lockVal)
    const maxRetryCount = 3
    for (let i = 0; i <= maxRetryCount; ++i) {
        console.log('retryCount: ', i)
        const lockResult = await redisClient.set(lockKey, lockVal, {EX: 5, NX: true})
        if (lockResult) {
            console.log('lock success')
            break
        }
        console.log('lock fail, retry......')
        await new Promise((resolve) => setTimeout(resolve, 100))
        const cache = await getDataFromRedis()
        if (cache) {
            return cache
        }
        if (i >= maxRetryCount) {
            throw new Error('lock fail')
        }
    }
    try {
        const data = await getDataFromDb()
        await redisClient.set(
            'test_data_key',
            JSON.stringify({
                data
            }),
            {EX: 10}
        )
        return data
    } finally {
        const delResult = await redisClient.eval(
            "if redis.call('get',KEYS[1]) == ARGV[1] then" +
            "   return redis.call('del',KEYS[1]) " +
            'else' +
            '   return 0 ' +
            'end',
            {keys: [lockKey], arguments: [lockVal]}
        )
        console.log('release lock success')
    }
}

async function getDataFromDb(){
    // query db ...
}

async function getDataFromRedis(){
    // query redis
}
```
