# JavaScript — Web Workers

> **Phase 1 | Week 4 | Topic 2**
> **Difficulty:** Intermediate → Advanced
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

### What is it?

A **Web Worker** is a browser API that lets you run JavaScript in a **background thread**,
completely separate from the main UI thread.

> **Analogy:** Your main thread is a chef cooking at the stove. A Web Worker is a
> kitchen assistant working at a separate counter — chopping vegetables in parallel
> without ever touching the stove or getting in the chef's way.

JavaScript is single-threaded by design — one call stack, one thing at a time.
But the browser gives you an escape hatch: Web Workers let you spin up separate
threads for CPU-heavy work so the main thread stays free to handle UI, events,
and user interaction.

**Three types of Web Workers:**

| Type | Scope | Use Case |
|---|---|---|
| **Dedicated Worker** | One script only | Heavy computation for one page |
| **Shared Worker** | Multiple scripts/tabs | Shared state across browser tabs |
| **Service Worker** | Network proxy | Offline support, caching, push notifications |

This lesson focuses on **Dedicated Workers** (most common in interviews) with
coverage of Service Workers in Advanced Concepts.

### Why should a senior frontend engineer care?

| Scenario | Without Worker | With Worker |
|---|---|---|
| Sorting 100k records | UI freezes for 2–3 seconds | UI stays smooth |
| Image processing / canvas | Frame drops, janky animation | 60fps maintained |
| AI inference in browser (ONNX, WASM) | Page hangs | Background processing |
| CSV/Excel parsing | Tab becomes unresponsive | Instant feedback |

Senior interviews at Atlassian, Adobe, Google test whether you know **when and how**
to move work off the main thread. This is one of the most practical performance topics.

---

## 2. Why It Exists

### The problem JavaScript had to solve

JavaScript's single-threaded model is perfect for UI safety — no race conditions on
the DOM. But it creates a UX problem:

**Long-running synchronous code blocks everything.**

```javascript
// This freezes the UI for ~3 seconds
function sortHugeArray(arr) {
  return arr.sort((a, b) => a - b); // blocking
}

const data = Array.from({length: 10000000}, () => Math.random());
sortHugeArray(data); // UI frozen — user can't scroll, click, type
```

The call stack is blocked. No events can be processed. The page appears frozen.

Early solutions were inadequate:
- `setTimeout(() => {}, 0)` → breaks work into chunks but still runs on main thread
- `requestAnimationFrame` → same thread, just better timing
- Server-side processing → network latency, requires backend infrastructure

### Why Web Workers were introduced (2009)

HTML5 introduced Web Workers to solve this exact problem: **CPU-heavy work should
happen off the main thread.**

Key design constraints:
- **No DOM access** → workers can't touch the DOM to avoid race conditions
- **Message passing only** → `postMessage` API, no shared memory (initially)
- **Separate global scope** → workers run in `WorkerGlobalScope`, not `Window`

### Engineering Perspective

> **Why was this designed this way?**
> Safety first. Allowing multithreaded DOM access would require complex locking,
> introducing race conditions and deadlocks. The no-DOM constraint keeps the main
> thread as the single source of truth for UI state, while workers handle pure computation.

> **What are the alternatives?**
> - **`async`/`await` + `queueMicrotask()`** — breaks work into microtasks but still
>   blocks the main thread if the work is CPU-intensive.
> - **WebAssembly (WASM)** — faster execution but still runs on main thread unless
>   combined with Web Workers.
> - **Server-side offloading** — adds network latency, requires backend infrastructure.
> - **`SharedArrayBuffer` + Atomics** (advanced) — true shared memory between threads
>   but requires careful synchronization.

> **What trade-offs does it introduce?**
> - **No DOM** — you can't manipulate UI from a worker. Must serialize results and
>   send them back.
> - **Message serialization cost** — `postMessage` clones data (structured clone
>   algorithm). Large objects have overhead. Use Transferable Objects to avoid this.
> - **Startup cost** — spinning up a worker takes ~10–50ms. Not suitable for tiny tasks.

> **How would a Staff Engineer think about this?**
> "Is this work CPU-bound or I/O-bound? If CPU-bound and takes >50ms, move it to a
> worker. If I/O-bound, keep it on main thread with async/await. If the data is
> huge (>1MB), use Transferable Objects or SharedArrayBuffer."

---

## 3. Fundamentals

### Core Concepts & Terminology

