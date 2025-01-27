---
layout: post
title: Caching Implementation in Backend Applications
date: '2025-01-26 23:56:03 +0530'
categories: [Animal, Insect]
tags: [bee]
author: nitin
description: A comprehensive guide to caching in Django.
toc: true
comments: true
---

## ✅ Task List

---

- [x]  How to capture cache hit-miss ratio?
- [x]  How to perform cache invalidation in Django?
- [ ]  Are libraries like django-cachealot useful?
- [ ]  Are LRU Cache relevant in cache clusters? How to implement it in redis?
- [ ]  Are there alternative cache systems that allow us to use read-through/write-through, i.e. where the cache interacts directly with the database?
- [ ]  ORM for cache, has it already been done somewhere? Can we add adapters to it to connect to the orms for databases?
- [ ]  How to handle concurrency in caching? Exclusive locks that are also present in the cache, or maybe an attribute within the cache model?
- [ ]  How to add ttl jitters, to prevent thundering herd problem on the db?
- [ ]  How to determine max ttl values for an application? Cache hit miss ration must be high and we need to see how long we’ll be okay with serving stale data? This needs to be evaluated on case-by-case basis
- [ ]  How to perform cache priming? Upon startup, or a periodic worker?
- [ ]  How will we handle cache & database failures?

## 🧠 Brainstorm

---

## Approach

- In-memory caching is not valid because worker restarts in our web servers need to happen very frequently. And this won’t work for bigger loads. Distributed caching will need to be used.
- Data that should be cached: 20% of data is read 80% of the time. So caching 20% of the data will improve 80% of request latencies.
- Cache hit-miss ratio must be captured for different resources. Based on this, we can decide the cache expiration policy (TTL).
- TTL jitters must be used to prevent sudden surge of requests to the database.
- Cache priming might need to be used to hot-load the data on the cache.
- Choose between lru, lfu, mru, fifo, rr (random replacement) must be selected.
- Choose between cache-aside and read-through for cache reads and write-around, write-through, write-back/write-behind, write-invalidate and refresh ahead for cache writes by weighing out the pros and cons of each strategy detailed below.

### Cache Read Strategies

- **Cache-Aside**: Cache and Database are independent systems, they don’t interact with each other. While reading, the application attempts to read from the cache first and returns response if the data was present (cache-hit). If it wasn’t present (cache-miss), the application will read from the database, add the data to cache and then return response to users.
    - Pros: faster reads, reduced load on db, independent cache data models (flexibility with how application interacts with cache and db), cache provider and dbs can be switched out anytime.
    - Cons: If cache hit-miss ratio is less than 1, then the caching is doing more bad than good. Application code must handle cache invalidation, consistency between cache and db, concurrent requests to cache.
- **Read-through:** Applications read from the cache. In case of cache-hit, the cache will return the data to the application, and the application to the end-user. In case of cache-miss, the cache will read from the database, update the data by itself and then return to the application, and the application to the end-user.
    - Pros: Simpler application code.
    - Cons: Lack of flexibility, harder to maintain separate cache models, and cache and databases cannot be switched out independently, since in this case they talk to each other.
    

### Cache Write Strategies

- **Write-Through:** Application writes to the cache and the cache writes to the database.
    - Pros: Data present in the cache and the database is always consistent unless data is individually updated in the cache or the database. Even first-time reads will be fast since it will be present in the cache. Application code is more maintainable since it only interacts with the cache, and doesn’t need to handle cache invalidation, concurrent requests and consistency.
    - Cons: Writes will be slower since they need to happen on both the cache and the database. All data that is written to the database will be written to the cache, which can cause cache pollution. There might be other data that could benefit from being present in the cache, but can’t be put in there since it is polluted with other data. Cache and db are not independent providers, so they cannot be switched to another provider at a later point in time without considering whether the cache and db know how to interact with each other.
