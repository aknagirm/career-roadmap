# JavaScript — Microtasks vs Macrotasks (Event Loop Deep Dive)

> **Phase 1 | Week 3 | Topic 1 of 30**
> **Difficulty:** Intermediate
> **Time to complete:** 2–3 hours
> **Status:** ⬜ Not Started

---

## Table of Contents

**Core**
1. [Introduction](#1-introduction)
2. [Why It Exists](#2-why-it-exists)
3. [Fundamentals](#3-fundamentals)
4. [Internal Working](#4-internal-working)
5. [Visual Architecture](#5-visual-architecture)
6. [Syntax & API](#6-syntax--api)
7. [Practical Examples](#7-practical-examples)
8. [Real-world Usage](#8-real-world-usage)
9. [Common Mistakes](#9-common-mistakes)
10. [Interview Questions](#10-interview-questions)
11. [Hands-on Exercises](#11-hands-on-exercises)
12. [Revision & Cheat Sheet](#12-revision--cheat-sheet)

**Good to Know**
13. [Performance & Optimisation](#13-performance--optimisation)
14. [Debugging](#14-debugging)
15. [Advanced Concepts](#15-advanced-concepts)
16. [References](#16-references)

---

## 1. Introduction

### What is a Task?

JavaScript is single-threaded — one call stack, one thing executing at a time.
Yet it handles timers, network requests, and user events without freezing. How?

When async work completes (a timer fires, a network response arrives, a user clicks),
the browser doesn't interrupt your running code. Instead it puts a **callback** into
a queue. Once the call stack is empty, the **Event Loop** picks the next callback
from the queue and runs it. Each such picked-up callback is called a **task**.

```
Call Stack empty?
    ↓ yes
Pick next callback from queue → run it  ← this is one "task"
    ↓ done
Call Stack empty again?
    ↓ yes
Pick next callback → run it
    ↓ ... forever
```

Every async operation — `setTimeout`, `fetch`, `addEventListener` — eventually
lands a callback in this queue as a task.

### So what are Macrotasks and Microtasks?

As JavaScript evolved, not all tasks turned out to be equal. Two distinct categories
emerged with different queues and different priorities:

---

**Macrotask** — a task scheduled by the browser's task scheduler, triggered by
external events or timers. The event loop picks up **one macrotask per turn**.

```
What produces macrotasks:
  setTimeout(fn, delay)      → fn queued as macrotask
  setInterval(fn, delay)     → fn queued as macrotask
  User events                → click, keydown, scroll
  I/O callbacks              → file read, network (Node.js)
  MessageChannel.postMessage → fn queued as macrotask
```

---

**Microtask** — a task scheduled to run **immediately after the current task
finishes**, before anything else. The event loop drains the **entire microtask
queue** after every single macrotask — before rendering, before the next macrotask.

```
What produces microtasks:
  Promise.then(fn)           → fn queued as microtask
  Promise.catch(fn)          → fn queued as microtask
  Promise.finally(fn)        → fn queued as microtask
  queueMicrotask(fn)         → fn queued as microtask
  await (inside async fn)    → code after await = microtask continuation
```

---

> **Analogy:** A customer service desk. A macrotask is serving a new customer — you
> take one ticket, serve them fully, then move on. A microtask is a sticky note that
> customer hands you saying "also do this before you call the next number" — you must
> handle all sticky notes before calling the next customer, no matter how many pile up.

### Why does the distinction matter?

```javascript
console.log('A');
setTimeout(() => console.log('B'), 0);          // macrotask
Promise.resolve().then(() => console.log('C')); // microtask
console.log('D');

// Output: A → D → C → B
//                ↑     ↑
//           microtask  macrotask
//           wins       loses
```

Even though `setTimeout` has 0ms delay, it is still a macrotask. The Promise `.then`
is a microtask. Sync code finishes first, then all microtasks drain, then macrotasks.

This ordering is the source of most async interview questions.

---

## 2. Why It Exists

JavaScript needs to handle async operations without blocking the main thread. It
defers callbacks until the call stack is empty — but not all deferred work is equal:

- **Promises** represent an already-completed operation. Their callbacks should run
  *as soon as possible* — before yielding to the browser for rendering or new events.
- **Timers and I/O** represent future work scheduled externally. They can wait for
  the next full turn of the event loop.

A single flat queue would either delay Promises (inconsistent behaviour) or starve
rendering (UI freezes). Two tiers solves this cleanly.

**Historical path:**
- **ES5:** Only one queue existed — what we now call macrotasks. `setTimeout` and
  DOM events. No Promises. No concept of priority.
- **ES6 (2015):** Promises introduced. Spec mandated `.then` callbacks run as
  microtasks — a new queue with higher priority, draining completely after every task.
- **ES2020+:** `queueMicrotask()` added to schedule microtasks explicitly without
  constructing a Promise.

---

## 3. Fundamentals

### Priority order

| Queue | Priority | What goes in | Drained when |
|---|---|---|---|
| **Call Stack** | 1st | Synchronous code | Per line |
| **Microtask Queue** | 2nd | Promise callbacks, `queueMicrotask`, `await` continuations | After EVERY task, completely |
| **Macrotask Queue** | 3rd | `setTimeout`, `setInterval`, I/O, UI events | One per event loop turn |

### The Event Loop algorithm

```
1. Run one macrotask (the initial script counts as the first macrotask)
2. Drain microtask queue completely:
     while (microtaskQueue.length > 0) {
       run(microtaskQueue.shift())
       // new microtasks added here are also processed before we stop
     }
3. Render frame if needed (browser only)
4. Pick next macrotask → go to step 1
```

> **Key rule:** The entire microtask queue drains after EVERY macrotask.
> There is no way for a macrotask to "cut in line" ahead of pending microtasks.

---

## 4. Internal Working

### Execution trace — the classic example

```javascript
console.log('1');                               // sync
setTimeout(() => console.log('2'), 0);          // macrotask
Promise.resolve().then(() => console.log('3')); // microtask
console.log('4');                               // sync
```

| Step | Action | Output |
|---|---|---|
| 1 | `console.log('1')` runs | `1` |
| 2 | `setTimeout` registered → macrotask queue: `[cb2]` | — |
| 3 | `Promise.then` registered → microtask queue: `[cb3]` | — |
| 4 | `console.log('4')` runs | `4` |
| 5 | Stack empty → drain microtasks → `cb3` runs | `3` |
| 6 | Pick next macrotask → `cb2` runs | `2` |

**Output: `1 → 4 → 3 → 2`**

---

### How `async/await` maps to microtasks

`await` is syntactic sugar for `.then()`. Every `await` splits the function into
a sync part and a microtask continuation:

```javascript
async function foo() {
  console.log('A');        // sync — runs when foo() is called
  await Promise.resolve(); // suspends foo(), schedules resume as microtask
  console.log('B');        // microtask continuation
}

console.log('start');
foo();
console.log('end');

// Output: start → A → end → B
```

When `await` is hit:
1. The expression is evaluated (sync)
2. `foo` is **suspended** — its state is saved
3. A microtask is queued to resume `foo` when the Promise resolves
4. Control returns to the caller immediately

`end` prints before `B` because the caller's sync code hasn't finished yet.
Microtasks only drain after the call stack is empty.

**Equivalent using `.then`** — identical output:

```javascript
function foo() {
  console.log('A');
  return Promise.resolve().then(() => {
    console.log('B');
  });
}
```

---

### Microtask starvation

```javascript
function infinite() {
  Promise.resolve().then(infinite); // queues itself forever
}
infinite();
// Browser freezes — microtask queue never empties, rendering never happens
```

The microtask queue must fully drain before rendering. An infinite chain = browser
never paints. Same effect as an infinite synchronous loop.

---

## 5. Visual Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        JAVASCRIPT RUNTIME                       │
│                                                                 │
│  ┌──────────────────┐        ┌─────────────────────────────┐   │
│  │    CALL STACK    │        │       WEB APIS / NODE       │   │
│  │                  │──────► │  setTimeout, fetch, I/O     │   │
│  │  [current task]  │        │  (registers callbacks,      │   │
│  │                  │        │   then queues them later)   │   │
│  └────────┬─────────┘        └─────────────────────────────┘   │
│           │ empty?                                              │
│           ▼                                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │               EVENT LOOP                                 │  │
│  │  Step 1: Run one macrotask (or initial script)           │  │
│  │         ↓                                                │  │
│  │  Step 2: Drain ENTIRE microtask queue ◄──────────────┐   │  │
│  │         ↓                             (new microtasks │   │  │
│  │  Step 3: Render frame (if needed)      loop back)     │   │  │
│  │         ↓                                             │   │  │
│  │  Step 4: Pick next macrotask → Step 1                  │  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─────────────────────┐   ┌─────────────────────────────────┐ │
│  │   MICROTASK QUEUE   │   │       MACROTASK QUEUE           │ │
│  │  Promise.then()     │   │  setTimeout callbacks           │ │
│  │  queueMicrotask()   │   │  setInterval callbacks          │ │
│  │  await continuations│   │  UI event handlers              │ │
│  └─────────────────────┘   └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Execution order across one full cycle

```
Script runs (macrotask 1)
  ├─ sync code runs
  ├─ setTimeout → macrotask queue
  ├─ Promise.then → microtask queue
  └─ sync code runs
        ↓
Microtask queue drained (ALL of them, including any newly queued)
  └─ Promise callbacks run
        ↓
Browser renders frame (if needed)
        ↓
Next macrotask: setTimeout callback runs
        ↓
Microtask queue drained again
        ↓
... and so on
```

---

## 6. Syntax & API

### `setTimeout` / `setInterval` — macrotask

```javascript
// Even 0ms delay — still a macrotask, runs after ALL pending microtasks
setTimeout(() => console.log('macro'), 0);

Promise.resolve().then(() => console.log('micro'));
// Output: micro → macro
```

### `Promise.then` — microtask

```javascript
// .then is always async (microtask) — even on an already-resolved Promise
const p = Promise.resolve(42);
p.then(v => console.log(v)); // does NOT run synchronously
console.log('this runs first');
// Output: "this runs first" → 42

// Promise executor runs SYNCHRONOUSLY — only .then is async
new Promise((resolve) => {
  console.log('executor — sync'); // prints immediately
  resolve();
}).then(() => console.log('then — microtask'));
console.log('after new Promise');
// Output: executor — sync → after new Promise → then — microtask
```

### `queueMicrotask` — explicit microtask

```javascript
queueMicrotask(() => console.log('microtask'));
console.log('sync');
// Output: sync → microtask
```

Use this instead of `Promise.resolve().then()` when you just need microtask
timing — it's more explicit and slightly lighter (no Promise object created).

### `async/await` — microtask at every `await`

```javascript
async function load() {
  console.log('1 - sync');
  const res = await fetch('/api');   // suspends here
  console.log('2 - after fetch');    // microtask continuation
  const data = await res.json();     // suspends again
  console.log('3 - after json');     // microtask continuation
}

load();
console.log('4 - sync after load()');
// Output: 1 → 4 → (fetch resolves) → 2 → (json resolves) → 3
```

---

## 7. Practical Examples

### Example 1 — Classic interview question

```javascript
console.log('start');

setTimeout(() => console.log('setTimeout 1'), 0);
setTimeout(() => console.log('setTimeout 2'), 0);

Promise.resolve()
  .then(() => console.log('promise 1'))
  .then(() => console.log('promise 2'));

console.log('end');
```

**Trace:**
1. `start` — sync
2. ST1, ST2 → macrotask queue
3. `promise 1` callback → microtask queue
4. `end` — sync
5. Drain microtasks: `promise 1` runs → chains `promise 2` → `promise 2` runs
6. Macrotask: `setTimeout 1`
7. Macrotask: `setTimeout 2`

**Output: `start → end → promise 1 → promise 2 → setTimeout 1 → setTimeout 2`**

---

### Example 2 — Promise inside setTimeout

```javascript
setTimeout(() => {
  console.log('timeout');
  Promise.resolve().then(() => console.log('promise inside timeout'));
  console.log('still in timeout');
}, 0);

Promise.resolve().then(() => console.log('promise outside'));
```

**Output: `promise outside → timeout → still in timeout → promise inside timeout`**

The entire timeout callback (including `still in timeout`) runs to completion as
one macrotask before any microtask inside it gets to run.

---

### Example 3 — async/await with setTimeout

```javascript
async function main() {
  console.log('A');
  await null;
  console.log('B');
  await null;
  console.log('C');
}

console.log('1');
main();
setTimeout(() => console.log('D'), 0);
console.log('2');
```

**Output: `1 → A → 2 → B → C → D`**

`D` is a macrotask — it always loses to the microtask continuations B and C,
even though the `setTimeout` was registered before `2` printed.

---

### Example 4 — Promise executor trap

```javascript
console.log('1');
new Promise((resolve) => {
  console.log('2');  // sync — runs immediately
  resolve();
}).then(() => console.log('3'));
console.log('4');
```

**Output: `1 → 2 → 4 → 3`**

The executor runs synchronously. Only `.then` is the microtask.
This catches most people in interviews.

---

## 8. Real-world Usage

### Angular — Zone.js and Change Detection

Zone.js monkey-patches all async APIs (`setTimeout`, `Promise`, `fetch`,
`addEventListener`) at bootstrap. When any async operation completes, Zone.js
detects it and triggers Angular's Change Detection.

```typescript
// Why CD triggers automatically after any async event
ngOnInit() {
  this.service.getData().subscribe(data => {
    this.items = data;
    // No detectChanges() needed — Zone.js triggers CD after this macrotask
    // completes (including all its microtask drains)
  });
}
```

```typescript
// NgZone.runOutsideAngular — prevent CD on heavy polling
this.ngZone.runOutsideAngular(() => {
  setInterval(() => {
    this.updateInternalCounter(); // macrotask — CD not triggered
    if (this.needsRender) {
      this.ngZone.run(() => {
        this.displayValue = this.internalCounter; // re-enter zone → CD triggered
      });
    }
  }, 100);
});
```

### Why `async/await` order matters in Angular

```typescript
async loadUser() {
  const user = await this.userService.get(); // suspends
  this.user = user; // microtask continuation
  // Angular CD hasn't run yet — this.user is set but view not updated
  // Zone.js will trigger CD after this entire async function settles
}
```

---

## 9. Common Mistakes

### 1 — `setTimeout(fn, 0)` runs before Promise

```javascript
setTimeout(() => console.log('A'), 0);
Promise.resolve().then(() => console.log('B'));
// Expected by most: A → B
// Actual: B → A   ← Promise (microtask) always beats setTimeout (macrotask)
```

### 2 — Promise executor is async

```javascript
let x = 0;
new Promise((resolve) => { x = 1; resolve(); }); // executor is SYNC
console.log(x); // 1, not 0
```

### 3 — `await` doesn't block the caller

```javascript
async function foo() {
  await something();
  console.log('B'); // does NOT run before the caller continues
}
foo();
console.log('A'); // always prints before B
```

### 4 — Microtask infinite loop

```javascript
// Freezes the browser silently
Promise.resolve().then(function self() {
  Promise.resolve().then(self);
});
```

### 5 — Code after `await` is not synchronous even with resolved Promises

```javascript
async function test() {
  await Promise.resolve(); // already resolved — doesn't matter
  console.log('still async'); // still a microtask, not sync
}
test();
console.log('this prints first'); // always
```

---

## 10. Interview Questions

### Beginner

**Q: What is the difference between a microtask and a macrotask?**
> A macrotask is a unit of work the event loop picks up one at a time —
> `setTimeout`, UI events, I/O. A microtask runs immediately after the current
> task finishes — Promise `.then` callbacks, `await` continuations. The entire
> microtask queue drains between every two macrotasks.

**Q: Does `setTimeout(fn, 0)` run before or after `Promise.resolve().then(fn)`?**
> After. Promise `.then` is a microtask — higher priority. `setTimeout` is a
> macrotask — it waits until all microtasks have drained.

**Q: Is the Promise executor synchronous or asynchronous?**
> Synchronous. Only `.then` / `.catch` / `.finally` callbacks are microtasks.

---

### Intermediate

**Q: What is the output?**
```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');
```
> `1 → 4 → 3 → 2`. Sync runs first, then microtasks, then macrotasks.

**Q: What happens if a microtask queues another microtask?**
> It is added to the current microtask drain cycle and also runs before any
> macrotask. If this is infinite, rendering is starved entirely.

**Q: Why does `end` print before `B` here?**
```javascript
async function foo() { console.log('A'); await null; console.log('B'); }
foo();
console.log('end');
```
> `await` suspends `foo` and immediately returns control to the caller. `end`
> is still synchronous code on the call stack — it runs before the microtask
> queue gets a chance to resume `foo`.

---

### Advanced

**Q: Explain the output:**
```javascript
async function a() { console.log('a1'); await b(); console.log('a2'); }
async function b() { console.log('b1'); }
console.log('start');
a();
console.log('end');
```
> `start → a1 → b1 → end → a2`.
> `a()` called: `a1` logs. `await b()` calls `b` synchronously: `b1` logs.
> `b` returns, `await` suspends `a` — `a2` queued as microtask. Control returns:
> `end` logs. Stack empties → microtask drains → `a2` logs.

**Q: How does Zone.js use macrotask patching to trigger Angular CD?**
> Zone.js wraps all async APIs at bootstrap. When a callback completes (including
> all its microtasks), Zone.js detects the zone becoming stable and notifies
> Angular to run Change Detection. This is why Angular updates the view after
> any async operation without you calling `detectChanges()`.

---

### Staff Level

**Q: A loop processing 50,000 items freezes the UI. How do you fix it?**
> Root cause: either a long synchronous block or 50,000 microtask boundaries
> draining without giving the browser a chance to render.
>
> Fix — chunk the work with macrotask yields:
> ```javascript
> async function process(items) {
>   const CHUNK = 200;
>   for (let i = 0; i < items.length; i += CHUNK) {
>     items.slice(i, i + CHUNK).forEach(processOne);
>     await new Promise(r => setTimeout(r, 0)); // yield to browser
>     // Chrome 115+: await scheduler.yield(); // better — higher priority re-queue
>   }
> }
> ```
> Each `setTimeout` creates a macrotask boundary — the browser gets to render
> and handle input between chunks.

---

## 11. Hands-on Exercises

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
4. `asyncFn()`: `async start` — sync, then `await` suspends → microtask queue: `[p1, resume]`
5. `script end` — sync
6. Drain microtasks:
   - `p1` → `promise 1`, chains `p2`
   - `resume` → `async end`
   - `p2` → `promise 2`
7. Macrotask: `setTimeout`
</details>

---

### Exercise 2 — Fix the starvation bug

```javascript
// Freezes the browser. Fix it.
async function processAll(items) {
  const results = [];
  for (const item of items) {
    results.push(await heavyCompute(item)); // 100,000 microtask boundaries
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
    if (i % CHUNK === 0) {
      await new Promise(r => setTimeout(r, 0)); // macrotask yield → allows render
    }
  }
  return results;
}
```
</details>

---

### Exercise 3 — Implement from scratch

Implement `myQueueMicrotask(fn)` without using `queueMicrotask()`.

<details>
<summary>Solution</summary>

```javascript
function myQueueMicrotask(fn) {
  Promise.resolve().then(fn);
}

myQueueMicrotask(() => console.log('microtask'));
console.log('sync');
// Output: sync → microtask
```
</details>

---

## 12. Revision & Cheat Sheet

### 5-Minute Summary

- **Macrotask** — `setTimeout`, `setInterval`, UI events — one per event loop turn
- **Microtask** — Promise `.then`, `queueMicrotask`, `await` continuations — entire queue drains after every macrotask
- **Order** — sync → microtasks → macrotasks
- **Promise executor** — synchronous, only `.then` is async
- **`setTimeout(fn, 0)`** — still a macrotask, always loses to pending microtasks
- **`await`** — suspends the function, gives control back to caller, resumes as microtask
- **Starvation** — infinite microtask chain starves rendering

### Flash Cards

| Q | A |
|---|---|
| Micro or macro runs first? | Microtask |
| Is `setTimeout(fn, 0)` a microtask? | No — macrotask |
| Is Promise `.then` a microtask? | Yes |
| Is the Promise executor sync or async? | Sync |
| When does microtask queue drain? | After every macrotask, completely |
| What does `await` produce? | Microtask continuation |
| New microtask queued during microtask drain? | Also runs before next macrotask |
| Explicit microtask API? | `queueMicrotask(fn)` |
| What causes microtask starvation? | Infinite recursive microtask chain |
| `end` before `B` in async fn — why? | `await` returns control to caller; caller's sync code runs first |

### Cheat Sheet

```
┌─────────────────────────────────────────────────────────────┐
│         MICROTASKS vs MACROTASKS — CHEAT SHEET              │
├─────────────────────────────────────────────────────────────┤
│ MACROTASKS          │ MICROTASKS                            │
│  setTimeout         │  Promise.then / .catch / .finally     │
│  setInterval        │  queueMicrotask()                     │
│  UI events          │  await continuations                  │
│  I/O callbacks      │                                       │
│  One per loop turn  │  Entire queue drains each turn        │
├─────────────────────────────────────────────────────────────┤
│ EVENT LOOP ORDER                                            │
│  1. Run one macrotask (script = first macrotask)            │
│  2. Drain ALL microtasks (incl. newly queued ones)          │
│  3. Render if needed                                        │
│  4. Pick next macrotask → repeat                            │
├─────────────────────────────────────────────────────────────┤
│ KEY RULES                                                   │
│  Promise executor    → SYNC                                 │
│  setTimeout(fn, 0)   → MACRO (min ~1ms, never instant)      │
│  await               → suspends fn, resumes as microtask    │
│  Infinite microtasks → starvation (browser freezes)         │
├─────────────────────────────────────────────────────────────┤
│ OUTPUT RULE OF THUMB                                        │
│  sync → microtasks → macrotasks                            │
├─────────────────────────────────────────────────────────────┤
│ CLASSIC TRAP                                                │
│  setTimeout(()=>log('A'), 0)                                │
│  Promise.resolve().then(()=>log('B'))                       │
│  → B then A  (micro beats macro every time)                 │
└─────────────────────────────────────────────────────────────┘
```

---

---

# Good to Know

> These topics are not tested directly in most frontend interviews but build
> deeper understanding. Read when you have time or before Staff-level rounds.

---

## 13. Performance & Optimisation

### Microtask flooding

Too many microtasks in one turn delays rendering. Every `await` and `.then` is
a boundary — 100,000 in a row means the browser never gets to paint between them.

```javascript
// Bad — 100,000 awaits before any render
async function bad(items) {
  for (const item of items) await process(item);
}

// Good — yield to browser every 100 items
async function good(items) {
  for (let i = 0; i < items.length; i++) {
    process(items[i]);
    if (i % 100 === 0) await new Promise(r => setTimeout(r, 0));
  }
}
```

### `scheduler.yield()` (Chrome 115+)

Better than `setTimeout` for yielding — re-queues at higher priority so it
doesn't wait behind other `setTimeout` callbacks:

```javascript
if (i % 100 === 0) await scheduler.yield();
```

### `setTimeout(fn, 0)` minimum delay

The HTML spec mandates a minimum of **4ms** for nested `setTimeout` (5+ levels
deep). Top-level is typically ~1ms. Never actually 0. Use `queueMicrotask` when
you need the smallest possible delay.

### `Promise.all` vs sequential `await`

```javascript
// Sequential — each waits for the previous (slow if independent)
const a = await fetchA();
const b = await fetchB();

// Parallel — both fire at once (fast)
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

---

## 14. Debugging

### Chrome DevTools — Performance tab

1. DevTools → **Performance** → Record
2. Trigger the operation
3. Stop recording
4. Look at the **Main** thread row — long yellow blocks = long tasks
5. Microtask checkpoints appear as small blocks between tasks

### Logging queue boundaries manually

```javascript
function logTurn(label) {
  setTimeout(() => console.log(`[macro] ${label}`), 0);
  Promise.resolve().then(() => console.log(`[micro] ${label}`));
  console.log(`[sync] ${label}`);
}
logTurn('A');
logTurn('B');
// sync A → sync B → micro A → micro B → macro A → macro B
```

### Detecting frame starvation

```javascript
let last = performance.now();
function monitor() {
  const now = performance.now();
  if (now - last > 100) console.warn(`Frame gap: ${(now - last).toFixed(0)}ms`);
  last = now;
  requestAnimationFrame(monitor);
}
monitor();
```

---

## 15. Advanced Concepts

### Multiple macrotask queues (browser internals)

Browsers maintain several macrotask queues with different priorities:
- **User interaction** — click, keyboard (highest)
- **Timer** — `setTimeout`, `setInterval`
- **Networking** — fetch, XHR responses
- **Rendering** — internal paint tasks

A user click can interrupt a long series of `setTimeout` callbacks because it
sits in a higher-priority queue.

### `MessageChannel` — macrotask without the 4ms floor

```javascript
// Used by React's scheduler internally — more precise than setTimeout
const ch = new MessageChannel();
ch.port1.onmessage = () => console.log('MessageChannel macrotask');
ch.port2.postMessage(null);
```

### `MutationObserver` — microtask for DOM changes

Fires a microtask when the DOM changes. Useful for detecting third-party DOM
mutations. Rarely tested in interviews but good to know it's a microtask.

### `requestAnimationFrame` — its own queue

Not a microtask or macrotask — runs just before the browser paints, after
microtasks drain. Use for animations to sync with display refresh rate (~60fps).

### Node.js `process.nextTick`

Higher priority than Promise microtasks in Node.js. Runs before any Promise
`.then` callbacks. Added before Promises existed — Node.js docs recommend
`queueMicrotask()` over `nextTick` for new code.

```
Node.js order: sync → nextTick → Promise → setImmediate → setTimeout
```

### ECMAScript spec terminology

The spec calls the microtask queue the **PromiseJobs queue**. Guarantees:
1. `.then` callbacks are always microtasks
2. A microtask never interrupts running synchronous code
3. All microtasks process before the next macrotask

---

## 16. References

- [WHATWG HTML Spec — Event Loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)
- [MDN — Microtask guide](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth)
- [MDN — queueMicrotask()](https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask)
- [Jake Archibald — Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
- [V8 Blog — Fast async functions](https://v8.dev/blog/fast-async)
- [Node.js Event Loop docs](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick)