#### Main Thread
The default JavaScript thread. Handles: DOM rendering, user events, JS execution,
CSS animations. **Must never be blocked for more than ~50ms** (Interaction to Next
Paint budget).

#### Worker Thread
A separate OS-level thread spawned by the browser. Runs JS but:
- Has no access to `window`, `document`, or DOM APIs
- Has its own global scope: `WorkerGlobalScope` (or `DedicatedWorkerGlobalScope`)
- Has access to: `fetch`, `XMLHttpRequest`, `WebSockets`, `IndexedDB`, `setTimeout`

#### postMessage / onmessage
The only way main thread and worker communicate. Uses the **structured clone algorithm**
to copy data between threads. Data is **copied**, not shared (by default).

#### Structured Clone Algorithm
The serialization mechanism used by `postMessage`. Supports:
- Primitives, Arrays, Objects, Map, Set, Date, RegExp, ArrayBuffer, Blob, File
- Does NOT support: Functions, DOM nodes, Symbols, `undefined` (in some cases)

#### Transferable Objects
Special objects that can be **transferred** (not copied) between threads.
Transferring moves ownership — the original reference becomes unusable.
Supported: `ArrayBuffer`, `MessagePort`, `ImageBitmap`, `OffscreenCanvas`

#### SharedArrayBuffer
A fixed-length raw binary buffer that can be **shared** between multiple threads
with no copying. Requires `Atomics` for safe concurrent access.

#### Atomics
Low-level API for thread-safe operations on `SharedArrayBuffer`.
Methods: `Atomics.add()`, `Atomics.load()`, `Atomics.store()`, `Atomics.wait()`,
`Atomics.notify()`

### Worker Types Summary

| | Dedicated Worker | Shared Worker | Service Worker |
|---|---|---|---|
| Scope | One script | Multiple tabs | Network proxy |
| Communication | `postMessage` | `MessagePort` | `fetch` events |
| DOM access | ❌ | ❌ | ❌ |
| Lifetime | Until page closes | Until all pages close | Persistent (background) |
| Use case | Heavy computation | Cross-tab state | Offline, caching, push |

---

## 4. Internal Working

### How the Browser Creates a Worker

```
Main Thread calls: new Worker('worker.js')
    ↓
Browser OS thread pool allocates a new thread
    ↓
V8 creates a NEW isolate (separate JS engine instance)
    ↓
worker.js is downloaded/parsed/compiled in that isolate
    ↓
Worker is ready — main thread and worker run in PARALLEL
```

Key point: each worker gets its **own V8 isolate** — completely separate heap,
call stack, event loop. They share nothing by default.

### The postMessage / onmessage Lifecycle

```
Main Thread                          Worker Thread
──────────────                       ─────────────────
worker.postMessage(data)
    ↓
Structured Clone Algorithm clones data
    ↓
                           →         self.onmessage fires
                                          ↓
                                     Process data
                                          ↓
                           ←         self.postMessage(result)
                                     ↓
                           Structured Clone clones result
                                          ↓
worker.onmessage fires ←
    ↓
Update UI with result
```

### Structured Clone vs Transferable — Memory Impact

```
Data: ArrayBuffer of 100MB

postMessage(buffer)          → 100MB COPIED  → 200MB total memory used
postMessage(buffer, [buffer])→ 0MB copied    → 100MB total, buffer transferred
                                               (original reference becomes detached)
```

### SharedArrayBuffer + Atomics (Thread Safety)

```javascript
// Shared memory — both threads read/write the SAME bytes
const shared = new SharedArrayBuffer(4);
const view   = new Int32Array(shared);

// WITHOUT Atomics — race condition possible
view[0] += 1; // read-modify-write is NOT atomic

// WITH Atomics — thread-safe
Atomics.add(view, 0, 1); // atomic increment
```

---

## 5. Visual Architecture

### Browser Threading Model

