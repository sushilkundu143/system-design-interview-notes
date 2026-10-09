# Browser Caching: A Simple, Detailed Guide

[All guides](../README.md) | [Caching overview](02-caching.md) |
[Redis caching](02a-redis-caching.md) | [CDN caching](02b-cdn-caching.md)

## How to use this guide

Start with HTTP caching and the header examples.
Then read the React, logout, and deployment sections.
Use the questions to practice explaining the differences between cache types.

## 1. What is browser caching?

The browser can keep copies of downloaded responses so it does not need to
download them again every time.

**Everyday example:** You save a map on your phone instead of downloading the
same map on every trip. You still need a way to know when the map changes.

Example:

```text
First visit:
Browser -> server -> download app.js -> save eligible copy

Later visit:
Browser -> reusable local copy -> avoid another download
```

This can reduce loading time, transferred bytes, and server requests.

## 2. Not all browser caches are the same

| Type | Simple explanation |
| --- | --- |
| HTTP cache | Browser-managed saved responses using HTTP rules |
| Service-worker cache | Responses explicitly saved by application code |
| React data cache | API data held by a library/application for UI reuse |
| Router cache | Framework-specific data used for navigation |
| `localStorage` / IndexedDB | Application-managed storage, not automatic HTTP caching |
| Back-forward cache | A saved page state used when navigating back or forward |

Clearing one does not necessarily clear the others.
This is why "I disabled browser caching but still see old data" can happen.

## 3. Freshness and validation

### Freshness: may I use this copy without asking?

```http
Cache-Control: max-age=60
```

The response can be considered fresh for 60 seconds, accounting for its age.
The browser may reuse a fresh response without a network request.

### Validation: has this saved copy changed?

After a copy becomes stale, the browser can ask the server whether it is still valid.
If unchanged, it reuses the saved body.

Terms:

- **Fresh:** usable under its current freshness policy.
- **Stale:** its freshness period has ended.
- **Validator:** a value used to check whether the saved representation changed.

Fresh does not necessarily mean "the database has not changed."
It means the cache policy permits reuse.

## 4. Headers with easy examples

### `no-cache`: ask before reusing

```http
Cache-Control: no-cache
```

The browser may save it, but must validate before reuse.

### `no-store`: do not save it in HTTP caches

```http
Cache-Control: no-store
```

Useful for sensitive responses where storage is inappropriate.
It does not automatically delete data that your application separately saved.

### `private`: do not share-cache it

```http
Cache-Control: private, no-cache
```

A private browser cache may store it and validate before reuse.
A shared cache, such as a CDN, must not store it under this policy.

`private` is not encryption and does not protect against malicious JavaScript.

### `immutable`: this version will not change while fresh

```http
Cache-Control: public, max-age=31536000, immutable
```

Use with versioned files such as `app.8f3c2a.js`.
Changed content must get a new URL.

### `must-revalidate`: do not reuse an expired copy without checking

It restricts reuse once stale. Do not confuse it with `no-cache`, which requires
validation before reuse even if a response might otherwise still be fresh.

## 5. ETag and 304, step by step

An **ETag** is a label representing a particular version of a response.

First response:

```http
HTTP/1.1 200 OK
Cache-Control: no-cache
ETag: "catalog-42"
```

The browser stores the body and the label.
On reuse, it asks:

```http
If-None-Match: "catalog-42"
```

If unchanged:

```http
HTTP/1.1 304 Not Modified
ETag: "catalog-42"
Cache-Control: no-cache
```

The browser reuses its stored body. A 304 does not contain a new resource body.
If changed, the server returns a 200 with the new body and ETag.

**Important:** Validation saves downloading the body, but still involves a
network request. A fresh local hit can avoid that request entirely.

`Last-Modified` is another validator, using a modification time.
When both conditions are sent, `If-None-Match` takes precedence.

## 6. What policy should each resource use?

| Resource | Example policy | Reason |
| --- | --- | --- |
| Hashed JS/CSS | Long `max-age` with `immutable` | Content changes get a new filename |
| Public mutable HTML | `no-cache` with an ETag | Check for new releases |
| Public product data | Short `max-age` | Allow limited staleness |
| Sensitive account API | `no-store` | Avoid HTTP storage of private data |

These are examples, not universal defaults.
Choose policies based on privacy and how quickly changes must become visible.

Never give a mutable `/app.js` a year-long lifetime unless you can safely change
its URL whenever the content changes.

## 7. Browser `fetch` cache options

```js
const response = await fetch("/api/account-summary", {
  cache: "no-store",
  credentials: "same-origin",
});
if (!response.ok) {
  throw new Error(`Request failed: ${response.status}`);
}
const summary = await response.json();
```

