# Redis Caching: A Simple, Detailed Guide

[All guides](../README.md) | [Caching overview](02-caching.md) |
[CDN caching](02b-cdn-caching.md) | [Browser caching](02c-browser-caching.md)

## How to use this guide

Read sections 1-8 first. They explain the everyday ideas.
Then practice the questions in section 9. Section 10 covers advanced topics.
The numbers and TTLs below are examples, not rules for every application.

## 1. What is Redis?

Redis is a fast data store. Applications often use it to keep copies of data
that would otherwise take longer to fetch or calculate.

**Everyday example:** You keep frequently used documents on your desk instead
of walking to the filing cabinet each time.

- Database = the filing cabinet containing the official records.
- Redis cache = copies on your desk for quick access.
- Application = the person deciding which copy to use.

Redis has uses beyond caching, but this guide focuses on caching.

### Example in an application

A product page needs a product's name, description, and price.
Without caching, the API might query the database for every visit.
With Redis, it can reuse a saved copy for a limited time.

```text
User -> API -> Redis
                 |
          Copy found? Return it.
                 |
          No copy? Read database.
                 |
          Save a copy in Redis.
                 |
          Return the result.
```

The browser usually talks to your API, not directly to Redis.

## 2. Words you should know

| Word | Simple meaning |
| --- | --- |
| Cache hit | The needed data is already in the cache |
| Cache miss | The needed data is not available in the cache |
| TTL | Time to live: how long an entry can remain usable before it expires |
| Stale data | An old copy that no longer matches the source |
| Invalidation | Removing or replacing a copy because the source changed |
| Eviction | Removing entries because Redis needs space |
| Cache key | The name used to find a saved value |
| Source of truth | The system containing the authoritative data |

**Example:** A TTL of 300 seconds means the entry expires after five minutes.
It does not mean Redis automatically fetches a fresh copy when those minutes end.
Your application must load it again or arrange a background refresh.

## 3. When should you use Redis caching?

Consider it when:

- The same data is requested repeatedly.
- The database query or calculation is expensive.
- Slightly old data is acceptable.
- You have measured a performance problem that caching can help.

Good examples include public product descriptions and expensive reports.

Be careful with balances, permissions, and available stock.
An old product description may be acceptable; an old balance used to approve a
payment may not be.

**Important:** A displayed balance snapshot and the balance used to authorize a
transaction are different requirements. Use the authoritative financial system
for critical decisions.

## 4. The simplest pattern: cache-aside

Cache-aside means the application checks the cache and fills it when needed.

### Read steps

1. Check that the user is allowed to access the data.
2. Build the cache key.
3. Look in Redis.
4. If found, return the saved value.
5. Otherwise, query the database.
6. Save the result in Redis with a TTL.
7. Return the result.

Example key:

```text
product:v2:tenant-42:product-123
```

Here, `v2` identifies the data format and `tenant-42` identifies the organization.
Including the organization helps avoid returning another organization's data.
Authorization is still required; a well-named key does not replace it.

Illustrative Redis commands:

```text
SET product:v2:tenant-42:product-123 '{"name":"Keyboard"}' EX 300
GET product:v2:tenant-42:product-123
DEL product:v2:tenant-42:product-123
```

- `SET ... EX 300` stores a value for 300 seconds.
- `GET` reads it.
- `DEL` removes it.

These commands demonstrate the idea, not a complete application.
Application code also needs timeouts, validation, logging, and failure handling.

## 5. What happens when data changes?

Suppose a product price changes from 1,000 to 900.
If Redis still contains 1,000, users may see the old price.

A common approach:

1. Commit the change to the database.
2. Remove the product from Redis.
3. The next read loads the new value.

**Why commit first?** If you remove the cache before the database update, a reader
can fetch the old database value and put it back into Redis.

### The surprising problem: an old reader can still put old data back

Even deleting after the database update has a race:

```text
A reads the old price from the database.
B saves the new price and deletes the cache.
A finishes later and puts the old price into the cache.
```

A **race** means the outcome depends on the order in which overlapping work finishes.

Choose a solution based on your requirement:

- If a short stale period is acceptable, TTL plus reliable invalidation may work.
- If users must immediately see their own update, return the saved data and use
  a fresh-read path when needed.
- For stricter needs, use carefully coordinated writes or version checks.
- For critical decisions, read the authoritative source.

