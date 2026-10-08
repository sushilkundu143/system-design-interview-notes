# SPA, MPA, and Hybrid Architectures: Interview Guide

## 1. Separate navigation architecture from rendering

SPA/MPA describe how pages are organized and navigated. CSR/SSR/SSG/ISR describe
where and when UI output is generated.

They are different axes:

- A SPA can start with server-rendered HTML and hydrate.
- An MPA can include interactive React widgets.
- A framework can server-render route entry and use client navigation afterward.

Do not equate SPA with "CSR only" or MPA with "no JavaScript."

## 2. SPA: Single-Page Application

A SPA generally loads an application shell, then uses a client router to update
the visible view without a full document navigation for internal transitions.

```text
Initial URL -> HTML + application assets -> Initialize app
Internal navigation -> Client router -> Fetch needed code/data -> Update UI
```

### Benefits

- Smooth transitions and rich interaction.
- Shared client state can survive route changes while its owner stays mounted.
- Reuses loaded application code.
- Good fit for dashboards and long interactive workflows.

### Costs

- Large initial bundles can delay startup.
- Client routing, loading, errors, and focus management need careful design.
- Memory can grow across long sessions.
- A runtime error can affect a large application area.
- CSR-only content can complicate previews and indexing.

### Operational details

Deep links must be supported by the server/platform. A URL such as
`/accounts/42` cannot simply return 404 because the router lives in the browser.

Do not rewrite asset requests or every unknown URL indiscriminately to a
successful app shell. Configure API, asset, and true-not-found behavior correctly.

Deployments need to handle clients still running older code while new chunks
and APIs are released. Retaining immutable assets and maintaining compatibility
can reduce chunk-load failures.

## 3. MPA: Multi-Page Application

An MPA uses separate document navigations between pages.

```text
Request /accounts -> Server returns document
Click /profile -> Browser requests another document
```

The documents may be generated at request time or served statically. Individual
pages can use JavaScript, React, and progressive enhancement.

### Benefits

- Native document navigation, history, and form behavior.
- Page-specific assets can reduce initial application scope.
- Failures and ownership can be isolated by page.
- Works well for content sites and traditional server-rendered workflows.
- Meaningful HTML can remain usable before enhancement.

### Costs

- Full navigation can repeat initialization and UI work.
- In-memory React state generally does not survive a new document.
- Shared layouts and behavior need consistent implementation.
- Highly interactive cross-page workflows may need additional coordination.

Browser caches and back-forward caching can make document navigation efficient;
an MPA is not inherently slow.

Store necessary workflow state deliberately on the server, in the URL, or in
approved client persistence. Do not depend on a component remaining mounted
across documents.

## 4. Hybrid architectures

"Hybrid" is an umbrella term, not one fixed implementation.

### Server-rendered entry plus client navigation

A framework returns useful HTML for the first request, hydrates interactive
components, and performs later navigations through a client router.

Examples include suitably configured React/Next.js applications.

Trade-off: good initial content and rich navigation, but hydration, caching,
client-state lifetime, and server/client boundaries add complexity.

### Islands architecture

A mostly static/server-rendered document contains independently interactive
regions, such as a search widget and cart control.

```text
Server-rendered page
  +-- Static article
  +-- Interactive search island
  +-- Interactive feedback island
```

It can reduce shipped JavaScript when most content needs no interactivity.
Shared state and coordination across islands need explicit design.

### MPA shell with embedded SPA features

A server-rendered application can embed React for selected workflows while
keeping other routes as ordinary pages.

This is useful for incremental migration. Define ownership of navigation,
authentication, styles, and mounting/unmounting.

### Route-specific strategies

One product may use static marketing pages, an ISR catalog, and a dynamic
authenticated dashboard. Avoid forcing every route into the same lifecycle.

Hybrid is not synonymous with micro-frontends. Independent frontend deployment
is another architectural decision that can coexist with SPA or MPA.

## 5. Comparison

| Dimension | SPA | MPA | Hybrid |
| --- | --- | --- | --- |
| Internal navigation | Usually client-managed | Usually new document | Depends on boundary |
| Initial content | CSR shell or server-rendered entry | Separate HTML documents | Static/server content with selected interaction |
| In-memory state | Can persist in mounted owners | Usually resets across documents | Boundary-dependent |
| JavaScript cost | Can be large without splitting | Can be limited per page | Can target interactive areas |
| SEO | Depends on rendered content and metadata | Often straightforward with meaningful HTML | Can combine crawlable content and interaction |
| Failure scope | Can affect the shared app runtime | Often page-scoped | Depends on integration |
| Main complexity | Routing and long-lived client lifecycle | Cross-document coordination | Multiple boundaries and contracts |

No architecture automatically wins on performance, security, or accessibility.
Implementation and workload matter.