This example controls browser HTTP-cache behavior for this request.
It does not clear a React query cache, service-worker cache, or CDN.

| Option | Simple meaning |
| --- | --- |
| `default` | Follow normal HTTP cache rules |
| `no-store` | Do not use or populate HTTP cache |
| `no-cache` | Check a matching saved response before reuse |
| `reload` | Fetch from network and normally update HTTP cache |
| `force-cache` | Prefer a matching saved response, even if stale |
| `only-if-cached` | Use cache only; requires same-origin mode |

Server-side `fetch` in Next.js is a different runtime and may have additional,
version-specific framework caching rules.

## 8. React data caching

A query library may keep an API result in memory to avoid repeated UI fetches.
This cache is separate from the browser HTTP cache.

Example:

```text
React asks for profile.
Query cache has a reusable profile.
The UI uses it without calling fetch.
```

Even perfect HTTP headers cannot refresh data if the application never makes a
new request.

### After editing a profile

1. Wait for confirmed server success.
2. Use the committed result to update local state.
3. Invalidate affected detail and list queries.
4. Refetch where necessary.
5. Show errors and undo optimistic changes if the operation fails.

Query keys should distinguish users, organizations, and filters when those
change the result.

A library's `staleTime` is not an HTTP `max-age` header.

## 9. Service workers and offline caching

For a full walkthrough and interview practice, read the separate
[Service Worker guide](11-service-workers.md).

A service worker is application code that can intercept requests.
It can return responses saved in **Cache Storage**.

Common choices:

- **Cache-first:** return a saved copy; fetch only on a miss.
- **Network-first:** try the network; use a saved copy if appropriate.
- **Stale-while-revalidate:** show a saved copy and refresh in the background.
- **Network-only:** always use the network.

Cache Storage does not automatically enforce HTTP expiration for your code.
Your service-worker logic must define updates and cleanup.

**Banking example:** Offline public help may be useful. Offline account data is a
different privacy/freshness decision; do not save it casually.

### Why updates become tricky

An old worker can still control an open tab while a new worker is waiting.
If new code deletes files required by the old tab, navigation can fail.

Plan how workers activate, how caches change, and how users are notified.
Forcing immediate activation is not always safe.

## 10. Logout and account switching

Imagine user A logs out and user B logs in on the same browser.
The UI must not show A's saved data.

Consider:

1. Invalidate the server session.
2. Clear user-specific React/query state.
3. Cancel old requests so their responses cannot refill the new user's state.
4. Remove deliberately persisted private values.
5. Clear any user-specific service-worker entries.
6. Coordinate logout across tabs where required.
7. Recheck authentication when a previously saved page is restored.

The server must enforce access for every private request.
Cache cleanup alone is not authorization.

Do not assume `no-store` prevents every browser from restoring a page's state.
Browser behavior changes; handle restoration explicitly where needed.

## 11. Deployments and stale JavaScript

Common failure:

```text
An old tab expects page.111.js.
A deployment deletes it and uploads page.222.js.
The old tab later requests page.111.js.
That request fails.
```

Prevent this by:

- Uploading new assets before publishing new HTML.
- Using content-versioned filenames.
- Keeping old assets long enough for active tabs and rollback.
- Coordinating service-worker updates.
- Monitoring missing chunks.

An unlimited automatic reload loop is not a safe recovery strategy.

## 12. How to debug old content

1. Open DevTools and find the exact request.
2. Inspect `Cache-Control`, `ETag`, `Age`, and timings.
3. Check whether it came from memory, disk, network, or a service worker.
4. Temporarily disable HTTP cache to isolate that layer.
5. Inspect service-worker caches separately.
6. Inspect React/query/router state.
7. If the network still returns old data, check CDN and origin caches.

Do not only test a clean browser profile. Existing tabs and returning users
often reveal deployment and caching problems.

DevTools cache settings are debugging tools, not production freshness guarantees.

## 13. Detailed interview questions with simple answers

### Q1. What is the difference between `no-cache` and `no-store`?

`no-cache` allows storage but requires checking before reuse.
`no-store` instructs HTTP caches not to save the response.

**Follow-up:** Which would you choose for sensitive account details?

Usually `no-store`, alongside policies preventing separate application storage.

### Q2. What happens when a browser receives a 304?

It reuses the body it already has, while updating applicable response metadata.
The server has confirmed that the saved representation can still be used.

### Q3. Why can a 304 still take noticeable time?

It still needs a network round trip and server work.
It saves response-body transfer, not all request latency.

### Q4. Does an ETag force every request to reach the server?