```
┌─────────────────────────────────────────────────────────────┐
│                        BROWSER                              │
│                                                             │
│  ┌─────────────────────────────┐                            │
│  │       MAIN THREAD           │                            │
│  │  ┌──────────┐ ┌──────────┐  │                            │
│  │  │Call Stack│ │Event Loop│  │                            │
│  │  └──────────┘ └──────────┘  │                            │
│  │  DOM, CSSOM, Rendering      │                            │
│  │  User Events                │◄──── postMessage ────┐     │
│  └─────────────────────────────┘                      │     │
│                                                        │     │
│  ┌─────────────────────────────┐                      │     │
│  │     DEDICATED WORKER        │                      │     │
│  │  ┌──────────┐ ┌──────────┐  │──── postMessage ─────┘     │
│  │  │Call Stack│ │Event Loop│  │                            │
│  │  └──────────┘ └──────────┘  │                            │
│  │  NO DOM, NO window          │                            │
│  └─────────────────────────────┘                            │
│                                                             │
│  ┌─────────────────────────────┐                            │
│  │     SERVICE WORKER          │                            │
│  │  Network proxy              │                            │
│  │  Intercepts fetch requests  │                            │
│  └─────────────────────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

### postMessage Data Flow

```
Main Thread                  Worker Thread
─────────────────            ────────────────────
const w = new Worker(url)
w.postMessage({ data })  ──► self.onmessage = (e) => {
                               // e.data = copy of { data }
                               const result = heavyWork(e.data)
                               self.postMessage(result)
                             }
w.onmessage = (e) => {  ◄──
  updateUI(e.data)
}
```

### Transferable Objects — Ownership Transfer

```
Before transfer:
  Main Thread:  [ArrayBuffer ████████████] ← owns it
  Worker:       (none)

After postMessage(buffer, [buffer]):
  Main Thread:  [ArrayBuffer detached]     ← can no longer use
  Worker:       [ArrayBuffer ████████████] ← now owns it

After worker sends back:
  Main Thread:  [ArrayBuffer ████████████] ← ownership returned
  Worker:       [ArrayBuffer detached]
```

---

## 6. Syntax & API

### Creating a Dedicated Worker

```javascript
// main.js
const worker = new Worker('worker.js');         // from file
// OR inline worker (no separate file needed):
const blob   = new Blob([workerCode], { type: 'application/javascript' });
const worker = new Worker(URL.createObjectURL(blob));
```

### Sending & Receiving Messages

```javascript
// main.js — send data to worker
worker.postMessage({ type: 'SORT', payload: [3, 1, 2] });

// main.js — receive result from worker
worker.onmessage = (event) => {
  console.log('Result:', event.data);
};

// main.js — handle errors from worker
worker.onerror = (error) => {
  console.error('Worker error:', error.message, error.filename, error.lineno);
};
```

```javascript
// worker.js — receive messages
self.onmessage = (event) => {
  const { type, payload } = event.data;
  if (type === 'SORT') {
    const sorted = payload.sort((a, b) => a - b);
    self.postMessage(sorted);
  }
};

// worker.js — can also use addEventListener
self.addEventListener('message', (event) => { ... });
```

### Transferable Objects — zero-copy transfer

```javascript
// main.js
const buffer = new ArrayBuffer(1024 * 1024 * 100); // 100MB
// Transfer (not copy) — buffer is now unusable in main thread
worker.postMessage(buffer, [buffer]);
console.log(buffer.byteLength); // 0 — detached

// worker.js
self.onmessage = (e) => {
  const buffer = e.data; // 100MB, zero copy
  // process...
  self.postMessage(buffer, [buffer]); // transfer back
};
```

### Terminating a Worker

```javascript
// From main thread
worker.terminate(); // immediate, no cleanup

// From inside the worker
self.close(); // graceful
```

### Inline Worker (no separate file)

```javascript
const workerCode = `
  self.onmessage = (e) => {
    const result = e.data.reduce((sum, n) => sum + n, 0);
    self.postMessage(result);
  };
`;

const blob   = new Blob([workerCode], { type: 'application/javascript' });
const url    = URL.createObjectURL(blob);
const worker = new Worker(url);
URL.revokeObjectURL(url); // clean up after creation

worker.postMessage([1, 2, 3, 4, 5]);
worker.onmessage = (e) => console.log('Sum:', e.data); // 15
```

### SharedArrayBuffer + Atomics

```javascript
// main.js
const sab  = new SharedArrayBuffer(4);  // 4 bytes = 1 Int32
const view = new Int32Array(sab);

worker.postMessage(sab); // no copy — shared memory

// Poll for result
const interval = setInterval(() => {
  if (Atomics.load(view, 0) === 1) {
    clearInterval(interval);
    console.log('Worker done');
  }
}, 50);

