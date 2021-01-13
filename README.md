## Redis

This is my personal redis learning repositry. May also includes some architecture design senarios using redis and other mainstream programming languages or frameworks.

# Table of Contents

1. [What is redis](#What-is-redis?)
2. [Foundamental data types](#Foundamental-data-types:)
3. [Scenarios](#Scenarios)
4. [String type](#String-type)
5. [Hash type](#Hash-Type)
6. [List type](#List-Type)
7. [Set type](#Set-Type)
8. [Data consistency](#Data-consistency)
9. [Cache penetration](#Cache-penetration:)
10. [Hotspot invalid
    ](Hotspot-invalid)
11. [Cache avalanche
    ](Cache-avalanche)

### What is redis?

Redis is an open source (BSD licensed), `in-memory` data structure store, used as a database, cache, and message broker.

The philosiphy is key-value match.

### Foundamental data types:

1. String
2. Hash
3. List
4. Set
5. ZSet(ordered set.)

**_ hint: key is always a string. _**

### Scenarios

1. Cached database.
2. Ranking and sorting.
3. Counter application (like, view).
4. Social media.
5. Message queue.
6. others.

### String type

#### Some commands

1. set value:

   1. set age 23 ex 10
      **_ 20 seconds expire _**

   2. setnx name test

      **_ atomic lock, can use for limited stocks. if key not exists return true, otherwise return false _**

   3. set age 25 xx

      **_ if key exists return true, otherwise return false. _**

2. get value:

   get age

3. batch set value:

   mset country china australia usa

4. batch get value:

   mget country city address

#### How to set expire time correctly?

For the `Highly frequent requested` data, the ex time should be longer, otherwise should be shorter to save the memory.

#### String type senarios:

1. Counter.

   Article No. is 001. Count how many times the article has been viewed.

   ```
   INCR article:001
   GET article:001
   ```

2. Integrated environment session sharing.

### Hash Type

string type matching table for field and value, which is very suitable for storing objects.

#### general commands:

1. hset key filed value
2. hset user:1 name james // suc return 1, else 0
3. hget user:1 name
4. hdel user:1 age
5. hset user:1 name james
   hset user:1 age 23
   hlen user:1 // return 2
6. hmset user:2 name james age 23 height 180
7. hmget user:2 name age height
8. hvals user:2 // get all values of a key
9. hgetall user:2 // get all fields and values of a key
10. hincrby user:2 age 1 // age ++
    hincrbyfloat user:2 age 2

#### senario:

1. shopping cart:

   ```
   //adding products into cart
   hmset cart:001 prod:001 1
   hmset cart:001 prod:002 2
   hmset cart:001 prod:003 3
   //select all prod in cart
   hgetall cart:001
   //increase prod count
   hincr cart:001 prod:001
   //check the amount of prod
   hlen cart:001
   ```

   for data persistentcy, store the data when expiring.

### List Type:

1. can use `lpush` + `rpop` or `lpop` + `rpush` to implement a general queue.
2. can use `lpush` + `lpop` or `rpush` + `rpop` to implement a general stack.
3. can use `blpop` or `brpop` to implement a blocking queue.

#### Senario:

When a user have a lot of orders:

```

hmset order:1 orderId 1 money 36.6 time 13-01-2021
hmset order:2 orderId 2 money 37.6 time 13-01-2021
hmset order:3 orderId 3 money 38.6 time 13-01-2021

//push all orders to the list
lpush user:1:order order:1 order:2 order:3

//create a new order hash
hmset order:4 orderId 4 money 39.6 time 13-01-2021

//push the new order to the list
lpush user:1:order order4

//inspect all the orders
lrange user:1:order 0 -1

```

### Set Type

Set is suitable for social media:

1. user tags.
2. topic tags.
3. mutual connection/friends.
4.

A set can be using to store a lot of unique values, which means no duplicate value.

A set support CRUD operations, union set,intersection set, subtraction set.

#### general commands:

1. exists user
2. sadd user a b c
3. smembers user // return all elements(orderless)
4. srem user a // delete element a
5. scard user // count the elements.

#### senarios:

1. draw a prize.

```
//push user id into the pool
sadd prize:1 user:001
sadd prize:1 user:002
sadd prize:1 user:003
sadd prize:1 user:004

//inspect how many users in the pool
smembers prize:1

//draw the user(s)
spop prize:1 2

```

2. endorse/like a article/twitt

(use zset is we need it in order)

```
sadd article:001 u001
sadd article:001 u002
sadd article:001 u003

scard article:001   //return 3
```

3. the mutual connections between two users

```
sinter conn:user:001 conn:user:002
```

4. list the people you might know

```
sdiff conn:user:001 conn:user:002
```

### Data consistency:

In highly concurrent scenarios, database is the weakest one for concurrent at most of the cases.

Therefore, you need to use redis as a buffer, so that the request can firstly access redis, instead of directly accessing databases such as Mysql. This can greatly ease the pressure on the database.

Redis cache data loading can be divided into two modes:

1. lazy loading.

如果删除了缓存 Redis，还没有来得及写库 MySQL，另一个线程就来读取，发现缓存为空，则去数据库中读取数据写入缓存，此时缓存中为脏数据。

If we deleted the data in redis, and did not write into db yet, while another process jump in and read the data from redis. There will be no data, so it is called dirty data in the memory.

If we wrote into db first, but the writing process broke down before deleting cached data, the data would not be consistent.

#### Solution 1:

1. delete cache.
2. write into db.
3. sleep a certain time, determined by requirements.
4. delete cache again.

To make sure after request, the dirty data can be removed.

#### Solution 2:

1. delete cache.
2. write into db
3. trigger async write message queue.
4. mq accept and delete cache again.

5. active loading.

#### write process:

1. delete cache
2. update db.
3. subscribe db binlog and analyse the data tag.
4. write the tag into mq.
5. consume mq.
6. get the data from db and update cache

#### read process:

1. read cache.
2. if empty, read db.
3. async write data tag into mq.
4. consume mq.
5. read db and update cache.

### Cache penetration:

It refers to querying a data that does not exist.

Because the cache is missing, it needs to bedatabaseQuery, if the data is not found, it will not be written to the cache.

This will cause the non-existent data to be queried every time the request is made, causing the cache to penetrate.

If too many queries at the same time, the db will crash.

**_ Solution _**:

1. Cache empty objects. Turn null into a value.

You can also use a simpler and more rude method. If the data returned by a query is empty (whether the data does not exist or the system is faulty), we still cache the empty result, but its expiration time will be short. The maximum is no more than five minutes.

2. Bloom filter

All the parameters that may be queried are stored in hash form, and are checked first in the control layer.

If they are not met, they are discarded. The most common is to use the Bloom filter to hash all possible data into a large enough bitmap. A certain non-existent data will be intercepted by this bitmap, thus avoiding the underlying storage system. Check the pressure.

There are two problems with caching empty objects:

First, the null value is cached, which means that more keys are stored in the cache layer, which requires more memory space (if it is an attack, the problem is more serious). A more effective method is to set a shorter time for this type of data. The expiration time is allowed to be automatically culled.

Second, the data of the cache layer and the storage layer may be inconsistent for a period of time, which may have a certain impact on the business. For example, if the expiration time is set to 5 minutes, if the data is added by the storage layer at this time, there will be inconsistency between the cache layer and the storage layer data. In this case, the message system or other means can be used to clear the space in the cache layer. Object.

### Hotspot invalid

We usually set an expiration time for the cache. After the expiration time, the database will be deleted directly by the cache, thus ensuring the real-time performance of the data to a certain extent.

However, for some hot data with very high requests, once the valid time has passed, there will be a large number of requests falling on the database at this moment, which may cause the database to crash.

**_ Solution _**

Mutex lock

We can use the lock mechanism that comes with the cache. When the first database query request is initiated, the data in the cache will be locked; at this time, other query requests that arrive at the cache will not be able to query the field, and thus will be blocked waiting; After a request completes the database query and caches the data update value, the lock is released; at this time, other blocked query requests can be directly retrieved from the cache.
When a hotspot data fails, only the first database query request is sent to the database, and all other query requests are blocked, thus protecting the database. However, due to the use of a mutex, other requests will block waiting and the throughput of the system will drop. This needs to be combined with actual business considerations to allow this.

### Cache avalanche

If the cache goes down for some reason, the massive query request that was originally blocked by the cache will flock to the database like a mad dog. At this point, if the database can’t withstand this huge pressure, it will collapse.

**_ Solution _**

1. You can cache data by category and add different cache times

2. At the same time of caching, add a random number to the cache time, so that all caches will not fail at the same time

3. For the problem of the redis service hanging up, we can realize the high availability master-slave architecture of redis, and implement the persistence of redis. When the redis is down, read the local cache data, and restore the redis service to load the persistent data