No. A fresh response can be reused without validation.
The ETag helps when the cache policy or request requires a check.

### Q5. Why is data still old after using `cache: "no-store"`?

React state, a query cache, a service worker, a CDN, or the origin may still provide
old data. Identify which layer supplied the value.

### Q6. What is the difference between memory and disk cache?

They describe where the browser stores data.
Whether the response is reusable depends on cache rules, not just storage location.

### Q7. Is `localStorage` a browser HTTP cache?

No. It is application-managed storage with no automatic HTTP validation or TTL.
Users can modify it, so the server must not trust it for authorization.

### Q8. How should a profile edit update the screen?

Use the committed response, update/invalidate relevant query state, and refetch
when needed. Handle failure visibly.

**Follow-up:** Is invalidating Redis enough?

No. The frontend may still hold a separate copy.

### Q9. How do you prevent user A's data appearing for user B?

Scope keys by identity, clear private state on logout, cancel old requests,
remove persisted private data, and enforce server authorization.

### Q10. What is the back-forward cache?

A browser mechanism that can restore a whole suspended page, including state.
It is different from saved HTTP responses.

**Follow-up:** Why does it matter after logout?

A restored page might contain old UI state. Handle restoration and authentication;
do not rely on HTTP headers alone.

### Q11. Why use hashed filenames?

Changed content gets a new URL. Unchanged content stays reusable for a long time.
This is more reliable than adding a new timestamp to every request.

### Q12. Why can a service worker serve old files after deployment?

Its code may choose a saved response, or an old worker may still control the tab.
Inspect worker lifecycle and cache versions.

### Q13. What is stale-while-revalidate?

Return an allowed older copy quickly while refreshing it in the background.
It trades freshness for less waiting.

**Follow-up:** Is the HTTP directive the same as a service-worker strategy?

No. They have similar goals but different implementations and controls.

### Q14. Should you cache API data in a banking app?

Start with privacy and freshness requirements. Public help is different from
balances. Critical financial decisions must use authoritative data.

### Q15. How do browser and CDN lifetimes interact?

Browsers use their freshness policy; shared caches can use `s-maxage`.
HTTP age accounting prevents an already-aged response from simply restarting
its entire original lifetime when received.

The age of underlying database data is still a separate matter.

### Q16. How would you test a caching change?

Test first visits, repeat visits, expiry, unchanged/changed validators, old tabs,
new deployments, logout, account switching, service workers, and offline recovery.
Check correctness as well as speed.

### Q17. Does `private` mean data is secure?

It tells shared HTTP caches not to store the response.
It is not encryption, XSS prevention, or application-storage cleanup.

### Q18. What does offline mode need beyond saving files?

A clear offline indicator, safe data selection, update rules, and recovery.
Queued actions must not look like completed server transactions.

### Q19. How is browser `fetch` different from Next.js server `fetch`?

Browser `fetch` follows browser HTTP-cache semantics.
Next.js server behavior can add framework caches and revalidation rules.
Always identify the runtime and version.

### Q20. When should you avoid aggressive browser caching?

When data must change quickly, storage is inappropriate, or old copies could
mislead users. Cache static assets and private data with different policies.

## 14. Advanced terms made simple

- **`Vary`:** which request headers can select different response copies.
- **Weak ETag:** a validator for equivalent content; not suitable for every use
  requiring exact byte identity.
- **`Age`:** HTTP-response age information, often supplied by a shared cache.
- **`Clear-Site-Data`:** a response header that can clear categories of site data
  in supporting browsers; it has broad effects and does not replace logout logic.
- **Router cache:** navigation data held by a framework; behavior varies by version.

## 15. Practice exercise

Design caching for a React banking dashboard with static assets, account APIs,
profile editing, logout, and optional offline public help.

Explain each cache separately, its freshness policy, update behavior, account
switching, deployment safety, and your debugging plan.

## Interview answer in 30 seconds

> Browser HTTP caching reuses downloaded responses according to freshness and
> validation rules. I would cache versioned assets for a long time, validate
> mutable entry pages, and avoid storing sensitive account responses.
> React caches, service workers, and restored page state are separate concerns,
> especially during updates, logout, and deployments.

## References

- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching)
- [MDN: Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [MDN: Request.cache](https://developer.mozilla.org/en-US/docs/Web/API/Request/cache)
- [MDN: Cache API](https://developer.mozilla.org/en-US/docs/Web/API/Cache)
- [web.dev: back-forward cache](https://web.dev/articles/bfcache)
- [RFC 9111: HTTP caching standard](https://www.rfc-editor.org/rfc/rfc9111)