Deleting twice with a delay can reduce some races, but does not guarantee correctness.

## 6. Common problems explained simply

### A. Many requests ask for the same missing item

This is a **cache stampede**.

Imagine 1,000 users open a popular product just after its cache entry expires.
All 1,000 requests might query the database.

Possible fixes:

- Let one request load the data while others wait briefly.
- Refresh popular data before it expires.
- Serve the previous copy briefly, only if stale data is acceptable.
- Limit how many database reads may happen at once.

Making different entries expire at slightly different times is called **TTL
jitter**. It helps many-entry expiry bursts, but not one very popular expired key.

### B. Repeated requests ask for something that does not exist

This is **cache penetration**.

Example: repeated requests for product `999999`, which is not in the database.

Possible fixes:

- Reject invalid identifiers early.
- Briefly cache a genuine "not found" result.
- Remove that negative entry if the product is later created.

Do not cache "database unavailable" as "product not found."
Those are different outcomes.

### C. Redis becomes unavailable

If every request immediately falls back to the database, the database may overload.

Plan for:

- Short Redis timeouts.
- Limited database fallback traffic.
- Visible errors when a safe fallback is unavailable.
- Monitoring and recovery.

Fallback is not always safe. If Redis holds essential session or permission state,
you need a specific failure policy rather than pretending it is an ordinary miss.

### D. Redis runs out of memory

Redis can remove entries or reject memory-growing writes, depending on its policy.

- **LRU:** approximately remove entries not used recently.
- **LFU:** approximately remove entries used least often.
- **No eviction:** do not remove entries automatically; affected writes may fail.

TTL and eviction are different. An entry can be removed under memory pressure
before its TTL ends.

## 7. Choosing a TTL and cache key

### TTL questions

Ask:

1. How old may this data be?
2. How often does it change?
3. How costly is a miss?
4. How much traffic can the database handle?

Do not choose five minutes simply because another project did.

### Key questions

Does the result change by:

- User or organization?
- Language or currency?
- Search filters or pagination?
- Data format version?

Include the relevant differences in the key.
Do not use raw access tokens as keys or log sensitive identifiers unnecessarily.
Avoid creating unlimited keys from arbitrary user input.

## 8. What should you measure?

| Measure | What it tells you |
| --- | --- |
| Hit rate | How often Redis avoids a source read |
| API response time | Whether users actually benefit |
| Redis response time | Whether cache access is becoming slow |
| Memory and evictions | Whether the cache is under pressure |
| Errors and timeouts | Whether Redis is reliable |
| Database load during misses | Whether fallback is safe |
| Stale-data reports | Whether freshness is acceptable |

A 99% hit rate is not automatically success. The remaining 1% may be extremely
slow, or cached data may be wrong.

## 9. Detailed interview questions with simple answers

### Q1. Why use Redis instead of a JavaScript `Map`?

A `Map` belongs to one application process. If you run ten API instances, each
has its own copy. Redis can provide a shared cache for those instances.

A local `Map` can be faster because there is no network call, but keeping all
instances updated becomes harder.

**Follow-up:** When would you use both?

For a small, short-lived local cache in front of Redis, if you can clearly explain
how much staleness is allowed.

### Q2. Explain a cache hit and miss using a product page.

A hit means Redis has the product, so the API returns that copy.
A miss means the API loads the product from the database and saves a copy.

**Follow-up:** What happens on the first visit after a restart?

The cache may be empty. The system must handle extra database traffic safely.

### Q3. How do you keep Redis and the database consistent?

First define what "consistent" means for this endpoint.
A common approach is database commit followed by cache invalidation.
For stricter freshness, consider version checks or authoritative reads.

**Follow-up:** Is deletion enough?

No. Explain the old-reader race from section 5.

### Q4. How do you avoid a cache stampede?

Allow one request to fetch a missing popular value, coordinate the others,
and limit database concurrency.

**Follow-up:** Does doing this inside one API process protect all instances?

No. Each process may still start a fetch. Cross-instance coordination needs
additional design.

### Q5. What happens if Redis fails?

Use a bounded fallback if the source can safely handle it. Set timeouts,
limit retries, log the failure, and avoid overwhelming the database.

**Follow-up:** Would you always return a cached or empty answer?

No. Old or empty data may be unsafe or misleading. Return an explicit failure
when correctness cannot be maintained.

### Q6. How do you choose an eviction policy?

