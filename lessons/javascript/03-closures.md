# JavaScript — Closures

> **Phase 1 | Week 1 | Topic 3 of 30**
> **Difficulty:** Foundation → Intermediate
> **Time to complete:** 2–3 hours
> **Status:** ⬜ Not Started

---

## Table of Contents

**Core**
1. [Introduction](#1-introduction)
2. [How Closures Work Internally](#2-how-closures-work-internally)
3. [Practical Uses](#3-practical-uses)
4. [Closure Traps](#4-closure-traps)
5. [Common Mistakes](#5-common-mistakes)
6. [Interview Questions](#6-interview-questions)
7. [Hands-on Exercises](#7-hands-on-exercises)
8. [Revision & Cheat Sheet](#8-revision--cheat-sheet)

**Good to Know**
9. [Memory & Garbage Collection](#9-memory--garbage-collection)
10. [Closures in Angular & RxJS](#10-closures-in-angular--rxjs)

---

## 1. Introduction

### What is a closure?

A closure is a function that **remembers the variables from the scope where it was
defined**, even after that outer scope has finished executing.

```javascript
function outer() {
  const message = 'hello'; // defined in outer's scope

  function inner() {
    console.log(message);  // inner remembers message — this is a closure
  }

  return inner;
}

const fn = outer(); // outer() has finished — message "should" be gone
fn();               // prints "hello" — it's still there, captured by inner
```

`inner` closed over `message`. Even though `outer()` has returned, `message` is
kept alive in memory because `inner` still holds a reference to it.

### The one-line definition

> A closure is a function bundled together with its surrounding lexical environment.

### Why it matters

Closures are the mechanism behind:
- Private variables (no `private` keyword needed)
- Factory functions
- The module pattern
- Memoization / caching
- Every callback you've ever written in JavaScript
- The `for` loop + async trap (most common interview question)

---

## 2. How Closures Work Internally

When JavaScript creates a function, it attaches a reference to the **lexical
environment** where that function was defined. This is called the function's
`[[Environment]]` slot.

When the function executes and looks up a variable, it:
1. Checks its own local scope first
2. If not found, follows `[[Environment]]` to the outer scope
3. Keeps going up the scope chain until found or `ReferenceError`

This is just the scope chain — but a closure is the specific case where a function
outlives the scope it was defined in, keeping that scope's variables alive.

```javascript
function makeCounter() {
  let count = 0;           // lives in makeCounter's scope

  return function() {
    count++;               // this function closes over count
    return count;
  };
}

const counter = makeCounter(); // makeCounter() is done, but count survives
counter(); // 1
counter(); // 2
counter(); // 3
```

Each call to `counter()` mutates the same `count` variable — because it's the
same closure, holding a reference to the same environment.

### Two separate closures = two separate environments

```javascript
const c1 = makeCounter();
const c2 = makeCounter();

c1(); // 1
c1(); // 2
c2(); // 1  ← c2 has its OWN count — not shared with c1
```

Each call to `makeCounter()` creates a **new lexical environment** with its own
`count`. The two closures are independent.

---

## 3. Practical Uses

### 1 — Private variables (data encapsulation)

JavaScript has no `private` keyword at the function level. Closures give you
private state without classes:

```javascript
function makeUser(name) {
  let _loginCount = 0;    // private — not accessible outside

  return {
    getName:  () => name,
    login:    () => ++_loginCount,
    getCount: () => _loginCount,
  };
}

const user = makeUser('Mriganka');
user.login();
user.login();
console.log(user.getCount()); // 2
console.log(user._loginCount); // undefined — private
```

### 2 — Factory functions

A factory creates multiple instances, each with their own closed-over state:

```javascript
function makeMultiplier(factor) {
  return (n) => n * factor; // closes over factor
}

const double = makeMultiplier(2);
const triple = makeMultiplier(3);

double(5); // 10
triple(5); // 15
```

### 3 — Memoization (caching expensive results)

```javascript
function memoize(fn) {
  const cache = {};  // closed over by the returned function

  return function(n) {
    if (cache[n] !== undefined) {
      return cache[n]; // return cached result
    }
    cache[n] = fn(n);  // compute and store
    return cache[n];
  };
}

const expensiveCalc = memoize((n) => {
  console.log('computing...');
  return n * n;
});

expensiveCalc(4); // computing... → 16
expensiveCalc(4); // (no log) → 16  ← from cache
expensiveCalc(5); // computing... → 25
```

### 4 — The Module Pattern (pre-ES modules)

Before `import`/`export` existed, closures were the way to create modules with
private state and a public API:

```javascript
const bankAccount = (function() {
  let balance = 0;       // private

  return {
    deposit:     (n) => { balance += n; },
    withdraw:    (n) => { balance -= n; },
    getBalance:  ()  => balance,
  };
})(); // IIFE — immediately invoked, creates the closure once

bankAccount.deposit(1000);
bankAccount.withdraw(200);
console.log(bankAccount.getBalance()); // 800
console.log(bankAccount.balance);      // undefined — private
```

### 5 — Callbacks and event handlers

Every callback you pass to `setTimeout`, `addEventListener`, or `.then()` is a
closure — it captures the surrounding scope:

```javascript
function setupButton(buttonId, message) {
  document.getElementById(buttonId).addEventListener('click', () => {
    alert(message); // closes over message — works even after setupButton returns
  });
}

setupButton('btn1', 'Hello from button 1');
setupButton('btn2', 'Hello from button 2');
```

Each click handler closes over its own `message`. They don't interfere.

---

## 4. Closure Traps

These are the patterns that appear in interviews and catch people off guard.

### Trap 1 — `var` in a loop (the classic)

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 3, 3, 3   — NOT 0, 1, 2
```

**Why?** `var` is function-scoped — there is only **one** `i` shared across all
iterations. All three callbacks close over the same `i`. By the time they run
(100ms later), the loop has finished and `i` is `3`.

**Fix 1 — use `let`** (block-scoped, creates a new `i` per iteration):

```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2  ✅
```

**Fix 2 — IIFE to capture `i` at each iteration:**

```javascript
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 100);
  })(i); // j is a new variable in each IIFE scope, capturing i's current value
}
// Output: 0, 1, 2  ✅
```

---

### Trap 2 — Shared mutable state

```javascript
function makeAdders() {
  const adders = [];
  for (var i = 0; i < 3; i++) {
    adders.push(() => i); // all three close over the SAME i
  }
  return adders;
}

const adders = makeAdders();
adders[0](); // 3
adders[1](); // 3
adders[2](); // 3
```

Same `var` problem. Fix: use `let` or capture via IIFE/parameter.

---

### Trap 3 — Closure in a method loses `this`

```javascript
const obj = {
  name: 'Mriganka',
  greet() {
    setTimeout(function() {
      console.log(this.name); // undefined — 'this' is window/undefined in strict
    }, 100);
  }
};
obj.greet();
```

`function()` inside `setTimeout` has its own `this` — it doesn't inherit `obj`.
The closure captures the surrounding **scope** but not the surrounding **`this`**.

**Fix — arrow function** (no own `this`, inherits from enclosing lexical scope):

```javascript
const obj = {
  name: 'Mriganka',
  greet() {
    setTimeout(() => {
      console.log(this.name); // 'Mriganka' ✅ — arrow inherits this from greet
    }, 100);
  }
};
obj.greet();
```

---

### Trap 4 — Stale closure (the React/Angular trap)

A closure captures the **value of a variable at the time it was created** — or more
precisely, a reference to the variable. If the variable later changes in a way the
closure can't see (e.g. a new variable binding in a new render), you get a stale value.

```javascript
function makeLogger(value) {
  return () => console.log(value); // captures value at creation time
}

let x = 1;
const log = makeLogger(x);
x = 2;
log(); // 1 — captures the original binding, not the updated x
```

This is because `makeLogger` takes `value` as a **parameter** — a new binding each
call. Reassigning `x` outside doesn't affect `value` inside the closure.

Contrast with closing over a variable directly:

```javascript
let x = 1;
const log = () => console.log(x); // closes over x directly
x = 2;
log(); // 2 — same binding, sees the update
```

---

## 5. Common Mistakes

### 1 — Expecting `var` loop closures to capture per-iteration values

```javascript
const fns = [];
for (var i = 0; i < 5; i++) {
  fns.push(() => i);
}
fns[0](); // 5, not 0
fns[2](); // 5, not 2
// Fix: use let
```

### 2 — Accidentally creating memory leaks

```javascript
function attachHandler() {
  const largeData = new Array(1000000).fill('data'); // large object

  document.getElementById('btn').addEventListener('click', () => {
    console.log('clicked'); // closes over largeData — even if never used
  });
}
// largeData stays in memory as long as the event listener exists
// Fix: only close over what you need, or remove the listener when done
```

### 3 — Thinking `this` is captured by a closure

```javascript
function Timer() {
  this.seconds = 0;
  setInterval(function() {
    this.seconds++; // ❌ this is window/undefined, not the Timer instance
  }, 1000);
}

// Fix:
function Timer() {
  this.seconds = 0;
  setInterval(() => {
    this.seconds++; // ✅ arrow function inherits this from Timer constructor
  }, 1000);
}
```

---

## 6. Interview Questions

### Beginner

**Q: What is a closure?**
> A closure is a function that retains access to variables from its outer lexical
> scope, even after that outer function has returned. The function and its
> surrounding environment are bundled together.

**Q: What is the output?**
```javascript
function outer() {
  let x = 10;
  return function() { return x; };
}
const fn = outer();
console.log(fn());
```
> `10`. `fn` closes over `x` from `outer`'s scope. Even though `outer()` has
> returned, `x` is kept alive by the closure.

**Q: What is the output of the `var` loop?**
```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```
> `3, 3, 3`. `var` is function-scoped — one `i` shared by all callbacks. The loop
> completes before any callback runs, so all see `i = 3`. Fix: use `let`.

---

### Intermediate

**Q: How does the IIFE fix the `var` loop problem?**
> The IIFE creates a new function scope on each iteration and passes `i` as an
> argument. The argument `j` is a new binding per iteration, so each closure
> captures a different `j`.

**Q: What is the difference between closing over a variable vs closing over a value?**
> Closures always capture a **reference to the variable**, not a snapshot of its
> value at creation time. If the variable changes later, the closure sees the
> updated value — unless the variable is a parameter (new binding per call) or a
> `const` (can't be reassigned).

**Q: Why does `this` inside a `setTimeout` callback not refer to the outer object?**
> A regular `function` callback creates its own execution context with its own
> `this` — which defaults to `window` (or `undefined` in strict mode). Closures
> capture lexical scope (variables), not `this`. Arrow functions fix this because
> they have no own `this` — they inherit `this` from the enclosing scope.

---

### Advanced

**Q: Implement a `once(fn)` function that only executes `fn` the first time it's called.**

```javascript
function once(fn) {
  let called = false;
  let result;
  return function(...args) {
    if (!called) {
      called = true;
      result = fn(...args);
    }
    return result;
  };
}

const init = once(() => console.log('initialised'));
init(); // initialised
init(); // (nothing)
init(); // (nothing)
```

> Uses closure to remember `called` and `result` across calls. `fn` is only
> executed once regardless of how many times the returned function is called.

**Q: Implement `memoize(fn)` from scratch.**

```javascript
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
```

---

## 7. Hands-on Exercises

### Exercise 1 — Predict the output

```javascript
function makeCounter() {
  let count = 0;
  return {
    inc: () => ++count,
    dec: () => --count,
    get: () => count,
  };
}

const c = makeCounter();
c.inc(); c.inc(); c.inc();
c.dec();
console.log(c.get());
```

<details>
<summary>Answer</summary>
`2` — incremented 3 times (count = 3), decremented once (count = 2).
All three methods close over the same `count` variable.
</details>

---

### Exercise 2 — Fix the bug

```javascript
const fns = [];
for (var i = 0; i < 5; i++) {
  fns.push(() => i);
}
// fns[0]() should return 0, fns[3]() should return 3
```

Fix it two ways: using `let`, and using an IIFE.

<details>
<summary>Solution</summary>

```javascript
// Fix 1: let
for (let i = 0; i < 5; i++) {
  fns.push(() => i);
}

// Fix 2: IIFE
for (var i = 0; i < 5; i++) {
  (function(j) {
    fns.push(() => j);
  })(i);
}
```
</details>

---

### Exercise 3 — Implement from scratch

Build `makeCounter(start, step)` that:
- Starts counting from `start`
- Increments by `step` each time
- Has `increment()`, `decrement()`, `reset()`, `value()` methods
- `reset()` goes back to the original `start`

<details>
<summary>Solution</summary>

```javascript
function makeCounter(start = 0, step = 1) {
  let current = start;

  return {
    increment: () => { current += step; },
    decrement: () => { current -= step; },
    reset:     () => { current = start; }, // start is closed over, never changes
    value:     () => current,
  };
}

const c = makeCounter(10, 5);
c.increment(); // 15
c.increment(); // 20
c.decrement(); // 15
c.reset();     // 10
console.log(c.value()); // 10
```
</details>

---

### Exercise 4 — Implement `once`

Implement `once(fn)` — a function that wraps `fn` so it only executes on the
first call. Subsequent calls return the first result without re-executing.

<details>
<summary>Solution</summary>

```javascript
function once(fn) {
  let called = false;
  let result;
  return function(...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args);
    }
    return result;
  };
}
```
</details>

---

## 8. Revision & Cheat Sheet

### 5-Minute Summary

- A closure is a function + its lexical environment
- The function remembers variables from where it was **written**, not where it's called
- Closures keep outer variables alive even after the outer function returns
- Each call to the outer function creates a **new, independent** closure
- `var` in loops = one shared variable = classic closure trap → fix with `let` or IIFE
- Closures capture variable **references**, not values — changes to the variable are visible
- `this` is NOT captured by closures — use arrow functions to preserve `this`

### Flash Cards

| Q | A |
|---|---|
| What is a closure? | Function + its surrounding lexical environment |
| `var` loop + setTimeout prints what? | Last value of i (all closures share one i) |
| `let` loop + setTimeout prints what? | 0, 1, 2… (each iteration has own i) |
| Do closures capture values or references? | References to variables |
| Does a closure capture `this`? | No — use arrow function for that |
| Two calls to the same factory function — shared state? | No — each call creates a new environment |
| Where is closed-over data stored? | Heap (not stack — it outlives the function call) |

### Cheat Sheet

```
┌──────────────────────────────────────────────────────────┐
│                CLOSURES — CHEAT SHEET                    │
├──────────────────────────────────────────────────────────┤
│ DEFINITION                                               │
│  Function + lexical environment where it was defined     │
│  Outer scope variables stay alive as long as closure     │
│  holds a reference to them                               │
├──────────────────────────────────────────────────────────┤
│ PRACTICAL USES                                           │
│  Private variables     → var inside outer fn             │
│  Factory functions     → each call = new closure         │
│  Memoization           → cache closed over by wrapper    │
│  Module pattern        → IIFE + return public API        │
│  Callbacks/handlers    → capture surrounding context     │
├──────────────────────────────────────────────────────────┤
│ CLASSIC TRAPS                                            │
│  var + loop + async    → all callbacks share one var     │
│                           Fix: use let or IIFE           │
│  this in callback      → not captured by closure         │
│                           Fix: use arrow function        │
│  Stale closure         → captures ref, sees mutations    │
│                           but not rebinding of outer var │
├──────────────────────────────────────────────────────────┤
│ KEY RULES                                                │
│  let in loop           → new binding per iteration ✅    │
│  var in loop           → one shared binding ❌           │
│  Arrow fn              → inherits this from outer scope  │
│  Regular fn            → its own this (window/undefined) │
└──────────────────────────────────────────────────────────┘
```

---

---

# Good to Know

---

## 9. Memory & Garbage Collection

Closures keep their outer scope's variables alive in memory as long as the closure
itself is reachable. This is useful — but can cause memory leaks if misused.

```javascript
function attachHandler() {
  const largeData = new Array(1000000).fill('x');

  document.getElementById('btn').addEventListener('click', () => {
    console.log('clicked'); // closes over largeData even if it never uses it
  });
  // largeData stays in memory until the event listener is removed
}
```

**Fix:** Null out large variables or remove event listeners when the component is destroyed.

In Angular, this is why `ngOnDestroy` + `unsubscribe` / `takeUntilDestroyed` matters —
RxJS subscriptions are closures holding references to component state.

V8 is smart enough to only keep variables that are actually referenced by the closure,
not the entire outer scope. But it can't always detect this statically — explicit cleanup is safer.

---

## 10. Closures in Angular & RxJS

### Every RxJS subscription is a closure

```typescript
ngOnInit() {
  const userId = this.route.snapshot.params['id']; // captured in closure

  this.userService.getUser(userId).subscribe(user => {
    // userId accessed via scope chain — closure
    console.log(`Loaded user ${userId}`, user);
    this.user = user; // also closes over 'this'
  });
}
```

The subscribe callback closes over `userId` and `this`. If the component is
destroyed and the subscription isn't cleaned up, the closure keeps the component
instance in memory — a memory leak.

### `NgZone.runOutsideAngular` and closures

```typescript
this.ngZone.runOutsideAngular(() => {
  // This callback closes over 'this' (the component)
  setInterval(() => {
    this.counter++; // still works — closure holds component reference
  }, 1000);
});
```

The interval callback is a closure over `this`. Even outside Angular's zone,
the closure correctly accesses the component's properties.
