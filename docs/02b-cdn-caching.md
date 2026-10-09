# CDN Caching: A Simple, Detailed Guide

[All guides](../README.md) | [Caching overview](02-caching.md) |
[Redis caching](02a-redis-caching.md) | [Browser caching](02c-browser-caching.md)

## How to use this guide

Start with the delivery example and HTTP headers. Then study deployments and
the interview questions. Advanced topics are grouped near the end.
Policies below are examples; verify your CDN's actual settings.

## 1. What is a CDN?

CDN means **Content Delivery Network**.
It is a network of servers that can deliver website content closer to users.

**Everyday example:** A company has one factory but many local warehouses.
Customers receive popular products from a nearby warehouse instead of waiting
for every delivery from the factory.

- Origin server = the factory.
- CDN edge server = a local warehouse.
- Cached response = a stored copy ready to deliver.

Example:

```text
User in Pune -> CDN edge -> origin server in another region
                   |
           Copy available?
                   |
           Return it from the edge.
```

A CDN may also provide routing and security features.
Using a CDN does not mean every response is cached.

## 2. Basic terms

| Term | Simple meaning |
| --- | --- |
| Origin | Your original application or content server |
| Edge | A CDN server serving user requests |
| Hit | An eligible saved copy is available |
| Miss | The edge must fetch from another cache or the origin |
| TTL | How long a copy can remain fresh |
| Cache key | How the CDN identifies which copy to return |
| Purge | Explicitly remove a cached copy |
| Revalidation | Ask whether a stored copy is still current |

## 3. How a CDN request works

1. The browser requests a URL.
2. The CDN receives the request.
3. It checks whether this request may use a shared cache.
4. It looks for the matching copy.
5. If a fresh copy exists, it returns it.
6. Otherwise, it fetches from the origin or another cache layer.
7. If allowed, it saves the response for later requests.

Different edge locations can hold different copies.
A request can be a hit in one city and a miss in another.

## 4. What should you cache?

| Content | Typical choice | Why |
| --- | --- | --- |
| Versioned JS, CSS, images | Long cache lifetime | Same files can be reused by many users |
| Public help pages | Cache with update rules | Content is public and changes less often |
| Public product data | Cache with a freshness limit | Many users request the same data |
| Personalized dashboard | Usually bypass shared caching | Different users receive different content |
| Balances or private statements | Avoid shared storage | Privacy and correctness matter more |

A GET request is not automatically safe to share.
`GET /api/account` can contain private information.

## 5. Cache keys: returning the right copy

Suppose two URLs return different prices:

```text
/products/123?currency=INR
/products/123?currency=USD
```

If the CDN ignores `currency`, it might return the wrong price.

A key may use the host, path, query parameters, and selected headers/cookies,
depending on the CDN configuration.

Ask:

- Does language change the response?
- Does currency change it?
- Do filters or page numbers change it?
- Does a cookie make it personalized?

**Rule:** If something changes the response, either account for it in the key or
do not share-cache that response.

Ignoring tracking parameters can improve reuse, but only if they do not change
what the origin returns.

## 6. HTTP headers explained with examples

HTTP headers tell caches how a response may be reused.

### A. Long-lived static asset

```http
Cache-Control: public, max-age=31536000, immutable
```

Meaning:

- `public`: shared caches may store it.
- `max-age`: it can remain fresh for the stated number of seconds.
- `immutable`: its content will not change while fresh.

Use this for a filename that changes when its content changes:

```text
/assets/app.8f3c2a.js
```

Do not change the contents of that URL later.

### B. Public API with different browser and CDN lifetimes

```http
Cache-Control: public, max-age=60, s-maxage=300
```

Meaning:

- Browser: fresh for 60 seconds.
- Shared cache/CDN: fresh for 300 seconds, if its configuration honors this header.

`s-maxage` means the shared-cache freshness rule.

### C. Sensitive response

```http
Cache-Control: no-store
```

Meaning: do not store this response in HTTP caches.
Also configure sensitive routes to bypass CDN caching.

