# Web Workers: A Simple, Detailed Guide

[All guides](../README.md) | [Service workers](11-service-workers.md) |
[Frontend performance](07-performance.md)

## How to read this guide

1. Read sections 1-5 for the basic ideas.
2. Follow the small example in section 6.
3. Study communication, cancellation, React integration, and performance.
4. Practice the 50 interview questions, from basic to advanced.

This guide focuses on **Web Workers**. Service workers are a different type of
worker with a different purpose; their interview questions have a
[separate guide](11-service-workers.md).

No document can cover every possible interview question. This guide covers the
main concepts, common follow-ups, and realistic design scenarios.
Code examples are learning examples, not complete production applications.

## Contents

- [1. What is a Web Worker?](#1-what-is-a-web-worker)
- [2. Why the main thread becomes busy](#2-why-the-main-thread-becomes-busy)
- [3. What workers can and cannot do](#3-what-workers-can-and-cannot-do)
- [4. Different worker types](#4-different-worker-types)
- [5. Worker lifecycle and communication](#5-worker-lifecycle-and-communication)
- [6. Small working example](#6-small-working-example)
- [7. Copying, transferring, and sharing data](#7-copying-transferring-and-sharing-data)
- [8. Errors, progress, and cancellation](#8-errors-progress-and-cancellation)
- [9. React and Next.js integration](#9-react-and-nextjs-integration)
- [10. Choosing tasks and measuring performance](#10-choosing-tasks-and-measuring-performance)
- [11. Pools, queues, and memory](#11-pools-queues-and-memory)
- [12. Security, loading, and browser support](#12-security-loading-and-browser-support)
- [13. Advanced features in simple terms](#13-advanced-features-in-simple-terms)
- [14. Debugging and testing](#14-debugging-and-testing)
- [15. Interview questions and answers](#15-interview-questions-and-answers)
- [16. Practice design exercise](#16-practice-design-exercise)
- [17. Quick revision sheet](#17-quick-revision-sheet)

## 1. What is a Web Worker?

A Web Worker lets a browser run JavaScript in a separate execution thread,
away from the page's main thread.

**Everyday example:** A restaurant cashier handles customers while a chef prepares
a large order. If the cashier also cooks every order, the queue stops moving.

- Main thread = the cashier handling the interface.
- Worker = the chef doing a separate task.
- Messages = order slips and completed-order notifications.

```text
Main thread                         Worker
Show UI                             Wait for a task
Send calculation -----------------> Calculate
Keep handling user input
Receive result <------------------- Send result
Update UI
```

The worker cannot update the page directly. It sends the result back, and page
code decides what to show.

### Examples

- Parse or transform a large file.
- Filter a large local dataset.
- Perform expensive image processing.
- Run suitable compression or mathematical calculations.

A worker improves responsiveness by moving suitable work away from the UI.
It does not automatically make every calculation finish faster.

## 2. Why the main thread becomes busy

In a page, JavaScript and much UI work compete for the main thread.
A long synchronous calculation can delay input handling and rendering.

```js
// Doing a large calculation directly in a click handler can block the UI.
function calculateTotal(limit) {
  let total = 0;
  for (let number = 1; number <= limit; number += 1) {
    total += number;
  }
  return total;
}
```

An **event loop** is the mechanism that schedules JavaScript tasks and other
work. A worker has its own event loop; it is not a function secretly running
inside the page's event loop.

### Why `async` does not solve CPU-heavy work

```js
async function stillRunsOnMainThread(limit) {
  return calculateTotal(limit);
}
```

This still performs the calculation on the calling thread.
`async` changes how a function returns and waits; it does not create a new thread.

Promises do not move CPU work to a worker either.
Even repeatedly yielding to resolved promises can starve rendering because
microtasks are processed before the next task/render opportunity.

### Why `setTimeout` is different

A timer can schedule work later on the same thread.
Splitting work into small tasks can help the browser respond between chunks,
but each chunk still competes with the UI.

A worker is an option when there is enough independent computation to justify it.
First consider a better algorithm or reducing the amount of work.

## 3. What workers can and cannot do

| Usually available in suitable workers | Not available like page code |
| --- | --- |
| JavaScript computation | Direct DOM access |
| `fetch()` | `document` and page `window` |
| Timers and messages | React component state |
| IndexedDB | `localStorage` |
| Supported binary-data APIs | Synchronous calls into page functions |
| Selected canvas/graphics APIs where supported | Every browser API |

Available APIs depend on worker type and browser support.
The worker global is commonly accessed as `self`.
The main page and worker do not share ordinary objects automatically.

### Network requests do not usually need a worker

Browser `fetch()` already waits asynchronously.
Move a request to a worker when its surrounding processing benefits from it,
not simply because network access sounds like "background work."

## 4. Different worker types

| Type | Simple purpose | Lifetime/control |
| --- | --- | --- |
| Dedicated Worker | Work for the context that created it | Created with `new Worker()` |
| Shared Worker | Communicate with multiple compatible same-origin clients | Clients connect through ports |
| Service Worker | Handle controlled-client requests and supported background events | Registration/install/activation lifecycle |

**Dedicated Worker:** one dashboard sends it an expensive calculation.

**Shared Worker:** several tabs coordinate a suitable shared connection or task.
Support, browser partitioning, origin, and worker identity matter; do not assume
every tab always joins one universal worker.

**Service Worker:** provide a planned offline response or receive supported push.
It is not the usual choice for a long page-related calculation.

### Classic versus module workers

These are script-loading modes, not additional lifecycle types.

- Classic worker: default mode; can use `importScripts()`.
- Module worker: uses ES module `import`/`export`; created with `{ type: "module" }`.

Example for a supported bundler setup:

```js
const worker = new Worker(new URL("./calculate-worker.js", import.meta.url), {
  type: "module",
});
```

The bundler must understand this syntax and emit a usable worker asset.
Do not assume every React/Next.js version or build configuration handles it identically.

## 5. Worker lifecycle and communication

```text
Create worker
  -> script loads
  -> worker waits for messages
  -> page sends a task
  -> worker performs work
  -> worker sends result or error
  -> page updates UI
  -> owner terminates worker when appropriate
```

There is no dedicated-worker install/waiting/activate lifecycle like a service worker.

### The main methods/events

| API | Meaning |
| --- | --- |
| `new Worker(url)` | Create a dedicated worker |
| `worker.postMessage(data)` | Send data from page to worker |
| `self.postMessage(data)` | Send data from dedicated worker to page |
| `message` | Receive a message |
| `error` | Report an uncaught worker/script error |
| `messageerror` | Report a message that cannot be deserialized |
| `worker.terminate()` | Stop the dedicated worker from its owner |
| `self.close()` | Request closing from inside the dedicated worker |

Do not confuse an application message `{ type: "ERROR" }` with the browser's
`error` event. Handle both.

### Use a message contract

Agree on shapes before writing both sides:

```text
Request: { id, type: "SUM", limit }
Result:  { id, type: "RESULT", total }
Failure: { id, type: "ERROR", message }
```

An ID connects the response to the request.
It also helps detect stale results, cancellations, and overlapping jobs.
TypeScript types help developers, but incoming data still needs runtime checks
where the boundary requires them.

## 6. Small working example

This example sends a bounded summation task to a worker.
The calculation is deliberately simple to explain the mechanics. In real code,
this sum has a constant-time formula; improving the algorithm would be better
than adding a worker just for summation.

Serve the following files from the same website using an HTTP development server.
Do not rely on opening them directly with `file://`.

### Page HTML

```html
<button id="calculate" type="button">Calculate in worker</button>
<p id="status" role="status" aria-live="polite">Ready</p>
<script src="/main.js" defer></script>
```

### Worker file: `/sum-worker.js`

```js
self.addEventListener("message", (event) => {
  const data = event.data;
  const id = typeof data?.id === "string" ? data.id : null;

  if (
    id === null ||
    data.type !== "SUM" ||
    !Number.isSafeInteger(data.limit) ||
    data.limit < 1 ||
    data.limit > 5_000_000
  ) {
    self.postMessage({
      id,
      type: "ERROR",
      message: "Expected SUM with a string id and a limit from 1 to 5000000.",
    });
    return;
  }

  let total = 0;
  for (let number = 1; number <= data.limit; number += 1) {
    total += number;
  }
  self.postMessage({ id, type: "RESULT", total });
});
```

The bound limits work and keeps the result within JavaScript's safe integer range.
This is example-specific validation, not a universal five-million-item rule.

### Page file: `/main.js`

```js
const button = document.getElementById("calculate");
const status = document.getElementById("status");

function showUnavailable(message, error) {
  button.disabled = true;
  status.textContent = message;
  console.error(message, error);
}

if (!("Worker" in window)) {
  showUnavailable("This browser does not support workers.");
} else {
  try {
    const worker = new Worker("/sum-worker.js");
    let nextId = 0;
    let activeId = null;

    const failWorker = (message, error) => {
      activeId = null;
      worker.terminate();
      showUnavailable(message, error);
    };

    worker.addEventListener("message", (event) => {
      const data = event.data;
      if (activeId === null || data?.id !== activeId) return;

      if (data.type === "RESULT" && Number.isSafeInteger(data.total)) {
        status.textContent = `Result: ${data.total}`;
      } else if (data.type === "ERROR" && typeof data.message === "string") {
        status.textContent = `Calculation failed: ${data.message}`;
        console.error("Worker task failed:", data.message);
      } else {
        failWorker("Worker returned an invalid response.", data);
        return;
      }
      activeId = null;
      button.disabled = false;
    });

    worker.addEventListener("error", (event) => {
      failWorker("Worker failed. Reload to try again.", event.message);
    });
    worker.addEventListener("messageerror", (event) => {
      failWorker("Worker response could not be read.", event);
    });

    button.addEventListener("click", () => {
      if (activeId !== null) return;
      activeId = String(++nextId);
      button.disabled = true;
      status.textContent = "Calculating; the page can still handle other input.";
      try {
        worker.postMessage({ id: activeId, type: "SUM", limit: 5_000_000 });
      } catch (error) {
        failWorker("Could not send the calculation.", error);
      }
    });
  } catch (error) {
    showUnavailable("Could not create the worker.", error);
  }
}
```

### What to notice

1. The page creates one reusable worker, not one per render.
2. The page and worker communicate through messages.
3. The worker performs the loop.
4. Only the page updates the DOM.
5. Invalid requests and worker failures produce explicit errors.
6. The example allows one task at a time.

For a single page, the worker is normally tied to its owner.
For a component/route with a shorter lifetime, explicitly terminate it during
cleanup. Add an appropriate task timeout/recovery policy for production use.

## 7. Copying, transferring, and sharing data

### A. Copying: the normal message behavior

Most values sent with `postMessage()` use the **structured clone algorithm**:
a browser mechanism for creating an independent copy of supported values.

It supports more than JSON, including many binary types, dates, maps, and sets.
Functions and DOM nodes cannot normally be cloned this way.
Custom prototypes and property descriptors are not preserved as ordinary
class-instance behavior.

**Analogy:** Send a photocopy. Editing your original does not edit the received copy.

Large messages can cost time and memory.

### B. Transferring: give ownership instead of copying

```js
const buffer = new ArrayBuffer(1024);
worker.postMessage({ type: "PROCESS_BUFFER", buffer }, [buffer]);
// The sender's original buffer is now detached; do not keep using it.
```

The receiving worker needs a matching handler.

**Analogy:** Hand over the original document rather than photocopying it.
The previous owner cannot continue using the same transferred resource.

The transferable must also be reachable from the message payload if the receiver
needs to use it. Listing it only in the transfer list does not add it to the message.

For a typed array, transfer its underlying `ArrayBuffer`, not the view itself.
Multiple views of the same buffer are affected when that buffer is detached.

### C. Sharing: both sides see the same memory

`SharedArrayBuffer` lets contexts access shared memory.
It requires supported secure/cross-origin-isolated browser conditions.
It is shared, not transferred.

**Analogy:** Two people write on the same whiteboard.
They must coordinate so they do not overwrite each other's work.

Use `Atomics` and a carefully designed protocol where needed.
Start with ordinary messages unless measurement justifies shared-memory complexity.

## 8. Errors, progress, and cancellation

### Error types

| Failure | Where to handle it |
| --- | --- |
| Worker construction/security error | Around `new Worker()` |
| Script download or uncaught runtime error | Worker's `error` event |
| Uncloneable outgoing message | Around `postMessage()` |
| Incoming deserialization failure | `messageerror` event |
| Application task failure | A defined error message |
| Task takes too long | Application timeout/recovery policy |

Do not report a failed worker task as a successful empty result.

### Progress

Send progress messages at useful intervals, not after every item.
Thousands of messages can overload the main thread.
Keep live announcements understandable rather than announcing every tiny change.

### Ignoring stale results

Suppose a user types `r`, then `re`, then `react`.
A result for `r` should not replace the current results for `react`.

Keep a latest-request ID and only display responses belonging to it.
Ignoring an old result prevents incorrect UI, but does not stop its computation.

### Cancellation

- **Terminate:** immediately stop the dedicated worker; recreate it for future work.
- **Cooperative cancellation:** split work into chunks, yield between them, and
  check cancellation state.
- **Shared signal:** advanced shared-memory protocols can be checked during work.

A worker running a long synchronous loop cannot process a queued cancellation
message until that loop gives the worker's event loop a chance.
Simply sending `{ type: "CANCEL" }` does not interrupt an arbitrary loop.

If a task performs `fetch()`, an `AbortController` owned by the worker can cancel
that fetch. The main page can send a cancellation instruction; that is separate
from interrupting CPU computation.

## 9. React and Next.js integration

### React ownership pattern

1. Create the worker in an effect or a deliberately owned shared service.
2. Keep the worker reference in a ref, not ordinary component render logic.
3. Subscribe to messages/errors.
4. Send serializable data, not component functions or React objects.
5. Match responses to the latest intended request.
6. Update state on the main thread.
7. Remove listeners and terminate owned workers during cleanup.

Do not create a new worker on every render.
Do not put a React component inside a worker and expect it to update the DOM.

Development Strict Mode may run an extra setup/cleanup cycle.
Correct cleanup prevents leaked workers and duplicated subscriptions.

### A result can still block rendering

The worker may quickly calculate 100,000 rows, but rendering all those rows can
still freeze the page.

Use pagination/virtualization and small state updates where appropriate.
A worker moves computation, not every UI cost.

### Next.js

Create browser workers only on the client, not during server rendering.
`window` and browser `Worker` are not available in the server render environment.

For module workers, verify the bundler emits a loadable asset.
For public script URLs, verify the deployment base path, MIME type, and script policy.

Node.js `worker_threads` are a different server-side API.
They are not the browser `Worker` constructor.

### Worker helper libraries

Libraries such as Comlink can wrap message handling in function-like APIs.
Underlying operations still cross an asynchronous boundary.
Understand transfer, error propagation, lifetime, and cancellation rather than
assuming a helper makes remote work behave like an ordinary local function.

## 10. Choosing tasks and measuring performance

### Good candidates

- Large parsing/transformation tasks.
- Repeated analysis of data already loaded locally.
- Suitable media processing.
- Independent CPU-heavy calculations.

### Poor candidates

- Tiny calculations with more messaging overhead than work.
- DOM-heavy code.
- A normal small API request already using asynchronous fetch.
- Work that exposes private business logic/data better kept on the server.

### Measure the whole path

```text
Total time =
  worker startup, if needed
  + input transfer/copy
  + queue waiting
  + computation
  + result transfer/copy
  + UI update/rendering
```

Measure responsiveness separately from total duration.
A task might take slightly longer in a worker while the page remains much easier
to use. Conversely, a worker can harm performance if the messages are enormous.

Useful measures include input responsiveness/INP, main-thread long tasks,
task duration, queue depth, message size, and memory.

INP means **Interaction to Next Paint**: a measure of how quickly the page visibly
responds to user interactions.

## 11. Pools, queues, and memory

A **worker pool** is a bounded set of reusable workers.

**Analogy:** A fixed team of chefs rather than hiring a new chef for every order.

Benefits:

- Reuse startup cost.
- Handle independent jobs concurrently.
- Limit resources.

Responsibilities:

- Queue limits.
- Cancellation and priorities.
- Worker failure/replacement.
- Backpressure: slowing or rejecting incoming work when capacity is full.
- Task ordering and result IDs.

Do not create hundreds of workers just because you have hundreds of jobs.
CPU, memory, mobile battery, and device capability are limited.
`navigator.hardwareConcurrency` is a hint, not a command to occupy every core.

For search, keeping a reusable dataset in one worker can avoid resending the
entire dataset on every keystroke. Plan how to update or release that data.

Avoid sending huge result sets when the UI only needs a page of results.

## 12. Security, loading, and browser support

- Worker entry scripts normally need same-origin loading; module dependencies
  and imports have their own fetch/CORS rules.
- Follow Content Security Policy, including `worker-src` where applicable.
- Do not run arbitrary untrusted JavaScript just because it is in a worker.
- A worker is not a security sandbox for hostile code.
- Authentication and authorization still belong to the server.
- Do not log private payloads unnecessarily.
- Test actual deployed worker URLs and valid JavaScript responses.

Dedicated workers are broadly supported, but individual APIs and SharedWorker/
module-worker features vary. Feature-detect and test target browsers.

Unlike service-worker registration, ordinary dedicated workers do not universally
require HTTPS. HTTPS is still the correct production baseline, and particular
features such as shared memory require additional security conditions.

If workers are unavailable, consider a smaller task, chunked main-thread work,
server computation, or an explicit feature-unavailable state.
Do not silently run a huge blocking task and claim the same experience.

## 13. Advanced features in simple terms

### SharedWorker and MessagePort

Several compatible clients can connect to a shared worker.
They communicate through ports, with appropriate handlers/start behavior.

Use it only when cross-tab coordination is needed and supported.
It does not stay alive as a guaranteed permanent background server.

### SharedArrayBuffer and Atomics

Shared memory can reduce copying, but introduces race conditions.
`Atomics` provides operations to coordinate certain shared typed-array accesses.

Shared memory typically requires HTTPS and cross-origin isolation, often using
COOP/COEP headers. Check `crossOriginIsolated` and platform support.
Those headers can affect embedded third-party resources.

`Atomics.wait()` cannot block the browser main thread.
Do not add shared memory without a measured reason and a synchronization design.

### OffscreenCanvas

Some supported canvas drawing/processing can run in a worker using OffscreenCanvas.
The page still owns DOM layout and accessibility.
It is not a way to move all HTML rendering off the main thread.

### WebAssembly

WebAssembly can run supported computation in a worker.
Putting WebAssembly on the main thread can still block the UI.
Wasm threads/shared memory have additional support and isolation requirements.

### Nested workers

A worker may create other workers where supported.
Keep the number bounded and clean up the full owned task tree.

## 14. Debugging and testing

### Debugging checklist

1. Confirm the worker asset URL returns JavaScript, not an HTML router fallback.
2. Inspect network, origin, MIME, and CSP errors.
3. Open worker execution contexts in supported DevTools.
4. Add request IDs to non-sensitive diagnostics.
5. Inspect message shapes and clone/transfer errors.
6. Measure main-thread and worker work separately.
7. Verify cleanup on route changes.

### Test cases

| Scenario | Expected result |
| --- | --- |
| Small/large valid input | Correct result within resource limits |
| Invalid message | Explicit task error |
| Worker script unavailable | Visible operational failure |
| Older search finishes later | Current query remains displayed |
| Cancellation during busy loop | Designed stop/ignore behavior |
| Component unmount | Owned worker/listeners cleaned up |
| Transferred buffer reused by sender | Test detects detached ownership |
| Uncloneable payload | Send error is handled |
| Queue overload | Bounded backpressure behavior |
| Low-end device | Responsive UI and acceptable memory |
| Result renders many rows | Pagination/virtualization avoids UI freeze |
| Browser without needed feature | Planned fallback |

Pure functions can be unit tested without a browser.
Real-browser tests are needed for actual worker loading, messaging, transfer,
cleanup, and responsiveness. A mocked worker does not prove those behaviors.

## 15. Interview questions and answers

### Basics

#### Q1. What is a Web Worker?

A way to execute suitable JavaScript separately from the page's main thread.
The page communicates with it through messages.

#### Q2. What problem does it solve?

CPU-heavy page code can delay input and rendering.
A worker lets suitable computation proceed without occupying the main thread.

#### Q3. Does it make JavaScript multi-threaded?

It gives the application additional execution threads.
Ordinary code within each worker still uses its own event loop; creating a worker
does not make one function automatically execute on many cores.

#### Q4. How is it different from a service worker?

A dedicated worker usually performs page-related work.
A service worker has registration/lifecycle rules for controlled requests and
supported background events.

#### Q5. Can a worker modify the DOM?

No. Send the result to the page, which performs the DOM update.

#### Q6. Can it access React state or `localStorage`?

Not directly. Send needed state as data. Use appropriate worker-accessible storage
such as IndexedDB when persistence is required.

#### Q7. Does `async/await` replace a worker?

No. It helps manage asynchronous operations but does not move synchronous CPU
work off the calling thread.

#### Q8. Can `setTimeout` replace a worker?

It can split tasks on the same thread, which sometimes is enough.
A worker separates execution; choose based on workload and measurement.

#### Q9. Is every task faster in a worker?

No. Startup, communication, copying, and scheduling add overhead.
Responsiveness may improve even when total time does not.

#### Q10. What are good worker use cases?

Large parsing, transformations, calculations, and supported image/media work.
Small tasks and DOM-heavy operations are usually poor candidates.

### Creation and communication

#### Q11. How do you create and stop a dedicated worker?

Use `new Worker(url)`, then `worker.terminate()` when its owner no longer needs it.
The worker can call `self.close()` to close from its side.

#### Q12. How do the page and worker communicate?

Use `postMessage()` and `message` handlers.
Define request, result, and error formats.

#### Q13. Is `postMessage()` a synchronous function call into the worker?

No. It queues communication; the result arrives later.
Do not expect an ordinary immediate return value.

#### Q14. What can you send?

Values supported by structured cloning, such as plain data, arrays, maps, and
many binary types. Functions and DOM nodes are not ordinary cloneable payloads.

#### Q15. What is structured cloning?

A browser mechanism that copies supported data across contexts.
It supports more types than JSON, but does not preserve all custom object behavior.

#### Q16. Can you send a class instance with its methods?

Do not expect its custom prototype/method behavior to survive as a normal instance.
Send data and reconstruct any required behavior explicitly.

#### Q17. Why use request IDs?

To match responses with requests, ignore stale results, and track failures or
cancellations. This matters when several jobs overlap.

#### Q18. What is a module worker?

A worker loaded using ES modules, created with `{ type: "module" }`.
Use `import` rather than classic-worker `importScripts()`.

#### Q19. Why use `new URL(..., import.meta.url)`?

Supported bundlers can resolve the worker relative to the module and emit its asset.
Verify your actual build/runtime support.

#### Q20. Why did the worker fail to start?

Check its URL, JavaScript MIME response, script syntax, origin restrictions,
CSP, and build output. Some routers return HTML for a missing worker URL.

### Data and performance

#### Q21. What is a transferable object?

A resource whose ownership can be moved to another context instead of normally
copying its contents. An ArrayBuffer is a common example.

#### Q22. What happens after transferring an ArrayBuffer?

The sender's buffer is detached. Views sharing it are affected.
Design ownership so the sender does not keep using it.

#### Q23. Can you transfer a typed array?

Transfer its underlying transferable ArrayBuffer.
Include the typed array/buffer appropriately in the payload so the receiver can use it.

#### Q24. What is SharedArrayBuffer?

Memory both contexts can access. It is shared rather than transferred and requires
supported isolation/security conditions.

#### Q25. Why use Atomics?

To coordinate appropriate shared-memory operations.
Without a protocol, overlapping reads/writes can produce incorrect results.

#### Q26. What makes a worker slow?

Large cloned messages, expensive work, long queues, startup, CPU contention,
memory pressure, and large UI updates after results arrive.

#### Q27. Should you put every API request in a worker?

No. Fetch is already asynchronous. A worker is more useful for expensive processing
around the request.

#### Q28. How do you decide whether a worker helped?

Compare task duration, input responsiveness, long tasks, memory, and end-to-end
timing on realistic devices. Do not measure only the calculation loop.

#### Q29. What is a worker pool?

A bounded group of reusable workers processing queued jobs.
It needs limits, failure handling, cancellation, and backpressure.

#### Q30. How many workers should you create?

There is no universal number. Consider device CPU/memory, workload, startup cost,
and UI needs. Hardware concurrency is only a hint.

### Cancellation and reliability

#### Q31. How do you cancel a worker task?

Terminate the worker or implement cooperative cancellation.
Sending a message alone cannot interrupt a worker that is stuck in a synchronous loop.

#### Q32. Is ignoring a stale result the same as cancelling?

No. It prevents the old result from changing the UI, but computation may continue.

#### Q33. How do you keep search results correct while typing?

Assign a new ID to each query, display only the current ID, and reduce old work
through debouncing, queue limits, or cancellation.

#### Q34. How do you report progress?

Send bounded progress messages with the task ID.
Too many messages can hurt UI performance and overwhelm accessibility announcements.

#### Q35. How do you handle errors?

Handle construction/send exceptions, browser error/messageerror events, task-error
messages, and timeouts. Show an honest failure and offer a safe recovery path.

#### Q36. What happens when a worker is terminated?

It stops without a promise of finishing pending work or running cleanup code.
Do not rely on it to send a final result or persist state after termination.

#### Q37. How do you avoid memory leaks?

Terminate owned workers, remove listeners, release large datasets, and bound
queues. Repeated renders must not create untracked workers.

#### Q38. Can you abort fetch inside a worker?

Yes, with an AbortController owned in the worker.
The page can send an instruction to invoke it; worker CPU cancellation is a separate issue.

#### Q39. How should a worker recover from a crash?

Fail affected tasks explicitly, recreate it if policy allows, and retry only safe
operations. Restore necessary data and avoid infinite restart loops.

#### Q40. Does a worker guarantee completion after the page closes?

No. A dedicated worker is not a persistent backend job system.
Move required durable work to an appropriate server workflow.

### React, security, and advanced scenarios

#### Q41. Where should you create a worker in React?

In an effect or a deliberately owned service, keeping its reference outside
render-created instances. Clean up subscriptions and owned workers on unmount.

#### Q42. Why might React Strict Mode reveal duplicate work?

Development can perform an extra setup/cleanup cycle.
Missing cleanup can leave workers/listeners running.

#### Q43. Can you create a browser worker during Next.js server rendering?

No. Create it on the client.
Server-side Node worker threads use a different API and environment.

#### Q44. Why can the UI still freeze after moving computation to a worker?

Rendering a huge result, sorting again on the main thread, or heavy message
processing can still block it. Optimize the whole path.

#### Q45. What is SharedWorker useful for?

Compatible same-origin clients can coordinate through ports, such as sharing a
suitable connection. Check support and partitioning; it is not a universal
cross-tab permanent process.

#### Q46. Is a worker a safe place to run untrusted code?

No. It is not a complete security sandbox.
It can still access permitted same-origin/network resources and consume resources.

#### Q47. Does an ordinary dedicated worker require HTTPS?

Not universally as service-worker registration does.
Use HTTPS in production; some worker features require additional secure/isolation
conditions.

#### Q48. What are OffscreenCanvas and WebAssembly used for?

OffscreenCanvas can move supported canvas work into a worker.
WebAssembly can execute suitable computation there too.
Neither automatically moves DOM rendering or all application work.

#### Q49. How would you test a worker feature?

Test algorithm results and invalid messages, then use real-browser tests for loading,
messaging, ownership transfer, cancellation, cleanup, stale results, and low-end
device responsiveness.

#### Q50. When would you avoid a worker?

When a better algorithm or less data solves the problem, work is too small, DOM
access dominates, or server computation is more appropriate.
Choose the simplest measured solution.

## 16. Practice design exercise

Design a React report viewer that searches and transforms a large local dataset.

Explain:

1. Why computation belongs in a worker rather than only using `async`.
2. How the dataset is loaded once and updated.
3. Request/result/error shapes.
4. Latest-query handling and cancellation.
5. Copy versus transfer decisions.
6. Queue and memory limits.
7. Pagination/virtualization for rendering.
8. Worker cleanup, unsupported browsers, and visible failures.
9. Performance tests on low-end devices.

Do not make private data downloadable merely to demonstrate a frontend worker.
Server authorization and product privacy requirements still determine what the
browser is allowed to receive.

## 17. Quick revision sheet

- Workers move suitable computation away from the page's main thread.
- `async` and promises do not create worker threads.
- No direct DOM or React-state access.
- Communicate using messages and explicit contracts.
- Copy by default; transfer deliberately; share only with coordination.
- IDs prevent old responses from replacing current UI.
- Cancellation messages cannot interrupt an arbitrary synchronous loop.
- Reuse workers, limit queues, and clean up ownership.
- Browser workers and Node worker threads are different APIs.
- A responsive computation result can still trigger slow rendering.
- Measure total cost, responsiveness, and memory.

### Interview answer in 30 seconds

> A Web Worker runs suitable JavaScript separately from the page's main thread.
> I would use it for expensive computation that affects UI responsiveness.
> The page and worker exchange typed messages, with IDs for results and failures.
> I would control data-copy costs, cancellation, queue size, and cleanup, then
> measure the whole experience rather than assuming workers are always faster.

## References

- [MDN: Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [MDN: Using Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)
- [MDN: Transferable objects](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects)
- [MDN: Structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm)
- [MDN: SharedWorker](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker)
- [MDN: SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)
- [MDN: OffscreenCanvas](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)
- [Node.js: worker threads](https://nodejs.org/api/worker_threads.html)
