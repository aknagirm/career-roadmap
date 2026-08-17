# JavaScript — Microtasks vs Macrotasks (Event Loop Deep Dive)

> **Phase 1 | Week 3 | Topic 1 of 30**
> **Difficulty:** Intermediate
> **Time to complete:** 3–4 hours
> **Status:** ⬜ Not Started

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Why It Exists](#2-why-it-exists)
3. [Fundamentals](#3-fundamentals)
4. [Internal Working](#4-internal-working)
5. [Visual Architecture](#5-visual-architecture)
6. [Syntax & API](#6-syntax--api)
7. [Practical Examples](#7-practical-examples)
8. [Real-world Usage](#8-real-world-usage)
9. [Performance](#9-performance)
10. [Common Mistakes](#10-common-mistakes)
11. [Best Practices](#11-best-practices)
12. [Debugging](#12-debugging)
13. [Advanced Concepts](#13-advanced-concepts)
14. [Interview Questions](#14-interview-questions)
15. [Hands-on Exercises](#15-hands-on-exercises)
16. [Mini Project](#16-mini-project)
17. [Revision](#17-revision)
18. [Cheat Sheet](#18-cheat-sheet)
19. [References](#19-references)

---

## 1. Introduction

### What is a Task?

JavaScript is single-threaded — one call stack, one thing executing at a time.
Yet it handles timers, network requests, and user events without freezing. How?

When async work completes (a timer fires, a network response arrives, a user clicks),
the browser doesn't interrupt your running code. Instead, it puts a **callback** into
a queue. Once the call stack is empty, the **Event Loop** picks the next callback from
the queue and runs it. Each such picked-up callback is called a **task** (or a job).

```
Call Stack empty?
    ↓ yes
Pick next callback from queue → run it (this is one "task")
    ↓ done
Call Stack empty again?
    ↓ yes
Pick next callback → run it
    ↓ ... and so on forever
```

This queue-based callback system is what lets JavaScript appear concurrent despite
being single-threaded. Every async operation you write — `setTimeout`, `fetch`,
`addEventListener` — eventually lands a callback in this queue as a **task**.

### So what are Macrotasks and Microtasks?

As JavaScript evolved, it turned out **not all tasks are equal**. Two distinct
categories emerged, with different queues and different priorities:

---

**Macrotask (also just called "Task"):**
A task scheduled by the browser's task scheduler — typically representing work
triggered by external events or timers. The event loop picks up **one macrotask
per turn**, runs it to completion, then checks what to do next.

```
What produces macrotasks:
  setTimeout(fn, delay)      → fn is a macrotask callback
  setInterval(fn, delay)     → fn is a macrotask callback
  setImmediate(fn)           → Node.js only
  I/O callbacks              → file read, network (Node.js)
  User events                → click, keydown, scroll
  MessageChannel.postMessage → fn is a macrotask callback
```

Think of a macrotask as a **complete unit of work** — the event loop runs one,
then pauses to check if anything else needs to happen (rendering, higher-priority
work) before picking the next one.

---

**Microtask:**
A task that is scheduled to run **immediately after the current task finishes**,
before the event loop does anything else — before rendering, before the next
macrotask. The event loop drains the **entire microtask queue** after every
single macrotask.

```
What produces microtasks:
  Promise.then(fn)           → fn is a microtask callback
  Promise.catch(fn)          → fn is a microtask callback
  Promise.finally(fn)        → fn is a microtask callback
  queueMicrotask(fn)         → fn is a microtask callback
  await (inside async fn)    → code after await is a microtask continuation
  MutationObserver callback  → microtask
  process.nextTick(fn)       → Node.js only (even higher priority than Promise)
```

Think of a microtask as an **urgent follow-up** — something that must happen
before the current unit of work is considered truly done.

---

> **Analogy:** Imagine a customer service desk. A macrotask is serving a new
> customer — you take one ticket, serve them fully, then move on. A microtask
> is a sticky note that customer leaves saying "also do this before you call
> the next number" — you must handle all sticky notes before calling the
> next customer, no matter how many pile up.

### Why does the distinction matter?

```javascript
console.log('A');

setTimeout(() => console.log('B'), 0);   // macrotask — goes to macrotask queue

Promise.resolve().then(() => console.log('C')); // microtask — goes to microtask queue

console.log('D');

// Output: A → D → C → B
//                ↑     ↑
//           microtask  macrotask
//           wins       loses
```

Even though `setTimeout` has 0ms delay, it is a macrotask. The Promise `.then`
callback is a microtask. After the synchronous code (`A`, `D`) finishes, all
microtasks drain first (`C`), then the macrotask runs (`B`).

This ordering is the source of most async interview questions.

### Why should a senior frontend engineer care?

Output ordering is one of the most common JavaScript interview questions at every tier.
Beyond interviews, microtask/macrotask ordering directly affects:

| Situation | Why it matters |
|---|---|
| Promise chains inside event handlers | Determines when state updates are visible |
| `async/await` execution order | Every `await` is a microtask boundary |
| Angular Change Detection (Zone.js) | Zone patches macrotasks to trigger CD |
| RxJS schedulers | `asapScheduler` = microtask, `asyncScheduler` = macrotask |
| `queueMicrotask()` for perf tricks | Defer work without yielding to rendering |
| Node.js `process.nextTick` vs `Promise` | Different queues, different priority |

---

## 2. Why It Exists

### The problem JavaScript had to solve

JavaScript needs to handle async operations (I/O, timers, network) without blocking
the main thread. The solution is to defer callbacks until the call stack is empty,
then process them. But not all deferred work is equal:

- **Promises** represent the result of an already-initiated operation. Once resolved,
  their callbacks should run as soon as possible — before yielding control back to
  the browser for rendering or picking up a new event.
- **Timers and I/O** represent future work scheduled at the OS level. They can wait
  for the next full turn of the event loop.

A single flat queue would either delay Promises (bad — inconsistent, race-prone) or
starve rendering (bad — UI freezes). Two tiers solves this cleanly.

### The historical path

- **ES5 and earlier:** JavaScript only had one type of deferred callback — what we
  now call macrotasks. `setTimeout` and DOM events put callbacks in a single flat
  queue. Promises didn't exist; nested callbacks ("callback hell") were the async
  primitive. There was no concept of a "higher-priority" queue.
- **ES6 (2015):** Promises were introduced. The spec mandated that `.then` callbacks
  run as **microtasks** — a new, separate queue that drains completely after every
  macrotask. This gave Promises a predictable, high-priority execution slot.
- **ES2020+:** `queueMicrotask()` was added as an explicit API to schedule microtasks
  directly, without having to construct a Promise just to get microtask timing.

### Engineering Perspective

> **Why were microtasks given priority over macrotasks?**
> Promises represent *already-settled* async results. If a Promise is resolved, its
> `.then` handler should run as close to immediately as possible — similar to a
> synchronous continuation. Making them macrotasks would mean a rendering frame or
> timer could fire *between* a Promise settling and its handler running, causing
> observable inconsistencies. Microtasks fix this by draining completely before
> control returns to the browser.

> **What trade-off does this introduce?**
> A microtask that queues more microtasks creates an infinite loop that starves
> rendering and macrotasks entirely. The browser never gets a chance to render.
> This is the microtask version of the "blocking the main thread" problem.

---

## 3. Fundamentals

### The Three Queues (in priority order)

| Queue | Priority | Contents | Drained when |
|---|---|---|---|
| **Call Stack** | Highest | Currently executing synchronous code | Per line |
| **Microtask Queue** | 2nd | Promise callbacks, `queueMicrotask`, `MutationObserver` | After EVERY task, completely |
| **Macrotask Queue (Task Queue)** | 3rd | `setTimeout`, `setInterval`, I/O, UI events | One per event loop turn |

> **Key rule:** After every single macrotask (including the initial script execution),
> the event loop drains the **entire** microtask queue before moving on.

### What goes where

**Microtasks:**
- `Promise.then()` / `.catch()` / `.finally()`
- `async/await` (every `await` resumes as a microtask)
- `queueMicrotask(fn)`
- `MutationObserver` callbacks
- `process.nextTick` (Node.js — even higher priority than Promises in Node)

**Macrotasks:**
- `setTimeout(fn, delay)` — even `setTimeout(fn, 0)`
- `setInterval(fn, delay)`
- `setImmediate(fn)` — Node.js only
- I/O callbacks (file read, network response — Node.js)
- UI rendering (browser paints between macrotasks)
- `MessageChannel` / `postMessage`
- `requestAnimationFrame` (browser — technically its own queue, runs before paint)

### The Event Loop Algorithm (simplified)

```
1. Execute the current macrotask (e.g., the script itself, or a timer callback)
2. Drain the microtask queue completely:
     while (microtaskQueue.length > 0) {
       execute(microtaskQueue.shift())
       // New microtasks added during this step are also processed
     }
3. If a render is needed → render the frame (browser only)
4. Pick the next macrotask from the queue → go to step 1
```

---

## 4. Internal Working

### Step-by-step execution trace

```javascript
console.log('1');                          // sync

setTimeout(() => console.log('2'), 0);    // macrotask

Promise.resolve().then(() => console.log('3')); // microtask

console.log('4');                          // sync
```

**Trace:**

| Step | Call Stack | Microtask Queue | Macrotask Queue | Output |
|---|---|---|---|---|
| 1 | `console.log('1')` | — | — | `1` |
| 2 | `setTimeout(...)` registers timer | — | `() => log('2')` | — |
| 3 | `Promise.resolve().then(...)` | `() => log('3')` | `() => log('2')` | — |
| 4 | `console.log('4')` | `() => log('3')` | `() => log('2')` | `4` |
| 5 | Stack empty → drain microtasks | — | `() => log('2')` | `3` |
| 6 | Pick next macrotask | — | — | `2` |

**Output:** `1 → 4 → 3 → 2`

### How `async/await` maps to microtasks

```javascript
async function foo() {
  console.log('A');           // sync — runs immediately when foo() is called
  await Promise.resolve();    // suspends foo(), queues resumption as microtask
  console.log('B');           // microtask — runs after current task + all earlier microtasks
}

console.log('start');
foo();
console.log('end');
```

**Output:** `start → A → end → B`

`await` is syntactic sugar for `.then()`. When the engine hits `await expr`:
1. `expr` is evaluated (sync)
2. `foo`'s execution context is suspended and saved
3. A microtask is queued to resume `foo` when `expr` resolves
4. Control returns to the caller

### Microtask starvation

```javascript
function infinite() {
  Promise.resolve().then(infinite); // queues itself as a microtask forever
}
infinite();
// Result: browser freezes — microtask queue never empties, rendering never happens
```

The microtask queue is drained completely before rendering. An infinite microtask
chain means the browser never paints. This is the microtask equivalent of an
infinite synchronous loop.

### Node.js specifics: `process.nextTick`

In Node.js, `process.nextTick` callbacks run in a **separate "next tick queue"**
that has even higher priority than the Promise microtask queue.

```
Node.js priority order (highest → lowest):
  1. Call Stack (sync code)
  2. process.nextTick queue
  3. Promise microtask queue
  4. setImmediate (check phase of libuv event loop)
  5. setTimeout / setInterval
  6. I/O callbacks
```

```javascript
// Node.js only
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('Promise'));
setTimeout(() => console.log('setTimeout'), 0);

// Output: nextTick → Promise → setTimeout
```

---

## 5. Visual Architecture

### The Full Event Loop

```
┌─────────────────────────────────────────────────────────────────┐
│                        JAVASCRIPT RUNTIME                       │
│                                                                 │
│  ┌──────────────────┐        ┌─────────────────────────────┐   │
│  │    CALL STACK    │        │       WEB APIS / NODE       │   │
│  │                  │        │  setTimeout, fetch, I/O     │   │
│  │  [current task]  │──────► │  (callbacks registered      │   │
│  │                  │        │   here, then queued)        │   │
│  └────────┬─────────┘        └─────────────────────────────┘   │
│           │ empty?                                              │
│           ▼                                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │               EVENT LOOP                                 │  │
│  │                                                          │  │
│  │  Step 1: Run one macrotask (or initial script)           │  │
│  │         ↓                                                │  │
│  │  Step 2: Drain ENTIRE microtask queue ◄──────────────┐   │  │
│  │         ↓                             (new microtasks │   │  │
│  │  Step 3: Render frame (if needed)      added here     │   │  │
│  │         ↓                             loop back)      │   │  │
│  │  Step 4: Pick next macrotask → Step 1                  │  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─────────────────────┐   ┌─────────────────────────────────┐ │
│  │   MICROTASK QUEUE   │   │       MACROTASK QUEUE           │ │
│  │  Promise.then()     │   │  setTimeout callbacks           │ │
│  │  queueMicrotask()   │   │  setInterval callbacks          │ │
│  │  MutationObserver   │   │  I/O callbacks                  │ │
│  │  await continuations│   │  UI event handlers              │ │
│  └─────────────────────┘   └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Execution Order Visualised

```
Script runs (macrotask 1)
  ├─ sync code runs
  ├─ setTimeout → queued to macrotask queue
  ├─ Promise.then → queued to microtask queue
  └─ sync code runs
        ↓
Microtask queue drained (all of them)
  └─ Promise.then callback runs
  └─ any new microtasks queued here also run
        ↓
Browser render (if frame needed)
        ↓
Next macrotask: setTimeout callback runs
        ↓
Microtask queue drained again
        ↓
... and so on
```

### async/await suspension points

```
async function fetch() {
  console.log('A')    ← sync
  await step1()       ← SUSPENSION POINT — resumes as microtask
  console.log('B')    ← microtask continuation
  await step2()       ← SUSPENSION POINT — resumes as microtask
  console.log('C')    ← microtask continuation
}

Each 'await' = one microtask boundary
Code after 'await' never runs synchronously
```

---

## 6. Syntax & API

### `setTimeout` and `setInterval` — macrotasks

```javascript
// Macrotask — runs after current task + all pending microtasks
setTimeout(() => {
  console.log('macrotask');
}, 0); // 0ms delay means "as soon as possible" — but still a macrotask

// Repeating macrotask
const id = setInterval(() => {
  console.log('interval');
}, 1000);
clearInterval(id); // always clean up
```

### `Promise` — microtask

```javascript
// .then callback is always a microtask — even if Promise is already resolved
Promise.resolve('value')
  .then(v => console.log('microtask 1:', v))
  .then(() => console.log('microtask 2')); // chained — also a microtask

// Already-resolved promise — still async (microtask, not sync)
const p = Promise.resolve(42);
p.then(v => console.log(v)); // does NOT run synchronously
console.log('this runs first');
// Output: "this runs first" → 42
```

### `queueMicrotask` — explicit microtask scheduling

```javascript
// Direct microtask scheduling — no Promise wrapper needed
queueMicrotask(() => {
  console.log('queued microtask');
});

console.log('sync');
// Output: sync → queued microtask
```

### `async/await` — microtask at every `await`

```javascript
async function example() {
  console.log('1 - sync start');

  const result = await fetch('/api/data');  // suspends here
  console.log('2 - after first await');     // microtask continuation

  const json = await result.json();         // suspends again
  console.log('3 - after second await');    // microtask continuation
}

example();
console.log('4 - sync after calling example');
// Output: 1 → 4 → (after fetch resolves) → 2 → (after json) → 3
```

### `MutationObserver` — microtask

```javascript
const observer = new MutationObserver((mutations) => {
  // This callback is a MICROTASK — runs before next macrotask
  console.log('DOM mutated:', mutations.length);
});

observer.observe(document.body, { childList: true });
document.body.appendChild(document.createElement('div')); // triggers observer
console.log('sync after mutation');
// Output: "sync after mutation" → "DOM mutated: 1"
// (MutationObserver runs after current sync code, as microtask)
```

### `requestAnimationFrame` — its own queue (browser)

```javascript
// rAF is NOT a microtask or a regular macrotask — it runs just before rendering
requestAnimationFrame(() => {
  console.log('rAF — runs before next paint');
});

setTimeout(() => console.log('setTimeout'), 0);
Promise.resolve().then(() => console.log('Promise'));

// Output: Promise → rAF → setTimeout
// (microtasks first, then rAF before paint, then next macrotask)
```

---

## 7. Practical Examples

### Classic interview question — predict the output

```javascript
console.log('start');

setTimeout(() => console.log('setTimeout 1'), 0);
setTimeout(() => console.log('setTimeout 2'), 0);

Promise.resolve()
  .then(() => console.log('promise 1'))
  .then(() => console.log('promise 2'));

console.log('end');
```

**Step-by-step trace:**

1. `console.log('start')` — sync → output: `start`
2. `setTimeout 1` registered → macrotask queue: `[ST1]`
3. `setTimeout 2` registered → macrotask queue: `[ST1, ST2]`
4. `Promise.resolve().then(p1)` → microtask queue: `[p1]`
5. `console.log('end')` — sync → output: `end`
6. Call stack empty → drain microtasks:
   - `p1` runs → output: `promise 1` → `.then(p2)` queued → microtask queue: `[p2]`
   - `p2` runs → output: `promise 2`
7. Pick next macrotask: `ST1` → output: `setTimeout 1`
8. Drain microtasks (empty)
9. Pick next macrotask: `ST2` → output: `setTimeout 2`

**Output:** `start → end → promise 1 → promise 2 → setTimeout 1 → setTimeout 2`

---

### Intermediate — nested Promise inside setTimeout

```javascript
setTimeout(() => {
  console.log('timeout');
  Promise.resolve().then(() => console.log('promise inside timeout'));
}, 0);

Promise.resolve().then(() => console.log('promise outside'));
```

**Output:** `promise outside → timeout → promise inside timeout`

Why? The outer `Promise` is a microtask — runs first. Then `setTimeout` fires (macrotask).
*Inside* that macrotask, a new Promise is queued — but only after `timeout` logs.
After that macrotask finishes, microtasks drain again → `promise inside timeout`.

---

### Advanced — async/await mixed with setTimeout

```javascript
async function main() {
  console.log('A');
  await null;           // same as await Promise.resolve(null)
  console.log('B');
  await null;
  console.log('C');
}

console.log('1');
main();
setTimeout(() => console.log('D'), 0);
console.log('2');
```

**Output:** `1 → A → 2 → B → C → D`

Trace:
- `1` — sync
- `main()` called → `A` (sync inside async fn) → `await null` suspends → microtask queued
- `setTimeout(D)` — macrotask queue
- `2` — sync
- Stack empty → microtasks: resume `main` → `B` → `await null` → suspend again → microtask queued
- Drain microtasks: resume `main` → `C` → function ends
- Next macrotask: `D`

---

### Real interview trap — Promise constructor is sync

```javascript
console.log('1');

new Promise((resolve) => {
  console.log('2');    // executor runs SYNCHRONOUSLY
  resolve();
}).then(() => console.log('3'));

console.log('4');
```

**Output:** `1 → 2 → 4 → 3`

The Promise **executor** (the function passed to `new Promise(...)`) runs synchronously.
Only `.then` callbacks are microtasks.

---

## 8. Real-world Usage

### Angular — Zone.js and Change Detection

Zone.js works by monkey-patching all async APIs (`setTimeout`, `Promise`, `fetch`,
`addEventListener`, etc.). When any async operation completes, Zone.js is notified
and triggers Angular's Change Detection.

Understanding microtask/macrotask order explains a common Angular gotcha:

```typescript
// Component
ngOnInit() {
  // This runs in a macrotask (component init)
  this.dataService.getData().subscribe(data => {
    this.items = data;
    // Angular CD will run AFTER this entire macrotask completes
    // (Zone.js triggers CD after the macrotask + its microtasks finish)
  });
}
```

If you manually call `ChangeDetectorRef.detectChanges()` inside a microtask,
it runs before Angular's Zone-triggered CD — potentially causing double detection.

```typescript
// Why this pattern works
Promise.resolve().then(() => {
  this.value = 'updated';
  // Zone.js will detect this change in the microtask flush phase
  // No need for manual detectChanges() in most cases
});
```

### Angular — `NgZone.runOutsideAngular`

```typescript
constructor(private ngZone: NgZone) {}

ngOnInit() {
  // Heavy polling that shouldn't trigger CD on every tick
  this.ngZone.runOutsideAngular(() => {
    setInterval(() => {
      this.updateInternalState(); // macrotask — no CD triggered
      if (this.needsRender) {
        this.ngZone.run(() => {
          // Re-enter zone → triggers CD as a microtask
          this.visibleState = this.internalState;
        });
      }
    }, 100);
  });
}
```

### RxJS Schedulers map directly to microtask/macrotask

```typescript
import { of } from 'rxjs';
import { observeOn } from 'rxjs/operators';
import { asapScheduler, asyncScheduler, animationFrameScheduler } from 'rxjs';

// asapScheduler → schedules as MICROTASK (queueMicrotask / Promise)
of(1, 2, 3).pipe(observeOn(asapScheduler)).subscribe(console.log);

// asyncScheduler → schedules as MACROTASK (setTimeout)
of(1, 2, 3).pipe(observeOn(asyncScheduler)).subscribe(console.log);

// animationFrameScheduler → schedules before next paint (requestAnimationFrame)
of(1, 2, 3).pipe(observeOn(animationFrameScheduler)).subscribe(console.log);
```

Choosing the wrong scheduler causes subtle ordering bugs in RxJS pipelines.

### `queueMicrotask` for batching DOM updates

```javascript
// Instead of updating the DOM on every item in a loop,
// batch them into one microtask flush

let pendingUpdate = false;

function scheduleUpdate() {
  if (!pendingUpdate) {
    pendingUpdate = true;
    queueMicrotask(() => {
      flushAllUpdates();
      pendingUpdate = false;
    });
  }
}

items.forEach(item => {
  updateItem(item);    // mark dirty
  scheduleUpdate();    // only one microtask queued regardless of loop size
});
// All DOM updates happen in one batch after the current sync code finishes
```

---

## 9. Performance

### Microtask flooding — the hidden performance trap

Every `.then()` callback, every `await`, every `queueMicrotask()` call adds to the
microtask queue. The queue drains completely before rendering. Too many microtasks
in one turn = dropped frames.

```javascript
// Bad — creates 100,000 microtasks that all must drain before first paint
async function processBig(items) {
  for (const item of items) {
    await process(item); // 100,000 await boundaries
  }
}

// Better — batch processing, yield to renderer periodically
async function processBig(items) {
  const BATCH = 100;
  for (let i = 0; i < items.length; i += BATCH) {
    const chunk = items.slice(i, i + BATCH);
    chunk.forEach(process);
    await new Promise(r => setTimeout(r, 0)); // macrotask yield — allows render
  }
}
```

### `scheduler.yield()` — the modern solution (Chrome 115+)

```javascript
// Yields to the browser's scheduler, allowing higher-priority work (render, input)
// without going all the way to a macrotask

async function processWithYield(items) {
  for (let i = 0; i < items.length; i++) {
    process(items[i]);
    if (i % 50 === 0) {
      await scheduler.yield(); // yields to browser, resumes as high-priority task
    }
  }
}
```

### `setTimeout(fn, 0)` is NOT zero

The HTML spec mandates a minimum delay of **4ms** for nested `setTimeout` calls
(after 5 levels of nesting). Even at the top level, the delay is at least 1ms.
For truly minimal delay, use `queueMicrotask` or `Promise.resolve().then()`.

```javascript
// Measuring actual delay
const start = performance.now();
setTimeout(() => {
  console.log(`Actual delay: ${performance.now() - start}ms`);
  // Typically 1–4ms, never actually 0
}, 0);
```

### Promise resolution cost

Creating Promise chains has overhead. For tight loops, avoid unnecessary `.then()`:

```javascript
// Avoid — creates a new Promise object for each iteration
const results = await items.reduce(async (accP, item) => {
  const acc = await accP;
  return [...acc, await process(item)];
}, Promise.resolve([]));

// Better — Promise.all runs concurrently, one chain per item
const results = await Promise.all(items.map(item => process(item)));
```

---

## 10. Common Mistakes

### Mistake 1 — Assuming `setTimeout(fn, 0)` runs before Promise

```javascript
setTimeout(() => console.log('first?'), 0);
Promise.resolve().then(() => console.log('second?'));

// Actual output: "second?" → "first?"
// Promise (microtask) always beats setTimeout (macrotask)
```

This is the most common interview mistake. Timers always lose to Promises.

---

### Mistake 2 — Assuming Promise executor is async

```javascript
let x = 0;
new Promise((resolve) => {
  x = 1;        // runs SYNCHRONOUSLY
  resolve();
});
console.log(x); // 1 — NOT 0

// Only .then/.catch/.finally are async (microtasks)
```

---

### Mistake 3 — Expecting `await` to be instant

```javascript
async function test() {
  let x = 0;
  await Promise.resolve();
  x = 1; // This does NOT run synchronously after the await line
}

test();
console.log('x is still 0 here if you could read it synchronously');
```

Every `await` suspends the function. Code after `await` runs as a microtask continuation.

---

### Mistake 4 — Microtask infinite loop

```javascript
// Freezes the browser — never do this
function loop() {
  return Promise.resolve().then(loop);
}
loop();
// Microtask queue never empties → browser can never render → tab freezes
```

---

### Mistake 5 — Mixing `async/await` and `.then` ordering

```javascript
async function a() {
  await b();
  console.log('after b'); // runs as microtask after b() resolves
}

async function b() {
  // b resolves implicitly when it returns
  console.log('inside b');
}

a();
console.log('sync');
// Output: inside b → sync → after b
// NOT: inside b → after b → sync
```

`after b` runs as a microtask continuation, not synchronously.

---

### Mistake 6 — Node.js: confusing `process.nextTick` with `setTimeout`

```javascript
// Node.js
setTimeout(() => console.log('timeout'), 0);
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));

// Output: nextTick → promise → timeout
// process.nextTick has HIGHER priority than Promise microtasks in Node.js
```

---

## 11. Best Practices

1. **Never use `setTimeout(fn, 0)` when you need microtask timing** — use
   `queueMicrotask()` or `Promise.resolve().then()` instead.

2. **Keep microtask chains short** — the microtask queue must fully drain before
   rendering. Long chains cause frame drops.

3. **Use `scheduler.yield()` for long tasks** — breaks work into browser-prioritised
   chunks without falling all the way back to a `setTimeout`.

4. **Understand your framework's scheduler** — Angular/Zone.js uses macrotask
   patching. RxJS schedulers map to specific queue types. Knowing this prevents
   subtle ordering bugs.

5. **Never create infinite microtask loops** — always check that recursive microtask
   patterns have a termination condition.

6. **Use `Promise.all` for parallel async work** — don't `await` in sequence
   when operations are independent.

7. **In Node.js, prefer Promises over `process.nextTick`** — `nextTick` starves
   the I/O event loop if overused. It was added before Promises existed.

8. **Measure, don't guess** — use the Performance tab in DevTools to see actual
   task timing before optimising.

---

## 12. Debugging

### Chrome DevTools — Performance Tab

1. Open DevTools → **Performance** tab → click Record
2. Trigger the operation you want to inspect
3. Stop recording
4. In the flame chart, look at the **Main** thread row
5. Tasks are shown as grey blocks. Long yellow tasks = long macrotasks.
6. Inside each task, you can see microtask checkpoints (small blocks between tasks)

### Logging task boundaries

```javascript
// Manually observe event loop turn boundaries
function logTurn(label) {
  setTimeout(() => console.log(`[macrotask] ${label}`), 0);
  Promise.resolve().then(() => console.log(`[microtask] ${label}`));
  console.log(`[sync] ${label}`);
}

logTurn('A');
logTurn('B');
// Output:
// [sync] A
// [sync] B
// [microtask] A
// [microtask] B
// [macrotask] A
// [macrotask] B
```

### Detecting microtask starvation

```javascript
// Track if rendering is being starved by counting animation frames
let frameCount = 0;
let lastFrameTime = performance.now();

function monitorFrames() {
  frameCount++;
  const now = performance.now();
  const elapsed = now - lastFrameTime;
  if (elapsed > 100) { // > 100ms between frames = starvation
    console.warn(`Frame starvation detected: ${elapsed.toFixed(0)}ms gap`);
  }
  lastFrameTime = now;
  requestAnimationFrame(monitorFrames);
}
monitorFrames();
```

### Node.js — `--trace-event-categories`

```bash
node --trace-event-categories v8,node,node.async_hooks your-script.js
# Generates a trace.json file viewable in chrome://tracing
# Shows exact task/microtask boundaries
```

---

## 13. Advanced Concepts

### The HTML Event Loop Specification

The formal algorithm per the [WHATWG HTML spec](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model):

1. Select the oldest task from the task queue
2. Set it as the "currently running task"
3. Run it
4. Set "currently running task" to null
5. **Perform a microtask checkpoint:**
   - While microtask queue is not empty:
     - Take the oldest microtask
     - Run it (which may add more microtasks)
6. Update the rendering (if needed)
7. Go to step 1

### Multiple Task Queues (browser)

Browsers actually maintain **multiple macrotask queues** with different priorities:

- **User interaction queue** — click, keyboard events (highest priority)
- **Timer queue** — `setTimeout`, `setInterval`
- **Networking queue** — fetch responses, XHR
- **Rendering queue** — internal rendering tasks

The browser picks from whichever queue has the highest-priority task. This is why
a user click can interrupt a long series of `setTimeout` callbacks.

### `MessageChannel` as a macrotask (faster than setTimeout)

```javascript
// MessageChannel posts messages as macrotasks
// More predictable than setTimeout(fn, 0) and no 4ms minimum
const channel = new MessageChannel();
channel.port1.onmessage = () => console.log('MessageChannel macrotask');
channel.port2.postMessage(null);

// Used internally by React's scheduler and some polyfills
```

### Promises and the job queue (ECMAScript spec terminology)

In the ECMAScript spec, the microtask queue is called the **PromiseJobs queue**
(in older specs) or **microtask queue** in the HTML spec integration.

The spec guarantees:
1. Promise reactions (`.then` callbacks) are always scheduled as microtasks
2. A microtask never runs while the call stack is non-empty
3. All microtasks are processed before the next macrotask begins

### `async/await` — how many microtasks does one `await` take?

In modern V8 (after optimisations in Node.js 12+):

```javascript
async function fast() {
  await Promise.resolve(); // optimised: 2 microtask ticks in modern V8
}
```

Earlier V8 versions required 3 microtask ticks per `await`. The V8 team optimised
this significantly. The practical impact: deeply chained `async` functions are
faster than they used to be.

### `queueMicrotask` vs `Promise.resolve().then()`

Both schedule microtasks, but `queueMicrotask` is more explicit and has slightly
less overhead (no Promise object allocation):

```javascript
// Equivalent, but queueMicrotask is cleaner and marginally faster
queueMicrotask(() => doWork());
Promise.resolve().then(() => doWork()); // allocates a Promise object
```

---

## 14. Interview Questions

### Beginner

**Q: What is the difference between a microtask and a macrotask?**
> A macrotask is a unit of work picked up by the event loop one per turn — `setTimeout`,
> `setInterval`, I/O, UI events. A microtask runs immediately after the current task
> completes, before the next macrotask — Promise `.then` callbacks, `queueMicrotask`,
> `MutationObserver`. The entire microtask queue is drained between every two macrotasks.

**Q: What is the output of `setTimeout(fn, 0)` vs `Promise.resolve().then(fn)`?**
> The Promise `.then` callback (microtask) always runs before the `setTimeout`
> callback (macrotask), even with a 0ms delay. Microtasks have higher priority.

**Q: When does the microtask queue run?**
> After every task (including the initial script execution), before the browser
> renders the next frame and before the next macrotask is picked up.

---

### Intermediate

**Q: What is the output?**
```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');
```
> `1 → 4 → 3 → 2`. Sync code runs first (1, 4), then microtasks (3), then
> macrotasks (2).

**Q: Is the Promise executor synchronous or asynchronous?**
> Synchronous. The function passed to `new Promise((resolve, reject) => {...})` runs
> immediately and synchronously. Only `.then` / `.catch` / `.finally` callbacks are
> scheduled as microtasks.

**Q: What happens if a microtask queues another microtask?**
> The new microtask is added to the end of the microtask queue and is also processed
> in the current drain cycle — before any macrotask runs. If this happens infinitely,
> it starves rendering and macrotasks completely.

---

### Advanced

**Q: Explain the output of this code:**
```javascript
async function a() {
  console.log('a1');
  await b();
  console.log('a2');
}
async function b() {
  console.log('b1');
}
console.log('start');
a();
console.log('end');
```
> `start → a1 → b1 → end → a2`.
> `a()` is called synchronously: `a1` logs, then `await b()` is hit. `b()` is called
> synchronously: `b1` logs. `b()` returns (resolving to `undefined`). `await` suspends
> `a` — `a2` is queued as a microtask continuation. Control returns to the call site:
> `end` logs. Stack empties → microtask queue drains → `a2` logs.

**Q: How does Zone.js use macrotask patching to trigger Angular Change Detection?**
> Zone.js wraps all async APIs (`setTimeout`, `Promise`, `XHR`, `fetch`, `addEventListener`)
> at app bootstrap. When any async operation completes, Zone.js intercepts the callback
> and runs it inside the Angular zone. After the callback finishes (including all
> microtasks), Zone.js notifies Angular that the zone has become stable, triggering
> Change Detection. This is why Angular updates the UI after any async event —
> regardless of whether you explicitly called `detectChanges()`.

---

### Staff Level

**Q: You have a loop processing 50,000 items with async operations. The UI freezes.
How do you fix it without sacrificing throughput?**

> The root cause is either: (1) synchronous processing blocking the call stack, or
> (2) 50,000 microtask boundaries all draining before the browser can render.
>
> **Diagnosis:** Use the Performance tab — look for long tasks (>50ms) on the main thread.
>
> **Solution — chunked processing with macrotask yields:**
> ```javascript
> async function process(items) {
>   const CHUNK = 200;
>   for (let i = 0; i < items.length; i += CHUNK) {
>     items.slice(i, i + CHUNK).forEach(processOne);
>     // Yield to browser — allows rendering, input handling
>     await scheduler.yield(); // Chrome 115+
>     // Fallback: await new Promise(r => setTimeout(r, 0));
>   }
> }
> ```
> Each `scheduler.yield()` / `setTimeout` yields control back to the browser as a
> macrotask boundary, letting it render frames and handle user input between chunks.
> `scheduler.yield()` is preferred because it re-queues at higher priority than
> `setTimeout`, reducing total latency.

**Q: How would you implement a custom task scheduler that respects microtask/macrotask boundaries?**

> A real-world scheduler (like React's) needs to:
> 1. Break work into time-sliced chunks (< 5ms per chunk)
> 2. Use `MessageChannel` (not `setTimeout`) for macrotask scheduling — more precise, no 4ms floor
> 3. Use `requestAnimationFrame` for render-sensitive work
> 4. Use microtasks (`queueMicrotask`) only for truly urgent continuations
>
> ```javascript
> class Scheduler {
>   constructor() {
>     this.queue = [];
>     this.channel = new MessageChannel();
>     this.channel.port1.onmessage = () => this._flush();
>   }
>   schedule(fn, priority = 'normal') {
>     this.queue.push({ fn, priority });
>     this.channel.port2.postMessage(null); // trigger macrotask
>   }
>   _flush() {
>     const deadline = performance.now() + 5; // 5ms time slice
>     while (this.queue.length && performance.now() < deadline) {
>       this.queue.shift().fn();
>     }
>     if (this.queue.length) {
>       this.channel.port2.postMessage(null); // reschedule
>     }
>   }
> }
> ```

---

## 15. Hands-on Exercises

### Exercise 1 — Predict the output

```javascript
console.log('script start');

setTimeout(() => console.log('setTimeout'), 0);

Promise.resolve()
  .then(() => console.log('promise 1'))
  .then(() => console.log('promise 2'));

async function asyncFn() {
  console.log('async start');
  await Promise.resolve();
  console.log('async end');
}

asyncFn();
console.log('script end');
```

<details>
<summary>Answer</summary>

```
script start
async start
script end
promise 1
async end
promise 2
setTimeout
```

Trace:
1. `script start` — sync
2. `setTimeout` → macrotask queue
3. `Promise.resolve().then(p1)` → microtask queue: `[p1]`
4. `asyncFn()` called:
   - `async start` — sync
   - `await Promise.resolve()` — suspends, queues resume as microtask: `[p1, asyncResume]`
5. `script end` — sync
6. Drain microtasks:
   - `p1` runs → `promise 1` → queues `p2`
   - `asyncResume` runs → `async end`
   - `p2` runs → `promise 2`
7. Macrotask: `setTimeout` → `setTimeout`
</details>

---

### Exercise 2 — Fix the starvation bug

```javascript
// This function freezes the browser. Fix it.
async function processAll(items) {
  const results = [];
  for (const item of items) {
    results.push(await heavyCompute(item));
  }
  return results;
}

processAll(new Array(100000).fill(1));
```

<details>
<summary>Solution</summary>

```javascript
async function processAll(items) {
  const results = [];
  const CHUNK = 500;

  for (let i = 0; i < items.length; i++) {
    results.push(heavyCompute(items[i]));

    // Yield to browser every CHUNK items — allows rendering
    if (i % CHUNK === 0) {
      await new Promise(r => setTimeout(r, 0));
    }
  }

  return results;
}
```

The original has 100,000 microtask boundaries in a row — the microtask queue
never fully empties between them from the browser's rendering perspective.
Inserting a `setTimeout` yield every 500 items creates macrotask boundaries,
allowing the browser to render between chunks.
</details>

---

### Exercise 3 — Implement from scratch

Implement `myQueueMicrotask(fn)` without using `queueMicrotask()` directly.
It should schedule `fn` to run as a microtask.

<details>
<summary>Solution</summary>

```javascript
function myQueueMicrotask(fn) {
  Promise.resolve().then(fn);
}

// Usage
myQueueMicrotask(() => console.log('microtask'));
console.log('sync');
// Output: sync → microtask
```

Alternative using `MutationObserver` (for environments without Promise):

```javascript
function myQueueMicrotask(fn) {
  const node = document.createTextNode('');
  const observer = new MutationObserver(() => {
    observer.disconnect();
    fn();
  });
  observer.observe(node, { characterData: true });
  node.data = '1'; // trigger mutation → microtask
}
```
</details>

---

### Exercise 4 — Tricky Node.js ordering

Predict the output (Node.js):

```javascript
setImmediate(() => console.log('setImmediate'));
process.nextTick(() => console.log('nextTick 1'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick 2'));
setTimeout(() => console.log('setTimeout'), 0);
console.log('sync');
```

<details>
<summary>Answer</summary>

```
sync
nextTick 1
nextTick 2
promise
setTimeout   (or setImmediate — order between these two can vary)
setImmediate
```

Node.js priority: sync → nextTick queue → Promise microtask queue → setTimeout (timers phase) → setImmediate (check phase).
Note: `setTimeout(fn, 0)` vs `setImmediate` order is non-deterministic when called
from the main module (outside I/O callbacks). Inside an I/O callback, `setImmediate`
always runs before `setTimeout`.
</details>

---

## 16. Mini Project

### Build: Event Loop Visualiser (Node.js)

A script that demonstrates all queue types running in sequence, with timestamps
to show real timing:

```javascript
// event-loop-demo.js
const start = Date.now();
const t = () => `+${Date.now() - start}ms`;

console.log(`[${t()}] [SYNC] Script start`);

// Macrotask 1
setTimeout(() => {
  console.log(`[${t()}] [MACRO] setTimeout 1`);
  Promise.resolve().then(() => {
    console.log(`[${t()}] [MICRO] Promise inside setTimeout 1`);
  });
}, 0);

// Macrotask 2
setTimeout(() => console.log(`[${t()}] [MACRO] setTimeout 2`), 0);

// Microtask 1
Promise.resolve()
  .then(() => {
    console.log(`[${t()}] [MICRO] Promise 1`);
    return Promise.resolve();
  })
  .then(() => console.log(`[${t()}] [MICRO] Promise 2 (chained)`));

// Microtask via queueMicrotask
queueMicrotask(() => console.log(`[${t()}] [MICRO] queueMicrotask`));

// async/await
(async () => {
  console.log(`[${t()}] [SYNC] async IIFE start`);
  await null;
  console.log(`[${t()}] [MICRO] async IIFE after await`);
})();

console.log(`[${t()}] [SYNC] Script end`);

/*
Expected output:
[+0ms] [SYNC] Script start
[+0ms] [SYNC] async IIFE start
[+0ms] [SYNC] Script end
[+0ms] [MICRO] Promise 1
[+0ms] [MICRO] queueMicrotask
[+0ms] [MICRO] async IIFE after await
[+0ms] [MICRO] Promise 2 (chained)
[+1ms] [MACRO] setTimeout 1
[+1ms] [MICRO] Promise inside setTimeout 1
[+1ms] [MACRO] setTimeout 2
*/
```

Run it with `node event-loop-demo.js` and compare your mental model with the actual output.

---

## 17. Revision

### 5-Minute Summary

- **Macrotask:** `setTimeout`, `setInterval`, I/O, UI events — one per event loop turn
- **Microtask:** Promise `.then`, `queueMicrotask`, `MutationObserver`, `await` continuations
- **Rule:** After every macrotask, drain the **entire** microtask queue before the next macrotask
- **Promise executor** is **synchronous** — only `.then` callbacks are microtasks
- **`setTimeout(fn, 0)`** is still a macrotask — always runs after all pending microtasks
- **`async/await`** — every `await` is a microtask boundary, code after `await` is async
- **Starvation** — too many microtasks in one turn prevents rendering
- **Node.js** — `process.nextTick` > Promise microtasks > `setImmediate` > `setTimeout`

### Mind Map

```
Event Loop
├── Macrotask Queue
│   ├── setTimeout, setInterval
│   ├── I/O callbacks (Node)
│   ├── UI events (click, keydown)
│   └── MessageChannel
├── Microtask Queue (drained after EVERY macrotask)
│   ├── Promise.then / .catch / .finally
│   ├── queueMicrotask()
│   ├── MutationObserver
│   └── await continuations
└── Special
    ├── requestAnimationFrame (before paint)
    ├── process.nextTick (Node, higher than Promise)
    └── setImmediate (Node, after I/O)
```

### Top 10 Flash Cards

| Q | A |
|---|---|
| Does microtask or macrotask run first? | Microtask |
| Is `setTimeout(fn, 0)` a microtask? | No — macrotask |
| Is Promise `.then` a microtask? | Yes |
| Is the Promise executor sync or async? | Sync |
| When does the microtask queue drain? | After every macrotask, completely |
| What does `await` produce? | A microtask continuation |
| What happens if a microtask queues more microtasks? | They all drain before the next macrotask |
| Node.js: `nextTick` vs `Promise` priority? | `nextTick` is higher |
| API to schedule a microtask explicitly? | `queueMicrotask(fn)` |
| What causes microtask starvation? | Infinite recursive microtask chain |

---

## 18. Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│           MICROTASKS vs MACROTASKS — CHEAT SHEET                │
├─────────────────────────────────────────────────────────────────┤
│ MACROTASKS (Task Queue)                                         │
│   setTimeout, setInterval, setImmediate (Node)                  │
│   I/O callbacks, UI events, MessageChannel                      │
│   → One per event loop turn                                     │
│                                                                 │
│ MICROTASKS (Microtask Queue)                                    │
│   Promise.then / .catch / .finally                              │
│   queueMicrotask(), MutationObserver                            │
│   await continuations (every await = microtask boundary)        │
│   → Entire queue drained after EVERY macrotask                  │
│                                                                 │
│ EVENT LOOP ORDER                                                │
│   1. Run macrotask (or initial script)                          │
│   2. Drain ALL microtasks (including newly added ones)          │
│   3. Render frame (if needed)                                   │
│   4. Pick next macrotask → repeat                               │
│                                                                 │
│ KEY RULES                                                       │
│   Promise executor   → SYNC (not a microtask)                  │
│   setTimeout(fn, 0)  → MACRO (not instant, min ~1ms)           │
│   await null         → suspends fn, resumes as microtask        │
│   Infinite microtask → STARVATION (browser freezes)            │
│                                                                 │
│ NODE.JS PRIORITY (highest → lowest)                             │
│   sync → process.nextTick → Promise → setImmediate → setTimeout │
│                                                                 │
│ OUTPUT ORDER RULE OF THUMB                                      │
│   sync → microtasks → macrotasks                               │
│                                                                 │
│ CLASSIC TRAP                                                    │
│   setTimeout(()=>log('A'), 0)                                   │
│   Promise.resolve().then(()=>log('B'))                          │
│   → Output: B then A  (micro beats macro)                       │
│                                                                 │
│ FIX LONG TASK / STARVATION                                      │
│   await scheduler.yield()           // Chrome 115+              │
│   await new Promise(r=>setTimeout(r,0)) // macrotask yield      │
│   MessageChannel                    // faster than setTimeout   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 19. References

- [WHATWG HTML Spec — Event Loop Processing Model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)
- [MDN — In depth: Microtasks and the JavaScript runtime environment](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth)
- [MDN — queueMicrotask()](https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask)
- [V8 Blog — Fast async functions and promises](https://v8.dev/blog/fast-async)
- [Jake Archibald — Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
- [Node.js Event Loop — Official Docs](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick)
- [scheduler.yield() — MDN](https://developer.mozilla.org/en-US/docs/Web/API/Scheduler/yield)