### D. A response that must be checked before reuse

```http
Cache-Control: private, no-cache
ETag: "profile-12"
```

- `private`: shared caches must not store it.
- `no-cache`: it may be stored privately but must be validated before reuse.
- `ETag`: a label used to check whether the representation changed.

**Remember:** `no-cache` does not mean `no-store`.

## 7. Keeping content up to date

There are three common choices:

1. **TTL:** wait for the copy to expire.
2. **Purge:** remove the old copy explicitly.
3. **Versioned URL:** publish changed content at a new URL.

Example:

```text
Old release: app.111.js
New release: app.222.js
```

Versioned URLs are particularly useful for static assets.
Mutable HTML still needs a policy that lets users discover the new file.

### Why purging may not fix everything

The browser, service worker, application, or Redis might still hold old data.
Removing the CDN copy does not remove all these other copies.

Purge propagation also depends on the provider. Verify the relevant variants,
locations, and any requests that were already fetching old data.

## 8. Safe React deployment

1. Upload new versioned assets.
2. Verify they are available.
3. Publish HTML referencing them.
4. Refresh or purge mutable HTML when required.
5. Keep old assets for open tabs and rollback.
6. Monitor loading failures.

Why keep old assets?

An open tab may still request a lazy-loaded file from the previous release.
If you delete it immediately, that user's navigation can break.

## 9. Common problems

| Problem | Possible explanation | First check |
| --- | --- | --- |
| Stale content | Wrong TTL or another stale layer | Headers and content version |
| Low hit rate | Unique URLs, cookies, short TTL, bypass | Cache rules and key |
| Wrong language or currency | Key ignores a response difference | Key dimensions |
| High origin load after release | Too many entries purged together | Miss traffic |
| Missing JS chunk | Old asset removed or new asset not uploaded | Deployment order |
| Slow response despite a hit | Large download or slow user connection | Network timing and size |

Inspect a GET response:

```bash
curl -sS -D - -o /dev/null https://example.com/public/catalog
```

Look at `Cache-Control`, `Age`, `ETag`, and the provider's diagnostic headers.
Some providers use `X-Cache` or `CF-Cache-Status`; these names are not universal.

`Age` helps show how old an HTTP response is.
It does not tell you whether the origin generated it from old database data.

## 10. Detailed interview questions with simple answers

### Q1. How is CDN caching different from Redis?

The CDN caches eligible HTTP responses close to users.
Redis caches application data for backend services.

**Follow-up:** Can you use both?

Yes. A public response might be served entirely by the CDN; an origin request
that misses the CDN might still read data from Redis.

### Q2. How is CDN caching different from browser caching?

A browser cache helps one browser reuse previous responses.
A CDN can reuse eligible responses across many users.

### Q3. Can a CDN cache API responses?

Yes, if the response is safe to share and the key captures the differences.
JSON is a format; it does not determine cache safety.

**Follow-up:** Would you cache `/api/balance` publicly?

No. It is private and requires a carefully defined freshness policy.

### Q4. What happens on a cache miss?

The CDN fetches from another cache tier or the origin. This is generally slower
and adds upstream work.

**Follow-up:** What happens if many users miss together?

The origin can overload. Limit traffic and use supported request-sharing features.

### Q5. What is the difference between `max-age` and `s-maxage`?

`max-age` sets general freshness; `s-maxage` overrides that for shared caches.
Explain browser and CDN lifetimes separately.

### Q6. Why are users seeing old content after a purge?

Another layer might still hold it, a different variant might not have been purged,
or the origin might generate stale content again.

**Follow-up:** How would you investigate?

Trace one URL and content version from browser to edge to origin to source.

### Q7. Why do cookies reduce the cache hit rate?

Depending on configuration, they can cause cache bypass or create separate copies.
Do not ignore cookies that change the response just to improve hit rate.

### Q8. How can caching leak one user's data to another?

Private content may be cached as public, or the CDN may return a copy without
considering the necessary user context.

