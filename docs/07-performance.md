# Web and React Performance: Interview Guide

## 1. Define performance as user experience

Performance is not only a small JavaScript bundle or a high Lighthouse score.
Users need useful content quickly, responsive interactions, stable layouts, and
reliable completion of tasks.

Measure the full path:

```text
Navigation -> Network/server -> HTML/assets -> Parse/execute
           -> Render -> Interaction -> API -> Updated UI
```

Choose goals for supported devices, networks, locations, and critical journeys.
An office laptop on fast Wi-Fi is not representative of every banking customer.

## 2. Core Web Vitals

| Metric | Measures | Good threshold |
| --- | --- | --- |
| LCP: Largest Contentful Paint | When the largest eligible visible content renders | At most 2.5 seconds |
| INP: Interaction to Next Paint | Overall responsiveness of user interactions | At most 200 milliseconds |
| CLS: Cumulative Layout Shift | Unexpected visual movement | At most 0.1 |

Evaluate these at the **75th percentile**, separating mobile and desktop where
appropriate. These are current Web Vitals targets, not guarantees of overall
product quality. INP replaced FID as a Core Web Vital in 2024.

Other useful measurements:

- TTFB: Time to First Byte.
- FCP: First Contentful Paint.
- API and task completion latency at p95/p99.
- JavaScript transfer, parse, and execution cost.
- Long tasks, memory growth, and error rates.

Time to first byte includes more than application processing. A fast TTFB can
still lead to a slow page if rendering waits for large assets or scripts.

## 3. Lab versus field data

**Lab tests** run under controlled conditions. They are useful for diagnosis and
repeatable pre-release comparisons.

**Field data** measures real users. It captures actual device/network differences,
cache states, third-party effects, and interaction patterns.

Use:

- Browser performance traces for the main-thread timeline.
- React Profiler for component render/commit behavior.
- Lighthouse and WebPageTest for reproducible page diagnostics.
- Real User Monitoring for deployed user experience.
- Backend traces for dependencies and API latency.

Total Blocking Time can help diagnose lab responsiveness but is not the same
metric as real-user INP. A lab run with no meaningful interactions does not
establish that INP is good.

## 4. A practical investigation sequence

1. Reproduce the slow journey on a representative device/network.
2. Determine whether time is spent in network, server, JavaScript, or rendering.
3. Inspect request waterfalls, long tasks, and component commits.
4. Change the most important bottleneck.
5. Re-measure under comparable conditions.
6. Confirm the improvement in production without functional regressions.

Avoid optimizing a cheap component while users wait for a slow API.

## 5. Network and asset optimization

### JavaScript

- Split by route and substantial optional features.
- Load a heavy editor/chart only when needed.
- Remove unused dependencies and duplicate versions.
- Prefer tree-shakeable imports when the package supports them.
- Measure compressed transfer size and execution cost, not just source size.

Lazy loading adds another request and loading state. Do not split every tiny
component or delay above-the-fold content unnecessarily.

### Images

- Serve appropriate dimensions and modern formats when supported.
- Use responsive sources rather than sending desktop-sized images to phones.
- Reserve width/height or aspect ratio to prevent layout shifts.
- Do not lazy-load the primary LCP image.
- Give genuinely critical images appropriate loading priority.

### Fonts and CSS

- Limit font families, weights, and character sets.
- Use a suitable font-display policy.
- Reduce render-blocking resources.
- Keep critical styles small and avoid loading unrelated feature styles.
- Account for font changes that shift layout.

### Delivery

Use compression, versioned-asset caching, and CDNs where appropriate. Preload
only resources that are genuinely important; excess preloads compete with
critical work.

See the [caching guide](02-caching.md) for freshness and invalidation trade-offs.

## 6. Improve React rendering

### State locality and subscriptions

Keep frequently changing state near its users. Typing in a search box should
not force every unrelated feature to update.

Use narrow store selectors and separate unrelated Context values. Do not
duplicate derived state through synchronization effects.

### Memoization

`memo`, `useMemo`, and `useCallback` can reduce unnecessary work when identity
or computation cost matters.

They are not universal speed switches:

- A new object prop can defeat a memoized child's comparison.
- Memoization has its own cost.
- Context and local-state changes can still update a memoized component.
- Incorrect dependency lists can create stale behavior.

Profile before and after. Modern compiler setups may automate some memoization.

### Large collections

