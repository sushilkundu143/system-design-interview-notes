# Service Workers: A Simple, Detailed Guide

[All guides](../README.md) | [Browser caching](02c-browser-caching.md) |
[CDN caching](02b-cdn-caching.md)

## How to read this guide

1. Start with sections 1-6 to understand the basics.
2. Read the small example in section 7.
3. Study updates, offline actions, security, and debugging.
4. Practice the 50 interview questions, grouped from basic to advanced.

This is a broad study guide, not literally every possible interview question.
Examples are intentionally small. Production applications need testing, monitoring,
and policies for their specific users and data.

## Contents

- [1. What is a service worker?](#1-what-is-a-service-worker)
- [2. What can and cannot it do?](#2-what-can-and-cannot-it-do)
- [3. Service worker versus other browser features](#3-service-worker-versus-other-browser-features)
- [4. Registration, scope, and control](#4-registration-scope-and-control)
- [5. The lifecycle](#5-the-lifecycle)
- [6. Important events and methods](#6-important-events-and-methods)
- [7. Small offline-page example](#7-small-offline-page-example)
- [8. Caching strategies](#8-caching-strategies)
- [9. Updates and deployments](#9-updates-and-deployments)
- [10. Messages, storage, and multiple tabs](#10-messages-storage-and-multiple-tabs)
- [11. Push notifications and background sync](#11-push-notifications-and-background-sync)
- [12. Security, privacy, and logout](#12-security-privacy-and-logout)
- [13. React, Next.js, Workbox, and PWAs](#13-react-nextjs-workbox-and-pwas)
- [14. Debugging and testing](#14-debugging-and-testing)
- [15. Interview questions and answers](#15-interview-questions-and-answers)
- [16. Practice design exercise](#16-practice-design-exercise)
- [17. Quick revision sheet](#17-quick-revision-sheet)

## 1. What is a service worker?

A service worker is JavaScript that the browser runs separately from your page.
It can handle certain events, including network requests from pages it controls.

**Everyday example:** Think of a receptionist between you and a library.

- You ask for a book.
- The receptionist checks whether a saved copy is available.
- If not, they contact the main library.
- If the library cannot be reached, they may offer an approved alternative.

In a web application:

```text
Page requests a resource
          |
          v
Service worker, if the page is controlled
          |
          +--> saved response in Cache Storage
          |
          +--> fetch from network
          |
          +--> offline fallback
```

**Important:** A service worker does not automatically make a website offline.
You must write the behavior, save suitable resources, and define what to do when
a request fails.

### Why use one?

- Show useful content when the connection fails.
- Reuse static files with explicit application-managed rules.
- Receive supported push events.
- Retry suitable queued work through supported background-sync APIs.

Not every application needs a service worker. Ordinary HTTP caching is often
enough when offline behavior and background events are not requirements.

## 2. What can and cannot it do?

| It can | It cannot |
| --- | --- |
| Handle requests from controlled pages | Automatically control every site or every open tab |
| Use Fetch, Cache Storage, and IndexedDB | Directly access the page's DOM |
| Communicate through messages | Use `window` or `localStorage` |
| Handle supported push/sync events | Run forever like a dedicated backend server |
| Show notifications with applicable permission | Bypass authentication or CORS rules |

The service worker has its own global object, called `self`.
It does not share the page's variables or React state.

### It is event-driven, not always running

The browser can stop an idle service worker and start it for a later event.
Do not rely on global variables, `setInterval`, or an open WebSocket to keep
important background work alive.

Persist necessary state in IndexedDB or another appropriate storage mechanism.

## 3. Service worker versus other browser features

| Feature | Main job | Example |
| --- | --- | --- |
| Service worker | Handle network/background events for controlled clients | Offline help page |
| Web Worker | Run page-related computation away from the UI thread | Expensive data processing |
| HTTP cache | Reuse responses according to HTTP headers | Long-lived versioned CSS |
| Cache Storage | Explicitly store Request/Response pairs | Service-worker offline assets |
| IndexedDB | Store structured application data | A queue of permitted drafts |
| React query cache | Reuse data in the application's UI | Previously loaded product list |
| CDN | Deliver eligible responses through shared edge servers | Public images near users |

Service workers commonly use Cache Storage, but they are not the same thing.
Pages can also use Cache Storage in supported secure contexts.

## 4. Registration, scope, and control

### Requirements

- A supporting browser.
- A secure context, normally HTTPS.
- A same-origin service-worker script.
- A valid JavaScript response, not an HTML fallback page.

Localhost is normally treated as trustworthy for development.
Plain HTTP on an arbitrary LAN address is not equivalent to localhost.

### Registration example

Run this in page code:

```js
async function registerServiceWorker() {
  if (!("serviceWorker" in navigator)) {
    return; // Offline enhancement is optional; the normal page still works.
  }
  try {
    const registration = await navigator.serviceWorker.register("/sw.js", {
      scope: "/",
    });
    console.info("Service worker registered:", registration.scope);
  } catch (error) {
    console.error("Service worker registration failed:", error);
  }
}

registerServiceWorker();
```

Production apps should send operational failures to their monitoring system.
If offline support is essential, show its unavailability to the user too.

### What is scope?

Scope determines which client pages a registration can control.

```text
Script location: /app/sw.js
Default scope:   /app/
```

By default, placing a script in `/app/` does not let it control `/admin/`.
A broader permitted scope requires the appropriate `Service-Worker-Allowed`
response header and registration settings.

Scope is about the controlled page, not simply an allowlist of resource URLs.
A controlled page can make requests outside its own path, and the worker can
receive those fetch events. Cross-origin restrictions still apply.

### Registered is not the same as controlling this page

```js
const isControlled = navigator.serviceWorker.controller !== null;
```

The first registration can succeed while the currently open page remains
uncontrolled. A later navigation normally lets the active worker control it.
An active worker can use `clients.claim()` to take control of eligible existing
clients, but that changes behavior mid-session and needs care.

`navigator.serviceWorker.ready` waits for a registration with an active worker.
It is not a guarantee that the current page is already controlled.

## 5. The lifecycle

```text
Register / discover update
          |
          v
Install new worker and prepare required resources
          |
          v
Wait, if an older active worker still controls clients
          |
          v
Activate
          |
          v
Handle fetch, message, push, and other supported events
```

### A. Install: prepare

The new worker can save resources it needs before becoming usable.
If required installation work rejects, the new installation fails.
An already-working older worker can continue.

**Analogy:** Prepare the new receptionist's desk before handing over the job.

### B. Waiting: do not disrupt the old version

A new worker normally waits while the previous active worker controls clients.
This helps avoid replacing the worker underneath an application already using
an older version.

Other open tabs can keep the old version in use. Simply refreshing one tab may
not release all controlled clients.

### C. Activate: hand over

Activation is where a worker can perform necessary setup or safe cache cleanup.
Be careful: old page code may still need old assets if you force an early handover.

### Worker states you may see

`installing`, `installed` (often waiting), `activating`, `activated`, and `redundant`.
A redundant worker is no longer usable, for example after replacement or failure.

### Two methods that need special care

- `self.skipWaiting()`: asks a waiting worker to activate without waiting for
  existing users of the older version to leave.
- `self.clients.claim()`: lets the active worker control eligible existing clients.

These solve different problems. Neither guarantees compatibility between old
page code and a new worker.

## 6. Important events and methods

| Event or method | Simple explanation |
| --- | --- |
| `install` | Prepare a new worker |
| `activate` | Finish handover/setup |
| `fetch` | Handle eligible requests from controlled clients |
| `message` | Receive a message from a page or another context |
| `push` | Handle a supported push delivery |
| `notificationclick` | React when a user clicks a notification |
| `sync` | Retry registered work in supporting browsers |
| `event.waitUntil(promise)` | Ask the browser to keep event work alive until it settles |
| `event.respondWith(promise)` | Supply a response for a fetch event |

Call `respondWith()` during the fetch handler's synchronous execution.
You can pass it a promise that performs asynchronous work.

Similarly, start lifetime-extension work with `waitUntil()` while the event
allows it. Neither method guarantees unlimited execution or survival through a
browser/OS shutdown.

## 7. Small offline-page example

This example does just one thing:

> When a controlled same-origin page navigation cannot reach the network,
> show a public offline page.

It does not cache APIs, replay forms, or provide a full offline application.

### Required public offline page

Create `/offline.html` with no required external resources:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Connection unavailable</title>
  </head>
  <body>
    <h1>We could not reach the server</h1>
    <p>Check your connection, then try again. No action has been submitted.</p>
  </body>
</html>
```

Do not include account details or content that implies an operation succeeded.

### Service-worker script: `/sw.js`

```js
const OFFLINE_CACHE = "help-offline-v1";
const OFFLINE_URL = "/offline.html";

self.addEventListener("install", (event) => {
  event.waitUntil(
    (async () => {
      const cache = await caches.open(OFFLINE_CACHE);
      const response = await fetch(OFFLINE_URL, { cache: "reload" });
      if (!response.ok) {
        throw new Error(`Offline page download failed: ${response.status}`);
      }
      await cache.put(OFFLINE_URL, response);
    })()
  );
});

self.addEventListener("fetch", (event) => {
  const request = event.request;
  const url = new URL(request.url);
  if (
    request.method !== "GET" ||
    request.mode !== "navigate" ||
    url.origin !== self.location.origin
  ) {
    return;
  }

  event.respondWith(
    (async () => {
      try {
        return await fetch(request);
      } catch (error) {
        console.error("Navigation network request failed:", error);
        const cache = await caches.open(OFFLINE_CACHE);
        const fallback = await cache.match(OFFLINE_URL);
        if (fallback) return fallback;
        return new Response("Server unreachable; offline page is unavailable.", {
          status: 503,
          headers: { "Content-Type": "text/plain; charset=utf-8" },
        });
      }
    })()
  );
});
```

### What each part does

1. `install` saves a required public page.
2. A failed download fails the installation rather than claiming offline support.
3. The handler accepts only same-origin GET navigations.
4. It tries the network.
5. A network failure gets a clearly labelled fallback.
6. If the saved fallback was removed, it returns an explicit 503.

HTTP 404 and 500 responses do not normally reject `fetch()`.
This example passes them through instead of calling every server error "offline."
Normal HTTP caching can also satisfy `fetch()`; network-first here is not a
promise that every navigation contacts the origin.

The example deliberately omits immediate activation and automatic cleanup.
Those require a tested update policy; see the next sections.

### Try it

1. Serve the page, `/sw.js`, and `/offline.html` on localhost or HTTPS.
2. Register the worker from page code.
3. Verify installation succeeded.
4. Navigate again and confirm a controller exists.
5. Switch the browser to offline.
6. Navigate to a same-origin page and verify the fallback.
7. Restore the network and verify normal behavior.

First-time offline visitors will not have downloaded the worker or fallback yet.
No service worker can retroactively cache content they never received.

## 8. Caching strategies

A strategy is a rule for choosing between a stored copy and a network response.

| Strategy | Steps | Suitable example | Main risk |
| --- | --- | --- | --- |
| Cache-first | Saved copy; network on miss | Content-hashed assets | Mutable data can stay old |
| Network-first | Network; suitable fallback on failure | Offline-tolerant public pages | Slow failures delay fallback |
| Stale-while-revalidate | Saved copy now; refresh in background | Public news summaries | Users see old content |
| Network-only | Use fetch, no Cache Storage fallback | Sensitive operations | Needs connection |
| Cache-only | Only a deliberately saved copy | Preloaded offline reference | Missing items cannot load |

Here, "network-only" means no application Cache Storage fallback.
If you must also avoid browser HTTP caching, choose suitable fetch options and
server response policies. A CDN is yet another layer.

### Precaching versus runtime caching

- **Precaching:** save known resources during installation.
- **Runtime caching:** save eligible responses as users request them.

Example: precache an offline help page; runtime-cache selected public help images.

Keep the required precache small. A large mandatory download delays installation,
uses storage, and can make updates fragile.

### Cache Storage does not automatically expire entries by HTTP TTL

If your code chooses a saved response with `cache.match()`, you must apply your
own expiration/version rules. Do not assume `Cache-Control: max-age=60` makes
Cache Storage delete that response after a minute.

Choose a maximum entry count, an age policy, and cleanup rules.
Workbox can help implement these rules, but cannot choose your product policy.

### `Response.clone()`

A response body is usually a stream that can be consumed once.
If you need to return a response and store another readable copy, clone it before
either operation consumes the body.

Cloning is not free. Large responses can consume memory; cache selectively.

### Failed or cross-origin responses

- Only save responses your policy actually allows.
- Do not accidentally cache login redirects or transient failures as public content.
- CORS still applies to cross-origin fetches.
- A no-CORS response may be **opaque**: JavaScript cannot inspect its body,
  headers, or actual status.
- Avoid broad caching of opaque responses; validation and storage costs become harder.

## 9. Updates and deployments

### How does the browser find a new worker?

It checks the registered script for updates at applicable times.
You can also request a check with `registration.update()`.
Changed script bytes can lead to installing a new worker.

Keep a stable script URL such as `/sw.js`.
Do not assume changing page HTML instantly updates every open tab.
Worker-script HTTP caching and imported-script checking have specific rules;
check your browser/build setup and the `updateViaCache` registration option.

### A user-friendly update flow

1. Publish new versioned assets.
2. Publish the worker and entry page in a compatible order.
3. Detect a waiting worker, including one already waiting at page load.
4. Show a clear, accessible "Update available" prompt.
5. Protect unsaved work before the user accepts.
6. Ask the waiting worker to activate, if the design supports it.
7. On the appropriate `controllerchange`, reload once when safe.
8. Monitor update and loading failures.

Do not automatically reload forever.

### Old files and open tabs

An old React tab might request an old lazy-loaded chunk later.
If you remove it too soon, navigation fails.

Keep old assets for an appropriate compatibility window.
Delete only caches owned by this feature/application, not every cache on the origin.
Different applications can share an origin.

### Rollback

Deploy the corrected worker at the stable URL and make it compatible with existing
page versions and stored data. Older cached HTML can still affect what users see.

Removing `register()` from page code does not automatically remove previously
installed registrations.

## 10. Messages, storage, and multiple tabs

### Messages

Page to worker:

```js
navigator.serviceWorker.controller?.postMessage({
  type: "REFRESH_PUBLIC_HELP",
});
```

This only demonstrates sending. The worker must validate the message and implement
the intended behavior. A missing controller means nothing receives it.

Use message events or MessageChannel for replies. Match requests to responses
when multiple operations can overlap.

### Storage

- Cache Storage: HTTP request/response pairs.
- IndexedDB: structured records, queue items, metadata.
- Global variables: temporary only; they disappear when the worker stops.

Storage may be unavailable, reach quota, or be evicted.
Private browsing and browser settings can change availability.
Use `navigator.storage.estimate()` for estimates; do not treat them as a permanent
storage guarantee.

### Multiple tabs

Tabs can share a registration and storage.
One tab can trigger an update or logout affecting others.

Coordinate user-specific cleanup and requests. Use appropriate messaging, such
as BroadcastChannel where supported, without putting sensitive data into messages
unnecessarily.

## 11. Push notifications and background sync

### Push: the server initiates a delivery

Typical flow:

```text
User chooses notifications
  -> browser permission / subscription
  -> backend stores subscription
  -> backend sends through browser's push service
  -> browser can wake worker
  -> worker handles push and displays permitted notification
```

Push requires supported browser/platform behavior and permission.
Installation requirements differ by platform, especially on mobile.
It can work without an open page, but delivery is not an immediate or guaranteed
exactly-once channel.

Ask for notification permission after a meaningful user action, not immediately
on arrival. Handle denial and expired subscriptions.

Do not place balances, OTPs, or other sensitive details into lock-screen messages.
Validate notification click destinations rather than opening arbitrary supplied URLs.

### Background sync: retry suitable queued work

Example: a public feedback draft may be queued until a connection returns.

1. Save the permitted draft durably.
2. Register a supported sync task.
3. Let the worker try sending.
4. Mark/remove the item only after confirmed server success.
5. Handle retry, duplicate delivery, expired authentication, and permanent errors.

**Do not show "sent" while the action is merely queued.**

Background sync support is limited. Always provide a normal-page retry/manual
recovery path. Periodic background sync is a separate, more restricted feature,
not a reliable scheduler.

### Payments are not ordinary offline drafts

A payment requires explicit product/security decisions and server-side
idempotency, authorization, and current transaction validation.
Do not automatically replay a financial action because a connection returned.

**Idempotency** means a retry with the same operation identifier must not apply
the financial action twice.

## 12. Security, privacy, and logout

Because a service worker can influence many page requests and persist across
visits, its script and routing policy deserve careful protection.

- Serve trusted worker code over HTTPS.
- Do not allow user uploads or untrusted content to become worker scripts.
- Apply appropriate content-security and worker-loading policies.
- Limit scope and caching routes to what the feature needs.
- Treat Cache Storage as readable by same-origin code, not as secret storage.
- Avoid caching authentication responses, private statements, or account APIs casually.
- Do not assume a service worker replaces server authorization.

### `no-store` needs application cooperation

HTTP cache rules do not automatically enforce your explicit `cache.put()` policy.
Do not manually persist a response whose privacy policy requires no storage.

### Logout checklist

1. Invalidate the server session.
2. Clear user-specific UI/query state.
3. Cancel or safely discard old in-flight work.
4. Clear intentionally stored private entries and queued actions.
5. Notify other tabs as required.
6. Recheck authentication when a saved page is restored.

Deleting everything may break unrelated same-origin apps.
Identify ownership, identity, and pending-work behavior before cleanup.

## 13. React, Next.js, Workbox, and PWAs

### React

A service worker is not a React component or hook.
Register it from browser-side startup code and connect messages/update state to
your UI if needed.

Avoid repeated registration logic scattered across components.
A service worker does not automatically refresh React state or query caches.

### Next.js

- Register only in browser code, not during server rendering.
- A script under the configured public assets path can be served at a stable URL.
- Check base paths, headers, deployment routing, and script MIME type.
- Be cautious about caching personalized HTML and framework navigation responses.
- App Router, Pages Router, framework caches, and hosting behavior differ by version.

No universal service-worker rule is safe for every Next.js route.
Test real navigations and updates on the deployed platform.

### Workbox

Workbox provides helpers for precaching, routing, strategies, expiration,
and some background-sync workflows.
It reduces repeated implementation work, not the need to understand lifecycle,
privacy, or update compatibility.

### PWA

A Progressive Web App is a web experience with selected app-like capabilities.
A service worker can provide offline/background behavior; a web app manifest
describes installation-related metadata.

A manifest alone does not provide offline support.
Installability rules vary by browser, so do not claim every PWA must satisfy one
fixed checklist everywhere.

## 14. Debugging and testing

### DevTools checklist

In browsers offering these tools, inspect:

1. Registration scope and worker script URL.
2. Installing, waiting, and active workers.
3. Whether this page has a controller.
4. Worker console errors.
5. Cache Storage names and entries.
6. Network response source and timing.
7. Offline and update behavior.

Disabling HTTP cache is not the same as disabling service-worker interception.
An ordinary hard reload does not reliably remove registrations or Cache Storage.

### Essential test cases

| Scenario | Expected behavior |
| --- | --- |
| First-ever visit offline | No false promise of previously downloaded content |
| Offline after successful setup | Approved fallback works |
| API returns 401, 404, or 500 | Correct status handling, not fake success |
| New worker installation fails | Existing working version remains usable |
| Update with two tabs open | No unexpected broken navigation |
| Update with unsaved form | No data loss from forced reload |
| Old lazy-loaded chunk requested | Supported compatibility behavior |
| Storage full or evicted | Clear failure/recovery |
| Logout while request is running | No old-user data repopulation |
| Retry of queued action | No duplicate application of an operation |
| Browser lacks sync/push support | Usable fallback experience |
| Worker rollback | Compatible recovery |

Run real-browser tests on localhost or HTTPS.
Unit tests help test logic but do not prove the browser lifecycle works.
Also test slow connections, real server errors, existing sessions, and supported
mobile platforms.

### Metrics

Track installation failures, cache write failures, fallback usage, update
completion, chunk errors, queue age/retry failures, and user-visible latency.
Avoid recording private response bodies or sensitive messages.

## 15. Interview questions and answers

### Basics

#### Q1. What is a service worker?

An event-driven browser script separate from the page. It can handle requests
from controlled clients and supported background events.

**Follow-up:** Does it run forever?

No. The browser can stop it when idle.

#### Q2. Why would you use one?

For planned offline behavior, application-managed caching, push, or supported
background retry. Do not add one merely because the app uses React.

#### Q3. How is it different from a Web Worker?

A Web Worker mainly helps a page perform computation away from the UI thread.
A service worker has a registration/lifecycle and can handle network/background
events for controlled clients.

#### Q4. Can it access the DOM?

No. Send messages to a page and let page code update the DOM.

#### Q5. Can it use `localStorage`?

No. Use appropriate worker-accessible storage such as IndexedDB or Cache Storage.

#### Q6. Why is HTTPS required?

A worker can influence requests across visits. Secure delivery helps prevent
network attackers from replacing its code. Localhost has a development exception.

#### Q7. Can it be loaded from a CDN on another origin?

The registered script must be same-origin. Imported dependencies and other
requests have their own rules; they do not make cross-origin registration valid.

#### Q8. Is a service worker required for every PWA?

Browser requirements vary. It is useful for offline/background features, but
installability and manifests are separate concerns.

#### Q9. Does registering one make the first page offline-ready immediately?

No. Installation, activation, control, and necessary downloads must happen first.

#### Q10. What is scope?

The URL range of client pages the registration can control.
It is not simply the set of asset paths the worker may see.

### Lifecycle and updates

#### Q11. Explain install, waiting, and activate.

Install prepares the worker. Waiting avoids replacing a worker still used by
clients. Activate performs handover/setup before normal event handling.

#### Q12. What happens if required precaching fails during installation?

If the installation promise rejects, the new worker does not successfully install.
An existing older worker can keep working.

#### Q13. Why is a new worker stuck waiting?

The previous worker may still control an open tab.
Check all clients rather than repeatedly refreshing only one page.

#### Q14. What does `skipWaiting()` do?

It asks the new worker to activate without the normal wait for old clients.
It does not update the JavaScript already running in those pages.

#### Q15. What does `clients.claim()` do?

It lets an active worker control eligible existing clients.
It is different from making a waiting worker active.

#### Q16. Does `serviceWorker.ready` mean the page is controlled?

No. It indicates an active registration is available.
Check `navigator.serviceWorker.controller` for current page control.

#### Q17. What is `controllerchange`?

An event on the page's service-worker container indicating its controller changed.
Use it carefully in update flows; unconditional reloads can lose work or loop.

#### Q18. How does the browser detect an update?

It checks the registered worker script and relevant dependencies under update
rules. Changed bytes can create a new worker.
`registration.update()` requests a check, not instant takeover.

#### Q19. Should the worker script URL change on every deployment?

Usually keep a stable URL and change the script content.
Version cached assets and metadata rather than creating uncontrolled registration
and rollback complexity.

#### Q20. Why not immediately delete every old cache?

Old tabs may still need old assets, especially lazy-loaded chunks.
Some caches may belong to other applications. Cleanup needs ownership and
compatibility rules.

### Fetch and caching

#### Q21. What does `respondWith()` do?

It tells the browser which response to use for a fetch event.
Call it synchronously in the handler and pass an asynchronous response promise.

#### Q22. What does `waitUntil()` do?

It extends an event's lifetime for associated promise work.
It does not turn the worker into a permanent process.

#### Q23. Does `fetch()` reject for 404 or 500?

Normally no. It resolves with an HTTP response.
Check its status/`ok` when your policy needs to distinguish errors.

#### Q24. What is cache-first?

Use a saved response; fetch only when missing. Good for versioned static files,
dangerous for mutable data without update rules.

#### Q25. What is network-first?

Try fetch and use an approved saved fallback on suitable failure.
Define timeout and HTTP error behavior; do not call every failure "offline."

#### Q26. What is stale-while-revalidate?

Return a saved copy quickly and refresh in the background.
Use it only when temporary old content is acceptable.

**Follow-up:** Is it the same as the HTTP header?

No. Application Cache Storage logic and HTTP caching are separate mechanisms.

#### Q27. What is precaching versus runtime caching?

Precache known files during setup. Runtime-cache selected responses as requests
occur. Required precaching should stay small and reliable.

#### Q28. Does Cache Storage follow `max-age` automatically?

Not as an application expiration policy. Your code must decide whether a matched
entry may be returned and when to delete/update it.

#### Q29. Why use `Response.clone()`?

The response body is a stream. Clone before consuming it when you need both a
readable returned response and a separately stored copy.

#### Q30. What is an opaque response?

A response whose details are hidden from JavaScript, commonly from a no-CORS
cross-origin request. You cannot inspect its true status or body.
Avoid caching it broadly without understanding the risks.

#### Q31. Does a service worker bypass CORS?

No. It cannot simply read a cross-origin private response that page code is not
allowed to read.

#### Q32. Why is the UI stale after disabling the HTTP cache?

The worker may still return Cache Storage data, or React/router state may hold
another copy. Isolate the actual layer serving the value.

### Background features and communication

#### Q33. How can a page communicate with a worker?

Use `postMessage` and message events; MessageChannel can help with replies.
Validate message types/data and handle missing controllers.

#### Q34. What happens to global variables when the worker stops?

They are lost. Persist necessary state and reload it when handling events.

#### Q35. Can a service worker receive push with no page open?

Supported platforms can wake it for push. Permission and platform rules still
apply, and delivery is not an immediate guaranteed channel.

#### Q36. How should notification permission be requested?

Explain the benefit and ask after a meaningful user action. Handle denial.
Do not repeatedly prompt or expose sensitive lock-screen information.

#### Q37. What does background sync solve?

It can retry registered suitable work when the browser allows it.
Support and scheduling are limited; provide a normal-page retry path.

#### Q38. Does background sync guarantee exactly-once execution?

No. Use durable queue state, safe retries, and server-side idempotency.
Only mark an item complete after confirmed success.

#### Q39. Can periodic sync replace a backend scheduler?

No. Support, permission, timing, and browser activity constraints prevent treating
it as a reliable cron job.

#### Q40. Should a payment be replayed automatically when online?

Not as a generic offline feature. It needs explicit product requirements,
current authorization/validation, idempotency, and a clear user-facing status.

### Security, frameworks, and scenarios

#### Q41. Is Cache Storage safe for secret data?

It is not a secret vault. Same-origin code may access it, and data can persist.
Avoid unnecessary private storage and protect against XSS.

#### Q42. Does `no-store` prevent all manual service-worker storage?

Do not rely on it to enforce explicit Cache Storage code.
The application's caching policy must honor sensitive-data requirements.

#### Q43. What should happen on logout?

Invalidate the session, clear scoped UI/storage/queue state, handle in-flight
requests, and coordinate other tabs. Server authorization remains essential.

#### Q44. What if storage is full or the cache is evicted?

Handle failed writes and missing entries visibly. Use network or explicit
fallback/error behavior. Do not promise permanent offline availability.

#### Q45. How do you add one to React or Next.js?

Register from browser startup code, serve a same-origin script, and connect update
messages to the UI. Check scope, framework routes, hosting, and private responses.

#### Q46. What does Workbox give you?

Reusable tools for caching/routing and related workflows.
It does not decide which data may be saved or solve all update compatibility.

#### Q47. Why does a new deployment break an old tab?

Its page code may ask for assets that were removed or replaced, or a new worker
may be incompatible with the old page. Retain versioned assets and test mixed
versions deliberately.

#### Q48. How do you remove a service worker safely?

Find the intended registration and unregister it; stop registering it again.
Existing controlled clients may remain controlled until their lifecycle ends,
and unregistering does not automatically delete Cache Storage.
Plan cache cleanup and reload/migration without disrupting unrelated apps.

#### Q49. What would you test before releasing an update?

Offline first/return visits, installation failure, two tabs, unsaved forms,
old chunks, logout, quota failures, supported browsers, and rollback.
Use real-browser lifecycle tests.

#### Q50. When would you choose not to use a service worker?

When HTTP caching already meets the need, offline/background features are not
required, or update/privacy complexity outweighs measured benefits.
The simplest suitable architecture is usually the better starting point.

## 16. Practice design exercise

Design a banking app with offline public help and push notifications.
Private account APIs must stay online and must not be saved for offline reuse.

Explain:

1. Registration and minimum necessary scope.
2. Public-resource allowlist and sensitive-route exclusions.
3. Offline page and clear transaction status.
4. Update prompt, unsaved forms, and old-tab compatibility.
5. Notification permission and private-content rules.
6. Logout, multiple tabs, and in-flight work.
7. Unsupported-browser behavior.
8. Tests, monitoring, and rollback.

## 17. Quick revision sheet

- Registration, activation, and page control are different.
- A service worker is event-driven and may stop when idle.
- No DOM, `window`, or `localStorage`.
- HTTPS normally required; localhost is suitable for development.
- Scope determines controlled clients.
- Cache Storage is not the HTTP cache.
- `fetch()` does not reject ordinary HTTP error responses.
- `respondWith()` supplies a response; `waitUntil()` tracks event work.
- `skipWaiting()` and `clients.claim()` have different roles.
- Updates must protect old tabs and unsaved work.
- Push/sync behavior depends on platform support and permissions.
- Queued is not the same as sent.
- Private data and financial operations require explicit policies.

### Interview answer in 30 seconds

> A service worker is an event-driven browser script separate from the page.
> It can handle requests from controlled pages and supported push or sync events.
> I would use it for clearly defined offline or background features, choose
> caching strategies per resource, and test its install/update lifecycle.
> Privacy, old-tab compatibility, and truthful failure states are essential.

## References

- [MDN: Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [MDN: Using service workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers)
- [web.dev: service-worker lifecycle](https://web.dev/articles/service-worker-lifecycle)
- [MDN: Cache API](https://developer.mozilla.org/en-US/docs/Web/API/Cache)
- [MDN: Background Synchronization API](https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API)
- [MDN: Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
- [Workbox documentation](https://developer.chrome.com/docs/workbox/)
