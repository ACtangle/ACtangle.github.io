---
title: redis之锁相关知识和场景
toc: true
date: 2020-01-02 17:40:56
thumbnail: /gallery/thumbnails/redis之锁相关知识.png
tags: redis
categories: 笔记
---
#### 注意：笔记只对作者有用，不能保证可靠性
<!--more-->
#### 1.加锁方法
###### 目的：解决死锁问题
###### 场景：商品秒杀，key为商品id，value为当前时间和超时时间
###### 内容：某一个时间点上，多个用户秒杀某个商品，其中开启了多个线程去处理，可能造成多个线程内有某个线程获取锁后由于其他方面如网络堵塞造成锁无法释放，使得时间超时，这个时候应该解锁，不能造成死锁。
###### 关键点：1.锁只能被单个线程占用；
###### 2.当两个线程同时进入加锁方法时且进入到锁过期代码段，则只能有一个线程会去跑redisTemplate.opsForValue().getAndSet(key,value)这段代码
###### 3.getAndSet方法获取的是上个占用锁的时间，且将当前进入该代码段的线程的value置位改key新的value值，所以当第二个线程进来时，原先的value已经被修改了，则判断不相等。
```
/**
  * key为商品id,value为当前时间+超时时间
 */
public boolean lock(String key, String value) {
    //如果不存在则可以加锁判断为true返回为true
    if(redisTemplate.opsForValue().setIfAbsent(key,value)){
        return true; 
    }
    //获取上一次占用锁的value值，方便理解这里使用时间
    String currentValue = redisTemplate.opsForValue().get(key);
    //如果锁过期了
    if(!StringUtils.isEmpty(currentValue) &&
       Long.parseLong(currentValue) < System.currentTimeMillis()) {
       //获取上一个锁的时间
       String oldValue = redisTemplate.opsForValue().getAndSet(key,value);
       if(!StringUtils.isEmpty(oldValue) &&
         oldValue.equals(currentValue)) {
            return true;
         }
    }
    return false;
}
```

#### 1.释放锁
###### 目的：释放占用的锁
###### 场景：释放锁
###### 内容：拿到当前的时间值与redis的value值对应，相等则删除redis对应key释放锁
###### 关键点：释放锁

```
/**
  * 拿到当前的value值与redis对应key的值作比较，相等则删除，否则抛异常
 */
public void unlock(String key,String value) {
    try{
        String currentValue = redisTemplate.opsForValue().get(key);
        if(StringUtils.isEmpty(currentValue) && currentValue.equals(value) {
            redisTemplate.opsForValue().getOperations().delete(key);
        }
    }catch(Exception){
        log.error("[redis分布式锁机制]解锁异常:{}",e);
    }
    
}
```