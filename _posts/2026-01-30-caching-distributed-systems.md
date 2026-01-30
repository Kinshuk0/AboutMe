---
layout: post
title: "Caching in Distributed Systems: More Than Just Redis"
date: 2026-01-30
tags: [caching, distributed-systems, redis, backend, system-design]
---

Caching seems simple until it isn't. Put frequently accessed data closer to where it's needed, reduce database hits, make things faster. But in distributed systems? That's where it gets interesting.

## The Cache Invalidation Problem

There's a famous quote: "There are only two hard things in Computer Science: cache invalidation and naming things."

I used to think this was a joke. Then I spent a week debugging why some users saw stale product prices while others didn't. Not a joke anymore.

## Caching Strategies

There are a few patterns you'll see everywhere:

**1. Cache-Aside (Lazy Loading)**

```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if user is None:
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        cache.set(f"user:{user_id}", user, ttl=3600)
    return user
```

The app checks cache first, falls back to DB, then populates cache. Simple and flexible, but you can have cache misses on first request.

**2. Write-Through**

```python
def update_user(user_id, data):
    db.update("UPDATE users SET ... WHERE id = ?", data, user_id)
    cache.set(f"user:{user_id}", data, ttl=3600)
```

Write to DB and cache at the same time. Data is always fresh in cache, but writes are slower.

**3. Write-Behind (Write-Back)**

Write to cache immediately, async write to DB later. Fast writes but risky - what if the cache crashes before DB sync?

**4. Read-Through**

Cache sits in front of DB. App only talks to cache, cache handles DB fetches. Cleaner separation but less control.

## The Multi-Node Problem

Here's where distributed systems make things fun. You have 5 app servers, each could have its own local cache. User updates their profile on Server A, but their next request hits Server B with stale cache.

Options:

**Shared Cache (Redis/Memcached)**

Everyone uses the same remote cache. Slower than local memory but consistent.

```
[App Server 1] ─┐
[App Server 2] ─┼─► [Redis Cluster] ─► [Database]
[App Server 3] ─┘
```

**Local + Shared (Two-Level)**

Hot data in local memory, everything else in Redis. But now you have two caches to invalidate.

**Cache Invalidation via Pub/Sub**

When data changes, broadcast to all nodes to invalidate their local cache.

```python
def update_user(user_id, data):
    db.update(...)
    redis.publish("cache_invalidate", f"user:{user_id}")

def cache_listener():
    for message in redis.subscribe("cache_invalidate"):
        local_cache.delete(message)
```

## Cache Eviction Policies

When cache is full, something has to go:

- **LRU (Least Recently Used)** - evict what hasn't been accessed lately. Usually the right choice.
- **LFU (Least Frequently Used)** - evict what's accessed least often. Good for skewed access patterns.
- **FIFO** - evict oldest entries. Simple but often not optimal.
- **TTL-based** - everything expires after a set time. Prevents staleness but doesn't adapt to access patterns.

Redis default is a variation of LRU. For most cases, don't overthink it.

## Thundering Herd

Cache expires. 1000 requests hit at the same time. All see cache miss. All query DB. DB dies.

Solutions:

**1. Cache Locking**

Only one request fetches from DB, others wait.

```python
def get_user_safe(user_id):
    user = cache.get(f"user:{user_id}")
    if user is None:
        lock = cache.acquire_lock(f"lock:user:{user_id}")
        if lock:
            user = db.query(...)
            cache.set(f"user:{user_id}", user)
            lock.release()
        else:
            time.sleep(0.1)  # wait for other request
            return get_user_safe(user_id)
    return user
```

**2. Background Refresh**

Refresh cache before it expires. No one ever sees a miss.

**3. Stale-While-Revalidate**

Serve stale data immediately, refresh in background.

## Consistency vs Performance

This is the real tradeoff. Strong consistency means always fresh data but more latency and complexity. Eventually consistent means faster but users might see stale data for a bit.

For most apps? Eventually consistent is fine. User profile updated 2 seconds late won't break anything.

For financial transactions? Yeah, you probably want consistency.

## When NOT to Cache

Not everything should be cached:

- **Highly personalized data** - low hit rate, not worth the memory
- **Rapidly changing data** - constant invalidation overhead
- **Large objects** - takes up cache space, probably not accessed enough
- **Write-heavy workloads** - invalidation cost outweighs read savings

## My Go-To Stack

For most projects:

- **Redis** for shared cache (it's battle-tested)
- **TTL-based expiration** for simplicity
- **Cache-aside pattern** for flexibility
- **Pub/sub invalidation** when I need freshness

Start simple, add complexity when you need it.

## Quick Checklist

Before adding caching to your system:

1. Is this actually a bottleneck? Profile first.
2. What's the acceptable staleness? (seconds? minutes? hours?)
3. How will you invalidate? (TTL? event-based? manual?)
4. What happens when cache is unavailable?
5. How will you monitor hit rate?

## Wrapping Up

Caching isn't just slapping Redis in front of everything. It's about understanding your access patterns, consistency requirements, and failure modes. Get it right and your system flies. Get it wrong and you're debugging phantom data issues at 2am.

Start with cache-aside and TTL. Add complexity only when profiling shows you need it.

---

*Building something that needs caching? Let's chat - [007kinshuk@gmail.com](mailto:007kinshuk@gmail.com)*
