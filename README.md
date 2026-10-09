# System Design Interview Notes

Practical interview guides for engineers preparing for senior, staff, principal,
and technical-lead roles. The examples emphasize web applications, React/Next.js,
and backend reliability.

## Guides

| Topic | What you will learn |
| --- | --- |
| [Database bottlenecks](docs/01-database-bottlenecks.md) | Diagnosing slow databases; indexes, partitioning, sharding, rate limiting, and concurrency controls |
| [Caching overview](docs/02-caching.md) | How browser, CDN, and Redis caches work together; freshness, consistency, invalidation, and troubleshooting |
| [Redis application caching](docs/02a-redis-caching.md) | Cache-aside, write races, stampedes, eviction, clustering, failover, observability, and detailed interview questions |
| [CDN caching](docs/02b-cdn-caching.md) | Edge request lifecycles, cache keys, HTTP policies, purging, origin protection, private content, and detailed interview questions |
| [Browser caching](docs/02c-browser-caching.md) | HTTP freshness and validation, ETags, service workers, React data caches, secure logout, deployments, and detailed interview questions |
| [Service workers](docs/11-service-workers.md) | Simple explanations of lifecycle, scope, offline caching, updates, push, background sync, security, debugging, and 50 interview questions |
| [Web workers](docs/12-web-workers.md) | Main-thread performance, worker types, messages, transferable data, cancellation, React integration, debugging, and 50 interview questions |
| [Debounce and throttle](docs/13-debounce-and-throttle.md) | Simple timing examples, use cases, implementations, React cleanup, request races, testing, and 25 interview questions |
| [CSR, SSR, SSG, and ISR in Next.js](docs/03-nextjs-rendering-patterns.md) | Rendering lifecycles, hydration, caching, revalidation, and choosing patterns per route |
| [Event-driven architecture](docs/04-event-driven-architecture.md) | Push/pull ingestion, scheduled jobs, queues, streams, delivery guarantees, ordering, and recovery |
| [State management and composition patterns](docs/05-state-management-and-composition.md) | State ownership, reducers, Context, Redux, server-state caching, reusable components, and enterprise React architecture |
| [Accessibility](docs/06-accessibility.md) | WCAG, semantic HTML, keyboard navigation, forms, focus management, and accessible React testing |
| [Performance](docs/07-performance.md) | Core Web Vitals, profiling, network and rendering optimization, performance budgets, and production monitoring |
| [Security](docs/08-security.md) | Trust boundaries, XSS, CSRF, authorization, sessions, dependencies, and secure frontend design |
| [CI/CD for non-functional requirements](docs/09-cicd-nfrs.md) | Measurable quality gates, accessibility/performance/security checks, safe releases, SLOs, and rollback |
| [SPA, MPA, and hybrid architectures](docs/10-spa-mpa-hybrid.md) | Navigation and rendering models, trade-offs, architecture selection, and incremental migration |

## How to study

1. Read the mental model and lifecycle for each topic.
2. Explain the example aloud without looking at the guide.
3. Practice the trade-offs and failure scenarios.
4. Use the interview questions to test your reasoning.
5. Follow the official references when implementing a specific technology.

## Scope and assumptions

- Code snippets are illustrative unless explicitly stated otherwise. They are
  not a complete production application or a runnable sample project.
- Database calls and service names in examples are placeholders.
- Next.js behavior depends on version, router, deployment platform, and caching
  configuration. The rendering guide labels its examples accordingly.
- Performance limits and TTLs are examples, not universal recommendations.
- Banking examples explain engineering decisions; they do not replace security,
  compliance, or financial-correctness requirements.

## Suggested interview approach

Start with requirements: traffic, data volume, freshness, latency, consistency,
availability, security, and cost. Then propose the simplest architecture that
meets them. Explain its failure modes, how you would measure it, and what would
justify adding complexity.
