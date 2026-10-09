# CDN, Redis, and Browser Caching

## Separate deep-dive guides

Use this document as the cross-layer overview. Each topic now has its own
detailed guide with implementation examples, failure scenarios, and interview
questions with answer guidance:

1. [Redis application caching](02a-redis-caching.md): backend caching patterns,
   consistency races, stampede prevention, memory management, clustering, and recovery.
2. [CDN caching](02b-cdn-caching.md): edge behavior, cache keys, HTTP policies,
   invalidation, deployment safety, and origin protection.
3. [Browser caching](02c-browser-caching.md): freshness and validation, browser
   storage distinctions, service workers, React data caching, and logout safety.

Recommended order: browser caching, CDN caching, then Redis. Finally, return to
this overview to explain how the layers interact.

## 1. Mental model

A cache stores a reusable copy of data or a response. It trades freshness and
invalidation complexity for lower latency and less repeated work.

| Layer | Location | Common contents | Main benefit |
| --- | --- | --- | --- |
| Browser HTTP cache | User's device | Images, scripts, styles, HTTP responses | Avoid repeated network transfers |
| CDN cache | Distributed edge servers | Public assets, pages, explicitly cacheable API responses | Reduce network distance and origin load |
| Redis application cache | Backend infrastructure | Query results, computed objects | Avoid repeated database/computation work |

A cache **hit** finds a usable entry; a **miss** requires fetching or computing it.
An existing entry can still be stale, unusable for this request, or require
validation.

## 2. CDN caching

A Content Delivery Network has geographically distributed servers. A nearby
edge server can return cached content instead of every request reaching the
origin application.

```text
User -> Nearby CDN edge
             |
        Cache miss
             |
       Origin server
```

### Request lifecycle

1. The edge computes a cache key.
2. It checks freshness and eligibility.
3. On a usable hit, it serves the stored response.
4. On a miss, it requests the origin response.
5. It stores the response only if headers and CDN policy permit it.

The cache key may include path, query parameters, selected headers, and other
configured fields. An incorrect key can serve the wrong content.

### Public versus private content

Public product information may be suitable for shared caching. Account balances,
statements, and other personalized banking data must not leak through a shared
cache.

- `public`: permits shared caching, subject to HTTP rules and CDN configuration.
- `private`: permits a private cache such as the browser, but not a shared cache.
- `no-store`: instructs caches not to store the response.
- `no-cache`: allows storage but requires validation before reuse.

For sensitive authenticated responses, use an intentional policy such as
`Cache-Control: no-store` and verify the CDN does not override it.
`private` alone is not the same as forbidding storage.

### Example CDN policy

```http
Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=30
```

- Browser freshness: 60 seconds.
- Shared-cache freshness: 300 seconds because `s-maxage` overrides `max-age`
  for shared caches.
- A supporting cache can serve stale content within the additional 30-second
  window while validating in the background.

The directives work only where the relevant cache supports and respects them.

## 3. Browser HTTP caching

The browser stores eligible responses in memory or on disk.

```text
First visit: Browser -> Network -> app.abc123.js -> Local cache
Later visit: Browser -> Fresh local app.abc123.js
```

### Freshness versus validation

A fresh response can be reused without a network request. A stale response may
need a conditional request:

```http
If-None-Match: "product-version-7"
```

The server compares this with the current representation:

- Unchanged: `304 Not Modified`; the browser reuses the stored body.
- Changed: `200 OK` with the new representation.

An ETag is a validator, not a timer. Cache-Control determines freshness.

### Versioned assets

Content-hashed assets can use long-lived caching:

```http
Cache-Control: public, max-age=31536000, immutable
```

When the asset changes, its URL changes, for example from `app.abc123.js` to
`app.def456.js`. Apply this policy to immutable versioned resources, not mutable
HTML or user-specific API data.

### What browser HTTP caching is not

- `localStorage` and `sessionStorage` are explicit application-managed storage.
- Service workers can implement their own Cache API strategy.
- The back-forward cache can preserve a page snapshot for history navigation.
- A frontend query library may maintain a separate in-memory data cache.

These mechanisms can produce different behavior from HTTP caching. Do not assume
clearing one clears all others.

## 4. Redis application caching

Redis is a fast, primarily in-memory data store. It can cache backend data and
also support other features such as counters and coordination. Persistence and
replication depend on configuration; Redis is not simply "always temporary."

### Cache-aside reads

The application checks Redis, then queries the database on a miss:

```js
async function getProduct(id) {
  const key = `product:v1:${id}`;
  const cached = await redis.get(key);

  if (cached !== null) {
    return JSON.parse(cached);
  }

  const product = await database.getProduct(id);
  await redis.set(key, JSON.stringify(product), { EX: 60 });
  return product;
}
```

This example assumes the Redis client supports the shown options and the
database returns a valid product. Production code must handle missing products,
dependency failures, serialization, observability, and concurrency explicitly.

### Updating the database and cache

The database price changes from 1,000 to 1,200. Redis does not discover that change
automatically.

Option A: invalidate after the database commit.

```js
async function updatePrice(id, price) {
  await database.updateProductPrice(id, price);
  await redis.del(`product:v1:${id}`);
}
```