- **Write-Around:** Application writes to the database directly bypassing the cache. In order to ensure consistency, it will need to invalidate the cache for every write.
    - Pros: Write operations will be faster since writes happen only at the database, and only invalidation occurs in the cache, as opposed to writing in the cache also. Prevents cache pollution, as only data that is read will be filled in the cache. Cache and Databases are independent of each other, so application has flexibility in terms of deciding which cache and database systems to use.
    - Cons: First-time reads will have greater latency since cache is only invalidated but not updated. This means that in case data is updated in the database often, cache hit-miss ratio will be very less, making it a bad option for write-heavy workloads. Application has less maintainable code since it has to interact with both the cache and database, and also has to handle cache invalidation, concurrent requests and consistency between cache and database.
- **Write-Behind/Write-back:**  Application writes only to the cache and the cache system asynchronously updates the database. Main purpose is to reduce latency while writing into both cache and database.
    - Pros: Writes will be fast since updates are only made to the cache and not the database. We can benefit from batching, coalescing, rate limiting, and time-shifting techniques while making updates to the database, to reduce load on the database. Application code will be simpler since the cache system takes care of interactions with the database, so the application doesn’t have to handle cache invalidation and concurrent requests.
    - Cons: Not feasible for applications that require strong data integrity, ones that require ACID guarantees. Strong Consistency will not be maintained between the cache and the database, only eventual consistency is possible. Cache system failing before updating the database would cause data to be lost.
- **Refresh-Ahead:** When a read request on a cache key that is about to expire is made, the cache returns the results and asynchronously fetches from the database and reloads itself, while also updating the TTL.
    - Pros: Read latency caused by TTL expiry of cache keys will be prevented, especially when future data access patterns are predictable.
    - Cons: Application code might become more complex when the cache cannot do this by itself. Not suitable when future data access patterns are unpredictable, as it will cause unnecessary reloads from the database.

### Points to Remember during Implementation

- In case of write-through cache
    - TTLs will still have to be set so that unused data is not present in the cache.
- In case of write-around cache
    - Do not invalidate the cache first and then update the db. If another read api was called before the db was updated, it would encounter a cache-miss, read unchanged data from the db and repopulate it in the cache, leading to stale cache data.
- In case of write-behind cache
    - TTLs will need to be implemented carefully, must be sufficiently more than the time it takes to update the database to avoid losing data.
    - In case of database failures, care must be taken to implement retry mechanisms to ensure eventual consistency.
- In case of refresh-ahead cache
    - Application must ensure that the async reload from the database doesn’t happen after a write-through operation updates the cache, or else it will cause stale data to overwrite fresh data in the cache. It must check last_updated timestamp to avoid this scenario.

## Implementation

- Identify what data can be cached ~ approx 20% of data is read 80% of the time in most cases, so we need to identify that data. This can be done by adding a counter variable in the cache per database query and incrementing it whenever that query is performed on the database. We can then perform a load test with a mix of actions with weights, and then inspect the cache later to find out which data is more frequently read.
- How to implement cache-aside, read-through, write-through, write-around, write-back, and refresh-ahead strategies in Django? Configuring the cache system to integrate with db will make maintainability more difficult, since db configurations have to be provided in Java, db table mappings must be updated every time a table changes. Moreover, the data models in the cache and db cannot be independent.
    - All the caching strategies will be implemented by having the application interact with the cache and the db independently. Configuring the cache system to integrate with db will make maintainability more difficult, since db configurations have to be provided in Java, db table mappings must be updated every time a table changes. Moreover, the data models in the cache and db cannot be independent.
    - Exploring cache libraries might allow us to make minimal code changes thereby keeping the code maintainable, and automatically handle cache invalidations. It also eliminates the need for cache models and allows the orms for the db to be reused, while still providing flexibility to store a different query for the same model in the cache. However, each library (cachealot, cacheops, cache-machine) must be evaluated, and its reliability must be verified.
    - Cachealot - seems good and reliable. However it invalidates the entire table, and might have a low cache hit-miss ratio. Need to check if they allow changing ttls, or setting ttl jitters.
    - Cacheops - seems good, more maintained than cachealot and cachemachine. Need to test this out.
