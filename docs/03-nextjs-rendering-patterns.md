# Rendering Patterns: CSR, SSR, SSG, and ISR in Next.js

## 1. Separate rendering from caching

Rendering creates UI output. Caching stores output or data for reuse.

These are related but separate decisions:

- Where does rendering happen: browser or server?
- When does it happen: build time, request time, or regeneration time?
- What data/output is cached, and for how long?
- Does the browser need JavaScript for interactive behavior?

Next.js supports mixing strategies across routes and within a page. A static
product page can contain a client-side cart.

## 2. Comparison

| Pattern | Initial rendering | Typical freshness | Main trade-off |
| --- | --- | --- | --- |
| CSR: Client-Side Rendering | Browser after JavaScript runs | According to client fetching | More work and data fetching before content appears |
| SSR: Server-Side Rendering | Server at request time | According to fetched data/cache policies | Server work and latency per request |
| SSG: Static Site Generation | Build time, or configured on-demand generation | Until output is replaced/rebuilt | Fast delivery, but data can become stale |
| ISR: Incremental Static Regeneration | Static generation plus later regeneration | Controlled, not necessarily immediate | Fast delivery with eventual freshness |

Static delivery often has low time to first byte, but large JavaScript bundles
and expensive client work can still harm usability. SSR is not automatically
fast or automatically uncached at every layer.

## 3. CSR

### Lifecycle

```text
Request -> HTML shell -> Download JavaScript -> Run React
         -> Fetch API data -> Render content -> Interactive UI
```

CSR is appropriate for interactive, authenticated screens where search-engine
indexing is not central, such as internal dashboards.

Benefits:

- Rich client interactions.
- Navigation and updates without full document reloads.
- Can use a static host plus backend APIs.

Costs:

- Initial content can wait for JavaScript and API calls.
- Large bundles and request waterfalls hurt performance.
- Loading, error, cancellation, and retry behavior require explicit design.
- SEO and previews can be harder if meaningful content is absent from initial
  HTML, even though some crawlers execute JavaScript.

### Client Components are not the same as CSR-only pages

In the Next.js App Router, `"use client"` establishes a Client Component
boundary. It does not mean that the component is never prerendered on the server.
Client Components can contribute initial server-rendered HTML and later hydrate.

Browser-only fetching inside an effect happens after the component mounts in the
browser. A client data library can manage caching, deduplication, and retries.
Do not use client rendering as a security boundary: authorization must be
enforced on the server.

## 4. SSR

### Lifecycle

```text
Request -> Server fetches data -> Server renders HTML
        -> Browser shows HTML -> JavaScript hydrates interactive parts
```

Pages Router example:

```jsx
export async function getServerSideProps() {
  const response = await fetch("https://api.example.com/products");

  if (!response.ok) {
    throw new Error("Unable to fetch products");
  }

  return {
    props: { products: await response.json() },
  };
}

export default function ProductsPage({ products }) {
  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
}
```

`getServerSideProps` runs on the server for relevant requests; it is not shipped
as browser code. On client navigation, Next.js can request page data from the
server rather than fetching a complete new HTML document.

Benefits:

- Meaningful content in the initial HTML.
- Suitable for request-specific information.
- Server-only credentials can remain on the server if not exposed in props.

Costs:

- Data fetching can delay the response.
- High traffic requires sufficient server capacity.
- Dependencies can fail during user requests.
- Personalized responses require safe cache policies.

Request-time rendering does not guarantee current data: an upstream API or Redis
cache may still return an older value.

## 5. SSG

### Lifecycle

```text
Build -> Fetch content -> Generate static output -> Deploy
User request -> Serve generated output
```

Pages Router example:

```jsx
export async function getStaticProps() {
  const response = await fetch("https://api.example.com/articles");

  if (!response.ok) {
    throw new Error("Unable to fetch articles");
  }

  return {
    props: { articles: await response.json() },
  };
}
```

Without a revalidation policy, this page does not refresh simply because the
source data changed. New output normally requires another build/deployment or
explicit supported regeneration.

