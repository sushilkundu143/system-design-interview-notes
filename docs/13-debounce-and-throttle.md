# Debounce and Throttle: A Simple, Detailed Guide

[All guides](../README.md) | [Performance](07-performance.md) |
[Web Workers](12-web-workers.md)

## How to read this guide

Start with the two everyday examples. Then compare the use cases, read the
small implementations, and practice the interview questions.

Examples demonstrate specific timing policies, not every production option.
Delays such as 300 ms are starting points to measure, not universal rules.

## Contents

- [1. Why do we need them?](#1-why-do-we-need-them)
- [2. Debounce: wait until activity pauses](#2-debounce-wait-until-activity-pauses)
- [3. Throttle: limit the rate](#3-throttle-limit-the-rate)
- [4. Which one should you choose?](#4-which-one-should-you-choose)
- [5. Leading, trailing, and maxWait](#5-leading-trailing-and-maxwait)
- [6. Simple debounce implementation](#6-simple-debounce-implementation)
- [7. Simple throttle implementation](#7-simple-throttle-implementation)
- [8. Practical library examples](#8-practical-library-examples)
- [9. React and asynchronous requests](#9-react-and-asynchronous-requests)
- [10. Alternatives and important limitations](#10-alternatives-and-important-limitations)
- [11. Testing checklist](#11-testing-checklist)
- [12. Interview questions and answers](#12-interview-questions-and-answers)
- [13. Quick revision and practice](#13-quick-revision-and-practice)

## 1. Why do we need them?

Events can happen much faster than useful work needs to run:

- Each character typed can trigger a search.
- Scrolling can trigger many events.
- Resizing a window can repeatedly trigger calculations.

Running expensive work for every event can waste requests and make the UI slow.

Debounce and throttle control **when or how often the function runs**.
They do not make the function's internal work cheaper.

## 2. Debounce: wait until activity pauses

Debounce delays execution until events stop for a specified time.
Each new event resets the timer.

**Everyday example:** An elevator waits before closing its door. Each new passenger
entering restarts the waiting period.

### Search example

Suppose a user types a character every 100 ms:

```text
Time:        0    100   200   300   400             700 ms
Text:        R    Re    Rea   Reac  React
Search:                                           "React"
```

With a 300 ms trailing debounce, the search starts 300 ms after the final event.
If the user pauses longer than 300 ms halfway through, an earlier search can
also start. Debounce does not promise exactly one request per typing session.

### Where to use it

| Use case | Why debounce helps |
| --- | --- |
| Search suggestions | Search after a typing pause |
| Username availability | Avoid checking every character |
| Draft autosave | Save after a pause |
| Expensive resize recalculation | Recalculate after resizing settles |
| Multiple quick filter changes | Apply the latest combination after a pause |

### Important limitation

If activity never stops, a simple trailing debounce can keep waiting forever.
For autosave, use a suitable maximum-wait policy or another periodic save mechanism.

Debounce the expensive search/save, not the immediate input value.
Users should still see their typing immediately.

## 3. Throttle: limit the rate

Throttle limits execution during repeated events, according to its configured
leading/trailing policy.

**Everyday example:** A status board updates periodically rather than redrawing
for every tiny change.

### Scroll example

```text
Scroll events: |||||||||||||||||||||||||||||||||||
Updates:       ^         ^         ^         ^
                   at a controlled rate
```

With a 200 ms interval, updates happen periodically during continued activity.
Exact timing depends on the implementation and browser scheduling.

### Where to use it

| Use case | Why throttle helps |
| --- | --- |
| Scroll-position display | Update periodically while scrolling |
| Pointer movement processing | Limit expensive calculations |
| Live resize preview | Continue updating at a controlled rate |
| Interaction telemetry | Limit sends during frequent events |
| Progress display | Avoid updating for every tiny progress change |

Some trailing implementations preserve the latest event for a final update.
Others, including the minimal implementation below, drop calls during the wait.
State the policy before discussing expected results.

## 4. Which one should you choose?

| Question | Debounce | Throttle |
| --- | --- | --- |
| Memory aid | Wait until quiet | Limit the rate |
| During continuous activity | May keep waiting | Continues periodically |
| New events reset a quiet-period timer | Yes | Not in the same way |
| Common example | Search input | Scroll updates |
| Typical goal | Use the final value after a pause | Keep responding during activity |

Decision examples:

- "Search when the user pauses." -> Debounce.
- "Update coordinates while dragging." -> Throttle or frame-aligned rendering.
- "Detect when an element becomes visible." -> Consider IntersectionObserver.
- "Update a visual before the next paint." -> Consider requestAnimationFrame.
- "Reject excessive API requests from any client." -> Server-side rate limiting.

## 5. Leading, trailing, and maxWait

### Leading

Execute at the beginning of a burst. This provides an immediate response.

### Trailing

Execute after the relevant waiting period, typically using the latest arguments.

### maxWait

Limit how long a debounce can postpone execution during continued calls.
Useful when a draft should eventually be saved even during continuous typing.

### Example configurations

| Task | Possible policy |
| --- | --- |
| Search | Trailing debounce |
| Autosave | Trailing debounce plus maxWait |
| Scroll indicator | Leading and trailing throttle |
| Immediate action with repeated clicks ignored | Leading-only policy, plus proper pending-state handling |

Leading/trailing interactions differ by implementation.
For example, enabling both does not necessarily make a single isolated call
execute twice. Read and test the library's actual behavior.

### Cancel and flush

- **Cancel:** discard a pending execution/reset timing state.
- **Flush:** execute pending work immediately, if supported.

Flushing on unmount is not automatically safe for autosave. Authentication,
validation, request completion, and error handling still matter.

## 6. Simple debounce implementation

This implementation is **trailing-only** and supports cancellation.
It does not implement leading calls, maxWait, or flush.

```js
function debounce(fn, delay) {
  let timer;

  function debounced(...args) {
    const context = this;
    clearTimeout(timer);
    timer = setTimeout(() => {
      timer = undefined;
      fn.apply(context, args);
    }, delay);
  }

  debounced.cancel = () => {
    clearTimeout(timer);
    timer = undefined;
  };

  return debounced;
}
```

Usage:

```js
const searchAfterPause = debounce((query) => {
  console.log("Search:", query);
}, 300);

searchAfterPause("R");
searchAfterPause("Re");
searchAfterPause("React");
// One call with "React" after approximately 300 ms of inactivity.
```

### Why preserve arguments and `this`?

The eventual call should receive the latest arguments and the intended receiver.
`fn.apply(context, args)` preserves that receiver for a normal function.
Arrow functions retain their own lexical `this`.

The wrapper does not return a promise for each future execution.
Do not assume `await searchAfterPause(...)` waits for a delayed search.
Async callbacks must handle their failures; this timer wrapper does not do that.

## 7. Simple throttle implementation

This implementation is **leading-only**.
It runs immediately, then drops calls during the interval.
It does not produce a final trailing call.

```js
function throttle(fn, interval) {
  let timer;

  function throttled(...args) {
    if (timer !== undefined) return;
    timer = setTimeout(() => {
      timer = undefined;
    }, interval);
    fn.apply(this, args);
  }

  throttled.cancel = () => {
    clearTimeout(timer);
    timer = undefined;
  };

  return throttled;
}
```

Example with a 200 ms interval:

```text
Call at 0 ms:   run
Call at 50 ms:  drop
Call at 150 ms: drop
After timer completes:
Next call:     run
```

If the final event happened at 150 ms, this implementation does not run it later.
Choose a trailing-capable implementation when preserving the final value matters.

These small implementations assume valid function/delay inputs.
Use a tested library when production requirements include more options and edge cases.

## 8. Practical library examples

The following assumes Lodash is already installed and imported.
It is not a dependency installation instruction.

```js
import debounce from "lodash/debounce.js";
import throttle from "lodash/throttle.js";

// Placeholder domain functions must implement their own error handling.
const search = debounce(fetchSuggestions, 300);

const updateScroll = throttle(handleScroll, 200, {
  leading: true,
  trailing: true,
});

const saveDraft = debounce(persistDraft, 1000, {
  maxWait: 5000,
});
```

The save policy schedules work; it does not guarantee that writes complete in order.
For overlapping saves, use suitable sequencing or server version checks.

Library throttle implementations can use debounce internally with maxWait.
Their boundary behavior may differ from the minimal throttle in section 7.

## 9. React and asynchronous requests

### Common React mistakes

1. Creating a new debounced function on every render.
2. Capturing old state in a delayed callback.
3. Forgetting to cancel timers on cleanup.
4. Delaying the controlled input itself.
5. Letting old API responses overwrite new ones.

Each newly created debounce instance has its own timer.
Instances created across renders do not automatically share one quiet period.

### Simple React search example

This example uses an effect-based delay rather than a debounce utility.
It assumes `/api/search` returns an array of `{ id: string, title: string }`.
It includes cancellation, visible errors, and protection from outdated results.

```jsx
import React, { useEffect, useState } from "react";

export default function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [status, setStatus] = useState("Type to search.");

  useEffect(() => {
    const trimmed = query.trim();
    let active = true;
    const controller = new AbortController();
    setResults([]);

    if (!trimmed) {
      setStatus("Type to search.");
      return () => {
        active = false;
        controller.abort();
      };
    }

    setStatus("Waiting for a typing pause.");
    const timer = setTimeout(async () => {
      setStatus("Searching.");
      try {
        const response = await fetch(
          `/api/search?q=${encodeURIComponent(trimmed)}`,
          { signal: controller.signal }
        );
        if (!response.ok) {
          throw new Error(`Search request failed: ${response.status}`);
        }
        const data = await response.json();
        if (
          !Array.isArray(data) ||
          !data.every(
            (item) =>
              item !== null &&
              typeof item.id === "string" &&
              typeof item.title === "string"
          )
        ) {
          throw new Error("Search returned an invalid response.");
        }
        if (!active) return;
        setResults(data);
        setStatus(`${data.length} results.`);
      } catch (error) {
        if (!active && controller.signal.aborted) return;
        console.error("Search failed:", error);
        if (active) setStatus("Search failed. Please try again.");
      }
    }, 300);

    return () => {
      active = false;
      clearTimeout(timer);
      controller.abort();
    };
  }, [query]);

  return (
    <section>
      <label htmlFor="search-query">Search</label>
      <input
        id="search-query"
        value={query}
        onChange={(event) => setQuery(event.target.value)}
      />
      <p role="status">{status}</p>
      <ul>
        {results.map((result) => (
          <li key={result.id}>{result.title}</li>
        ))}
      </ul>
    </section>
  );
}
```

What happens:

1. The input updates immediately.
2. Each query change cleans up the previous timer/request.
3. A request starts after the pause.
4. Aborted/outdated work cannot apply a successful result to the current UI.
5. Genuine failures are logged and shown.

Aborting a fetch does not guarantee the server stops processing it.
The `active` guard handles result ownership separately.
Use repository-standard notifications/monitoring in a real app.

### Autosave needs additional care

Two saves can finish out of order. Debounce reduces how often saves start, not
which write becomes final.

Consider serialized writes, version/ETag checks, and clear queued/saving/saved/error
states. Do not claim content is saved before the server confirms it.

## 10. Alternatives and important limitations

### requestAnimationFrame

Useful for scheduling visual updates around rendering opportunities.
Use a pending-frame guard to coalesce many events into one scheduled update.
Registering a new callback for every event does not automatically coalesce them.

It is not a fixed millisecond throttle and may pause in background tabs.
A heavy callback can still block rendering.

### IntersectionObserver

Useful for visibility-driven behavior such as lazy loading or an infinite-scroll
sentinel. Often simpler than repeatedly calculating positions on scroll.

Still guard against overlapping loads and handle loading errors.

### Web Workers

If each permitted call still performs heavy independent CPU work, a worker may
help. Debounce/throttle control frequency; workers move suitable computation
away from the main thread.

### Payment buttons

Do not treat debounce as payment correctness.
Track pending submission in the UI and enforce server-side idempotency so retries
cannot apply the same operation twice.

### Server-side rate limiting

Client-side throttle can be bypassed.
The backend must enforce limits independently for protection and fairness.

### Timer delays are not exact

Busy main threads and background-tab policies can delay execution.
Do not use these timers for precise deadlines or guaranteed background jobs.

## 11. Testing checklist

Use fake timers for deterministic timing tests, plus integration tests for UI
cleanup and asynchronous behavior.

| Test | What to verify |
| --- | --- |
| Debounce burst | One trailing call with latest arguments |
| Two separated bursts | Both allowed executions occur |
| Continuous calls | Basic debounce waits; maxWait policy eventually runs |
| Leading-only throttle | First runs; calls in wait window are dropped |
| Trailing throttle | Final value follows documented library behavior |
| Cancel | Pending call is discarded and state resets |
| Flush | Pending call runs once if supported |
| Receiver/arguments | Intended `this` and arguments survive |
| Component rerender | Debounce is not unintentionally recreated |
| Component unmount | No delayed owned work updates unmounted UI |
| Out-of-order responses | Old data cannot replace new results |
| Failed request/save | Visible error, not fake success |

At exact timer boundaries, event ordering matters.
Test the chosen implementation rather than assuming all throttle functions behave
identically.

## 12. Interview questions and answers

### Q1. What is debounce?

Delay execution until activity pauses for the specified period.
Each new event resets the quiet-period timer.

### Q2. What is throttle?

Limit execution during repeated activity according to an interval and
leading/trailing policy.

### Q3. What is their main difference?

Debounce normally waits for a pause; throttle continues at a controlled rate.

### Q4. Which would you choose for search?

Usually trailing debounce. Update typing immediately, and delay only the search.

### Q5. Which would you choose for scrolling?

Throttle for periodic work, requestAnimationFrame for suitable visual updates,
or IntersectionObserver for visibility detection.

### Q6. Can debounce guarantee one request while typing?

No. A long enough pause can start a request before typing resumes.
It also does not coordinate different users or components.

### Q7. What happens during continuous typing?

A simple trailing debounce can wait indefinitely.
Use maxWait or a separate periodic policy when work must eventually run.

### Q8. What are leading and trailing calls?

Leading happens at the start of a burst. Trailing happens after the relevant
waiting period, often using the latest arguments.

### Q9. Does enabling both always execute twice?

No. Behavior depends on the implementation and whether calls repeat.
Read the library's rules and test isolated calls as well as bursts.

### Q10. What is maxWait useful for?

Preventing continuous activity from postponing debounced work forever.
Autosave is a common example.

### Q11. What is cancel versus flush?

Cancel discards pending work; flush executes it immediately when supported.
Choose explicitly during cleanup rather than assuming saving on unmount is safe.

### Q12. Why preserve arguments and `this`?

The delayed call should use the intended receiver and latest values.
Otherwise a method can operate on the wrong object or stale input.

### Q13. Why can a React debounce fail?

Recreating it on every render creates independent timers.
Stale closures and missing cleanup are other common causes.

### Q14. How do you avoid stale closures?

Pass current values as arguments, use correct dependencies, or an appropriately
managed callback ref. Do not freeze a callback with incomplete dependencies.

### Q15. Does debounce prevent stale search results?

No. Older requests can finish later.
Abort unnecessary work and/or use result ownership IDs/guards.

### Q16. Is aborting fetch enough to stop backend work?

No. The server may continue. Protect displayed-result ownership and make
backend operations safe independently.

### Q17. How would you debounce autosave?

Delay after a pause, consider maxWait, and coordinate overlapping writes.
Show saved only after confirmation, and handle failure/retry.

### Q18. Would you debounce a payment button?

Not as the guarantee against duplicates.
Use pending-state UX and server-side idempotency.

### Q19. Is browser throttle the same as rate limiting?

No. Browser scheduling improves UX/resource use.
Server limits enforce protection and cannot trust client behavior.

### Q20. Does throttle fix an expensive function?

Only its frequency. Improve the algorithm, reduce work, or use a suitable worker
if each execution still blocks the UI.

### Q21. How do you choose a delay?

Balance response expectations, operation cost, traffic, and measurements.
Do not assume 300 ms is correct for every action.

### Q22. Are timer delays exact?

No. Execution waits until browser scheduling permits it.
Background tabs can have additional restrictions.

### Q23. How do you test debounce and throttle?

Use fake timers for timing/arguments/options, then integration tests for cleanup,
request races, and errors.

### Q24. Can you await a debounced function?

Not automatically. A simple timer wrapper does not return a promise for each
eventual execution. Define async result/error semantics or use an appropriate
library abstraction.

### Q25. What is a common throttle edge case?

A leading-only throttle can discard the final event.
If the final position/value matters, choose and test trailing behavior.

## 13. Quick revision and practice

- Debounce: wait until quiet.
- Throttle: control the rate.
- State the leading/trailing policy.
- Use maxWait when continuous activity must not postpone work forever.
- Keep input feedback immediate.
- Cancel owned timers and handle async failures.
- Debounce does not order API responses or database writes.
- Client-side scheduling does not enforce server correctness.

### Practice exercise

Design a product search with a controlled input, debounced requests, latest-result
protection, accessible status messages, errors, and an infinite-scroll sentinel.
Explain why each part uses debounce, an observer, or neither.

### Interview answer in 30 seconds

> Debounce waits for a pause before running work, while throttle limits how often
> work runs during continued activity. Search often uses debounce; periodic scroll
> updates may use throttle. I would choose leading/trailing behavior explicitly,
> clean up timers, and separately handle stale async results and failures.

## References

- [Lodash: debounce](https://lodash.com/docs/4.17.15#debounce)
- [Lodash: throttle](https://lodash.com/docs/4.17.15#throttle)
- [MDN: setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout)
- [MDN: requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame)
- [MDN: IntersectionObserver](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver)
- [React: useEffect](https://react.dev/reference/react/useEffect)