## 6. State, data, and navigation

For an SPA:

- Keep route state in the URL when it should be shareable.
- Scope stores/providers to their intended lifetime.
- Clear user-specific data on identity changes.
- Handle obsolete requests during fast navigation.

For an MPA:

- Persist important multi-step work intentionally.
- Use server session/draft models where appropriate.
- Expect document lifecycle and input restoration behavior to vary.

For hybrid:

- Define which system owns each URL.
- Avoid duplicate routers competing for history.
- Avoid multiple isolated stores pretending to be one shared source of truth.
- Set contracts for cross-boundary events and data.

See [state management](05-state-management-and-composition.md) for ownership.

## 7. NFR implications

### Accessibility

SPAs must handle route titles, announcements, and focus without relying solely
on a document reload. MPAs still need accessible content and controls. Hybrid
boundaries must not duplicate landmarks or create confusing focus behavior.

### Performance

Compare initial load, route transitions, hydration, and long-session memory.
MPAs can benefit from asset caching; SPAs can benefit from reuse. Both can be
slow if they ship excessive code or wait on poor APIs.

### Security

Enforce backend authorization in every architecture. Keep secrets off the
client and prevent private shared caching. Architecture does not remove XSS,
CSRF, or dependency risks.

### Reliability

Plan error boundaries, unavailable routes, offline/reconnect behavior where
required, and compatibility during deployments. A client error boundary does
not catch every async or server failure.

## 8. Choosing an architecture

Ask:

1. Is the product mostly content or continuous interaction?
2. Are indexing and social previews important?
3. What are device/network constraints?
4. Must long workflows preserve drafts across navigation?
5. What existing backend/rendering infrastructure exists?
6. How are teams deploying and owning features?
7. What operational complexity can the organization support?

| Product | Reasonable starting point |
| --- | --- |
| Documentation/public content | Static MPA or server-rendered content with small interactive areas |
| Complex internal dashboard | SPA or server-rendered app with client navigation |
| Public catalog plus cart | Hybrid static/ISR content and interactive cart |
| Existing server banking portal | Incremental React islands or embedded workflows where appropriate |
| Highly interactive authenticated banking app | SPA/hybrid with deliberate loading, session, and workflow design |

These are starting points, not mandatory prescriptions.

## 9. Incremental migration example

**Situation:** A server-rendered banking portal needs a richer alert-preferences
experience.

1. Establish measurable UX and quality goals.
2. Choose the alert feature as a bounded migration surface.
3. Reuse authentication and backend authorization.
4. Mount React in a defined region or route.
5. Keep one owner for breadcrumbs, navigation, and page layout.
6. Scope feature state and avoid duplicate providers.
7. Verify direct links, back/forward navigation, and refresh.
8. Test accessibility and performance against the existing flow.
9. Roll out behind a flag with a documented fallback.
10. Expand only after operational evidence supports it.

Do not call a full rewrite necessary simply because React is preferred.

## 10. Common mistakes

- Treating SPA as automatically faster.
- Treating MPA as incapable of modern interaction.
- Assuming SSR means no hydration or client code.
- Claiming hybrid combines benefits without any new costs.
- Losing unsaved data at routing boundaries.
- Mounting duplicate routers or leaking event subscriptions.
- Replacing native navigation without focus/title handling.
- Making a public shell cacheable while accidentally including private data.

## 11. Interview questions

**Can an SPA support SEO?**

Yes, especially with server-rendered or prerendered meaningful content and
correct metadata. A CSR-only shell may need additional attention.

**Why choose MPA for a content site?**

It can provide straightforward document delivery, small per-page assets,
progressive enhancement, and simple navigation semantics.

**When is hybrid useful?**

When requirements differ across routes or when migrating an existing application
incrementally without taking on a full rewrite.

**Does an SPA keep state forever?**

No. State survives only while its owning component/store survives. Remounts,
reloads, logout, and deliberate resets can remove it.

**How would you prove the architecture choice was good?**

Measure critical journeys, Web Vitals, accessibility, reliability, delivery
complexity, and maintenance cost against the requirements.

## 12. Interview summary

> SPA, MPA, and hybrid describe navigation and application boundaries, while
> CSR/SSR/SSG/ISR describe rendering. I choose based on interaction, content,
> state lifetime, device constraints, and operations, and use hybrid boundaries
> or incremental migration when different parts need different approaches.

## References

- [MDN single-page applications](https://developer.mozilla.org/en-US/docs/Glossary/SPA)
- [Next.js server and client components](https://nextjs.org/docs/app/getting-started/server-and-client-components)
- [Astro islands architecture](https://docs.astro.build/en/concepts/islands/)
- [Rendering patterns guide](03-nextjs-rendering-patterns.md)
- [CI/CD and NFRs guide](09-cicd-nfrs.md)