Prefer bypassing shared caching for private responses. If you deliberately serve
private data at the edge, authorization must apply to every request, including hits.

### Q9. Why use content-hashed filenames?

The URL changes when the file changes, so old files can be cached safely.
It also helps old tabs and new releases coexist.

### Q10. How would you remove public content urgently?

Update the origin, purge all relevant copies, and verify results.
Previously downloaded browser copies cannot simply be recalled.
Plan ahead if content requires reliable revocation.

### Q11. Can a CDN cache a 404?

Yes, depending on policy. A short cached "not found" response can reduce repeated
work. A long-lived cached 404 can hide an asset uploaded later.

### Q12. Does a high cache hit rate guarantee a fast website?

No. Large files, slow networks, and expensive frontend rendering can still be slow.
Measure user performance, not just edge statistics.

### Q13. What should you monitor?

Hit rate, response time, origin traffic, errors, purge behavior, and wrong/stale
content reports.

Also compare **request hit rate** with **byte hit rate**:
serving many tiny files from cache is different from avoiding large downloads.

### Q14. How do you choose a TTL?

Start with how quickly updates must become visible. Consider update frequency,
origin cost, and how reliably you can purge. Do not apply one TTL to every route.

### Q15. What is `Vary`?

It tells HTTP caches which request headers can select different responses.
For example, `Vary: Accept-Encoding` relates to compressed representations.
Check your CDN's support and key settings rather than assuming every variation
is handled automatically.

### Q16. Does a Next.js revalidation call clear the CDN?

Not necessarily. It depends on the framework version, hosting platform, and
integration. A separately managed CDN may need its own invalidation.

### Q17. Why can a signed URL be risky with caching?

If the CDN ignores the signature and does not verify access on a cache hit,
an unauthorized request might receive a previously cached private file.
Access checks and key rules must work together.

### Q18. When would you avoid CDN caching?

When data is private, changes must be visible immediately, or the response is
not reusable. You can still use a CDN for delivery/security without caching it.

## 11. Advanced ideas explained simply

### Stale-while-revalidate

Serve an allowed older copy while fetching an update in the background.
This reduces waiting, but users deliberately receive slightly old content.
Use a bounded stale window only where the product accepts it.

Example HTTP policy for public content:

```http
Cache-Control: public, max-age=60, stale-while-revalidate=30
```

Actual behavior depends on the cache's support and configuration.

### Stale-if-error

Serve an allowed old copy when the origin fails. Useful for some public content,
not automatically safe for balances, permissions, or revoked documents.

### Request collapsing

When many requests need the same missing copy, the CDN may share one upstream
fetch instead of making many. Check provider support.

### Origin shielding

An extra cache between the edge servers and your origin.
It can reduce repeated origin requests from different locations.

### Cache poisoning

A response affected by an input gets saved under a key that ignores that input.
Other users then receive the wrong response. Ensure the CDN and origin agree
about which host, path, headers, and parameters change the output.

### Multiple CDNs

Using more than one CDN can improve resilience, but adds routing, configuration,
and purge coordination work. Each provider has its own cached copies.

## 12. Practice exercise

Design delivery for a storefront with public product pages, React assets,
private carts, and urgent product updates.

Explain what is cached, the cache keys, browser/CDN TTLs, deployment order,
purging, origin protection, and monitoring.

## Interview answer in 30 seconds

> A CDN serves eligible content from edge servers closer to users.
> I would cache versioned assets aggressively and public responses according to
> their freshness needs. Private content normally bypasses shared caching.
> Correct cache keys, safe deployments, purging, and origin protection matter
> as much as the hit rate.

## References

- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching)
- [RFC 9111: HTTP caching standard](https://www.rfc-editor.org/rfc/rfc9111)
- [Cloudflare: cache keys](https://developers.cloudflare.com/cache/how-to/cache-keys/)
- [Cloudflare: cache diagnostics](https://developers.cloudflare.com/cache/concepts/cache-responses/)
- [CloudFront: expiration](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Expiration.html)