- How to implement cache orms in Django? - through cacheops itself, where django orm is reused.
- How to capture cache hit-miss ratio, to determine the ttl policy, in Django? Or can this be done using some redis tool itself or a third party extension for redis? Cacheops supports recording of this parmeter.
- How to implement ttl jitters in Django? - Need to check this, we might need to override the library’s methods to implement this ourselves.
- How to configure cache eviction policies like lru, lfu, etc.? - configured at a global level - devops to implement this
- How to implement cache-priming on startup in Django? We must make sure it doesn’t happen for on every container startup.  - Within setup command, a publish event can trigger its own consumer to prime the cache

### Django Cacheops Evaluation

- [x]  Ensure that there is a valid load test script for the application with weighted tests that closely simulates real-world load. Calculate a standard load test for the application
- [x]  Identify data that is read often and won’t change often. We can inspect into selectors and services and infer this. This is the data that can be cached.
    - [x]  Currently caching for all the selectors except the select related query
- [x]  After identifying potential candidates, since we are always going to be using cache-aside and write-around cache, we can proceed with including the queries that need to be cached in the cacheops settings.
- [x]  Review the load tests before and after caching, and ensure that the throughput of the application has actually increased.
    - [x]  With cache - for a simple single db read api running 1 worker 2 threads -  300 users, at 500 rps at 1s 95th %ile
    - [x]  Without cache - same api same runtime configuration - 70 users, 93.5 rps at 1 sec 95th %ile response times.
        
        ![Screenshot 2025-01-21 at 4.50.15 PM.png](Caching%20Implementation%20in%20Backend%20Applications%20184610ebcb89807ea25cdef2d9ac3d00/Screenshot_2025-01-21_at_4.50.15_PM.png)
        
- [x]  Perform a load test with weighted tests, and review the hit-miss ratio. If it is low, then there could be two possible reasons:
    - [x]  The application is caching data that is changed often so it keeps getting invalidated. Prevent caching of such data.
    - [x]  The TTL set on the data is too short so the cache expires often. Increase TTL for such data, but not too high to prevent the application from serving stale data.
- [x]  Test if invalidation is happening properly.
- [x]  Test if caches are modified mid-transaction - Doesn’t happen, unless we explicitly set something in cache
- [x]  What happens if redis went down and came back up again, we need it to not restore any values from the previous session since they might have been changed in the db - we need to change the redis config to remove volume mappings and to not persist anything —save “” —appendonly no
- [x]  Is there a way to force read from the db even when the cache is up? Yes, there is a method called nocache() for this.
- [x]  Test if concurrent requests are being handled properly
    - [x]  Could any of the read phenomenas happen?
        - [x]  Dirty Reads - since it doesn’t update mid-transaction, this won’t be possible
        - [x]  Non Repeatable Reads - yes, this is possible since if we read the same data twice within a transaction, if the cache was updated by another concurrent transaction, it would read different data. Perhaps there is a way to bypass the cache in this scenario? Since there are no isolation levels that we can set in a cache. It is equivalent to a read committed level always. Yes, the nocache() method will allow us to do this.
        - [x]  Phantom Reads - Again, this could happen in the cache, unless we explicitly bypass it for these queries. Yes, the nocache() method will allow us to do this.
        - [x]  Lost Updates - Select for update bypasses the cache and always hits the database, so this will work as expected. And whichever transaction completes first will update the cache with whatever value is present in the db as expected.
    - [x]  Could the orders of the cache writes change post db transaction? Don’t think so, this is probably handled by the library but better to check open and closed issues for this. Need to look into 7 potential issues on github.
- [x]  Test if consistency is being maintained
- [x]  Multiple databases support - db_agnostic → False does the job
- [ ]  Redis Cluster Support
- [ ]  Make a note of what queries can be cached and what operations invalidate it and what operations do not.
- [ ]  Settings like disabling cache without having to add and remove it from installed apps
- [ ]  Tenant based cache prefixing - is this needed? → Can be done using threadlocals in a gthread environment, need to figure something out for asyncio use cases