// worker.js
self.onmessage = (e) => {
  const view = new Int32Array(e.data);
  // do heavy work...
  Atomics.store(view, 0, 1);   // signal main thread
  Atomics.notify(view, 0, 1);  // wake up waiting threads
};
```

---

## 7. Practical Examples

### Simple — Offload a sort

```javascript
// worker.js
self.onmessage = (e) => {
  const sorted = [...e.data].sort((a, b) => a - b);
  self.postMessage(sorted);
};

// main.js
const worker = new Worker('worker.js');
const data   = Array.from({ length: 1000000 }, () => Math.random());

console.time('sort');
worker.postMessage(data);
worker.onmessage = (e) => {
  console.timeEnd('sort'); // logs time taken, UI never froze
  console.log('First 5:', e.data.slice(0, 5));
};
```

### Intermediate — Worker pool pattern

For repeated tasks, reuse workers instead of creating/destroying them each time:

```javascript
class WorkerPool {
  constructor(url, size = navigator.hardwareConcurrency || 4) {
    this.workers  = Array.from({ length: size }, () => new Worker(url));
    this.queue    = [];
    this.idle     = [...this.workers];
  }

  run(data) {
    return new Promise((resolve, reject) => {
      const task = { data, resolve, reject };
      if (this.idle.length > 0) {
        this._dispatch(task);
      } else {
        this.queue.push(task);
      }
    });
  }

  _dispatch(task) {
    const worker = this.idle.pop();
    worker.onmessage  = (e) => {
      task.resolve(e.data);
      if (this.queue.length > 0) {
        this._dispatch(this.queue.shift());
      } else {
        this.idle.push(worker);
      }
    };
    worker.onerror = (e) => task.reject(e);
    worker.postMessage(task.data);
  }
}

// Usage
const pool = new WorkerPool('sort-worker.js', 4);
const results = await Promise.all(chunks.map(chunk => pool.run(chunk)));
```

### Advanced — WASM + Worker (AI inference pattern)

```javascript
// ai-worker.js — runs ONNX model inference off main thread
importScripts('https://cdn.jsdelivr.net/npm/onnxruntime-web/dist/ort.min.js');

let session;

self.onmessage = async (e) => {
  if (e.data.type === 'LOAD') {
    session = await ort.InferenceSession.create(e.data.modelUrl);
    self.postMessage({ type: 'READY' });
  }
  if (e.data.type === 'INFER') {
    const tensor = new ort.Tensor('float32', e.data.input, [1, e.data.input.length]);
    const output = await session.run({ input: tensor });
    self.postMessage({ type: 'RESULT', data: output });
  }
};
```

---

## 8. Real-world Usage

### In Angular

Angular CLI apps run in a browser — Web Workers integrate naturally:

```bash
ng generate web-worker app  # Angular CLI scaffolds the worker
```

```typescript
// app.component.ts
if (typeof Worker !== 'undefined') {
  const worker = new Worker(new URL('./app.worker', import.meta.url));
  worker.onmessage = ({ data }) => {
    console.log('Result from worker:', data);
  };
  worker.postMessage({ type: 'PROCESS', payload: largeDataSet });
}
```

Common Angular use cases:
- Heavy data transformation before rendering
- CSV/Excel file parsing
- Image manipulation with Canvas API
- Search/filter over large datasets

### In Your AI Project (convopro_chatgpt)

```javascript
// Move embedding computation off main thread
const embeddingWorker = new Worker('embedding-worker.js');

embeddingWorker.postMessage({ text: userInput });
embeddingWorker.onmessage = ({ data: embedding }) => {
  // embedding computed in worker, main thread free for UI
  vectorDb.search(embedding).then(updateChatUI);
};
```

### Browser Support

Web Workers: supported in all modern browsers since 2010. 98%+ global support.
SharedArrayBuffer: requires HTTPS + cross-origin isolation headers:
```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

---

## 9. Performance

### When to use a Worker (the 50ms rule)

Chrome's RAIL model says: any task taking >50ms on the main thread is perceptible
as jank. If your computation takes longer, move it to a worker.

```javascript
// Measure before deciding
console.time('task');
const result = heavyTask(data);
console.timeEnd('task');
// > 50ms? → Worker candidate
```

### postMessage serialization cost

```
Structured clone of 1MB object:  ~1–2ms   (acceptable)
Structured clone of 10MB object: ~10–20ms (noticeable)
Structured clone of 100MB:       ~100ms+  (use Transferable!)
```

### Worker startup cost