For long transaction histories, consider server pagination and virtualization.
Bound DOM size, but preserve keyboard focus, accessible content, and stable keys.

### Expensive computation

Move suitable CPU-heavy work into a Web Worker or process it in bounded chunks.
Workers cannot directly manipulate the DOM and introduce serialization and
coordination overhead.

Use transitions or deferred values for nonurgent React updates when appropriate.
They can improve responsiveness; they do not eliminate the computation or move
it automatically off the main thread.

## 7. API and interaction performance

Avoid sequential waterfalls when requests are independent:

```js
const [accounts, preferences] = await Promise.all([
  loadAccounts(),
  loadPreferences(),
]);
```

Parallelism is appropriate only if dependencies and capacity permit it.
Do not fire hundreds of requests concurrently.

Other techniques:

- Batch or aggregate data needed for one screen.
- Deduplicate requests through a query layer.
- Cancel obsolete requests where supported.
- Debounce search requests without making input typing sluggish.
- Cache reads under explicit freshness rules.
- Use optimistic feedback only when rollback and business semantics are safe.

If a transfer is pending, show pending status rather than claiming completion
for the sake of perceived speed.

## 8. Rendering architecture

SSR, SSG, and ISR can reduce initial browser work or enable cached delivery,
but expensive hydration and large client bundles can still cause poor INP.

Streaming can show ready sections sooner. Avoid request waterfalls inside
server-rendered trees.

Choose rendering by route and content; consult the
[rendering guide](03-nextjs-rendering-patterns.md).

## 9. Memory and lifecycle problems

Watch for:

- Event listeners and subscriptions that are never cleaned up.
- Intervals that continue after a feature unmounts.
- Large retained caches with no lifecycle policy.
- Detached DOM references.
- Repeated expensive parsing or allocations.

Test long sessions, repeated navigation, and background tabs. A page that starts
fast can become slow after an hour.

## 10. Budgets and production monitoring

Set measurable budgets per route and resource category:

| Example budget | Purpose |
| --- | --- |
| LCP p75 at most 2.5 seconds | Field loading target |
| INP p75 at most 200 milliseconds | Field responsiveness target |
| CLS p75 at most 0.1 | Field stability target |
| Agreed route JavaScript ceiling | Prevent asset growth |
| Agreed p95 API latency at defined load | Protect task responsiveness |

Asset and API budgets depend on baseline and business context. Specify whether
sizes are raw, compressed, initial-route, or total downloaded sizes.

CI can check bundle growth and controlled lab tests. Field percentiles need
production samples, not a single CI measurement.

Segment monitoring by route, device, geography, and release. Do not record
sensitive user inputs in performance telemetry.

## 11. Interview scenario

**Problem:** A transaction page loads slowly and typing into its filter freezes.

Investigation:

1. Trace reveals a sequential API waterfall.
2. The browser mounts thousands of rows.
3. Filtering recomputes a costly transformation on each keystroke.

Response:

- Fetch independent data concurrently.
- Introduce suitable pagination/virtualization.
- Keep input state local and optimize measured computation.
- Consider deferred rendering for the results.
- Verify focus/accessibility and field INP after deployment.

Do not invent a percentage improvement in an interview. Explain how you measured
the baseline and the actual result.

## 12. Interview questions

**Is fewer React renders always better?**

No. Cheap renders may be harmless; DOM work, computation, or network delays can
dominate. Optimize measured user-visible cost.

**How do you improve LCP?**

Identify its element, then reduce server delay, resource discovery delay,
download time, and render delay as applicable.

**How do you improve INP?**

Reduce input delay, handler work, and presentation delay. Break up long tasks
and minimize unnecessary work around interactions.

**Does caching solve all performance problems?**

No. It does not fix expensive client execution and can introduce stale data.

## 13. Interview summary

> I define performance targets for real journeys, combine field metrics with
> traces and profiling, and optimize the dominant network, server, or rendering
> bottleneck. I prevent regressions with budgets and confirm improvements in
> production while preserving accessibility and correctness.

## References

- [Web Vitals](https://web.dev/articles/vitals)
- [Optimize LCP](https://web.dev/articles/optimize-lcp)
- [Optimize INP](https://web.dev/articles/optimize-inp)
- [React Profiler](https://react.dev/reference/react/Profiler)
- [Chrome performance tools](https://developer.chrome.com/docs/devtools/performance)