For dynamic routes in the Pages Router, `getStaticPaths` selects paths generated
at build time. Its fallback setting controls whether other paths return 404 or
are generated on demand.

Benefits:

- Fast delivery from static hosting/CDNs.
- Low per-request application rendering cost.
- Good fit for documentation, marketing pages, and stable content.

Costs:

- Large sites can have long builds.
- Content may be stale until regenerated.
- User-specific information cannot safely be baked into shared static output.

## 6. ISR: the detailed lifecycle

ISR lets generated pages be updated without rebuilding the entire application.

Pages Router example:

```jsx
export async function getStaticProps({ params }) {
  const response = await fetch(
    `https://api.example.com/products/${params.id}`
  );

  if (!response.ok) {
    throw new Error("Unable to fetch product");
  }

  return {
    props: { product: await response.json() },
    revalidate: 60,
  };
}

export default function ProductPage({ product }) {
  return (
    <main>
      <h1>{product.name}</h1>
      <p>Price: {product.price}</p>
    </main>
  );
}
```

For `pages/products/[id].jsx`, this example also needs `getStaticPaths` and an
appropriate fallback policy; it is a lifecycle illustration, not a complete
dynamic-route implementation.

### What happens after 60 seconds?

1. Next.js has a successfully generated cached page.
2. Requests within its freshness period receive the cached output.
3. After the interval, the page becomes eligible for revalidation.
4. The next request typically receives the stale cached page.
5. Next.js reruns `getStaticProps` on the server in the background.
6. Your `fetch` calls the API again.
7. Next.js renders the page using the returned props.
8. Successful output replaces the cached version.
9. Later requests receive the new version.

```text
00s: Generate price 1,000
20s: Database price becomes 1,200
40s: Visitor sees cached 1,000
60s: Interval expires; no automatic timer-driven rebuild
75s: Visitor sees 1,000; background regeneration begins
77s: Regeneration finishes with 1,200
80s: Later visitor sees 1,200
```

If nobody requests the route, the interval alone does not regenerate it.

### How can it render without a new build?

The deployed server already has the compiled page code. It executes the data
fetching and rendering code again and saves the resulting page output.

This is regeneration of an affected route, not a new deployment or compilation
of the entire application.

### Does ISR rebuild only the changed price component?

Traditional page-level ISR regenerates the page output. It does not compare old
API data with new data and patch only the price component's cached HTML.

Distinguish:

- **Page regeneration:** Server produces new cached route output.
- **Data caching:** Some data sources may be reused while others refresh.
- **React reconciliation:** An existing client React tree updates necessary DOM
  elements when its state/props change.

An already-open browser page does not automatically update when ISR completes.
It needs a later navigation, refresh, fetch, or live-update mechanism.

### Unchanged data and failures

If the API returns the same price, regeneration can succeed with visually
identical output. ISR does not require a data change.

If regeneration fails, Next.js normally retains the last successful page and
can retry on a later request. Monitor failures instead of silently replacing
errors with incomplete "successful" data.

If Redis or an API returns stale data, a freshly regenerated page may still show
the old price.

## 7. Next.js App Router and version-sensitive caching

The App Router uses Server Components by default. Server Components can render
statically or dynamically; "server component" does not mean "render on every
request."

For supported App Router configurations **without Cache Components**, an
explicit cached fetch can look like:

```tsx
export default async function ProductPage() {
  const response = await fetch("https://api.example.com/products/42", {
    next: { revalidate: 60, tags: ["product:42"] },
  });

  if (!response.ok) {
    throw new Error("Unable to fetch product");
  }

  const product: { name: string; price: number } = await response.json();

  return (
    <main>
      <h1>{product.name}</h1>
      <p>Price: {product.price}</p>
    </main>
  );
}
```

The annotation illustrates the expected API shape; production code should
validate untrusted data at the boundary. Whether the whole route is static also
depends on the rest of the route and its use of dynamic APIs.

Avoid assuming all `fetch` calls are cached by default: defaults changed across
Next.js versions.

For request-specific fetching in applicable configurations, `cache: "no-store"`
expresses an uncached fetch. Do not combine incompatible cache options.

### Cache Components

When modern Next.js Cache Components are enabled, caching can be expressed with
`"use cache"`, `cacheLife`, and `cacheTag`. Partial prerendering can combine a
static shell with request-time sections behind Suspense boundaries.

Do not apply a legacy route-level `revalidate` recipe blindly to this mode:
supported configuration and behavior differ. Explain the deployed version and
mode before choosing APIs.

### On-demand revalidation

When content changes, an authorized server-side mutation or webhook can
invalidate affected cache entries.

Relevant APIs include:

- **Pages Router:** `res.revalidate(path)` for on-demand ISR.
- **App Router:** `revalidatePath(path)` for path-related invalidation.
- **Modern App Router:** `revalidateTag(tag, "max")` for stale-while-revalidate
  invalidation of tagged data.
- **Modern App Router:** `updateTag(tag)` in Server Actions for immediate
  expiration and read-your-own-writes use cases.

These APIs are not interchangeable. The timing of regeneration and visible UI
refresh depends on the API, call context, router, and version. For example,
path invalidation from a Route Handler does not mean every cached page is
eagerly regenerated immediately.

Protect webhook endpoints with authentication/signature verification. Never
allow arbitrary unauthenticated callers to purge your caches.

## 8. Hydration, streaming, and security

**Hydration** connects client React behavior to existing server-rendered HTML.
The initial server and client output should agree. Uncontrolled time, random
values, or browser-only state in the first render can cause mismatches.

**Streaming SSR** sends ready parts of the response while slower sections are
still rendering, often using Suspense. It can improve perceived responsiveness
but does not make slow dependencies disappear.

Do not send secrets as props or expose private data in generated shared pages.
Server rendering does not make serialized data invisible to the browser.

## 9. Choosing a pattern

| Use case | Reasonable starting point |
| --- | --- |
| Documentation or rarely changing marketing content | SSG |
| Public product catalog with bounded staleness | ISR |
| Personalized request-dependent page | Dynamic SSR with safe caching |
| Highly interactive internal dashboard | CSR or SSR shell plus client fetching |
| Public page with a live stock indicator | Static/ISR content plus a separately refreshed indicator |

For checkout, confirm price and availability on the server at transaction time.
A stale catalog page should never determine the authoritative amount charged.

## 10. Operational checks and interview questions

Test production behavior, not only the development server. Use the deployed
platform's cache diagnostics or a production build/start flow. ISR needs
compatible server/platform support; a plain static export cannot perform
runtime ISR.

If self-hosting across multiple instances, ensure cache persistence and
coordination match the deployment. Independent caches can produce inconsistent
freshness.

**Is ISR a cron job?**

Not in the traditional request-triggered model. A request after the interval
triggers regeneration.

**Does SSR always return the latest database state?**

No. Upstream caches and replicas may still be stale.

**Does a Client Component disable SSR?**

No. Client Components can be prerendered and hydrated.

**Can all four approaches exist in one application?**

Yes. Choose per route and combine static content with dynamic/client sections.

## 11. Interview summary

> CSR renders in the browser, SSR renders at request time, SSG generates reusable
> output ahead of requests, and ISR refreshes generated output without a full
> rebuild. I choose based on freshness, personalization, SEO, latency, and cost,
> and explain the router/version-specific cache behavior.

## References

- [Next.js Pages Router rendering](https://nextjs.org/docs/pages/building-your-application/rendering)
- [Pages Router ISR](https://nextjs.org/docs/pages/guides/incremental-static-regeneration)
- [App Router ISR](https://nextjs.org/docs/app/guides/incremental-static-regeneration)
- [Cache Components](https://nextjs.org/docs/app/getting-started/cache-components)
- [revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)
- [updateTag](https://nextjs.org/docs/app/api-reference/functions/updateTag)