Look at how data is accessed. LRU favors recently used data; LFU favors frequently
used data. Measure your workload and avoid mixing disposable entries with state
that must not be evicted.

### Q7. Would you cache a balance?

Possibly a clearly defined display snapshot, if the product allows it.
Do not use an old cached balance to authorize a financial transaction.

**Follow-up:** What would you ask first?

How fresh must the display be, and which system makes the financial decision?

### Q8. Why is Redis fast but not always fast enough?

It avoids many disk/database operations, but network calls, large values,
serialization, slow commands, and overloaded nodes still take time.

**Follow-up:** What is a hot key?

One key receiving a large share of requests. It can overload a node even when
average cluster usage looks low.

### Q9. What is the difference between expiration and invalidation?

Expiration happens when the TTL runs out.
Invalidation happens because the application knows the data should no longer be used.

### Q10. What is negative caching?

Briefly saving "this item does not exist" to avoid repeated database lookups.
Keep the TTL short enough for your requirements and invalidate on creation.

### Q11. What should happen after a user updates a profile?

Return the committed profile data, update the frontend's own state, and invalidate
the relevant backend cache entries. The frontend cache and Redis are separate layers.

### Q12. How do you cache search results?

Include normalized filters, sorting, page/cursor, and user/tenant scope when relevant.
Be careful: many filter combinations can create too many entries.

**Follow-up:** Why are lists harder to invalidate than individual records?

Changing one record can affect multiple search pages and filtered lists.

### Q13. How do you know Redis is worth adding?

Measure query cost, repeated reads, latency, and source load. Estimate the benefit
against operational complexity. A well-indexed database may already be sufficient.

### Q14. What should a load test cover?

Warm cache, empty cache, hot keys, expiry bursts, Redis failure, and recovery.
Check the database during misses, not just Redis throughput.

### Q15. What is cache avalanche?

A large set of entries expires together, or the cache fails entirely, causing
a sudden wave of source reads. Use staggered TTLs, limits, and gradual warming.

### Q16. How can invalidation events be made reliable?

Record the database update and an event to deliver using a reliable pattern such
as a transactional outbox. Retry delivery and make repeated invalidations safe.

An **outbox** is a database table recording messages that still need to be sent.
It reduces the risk of committing a change but losing its notification.
It does not, by itself, eliminate the old-reader race.

## 10. Advanced topics, still in plain English

### Replication

Another Redis node keeps a copy for availability. Copies can lag behind the main
node, so reading them can return older data. Failover may lose recent writes
when replication is asynchronous.

### Cluster

A Redis Cluster divides keys among multiple nodes to increase capacity.
It does not automatically solve hot keys. Some operations involving multiple
keys require those keys to be placed together.

### Sentinel

Sentinel monitors a non-clustered Redis setup and helps switch to a replica when
the main node fails. Monitoring/failover and distributing data are different jobs.

### Persistence

Redis can save snapshots or a log of changes to help recover data after restart.
For a rebuildable cache, this is a trade-off between recovery speed and extra cost.
Do not confuse cache persistence with a complete financial durability design.

### Pipelining

Send several commands without waiting after each one. This reduces network
round trips. It does not make the commands one atomic transaction.

### Locks

A Redis lock can let one worker refresh an item. Use an owner token and only
release your own lock. A worker may continue after its lock expires, so locks
alone do not guarantee correctness for payments or other critical work.

### Other cache patterns

| Pattern | Simple explanation | Main caution |
| --- | --- | --- |
| Read-through | A cache helper loads misses for the application | Still needs freshness rules |
| Write-through | Write path updates source and cache together | Two systems are not automatically one transaction |
| Write-behind | Save quickly, write to the source later | Must not lose accepted writes |
| Refresh-ahead | Refresh before popular data expires | Extra work and coordination |

## 11. Practice exercise

Design caching for a product catalog.

Explain the key, TTL, read steps, price-update steps, stampede prevention,
Redis outage behavior, and metrics. At checkout, verify price and stock using
the authoritative system.

## Interview answer in 30 seconds

> Redis is a shared, fast data store often used to avoid repeated database work.
> I would start with cache-aside, clear keys, and a TTL based on freshness needs.
> I would plan invalidation, concurrent read/write races, stampedes, and outages.
> For correctness-critical decisions, I would use the authoritative source.

## References

- [Redis eviction policies](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Redis distributed locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