```
new Worker(url): ~10–50ms first time (parse + compile worker.js)
```
**Solution:** Create workers at app startup, keep them alive, reuse them.

### CPU cores

```javascript
const cores = navigator.hardwareConcurrency; // e.g., 8
// Optimal worker pool size = cores - 1 (leave one for main thread)
```

### Benchmark: main thread vs worker

```
Task: sort 5 million numbers

Main thread:
  Sort time:    2800ms
  UI frozen:    YES — completely unresponsive

Web Worker:
  Sort time:    2900ms  (slightly slower — postMessage overhead)
  UI frozen:    NO — smooth 60fps throughout
```

The worker is slightly slower but the user experience is dramatically better.

---

## 10. Common Mistakes

### Mistake 1 — Trying to access the DOM from a worker

```javascript
// worker.js
document.getElementById('result').textContent = data; // ReferenceError: document is not defined
```
Workers have no DOM. Send the result back via `postMessage` and update DOM on main thread.

### Mistake 2 — Creating a new Worker for every task

```javascript
// Bad — expensive, creates new OS thread each time
button.addEventListener('click', () => {
  const w = new Worker('worker.js');
  w.postMessage(data);
  w.onmessage = (e) => { w.terminate(); updateUI(e.data); };
});
```
Use a worker pool or keep one worker alive and reuse it.

### Mistake 3 — Forgetting that postMessage COPIES data

```javascript
const obj = { value: 42 };
worker.postMessage(obj);
obj.value = 100; // This change does NOT reach the worker — already copied
```

### Mistake 4 — Not handling worker errors

```javascript
const worker = new Worker('worker.js');
// No onerror handler — silent failures in production
```
Always add `worker.onerror` to catch syntax errors, runtime errors, or failed imports.

### Mistake 5 — Using SharedArrayBuffer without Atomics

```javascript
// Race condition — two workers incrementing simultaneously
view[0] += 1; // NOT atomic: read → add → write (can be interrupted)

// Safe
Atomics.add(view, 0, 1); // atomic — cannot be interrupted
```

---

## 11. Best Practices

1. **Create workers at startup, not on demand** — avoids 50ms startup penalty per task
2. **Use a Worker Pool** — match pool size to `navigator.hardwareConcurrency - 1`
3. **Use Transferable Objects for large data** — zero-copy, avoids serialization overhead
4. **Always handle `onerror`** — silent worker failures are hard to debug
5. **Keep worker code focused** — one worker = one responsibility (sort, parse, infer)
6. **Terminate idle workers** — workers consume memory even when not running
7. **Use `URL.createObjectURL` for inline workers** — avoids needing a separate file in bundled apps
8. **Combine with WASM** — for peak performance: WASM runs computation, Worker keeps it off main thread
9. **Use `Atomics.wait` / `Atomics.notify` sparingly** — prefer `postMessage` for most coordination
10. **Test on low-end devices** — workers shine on mobile where CPU is constrained

---

## 12. Debugging

### Chrome DevTools — Threads Panel

1. Open DevTools → **Sources** tab
2. In the left panel, find **Threads** section
3. Click the worker thread to inspect its call stack, set breakpoints
4. Each worker gets its own isolated DevTools context

### Debugging postMessage flow

```javascript
// Add logging on both sides
worker.postMessage(data);
console.log('[Main] Sent:', data);

// worker.js
self.onmessage = (e) => {
  console.log('[Worker] Received:', e.data); // shows in DevTools Sources → worker thread
};
```

### Checking worker status

```javascript
// Workers don't expose a status property — track it yourself
class ManagedWorker {
  constructor(url) {
    this.worker = new Worker(url);
    this.busy   = false;
    this.worker.onmessage = (e) => {
      this.busy = false;
      this.onResult?.(e.data);
    };
  }
}
```

### Performance tab — spotting main thread blocks

1. DevTools → **Performance** tab → Record
2. Perform the heavy operation
3. Look for long **yellow** blocks on the Main thread row
4. If you see 500ms+ blocks → move that work to a Worker

### Memory — checking for worker leaks

Workers that are created but never terminated keep their memory allocated.
DevTools → **Memory** → Heap Snapshot → search for `Worker` objects.

---

## 13. Advanced Concepts

### Service Workers

Service Workers are a special type of worker that acts as a **network proxy**:

```javascript
// Register a service worker
navigator.serviceWorker.register('/sw.js').then(reg => {
  console.log('SW registered:', reg.scope);
});

// sw.js — intercept all fetch requests
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      return cached || fetch(event.request);
    })
  );
});
```

Key differences from Dedicated Workers:
- Persists even after the page closes (lives in browser background)
- Controls network requests for all pages in its scope
- Used for: offline-first apps, push notifications, background sync

### OffscreenCanvas — rendering in a Worker

```javascript
// Move canvas rendering off main thread
const canvas         = document.getElementById('myCanvas');
const offscreen      = canvas.transferControlToOffscreen();
const renderWorker   = new Worker('render-worker.js');

renderWorker.postMessage({ canvas: offscreen }, [offscreen]);

// render-worker.js
self.onmessage = (e) => {
  const canvas = e.data.canvas;
  const ctx    = canvas.getContext('2d');
  // All rendering happens in worker — zero impact on main thread
  ctx.fillRect(0, 0, 100, 100);
};
```

### Module Workers (modern)

```javascript
// Supports ES module imports in worker
const worker = new Worker('./worker.js', { type: 'module' });

// worker.js — can now use import
import { heavyFunction } from './utils.js';
```

### Comlink — RPC-style Worker API

[Comlink](https://github.com/GoogleChromeLabs/comlink) by Google Chrome Labs wraps
the `postMessage` API into a clean RPC interface:

```javascript
// worker.js
import * as Comlink from 'comlink';
Comlink.expose({ sort: (arr) => arr.sort() });

// main.js
const worker = new Worker('worker.js', { type: 'module' });
const api    = Comlink.wrap(worker);
const result = await api.sort([3, 1, 2]); // feels like a direct function call
```

---

## 14. Interview Questions

### Beginner

**Q: What is a Web Worker?**
> A browser API that lets you run JavaScript in a background thread, separate from
> the main UI thread. Used to offload CPU-heavy work without blocking the UI.

**Q: Can Web Workers access the DOM?**
> No. Workers have no access to `window`, `document`, or any DOM APIs. They
> communicate with the main thread exclusively via `postMessage`.

**Q: How do Web Workers communicate with the main thread?**
> Via the `postMessage` / `onmessage` API. Data is copied using the structured
> clone algorithm. For large data, Transferable Objects can transfer ownership
> instead of copying.

---

### Intermediate

**Q: What is the difference between postMessage copying and Transferable Objects?**
> `postMessage(data)` uses structured clone — creates a complete copy of the data.
> For large ArrayBuffers this is expensive in both time and memory.
> `postMessage(buffer, [buffer])` transfers ownership — zero-copy, the original
> reference becomes detached. Much faster for large binary data.

**Q: When would you NOT use a Web Worker?**
> When the task is fast (<50ms), when the overhead of postMessage serialization
> exceeds the benefit, when the worker needs DOM access, or when you need the
> result synchronously.

**Q: What is SharedArrayBuffer and why does it require special headers?**
> A fixed-length buffer shared between multiple threads without copying. Requires
> `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp`
> headers because shared memory + high-precision timers can enable Spectre-class
> side-channel attacks. The headers ensure cross-origin isolation.

---

### Advanced

**Q: How would you implement a Worker Pool?**
> Create N workers at startup (N = `navigator.hardwareConcurrency - 1`). Maintain
> an idle queue and a task queue. When a task arrives, dispatch it to an idle worker.
> When a worker finishes, return it to the idle pool or pick up the next queued task.
> This avoids the 50ms startup cost of creating new workers per task.

**Q: How would you do canvas rendering at 60fps without impacting UI?**
> Use `OffscreenCanvas`. Transfer control of the canvas to a worker via
> `canvas.transferControlToOffscreen()`. The worker handles all draw calls without
> touching the main thread. `requestAnimationFrame` runs in the worker context.

---

### Staff Level

**Q: How would you architect browser-side AI inference without blocking the UI?**
> Layer the solution: (1) ONNX Runtime Web or TensorFlow.js for model inference,
> (2) run inside a Dedicated Worker to keep main thread free, (3) use Transferable
> Objects for input tensors to avoid copying large float arrays, (4) pool multiple
> workers for concurrent requests, (5) stream partial results back via postMessage
> for perceived responsiveness. For models too large for browser memory, fall back
> to streaming API calls with SSE.

---

## 15. Hands-on Exercises

### Exercise 1 — Predict the output

```javascript
// main.js
const w = new Worker(URL.createObjectURL(new Blob([`
  self.onmessage = (e) => {
    self.postMessage(e.data * 2);
  };
`], { type: 'application/javascript' })));

w.onmessage = (e) => console.log('B:', e.data);
w.postMessage(21);
console.log('A');
```

**What is the output order? Why?**

<details>
<summary>Answer</summary>

```
A
B: 42
```
`console.log('A')` runs synchronously on the main thread.
The `onmessage` callback fires asynchronously when the worker responds — after the
current synchronous code completes.
</details>

---

### Exercise 2 — Fix the bug

```javascript
const worker = new Worker('worker.js');

function processData(data) {
  worker.postMessage(data);
}

// Bug: this is called 100 times rapidly — what goes wrong?
for (let i = 0; i < 100; i++) {
  processData(hugeDataArray);
}
```

<details>
<summary>Answer</summary>

Messages queue up. The worker processes them sequentially. If each takes 500ms,
the last message waits 50 seconds. Fix: use a worker pool, or debounce the calls,
or cancel pending work when new data arrives using a message ID system.
</details>

---

### Exercise 3 — Build from scratch

Create an inline Web Worker that:
1. Accepts an array of numbers
2. Computes the sum, min, max, and average
3. Returns all four values
4. Main thread updates the DOM without any freezing

```javascript
// YOUR IMPLEMENTATION
```

<details>
<summary>Solution</summary>

```javascript
const workerCode = `
  self.onmessage = ({ data: arr }) => {
    const sum = arr.reduce((a, b) => a + b, 0);
    self.postMessage({
      sum,
      min: Math.min(...arr),
      max: Math.max(...arr),
      avg: sum / arr.length,
    });
  };
`;

const worker = new Worker(
  URL.createObjectURL(new Blob([workerCode], { type: 'application/javascript' }))
);

worker.onmessage = ({ data }) => {
  document.getElementById('result').textContent =
    `Sum: ${data.sum} | Min: ${data.min} | Max: ${data.max} | Avg: ${data.avg.toFixed(2)}`;
};

const bigArray = Array.from({ length: 5000000 }, () => Math.random() * 1000);
worker.postMessage(bigArray);
```
</details>

---

## 16. Mini Project

### Build: Background Data Processor

Build a small web page that demonstrates Web Workers in action:

**Features:**
1. A button that triggers sorting 5 million numbers
2. A live counter ticking every 100ms to prove the UI is NOT frozen
3. Progress updates from the worker (split work into chunks, report % complete)
4. A "Cancel" button that terminates the worker mid-task

```
index.html
worker.js
main.js
```

**Starter structure:**

```javascript
// worker.js
self.onmessage = ({ data: { type, payload } }) => {
  if (type === 'SORT') {
    const CHUNK = 100000;
    const arr   = payload;
    // Sort in chunks and report progress
    for (let i = 0; i < arr.length; i += CHUNK) {
      arr.slice(i, i + CHUNK).sort((a, b) => a - b);
      self.postMessage({ type: 'PROGRESS', pct: Math.round((i / arr.length) * 100) });
    }
    self.postMessage({ type: 'DONE', result: arr.sort((a, b) => a - b) });
  }
};

// main.js
let worker;

document.getElementById('start').onclick = () => {
  worker = new Worker('worker.js');
  const data = Array.from({ length: 5000000 }, () => Math.random());

  worker.postMessage({ type: 'SORT', payload: data });
  worker.onmessage = ({ data: msg }) => {
    if (msg.type === 'PROGRESS') updateProgressBar(msg.pct);
    if (msg.type === 'DONE')     showResult(msg.result);
  };
};

document.getElementById('cancel').onclick = () => {
  worker?.terminate();
  resetUI();
};
```

---

## 17. Revision

### 5-Minute Summary

- **Web Worker** = background JS thread with its own V8 isolate. No DOM access.
- **Communication** = `postMessage` / `onmessage` only. Data is **copied** (structured clone).
- **Transferable Objects** = zero-copy transfer of `ArrayBuffer`. Original becomes detached.
- **SharedArrayBuffer** = true shared memory. Needs `Atomics` for thread safety + COOP/COEP headers.
- **50ms rule** = if a task blocks the main thread >50ms, move it to a worker.
- **Worker Pool** = create workers at startup, reuse them, match count to CPU cores.
- **Service Worker** = special worker that proxies network requests. Used for offline, caching, push.
- **OffscreenCanvas** = move canvas rendering to a worker for zero main-thread impact.

### Top 10 Flash Cards

| Q | A |
|---|---|
| Can a worker access the DOM? | No |
| How do workers communicate? | postMessage / onmessage |
| What algorithm copies data in postMessage? | Structured clone |
| What avoids copying in postMessage? | Transferable Objects |
| What is SharedArrayBuffer? | Shared binary memory between threads |
| What prevents race conditions on SharedArrayBuffer? | Atomics |
| What is the 50ms rule? | Tasks >50ms should move to a worker |
| What headers does SharedArrayBuffer require? | COOP: same-origin + COEP: require-corp |
| What is a Service Worker? | A network proxy worker for offline/caching |
| What is OffscreenCanvas? | Canvas API available in workers |

---

## 18. Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                  WEB WORKERS — CHEAT SHEET                       │
├──────────────────────────────────────────────────────────────────┤
│ CREATE                                                           │
│   new Worker('worker.js')                    from file           │
│   new Worker(url, { type: 'module' })        ES module worker    │
│   new Worker(URL.createObjectURL(blob))      inline worker       │
│                                                                  │
│ COMMUNICATE                                                      │
│   worker.postMessage(data)                   main → worker       │
│   self.postMessage(result)                   worker → main       │
│   worker.onmessage = (e) => e.data           receive in main     │
│   self.onmessage = (e) => e.data             receive in worker   │
│   worker.onerror = (e) => ...                handle errors       │
│                                                                  │
│ TRANSFERABLE (zero-copy)                                         │
│   worker.postMessage(buffer, [buffer])       transfer ownership  │
│   buffer.byteLength === 0 after transfer     detached            │
│                                                                  │
│ SHARED MEMORY                                                    │
│   new SharedArrayBuffer(bytes)               shared buffer       │
│   new Int32Array(sab)                        typed view          │
│   Atomics.add/load/store/wait/notify         thread-safe ops     │
│   Requires: COOP + COEP headers              security isolation  │
│                                                                  │
│ TERMINATE                                                        │
│   worker.terminate()                         from main (force)   │
│   self.close()                               from worker (clean) │
│                                                                  │
│ WORKER CANNOT ACCESS                                             │
│   window, document, DOM, localStorage                            │
│                                                                  │
│ WORKER CAN ACCESS                                                │
│   fetch, WebSocket, IndexedDB, setTimeout, crypto, WASM          │
│                                                                  │
│ WHEN TO USE                                                      │
│   Task > 50ms on main thread → Worker                           │
│   Large binary data → Transferable Objects                       │
│   Cross-tab state → Shared Worker                                │
│   Network proxy/offline → Service Worker                         │
│   Canvas rendering → OffscreenCanvas + Worker                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 19. References

### Official Documentation
- [MDN — Using Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)
- [MDN — SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)
- [MDN — Transferable Objects](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects)
- [HTML Spec — Web Workers](https://html.spec.whatwg.org/multipage/workers.html)

### Best Articles
- [The State of Web Workers in 2021 — Surma (Google Chrome)](https://web.dev/workers-overview/)
- [Use Web Workers to run JavaScript off the browser's main thread](https://web.dev/off-main-thread/)
- [SharedArrayBuffer and Atomics — Exploring JS](https://exploringjs.com/es2016-es2017/ch_shared-array-buffer.html)

### Best Videos
- [Web Workers — Surma (Google) — Chrome Dev Summit](https://www.youtube.com/watch?v=7Rrv9qFMWNM)
- [Parallel programming in JavaScript using Web Workers — Fireship](https://www.youtube.com/watch?v=Gd5J08uq8cQ)

### Best GitHub Repositories
- [Comlink — RPC-style Web Workers — GoogleChromeLabs](https://github.com/GoogleChromeLabs/comlink)
- [Workerize — run a module in a worker — developit](https://github.com/developit/workerize)
- [threads.js — high-level Web Worker abstraction](https://github.com/andywer/threads.js)

### RFC / Specification
- [TC39 — Shared Memory and Atomics](https://tc39.es/ecma262/#sec-atomics-object)
- [WHATWG — Web Workers Living Standard](https://html.spec.whatwg.org/multipage/workers.html)

---

> **Next topic:** `03-scope-chain-lexical-scope-hoisting.md`
> Say **"next topic"** when ready.