The next read misses Redis, loads 1,200, and repopulates the cache.

Option B: update Redis after the database commit using the complete updated
object:

```js
async function updatePrice(id, price) {
  const product = await database.updateProductPrice(id, price);
  await redis.set(
    `product:v1:${id}`,
    JSON.stringify(product),
    { EX: 60 }
  );
}
```

Both examples assume the database operation has committed before it returns.
Neither makes the database and Redis update atomic.

If Redis fails after the database commits, the database change has already
happened. Log and monitor the failure and use a durable repair/retry mechanism
where required. Avoid blindly retrying non-idempotent database operations.

### The stale-repopulation race

Even successful invalidation can race with another reader:

```text
Reader A: Misses cache; reads old database value
Writer B: Commits new value; deletes cache
Reader A: Stores its old value after the deletion
```

Possible mitigations include version-aware cache writes with atomic comparisons,
coordinated access, durable change events with repair, and bounded TTLs. The right
choice depends on consistency requirements. A database transaction alone does
not coordinate Redis.

For correctness-critical decisions, such as final payment amounts or balances,
validate against the authoritative data source rather than trusting an
eventually consistent cache.

## 5. Cache strategies

| Strategy | Behavior | Trade-off |
| --- | --- | --- |
| Cache-aside | Application fills cache on read misses | Simple; first miss costs more |
| Read-through | Cache abstraction loads missing values | Centralizes reads; depends on implementation |
| Write-through | Write path synchronously updates the cache-backed storage flow | Adds write latency and coordination concerns |
| Write-behind | Cache accepts writes, persists later | Lower write latency; introduces durability and ordering risks |
| Stale-while-revalidate | Serve old value while refreshing | Fast reads; deliberately allows bounded staleness |

Do not describe two independent writes to a database and Redis as automatically
consistent "write-through." Explain ordering, failure handling, and recovery.

## 6. TTL, invalidation, and eviction

- **TTL:** Time until an entry expires.
- **Invalidation:** Explicitly mark/remove an entry because it is no longer valid.
- **Eviction:** Remove entries to make room, according to memory policy.

TTL is a safety bound, not a guarantee of immediate freshness. In Redis, expired
keys are treated as unavailable when accessed, while physical cleanup uses
expiration mechanisms.

Choose TTL according to business freshness needs. Add bounded random jitter to
avoid many entries expiring at exactly the same moment.

## 7. Common failure modes

### Cache stampede

Many requests miss the same popular key and simultaneously query the database.

Mitigations:

- Coalesce concurrent refreshes ("single flight").
- Use carefully implemented per-key refresh coordination.
- Serve stale data during refresh when business requirements allow it.
- Add TTL jitter and refresh popular keys proactively.
- Bound database concurrency even if the cache is unavailable.

### Cache penetration

Repeated requests for nonexistent objects keep missing the cache and hitting
the database. Short-lived negative caching may help, but must not incorrectly
hide newly created objects or authorization differences.

### Cache outage

Blindly falling back to the database can turn a cache outage into a database
outage. Plan bounded fallback, load shedding, timeouts, and alerts.

### Incorrect cache keys

Include the dimensions that affect the result: tenant, permissions where
appropriate, locale, filters, pagination, and schema version.
Never reuse one user's private response for another user.

## 8. Multiple layers and stale ISR pages

A possible request path is:

```text
Browser -> CDN -> Next.js route cache -> Product API -> Redis -> Database
```

Not every application has all these layers.

Changing the database does not automatically invalidate them all:

1. Commit the product update.
2. Invalidate/refresh Redis.
3. Revalidate affected Next.js data and routes.
4. Purge or expire CDN responses when separately required.
5. Trigger a client refresh or notify open clients if live updates are required.

The correct order and guarantees depend on the deployment. If page regeneration
reads stale Redis data, it can generate a new page containing the old price.

### Troubleshooting checklist

- Does the database contain the new value?
- Does the API return the new value when inspected directly?
- Is Redis storing an old object?
- Did route regeneration succeed?
- Is the CDN serving an older response? Inspect `Age` and provider cache headers.
- Is the browser reusing a response or a frontend-library cache?
- Is a service worker involved?
- Was the page already open without a refresh or subscription?

## 9. Metrics and interview questions

Measure hit ratio per layer, miss latency, origin load, memory usage, evictions,
refresh failures, and business-visible staleness. A high hit ratio alone does
not prove correctness.

**What is the difference between no-cache and no-store?**

No-cache allows storage but requires validation before reuse. No-store forbids
storage by caches.

**Does deleting Redis update an open React page?**

No. The browser needs another fetch, refresh, polling cycle, or push notification.

**When should you avoid caching?**

When the freshness/security requirements cannot be met safely, or when cache
complexity outweighs savings.

## 10. Interview summary

> Browser caching avoids repeated downloads, CDN caching serves shared content
> near users, and Redis caching reduces backend data work. I design cache keys,
> freshness policies, invalidation, and outage behavior together, and validate
> correctness-critical actions against authoritative data.

## References

- [MDN HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching)
- [MDN Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [Redis SET](https://redis.io/docs/latest/commands/set/)
- [Redis key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
