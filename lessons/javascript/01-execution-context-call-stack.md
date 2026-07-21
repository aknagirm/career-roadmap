# JavaScript — Execution Context, Call Stack & Variable Environment

> **Phase 1 | Week 1 | Topic 1 of 30**
> **Difficulty:** Foundation
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

Every time JavaScript runs your code, it does not just execute lines top-to-bottom blindly.
Before running a single line, it creates an **Execution Context** — a structured internal
environment that holds everything needed to execute that piece of code.

> **Analogy:** A chef doesn't start cooking the moment they enter the kitchen.
> First they set up their station — ingredients laid out, tools ready, recipe in hand.
> That *setup* is the Execution Context. Only then does the cooking (execution) begin.

There are **three types** of Execution Context:

| Type | When Created | Scope |
|---|---|---|
| **Global Execution Context (GEC)** | Once, automatically when the script loads | Everything outside any function |
| **Function Execution Context (FEC)** | Every time a function is **called** | Inside that function |
| **Eval Execution Context** | Inside `eval()` (avoid in production) | Inside eval string |

The **Call Stack** is JavaScript's mechanism for tracking which execution context is
currently running and what to return to when it finishes. It follows **LIFO** —
Last In, First Out.

The **Variable Environment** is the part of each execution context that stores:
- Variable declarations (`var`, `let`, `const`)
- Function declarations
- The value of `this`
- A reference to the outer environment (scope chain)

### Why should a senior frontend engineer care?

Every JavaScript interview question about **scope, hoisting, closures, `this`, async
behaviour, and memory leaks** roots back to execution contexts. Understanding this deeply
means you stop memorising rules and start *reasoning* about code behaviour.

| Interviewer asks... | They are really testing... |
|---|---|
| "What is the output of this code?" | Can you simulate the call stack mentally? |
| "Why does `this` behave differently here?" | Do you know which execution context is active? |
| "Why is this `undefined` and not a ReferenceError?" | Do you understand how variable environment is built? |
| "Explain hoisting" | Do you understand the creation phase of execution context? |

---

## 2. Why It Exists

### The problem JavaScript had to solve

JavaScript was designed in 1995 for browsers. Brendan Eich built it in 10 days.
The core design decision was: **single-threaded**.

One thread means one thing executing at a time. No parallel execution.

This raised an immediate question: if JavaScript can only do one thing at a time,
**how does it know what "one thing" to do next?** And when a function calls another
function, which calls another — how does it track where to return?

The answer is the **Call Stack + Execution Context** model.

### Why not multithreading?

| Approach | Problem |
|---|---|
| Multithreading | Two threads modifying the DOM simultaneously → race conditions, corruption |
| Single thread + Call Stack | Predictable, safe DOM manipulation — one thing at a time |

Browsers manipulate a shared resource (the DOM). Multithreading would require complex
locking mechanisms. Single-threading with an event loop is simpler and safer for UI work.

### What problem does Execution Context solve?

Without execution context, JavaScript would have no answer to:
- Where does this variable live?
- What does `this` refer to inside this function?
- What variables can this function access from outside itself?
- When this function finishes, where do I return to?

Execution Context is the answer to all of these.

### Engineering Perspective

> **Why was this designed this way?**
> Single-threaded execution was a deliberate trade-off: simplicity and DOM safety over
> parallelism. The execution context model gives the engine a consistent, predictable
> way to manage scope and state.

> **What are the alternatives?**
> Web Workers allow background threads but cannot touch the DOM. SharedArrayBuffer
> enables shared memory between workers but requires careful synchronisation.

> **What trade-offs does it introduce?**
> Long-running synchronous code blocks the entire UI. This is why understanding the
> event loop (next topic) is critical — it's the escape hatch from this limitation.

> **How would a Staff Engineer think about this?**
> They would ask: "Is my long-running code on the main thread? Can I move it to a
> Worker? Can I break it into microtasks using `queueMicrotask()`?" Understanding
> execution context is the foundation of answering those questions.

---

## 3. Fundamentals

### Core Concepts & Terminology

#### Execution Context

An internal data structure created by the JavaScript engine. Contains:

1. **Variable Environment** — stores `var` declarations and function declarations
2. **Lexical Environment** — stores `let`, `const`, and the scope chain reference
3. **`this` binding** — what `this` refers to in this context

#### Call Stack

A stack data structure (LIFO) that tracks execution contexts.

- When a function is called → its execution context is **pushed** onto the stack
- When a function returns → its execution context is **popped** off the stack
- The context at the top of the stack is always the **currently executing** code

#### Variable Environment

The record inside an execution context that holds identifier-to-value mappings.

```
Variable Environment = {
  a: undefined,        // var declarations hoisted as undefined
  greet: fn,           // function declarations hoisted fully
  this: window/global
}
```

#### Lexical Environment

Similar to Variable Environment but for `let` and `const`. Also holds a reference
to the **outer lexical environment** — this reference chain is the **scope chain**.

#### Hoisting

During the **Creation Phase** of an execution context, before any code runs:
- `var` declarations are stored with value `undefined`
- `function` declarations are stored with their full function body
- `let` and `const` are stored but NOT initialised (Temporal Dead Zone)

#### Temporal Dead Zone (TDZ)

The period between when `let`/`const` is hoisted (creation phase) and when its
declaration line is actually executed. Accessing it in this window throws a
`ReferenceError`.

---

## 4. Internal Working

### How the JavaScript Engine processes your code

The engine processes each execution context in **two phases**:

---

### Phase 1: Creation Phase (before any code runs)

The engine scans the code and sets up the execution context:

1. Creates the **Variable Environment** and **Lexical Environment**
2. Determines the value of **`this`**
3. Sets up the **scope chain** (outer environment reference)
4. **Hoists** all declarations:
   - `var` → stored as `undefined`
   - `function` declarations → stored with full body
   - `let` / `const` → stored but uninitialized (TDZ)

### Phase 2: Execution Phase

The engine executes code line by line:
- Assigns actual values to variables
- Executes function calls (creating new execution contexts for each)
- Evaluates expressions

---

### V8 Engine Internals

V8 is Google's JavaScript engine (used in Chrome and Node.js).

```
Source Code
    ↓
Parser → AST (Abstract Syntax Tree)
    ↓
Ignition (Interpreter) → Bytecode
    ↓
TurboFan (JIT Compiler) → Optimised Machine Code
```

| Component | Role |
|---|---|
| **Parser** | Reads source code, checks syntax, builds AST |
| **Ignition** | Interprets AST into bytecode — fast startup |
| **TurboFan** | JIT-compiles hot (frequently run) functions to native machine code |
| **Orinoco (GC)** | Garbage collector — frees memory from unused execution contexts |

### Memory: Heap vs Stack

| Memory | Stores | Behaviour |
|---|---|---|
| **Call Stack** | Execution contexts, primitive values | Fixed size, fast, auto-managed |
| **Heap** | Objects, arrays, functions, closures | Dynamic size, managed by GC |

When a function is called:
- Its execution context (with local variables) goes on the **Call Stack**
- Any objects it creates go on the **Heap**
- When the function returns, the execution context is popped off the stack
- If nothing else holds a reference to those heap objects, GC can collect them

### Garbage Collection

V8 uses **generational garbage collection**:
- **Young Generation (Scavenger):** Short-lived objects. Collected frequently and fast.
- **Old Generation (Mark-Sweep/Compact):** Long-lived objects. Collected less frequently.

Closures can prevent GC if they hold references to variables longer than needed —
a common source of memory leaks.

---

## 5. Visual Architecture

### The Full Picture

```
┌─────────────────────────────────────────────────┐
│                  JAVASCRIPT ENGINE (V8)          │
│                                                  │
│  ┌──────────────────┐   ┌─────────────────────┐  │
│  │    CALL STACK     │   │        HEAP         │  │
│  │                  │   │                     │  │
│  │ ┌──────────────┐ │   │  { obj: ... }       │  │
│  │ │  Function B  │ │   │  [ array ]          │  │
│  │ │  EC          │ │   │  function() {}      │  │
│  │ ├──────────────┤ │   │  closures           │  │
│  │ │  Function A  │ │   │                     │  │
│  │ │  EC          │ │   └─────────────────────┘  │
│  │ ├──────────────┤ │                             │
│  │ │   Global     │ │                             │
│  │ │   EC         │ │                             │
│  │ └──────────────┘ │                             │
│  └──────────────────┘                             │
└─────────────────────────────────────────────────┘
```

### Execution Context Structure

```
Execution Context
┌─────────────────────────────────────┐
│  Variable Environment               │
│    var x = undefined → 10           │
│    function foo = [fn body]         │
│                                     │
│  Lexical Environment                │
│    let y = <TDZ> → 20               │
│    const z = <TDZ> → 30             │
│    outer: → [parent EC reference]   │
│                                     │
│  this binding                       │
│    (global) → window / global       │
│    (method) → calling object        │
│    (arrow)  → inherited from parent │
└─────────────────────────────────────┘
```

### Call Stack: Step by Step

```javascript
function multiply(a, b) { return a * b; }
function square(n)      { return multiply(n, n); }
function printSquare(n) { console.log(square(n)); }

printSquare(5);
```

```
Step 1:  [Global EC]                   ← script loads
Step 2:  [printSquare EC] [Global EC]  ← printSquare(5) called
Step 3:  [square EC] [printSquare EC] [Global EC]  ← square(5) called
Step 4:  [multiply EC] [square EC] [printSquare EC] [Global EC]  ← multiply(5,5)
Step 5:  multiply returns 25 → popped
         [square EC] [printSquare EC] [Global EC]
Step 6:  square returns 25 → popped
         [printSquare EC] [Global EC]
Step 7:  console.log runs, printSquare returns → popped
         [Global EC]
```

---

## 6. Syntax & API

### Observing Execution Context in Code

You cannot directly inspect the execution context as an object, but you can observe
its effects through these built-in mechanisms:

#### `this` — reflects the current execution context's this-binding

```javascript
// Global context
console.log(this);                    // window (browser) / {} (Node module)

// Function context (non-strict)
function show() { console.log(this); }
show();                               // window (browser)

// Function context (strict mode)
'use strict';
function show() { console.log(this); }
show();                               // undefined ← important difference

// Method context
const obj = {
  name: 'Mriganka',
  greet() { console.log(this.name); } // 'Mriganka' — this = obj
};
obj.greet();

// Arrow function — no own execution context, inherits parent's this
const obj2 = {
  name: 'Mriganka',
  greet: () => { console.log(this.name); } // undefined — this = window/global
};
obj2.greet();
```

#### `var` vs `let` / `const` — hoisting difference

```javascript
// var — hoisted as undefined (Variable Environment)
console.log(a);   // undefined (NOT ReferenceError)
var a = 10;
console.log(a);   // 10

// let — in TDZ until declaration line (Lexical Environment)
console.log(b);   // ReferenceError: Cannot access 'b' before initialization
let b = 20;

// const — same TDZ behaviour as let
console.log(c);   // ReferenceError
const c = 30;
```

#### Function declarations vs expressions — hoisting difference

```javascript
// Function declaration — fully hoisted
greet();          // "Hello" — works before the declaration line
function greet() { console.log("Hello"); }

// Function expression — variable hoisted as undefined, not the function
sayHi();          // TypeError: sayHi is not a function
var sayHi = function() { console.log("Hi"); };

// Arrow function expression — same as function expression
sayBye();         // TypeError
var sayBye = () => console.log("Bye");
```

#### `arguments` object — exists in FEC (not in arrow functions)

```javascript
function sum() {
  console.log(arguments);  // Arguments [1, 2, 3]
  return [...arguments].reduce((a, b) => a + b, 0);
}
sum(1, 2, 3); // 6

const sumArrow = () => {
  console.log(arguments);  // ReferenceError — arrow has no arguments object
};
```

---

## 7. Practical Examples

### Simple — Predict the output

```javascript
var x = 1;

function outer() {
  var x = 2;
  function inner() {
    var x = 3;
    console.log(x); // ?
  }
  inner();
  console.log(x);   // ?
}

outer();
console.log(x);     // ?
```

**Answer:** `3`, `2`, `1`
Each function call creates a new execution context with its own variable environment.
`x` in each context is independent.

---

### Intermediate — Hoisting trap

```javascript
console.log(typeof foo); // ?
console.log(typeof bar); // ?

var foo = 'hello';
function bar() { return 'world'; }
```

**Answer:** `"undefined"`, `"function"`

During creation phase:
- `foo` → hoisted as `undefined` (var)
- `bar` → hoisted as full function body (function declaration)

---

### Advanced — TDZ in a block

```javascript
let x = 'global';

function test() {
  console.log(x); // ?
  let x = 'local';
  console.log(x); // ?
}

test();
```

**Answer:** `ReferenceError` on the first `console.log(x)`.

Even though `x` exists in the global scope, the `let x` inside `test()` is hoisted
to the top of `test()`'s lexical environment, putting it in TDZ. JavaScript does NOT
fall back to the outer `x` — the local declaration shadows it immediately.

This is a very common senior interview trap.

---

### Real Interview Example — Stack overflow

```javascript
function recurse() {
  return recurse();
}

recurse(); // What happens?
```

**Answer:** `RangeError: Maximum call stack size exceeded`

Each call to `recurse()` pushes a new execution context onto the call stack with no
base case to pop them off. The stack has a finite size (~10,000–15,000 frames in V8).
When exceeded, V8 throws a stack overflow error.

---

## 8. Real-world Usage

### In Angular

Angular's Zone.js patches async browser APIs to detect when async operations complete
and trigger change detection. It relies entirely on execution context tracking.

```
User clicks button
    ↓
Event handler pushed onto Call Stack (new FEC)
    ↓
HTTP call made → goes to Web APIs (outside stack)
    ↓
Handler pops off stack
    ↓
HTTP response comes back → callback pushed to Task Queue
    ↓
Event Loop picks it up → pushed onto Call Stack
    ↓
Zone.js intercepts → notifies Angular → Change Detection runs
```

Without understanding execution contexts and the call stack, Zone.js's behaviour
appears magical. With it, it's logical.

### In RxJS

Every Observable subscription runs its operators inside function execution contexts.
The `this` binding issue is a common bug in RxJS pipelines:

```javascript
class MyComponent {
  value = 42;

  ngOnInit() {
    // Bug: loses `this` inside subscribe
    this.service.getData().subscribe(function(data) {
      console.log(this.value); // undefined — `this` is not the component
    });

    // Fix: arrow function inherits parent execution context's `this`
    this.service.getData().subscribe((data) => {
      console.log(this.value); // 42 — correct
    });
  }
}
```

### In async/await

`async` functions create their own execution context. When `await` is encountered,
the function is suspended — its execution context is saved — and control returns
to the caller. When the awaited promise resolves, the execution context is restored.

```javascript
async function fetchUser() {
  console.log('1 - before await');      // runs immediately in FEC
  const user = await getUser();          // FEC suspended, saved to microtask queue
  console.log('2 - after await', user); // FEC restored when promise resolves
}

console.log('A');
fetchUser();
console.log('B');

// Output: A → 1 - before await → B → 2 - after await [user]
```

---

## 9. Performance

### Call Stack Size

V8's call stack limit is approximately **10,000–15,000** frames depending on the
platform and available memory. Deep recursion hits this limit.

**Solution:** Convert deep recursion to iteration, or use trampolining.

```javascript
// Stack overflow risk for large n
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1); // each call = new EC on stack
}

// Iterative — O(1) stack space
function factorialSafe(n) {
  let result = 1;
  while (n > 1) result *= n--;
  return result;
}
```

### Variable Lookup Cost

Variable lookup traverses the scope chain (linked list of lexical environments).
Deeply nested functions with many outer scope references are slower to look up.

```javascript
// Slower — x is 5 scopes away, looked up on every iteration
function deep() {
  const x = 100;
  function level1() { function level2() { function level3() {
    for (let i = 0; i < 1000000; i++) { x; } // scope chain traversal
  }}}
}

// Faster — cache outer variable locally
function deepFast() {
  const x = 100;
  function level1() { function level2() { function level3() {
    const localX = x; // one lookup, then local access
    for (let i = 0; i < 1000000; i++) { localX; }
  }}}
}
```

### Memory: Execution Context Lifecycle

An execution context is eligible for GC when:
1. The function has returned (popped off stack)
2. No closure holds a reference to its variables

Closures that survive beyond the function's return keep the variable environment
alive in heap memory — a common memory leak pattern.

---

## 10. Common Mistakes

### Mistake 1 — Assuming `var` throws when accessed before declaration

```javascript
console.log(x); // Most people expect ReferenceError
var x = 5;      // Actual output: undefined
```
`var` is hoisted and initialised to `undefined` during creation phase.
`let`/`const` would throw. This distinction is a very common interview question.

---

### Mistake 2 — Assuming function expressions are hoisted like declarations

```javascript
greet(); // TypeError: greet is not a function
var greet = function() { return 'hello'; };
```
The variable `greet` is hoisted as `undefined`. The function body is NOT hoisted.
Calling `undefined()` throws a TypeError.

---

### Mistake 3 — `this` inside a callback losing context

```javascript
const user = {
  name: 'Mriganka',
  greetLater() {
    setTimeout(function() {
      console.log(this.name); // undefined — `this` is window/undefined in strict
    }, 1000);
  }
};
user.greetLater();
```
The callback creates its own execution context. `this` is not `user`.
Fix: arrow function, `.bind(this)`, or save `const self = this`.

---

### Mistake 4 — TDZ shadowing outer scope variable

```javascript
let x = 'outer';
function test() {
  console.log(x); // ReferenceError — NOT 'outer'
  let x = 'inner';
}
test();
```
The inner `let x` is hoisted to TDZ for the entire function scope, shadowing
the outer `x` before the declaration line is reached.

---

### Mistake 5 — Infinite recursion without noticing

```javascript
function processItems(items) {
  if (items.length === 0) return;
  processItems(items); // Bug: should be items.slice(1)
}
```
Call stack overflow. Always verify recursive base cases reduce the input.

---

## 11. Best Practices

1. **Prefer `const` > `let` > `var`**
   - `const` and `let` have block scope and TDZ protection
   - `var` leaks to function scope, hoists as `undefined` — a source of bugs

2. **Declare variables at the top of their scope**
   - Matches what the engine does in creation phase
   - Makes hoisting explicit and intentional

3. **Use arrow functions for callbacks to preserve `this`**
   - Arrow functions do not create their own execution context's `this` binding
   - Eliminates the most common `this` bug in Angular/RxJS code

4. **Avoid `var` in modern code**
   - No block scope, confusing hoisting, function-scoped leakage

5. **Use strict mode (`'use strict'`)**
   - `this` in function context becomes `undefined` instead of `window`
   - Prevents accidental globals
   - Catches silent errors

6. **Convert deep recursion to iteration or trampolining**
   - Protects against stack overflow in production

7. **Cache frequently accessed outer scope variables**
   - Reduces scope chain traversal in hot paths

---

## 12. Debugging

### Chrome DevTools — Call Stack Panel

1. Open DevTools → **Sources** tab
2. Set a breakpoint on any line
3. When paused, the **Call Stack** panel on the right shows the exact execution
   context stack at that moment
4. Click any frame to inspect its local variables (Variable Environment)
5. The **Scope** panel shows: Local → Closure → Script → Global

### Inspecting `this` at runtime

```javascript
function debug() {
  console.log('this =', this);
  debugger; // Pauses here — inspect `this` in DevTools console
}
```

### Detecting stack overflow

```javascript
// Node.js
try {
  function inf() { inf(); }
  inf();
} catch (e) {
  console.log(e instanceof RangeError); // true
  console.log(e.message); // "Maximum call stack size exceeded"
}
```

### Memory — checking for retained execution contexts (closures)

1. DevTools → **Memory** tab
2. Take a **Heap Snapshot**
3. Search for function names — if a function's scope is still in memory
   after it should have been freed, you have a closure leak
4. Use **Allocation Timeline** to see what's being retained over time

### Node.js Inspector

```bash
node --inspect-brk your-script.js
# Open chrome://inspect in Chrome
# Full DevTools for Node — same call stack, scope panels
```

---

## 13. Advanced Concepts

### ECMAScript Specification Terms

The spec uses precise terminology:

- **Environment Record** — the actual data structure (replaces the informal "Variable Environment")
  - **Declarative Environment Record** — `let`, `const`, `function`, `class`
  - **Object Environment Record** — `var` and `with` statements (uses an object)
  - **Global Environment Record** — combination of both at global scope
- **Outer Environment Reference** — the pointer that forms the scope chain

### Scope Chain Mechanics

Each Lexical Environment has a reference to its outer environment, forming a chain:

```
inner() EC
  → outer() EC
    → Global EC
      → null
```

Variable lookup traverses this chain. If not found anywhere → `ReferenceError`.

### Temporal Dead Zone — precise timing

```javascript
{
  // TDZ for `x` starts HERE (when the block's lexical environment is created)
  typeof x; // ReferenceError (even typeof doesn't save you from TDZ)
  let x = 5;
  // TDZ for `x` ends HERE
}
```

Note: `typeof undeclaredVar` returns `"undefined"` safely, but
`typeof tdzVar` throws `ReferenceError` if `tdzVar` is in TDZ.

### eval() and its own Execution Context

```javascript
eval('var x = 10'); // Creates its own execution context
// Pollutes the surrounding scope in non-strict mode
// Avoid in production — security risk, performance killer
```

### `with` statement (deprecated — never use)

Creates an object environment record, making property lookups ambiguous and
un-optimisable by V8. Forbidden in strict mode.

### Browser vs Node.js differences

| | Browser | Node.js |
|---|---|---|
| Global `this` | `window` | `{}` (module) or `global` (REPL) |
| Global scope pollution | `var x` → `window.x` | `var x` → module-scoped only |
| `arguments` object | Available in FEC | Available in FEC |

---

## 14. Interview Questions

### Beginner

**Q: What is an Execution Context?**
> An internal environment the JavaScript engine creates before executing code.
> It contains the Variable Environment (variables & functions), Lexical Environment
> (scope chain), and the `this` binding. Every function call creates a new one.

**Q: What is the Call Stack?**
> A LIFO data structure that tracks execution contexts. When a function is called,
> its context is pushed. When it returns, it's popped. The context at the top is
> always the currently executing code.

**Q: What is hoisting?**
> During the creation phase of execution context, `var` declarations are stored as
> `undefined` and function declarations are stored with their full body — before
> any code runs. `let`/`const` are hoisted too but stay in TDZ until their line.

---

### Intermediate

**Q: What is the difference between `var`, `let`, and `const` in terms of hoisting?**
> All three are hoisted. `var` is initialised to `undefined`. `let` and `const` are
> in the Temporal Dead Zone — accessing them before their declaration line throws
> a `ReferenceError`. Also, `var` is function-scoped; `let`/`const` are block-scoped.

**Q: Why does `this` inside a `setTimeout` callback refer to `window`?**
> Because the callback creates a new Function Execution Context. In non-strict mode,
> `this` in a standalone function defaults to the global object. The calling object
> context is not preserved. Arrow functions fix this by inheriting the parent context's
> `this` binding.

**Q: What is the Temporal Dead Zone?**
> The period from the start of a block scope until the line where `let`/`const` is
> declared. During TDZ, the variable exists in the lexical environment but is
> uninitialised. Accessing it throws `ReferenceError`.

---

### Advanced

**Q: Explain the two phases of execution context creation.**
> **Creation phase:** Engine scans the code, creates Variable and Lexical environments,
> determines `this`, sets up scope chain, hoists all declarations.
> **Execution phase:** Code runs line by line — variables are assigned actual values,
> functions are called (creating new execution contexts on the stack).

**Q: How does V8 optimise execution contexts?**
> V8's Ignition interpreter first converts code to bytecode. TurboFan JIT-compiles
> frequently executed ("hot") functions to optimised machine code. V8 uses hidden
> classes to optimise property lookups in object-based execution contexts. Functions
> that change shape (add/remove properties) deoptimise — V8 falls back to interpreted
> mode.

**Q: What causes a stack overflow and how do you prevent it?**
> Unbounded recursion — each call pushes a new execution context with no base case
> to pop them. Prevention: use iteration, ensure recursive base cases are correct,
> or use trampolining (returning a function instead of calling recursively, then
> running it in a loop) for functional-style recursion.

---

### Staff Level

**Q: How would you debug a memory leak caused by execution contexts not being garbage collected?**
> A closure keeping a reference to a large outer variable environment prevents GC.
> I'd use Chrome DevTools Memory tab → Heap Snapshot → look for detached closures
> holding references. Then I'd examine the Allocation Timeline to see what persists.
> Fix: null out large variables when done, avoid closures capturing unnecessary scope,
> use WeakMap/WeakRef for cache-like patterns.

**Q: How would you architect a system where long-running synchronous code doesn't block the UI?**
> The call stack being blocked is the root problem — a long-running EC monopolises
> the thread. Solutions: (1) Web Workers — move heavy computation off main thread.
> (2) `queueMicrotask()` / `scheduler.yield()` — break work into microtasks, yielding
> to the event loop between chunks. (3) `requestAnimationFrame` chunks for
> animation-sensitive work. (4) WASM for CPU-heavy computation. Understanding that
> each of these works by either moving work off the call stack or interleaving it
> with the event loop is the key insight.

---

## 15. Hands-on Exercises

### Exercise 1 — Predict the output (do NOT run first)

```javascript
var a = 1;
function outer() {
  console.log(a); // A: ?
  var a = 2;
  function inner() {
    console.log(a); // B: ?
  }
  inner();
  console.log(a); // C: ?
}
outer();
console.log(a); // D: ?
```

<details>
<summary>Answer</summary>

```
A: undefined  — var a inside outer() is hoisted to undefined (creation phase)
B: 2          — inner() looks up scope chain, finds a=2 in outer()'s environment
C: 2          — outer()'s a was assigned 2
D: 1          — global a is 1, never modified
```
</details>

---

### Exercise 2 — Identify and explain the bug

```javascript
const counter = {
  count: 0,
  increment: function() {
    setTimeout(function() {
      this.count++;
      console.log(this.count);
    }, 100);
  }
};
counter.increment(); // What logs? Why? How to fix?
```

<details>
<summary>Answer & Fix</summary>

Logs `NaN`. `this` inside the `setTimeout` callback is `window` (or `undefined` in strict).
`window.count` is `undefined`. `undefined++` is `NaN`.

Fix with arrow function:
```javascript
increment: function() {
  setTimeout(() => {
    this.count++; // arrow inherits `this` from increment's FEC = counter
    console.log(this.count); // 1
  }, 100);
}
```
</details>

---

### Exercise 3 — Write from scratch

Implement a simple call stack visualiser that logs when functions are entered and exited:

```javascript
// Wrap any function so it logs [PUSH] and [POP] with indentation
// showing nesting depth

function wrapWithStack(fn, name) {
  // YOUR IMPLEMENTATION
}

const multiply = wrapWithStack((a, b) => a * b, 'multiply');
const square   = wrapWithStack((n) => multiply(n, n), 'square');
const print    = wrapWithStack((n) => console.log('Result:', square(n)), 'print');

print(5);
// Expected output:
// [PUSH] print
//   [PUSH] square
//     [PUSH] multiply
//     [POP]  multiply → 25
//   [POP]  square → 25
// Result: 25
// [POP]  print → undefined
```

<details>
<summary>Solution</summary>

```javascript
let depth = 0;

function wrapWithStack(fn, name) {
  return function(...args) {
    const indent = '  '.repeat(depth);
    console.log(`${indent}[PUSH] ${name}`);
    depth++;
    const result = fn(...args);
    depth--;
    console.log(`${'  '.repeat(depth)}[POP]  ${name} → ${result}`);
    return result;
  };
}
```
</details>

---

## 16. Mini Project

### Build: Execution Context Visualiser

Create a small Node.js script that demonstrates execution context lifecycle:

**Goal:** A script that shows creation phase → execution phase → GC eligibility
for a set of function calls.

```javascript
// executionContextDemo.js

// ── 1. Demonstrate hoisting ──────────────────────────────────
console.log('\n=== HOISTING DEMO ===');
console.log('var before declaration:', typeof varDemo);   // undefined
console.log('fn before declaration:', typeof fnDemo);     // function
// console.log('let before declaration:', letDemo);       // would throw TDZ

var varDemo = 'assigned';
function fnDemo() { return 'hoisted'; }
let letDemo = 'let value';

// ── 2. Demonstrate call stack depth ─────────────────────────
console.log('\n=== CALL STACK DEPTH ===');

function measureStackDepth(depth = 0) {
  try {
    return measureStackDepth(depth + 1);
  } catch (e) {
    return depth;
  }
}
console.log('Max call stack depth:', measureStackDepth());

// ── 3. Demonstrate this binding in each context ──────────────
console.log('\n=== THIS BINDING ===');

function regularFn() {
  console.log('Regular fn this === global:', this === globalThis);
}

const obj = {
  method() {
    console.log('Method this === obj:', this === obj);
    const arrow = () => {
      console.log('Arrow inherits this === obj:', this === obj);
    };
    arrow();
  }
};

regularFn();   // true (non-strict)
obj.method();  // true, true

// ── 4. Demonstrate closure preventing GC ────────────────────
console.log('\n=== CLOSURE SCOPE RETENTION ===');

function createCounter() {
  let count = 0; // This variable environment stays alive via closure
  return {
    increment: () => ++count,
    getCount:  () => count,
  };
}

const counter = createCounter();
counter.increment();
counter.increment();
console.log('Closure-retained count:', counter.getCount()); // 2
```

**Run it:**
```bash
node executionContextDemo.js
```

**Extend it:**
- Add a memory usage reporter using `process.memoryUsage()`
- Compare memory before and after creating 10,000 closures
- Add a trampoline implementation for deep recursion

---

## 17. Revision

### 5-Minute Summary

- **Execution Context** = environment created before any code runs. Contains Variable Environment, Lexical Environment, `this` binding.
- **Two types:** Global EC (once) and Function EC (every call).
- **Two phases:** Creation (hoist, set up scope, bind `this`) → Execution (run code line by line).
- **Call Stack** = LIFO tracker of active execution contexts. Push on call, pop on return.
- **Hoisting:** `var` → `undefined`. Function declarations → full body. `let`/`const` → TDZ.
- **TDZ** = accessing `let`/`const` before declaration line → `ReferenceError`.
- **`this`** = determined at call time (not definition time) except in arrow functions (which inherit parent's `this`).
- **Stack overflow** = unbounded recursion exhausting the call stack size (~10K–15K frames in V8).
- **Closure memory leak** = a returned inner function holding a reference to outer EC's large variables, preventing GC.

### Mind Map

```
Execution Context
├── Types
│   ├── Global EC (once)
│   └── Function EC (every call)
├── Phases
│   ├── Creation → hoist, scope, this
│   └── Execution → run line by line
├── Contains
│   ├── Variable Environment (var, fn declarations)
│   ├── Lexical Environment (let, const, scope chain)
│   └── this binding
├── Call Stack (LIFO)
│   ├── Push on function call
│   ├── Pop on return
│   └── Overflow = RangeError
└── Hoisting
    ├── var → undefined
    ├── function → full body
    └── let/const → TDZ → ReferenceError
```

### Top 10 Flash Card Questions

| Q | A |
|---|---|
| What is created before any JS code runs? | Global Execution Context |
| What are the two phases of EC creation? | Creation Phase, Execution Phase |
| What is hoisted as `undefined`? | `var` declarations |
| What is in TDZ? | `let` and `const` before their declaration line |
| What does the Call Stack use (LIFO/FIFO)? | LIFO |
| What error does stack overflow throw? | `RangeError: Maximum call stack size exceeded` |
| Does an arrow function have its own `this`? | No — inherits from parent EC |
| Does an arrow function have `arguments`? | No |
| Where do objects live (Stack or Heap)? | Heap |
| What prevents GC of a variable environment? | A closure holding a reference to it |

---

## 18. Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│         EXECUTION CONTEXT & CALL STACK — CHEAT SHEET            │
├──────────────────────────────────────────────────────────────────┤
│ EXECUTION CONTEXT                                                │
│   Global EC    → created once, window/global as `this`           │
│   Function EC  → created on every function CALL                  │
│   Contents     → Variable Env + Lexical Env + this binding       │
│                                                                  │
│ CREATION PHASE (before code runs)                                │
│   var      → hoisted as undefined                                │
│   function → hoisted with full body                              │
│   let/const→ hoisted into TDZ (ReferenceError if accessed)       │
│   this     → bound based on call type                            │
│                                                                  │
│ THIS BINDING RULES                                               │
│   Default    → window (non-strict) / undefined (strict)          │
│   Implicit   → object before the dot: obj.method()              │
│   Explicit   → call/apply/bind                                   │
│   new        → newly created object                              │
│   Arrow      → inherited from surrounding lexical context        │
│                                                                  │
│ CALL STACK                                                       │
│   LIFO — last in, first out                                      │
│   Push  → function called                                        │
│   Pop   → function returns                                       │
│   Limit → ~10K–15K frames (V8) → RangeError on overflow         │
│                                                                  │
│ SCOPE CHAIN                                                      │
│   Each EC has outer reference → forms a chain to Global EC       │
│   Variable lookup: current → outer → ... → Global → ReferenceError│
│                                                                  │
│ COMMON INTERVIEW TRAPS                                           │
│   var before declaration → undefined (NOT ReferenceError)        │
│   let before declaration → ReferenceError (TDZ)                  │
│   this in setTimeout callback → window (not the object)          │
│   TDZ shadows outer scope variable                               │
│   typeof TDZ variable → ReferenceError (not "undefined")         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 19. References

### Official Documentation
- [ECMAScript Specification — Execution Contexts](https://tc39.es/ecma262/#sec-execution-contexts)
- [ECMAScript Specification — Environment Records](https://tc39.es/ecma262/#sec-environment-records)
- [MDN — Hoisting](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting)
- [MDN — `this`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)

### Best Articles
- [JavaScript Visualized: the JavaScript Engine — Lydia Hallie](https://dev.to/lydiahallie/javascript-visualized-the-javascript-engine-4cdf)
- [JavaScript Execution Context — How JS Works Behind the Scenes — FreeCodeCamp](https://www.freecodecamp.org/news/execution-context-how-javascript-works-behind-the-scenes/)
- [The Ultimate Guide to Hoisting, Scopes, and Closures in JavaScript — Tyler McGinnis](https://ui.dev/ultimate-guide-to-execution-contexts-hoisting-scopes-and-closures-in-javascript)

### Best Videos
- [JavaScript Execution Context — Full Tutorial — Akshay Saini (Namaste JS)](https://www.youtube.com/watch?v=iLWTnMzWtj4)
- [What the heck is the event loop anyway? — Philip Roberts (JSConf)](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
- [JavaScript Under the Hood — Franziska Hinkelmann (Google)](https://www.youtube.com/watch?v=Y3Mu24ogCwE)

### Best GitHub Repositories
- [javascript-algorithms — trekhleb](https://github.com/trekhleb/javascript-algorithms)
- [You-Dont-Know-JS (Scope & Closures) — getify](https://github.com/getify/You-Dont-Know-JS/tree/2nd-ed/scope-closures)

### RFC / Specification
- [TC39 ECMA-262 — Section 9: Executable Code and Execution Contexts](https://tc39.es/ecma262/#sec-executable-code-and-execution-contexts)

---

> **Next topic:** `02-scope-chain-lexical-scope-hoisting.md`
> Say **"next topic"** when ready.
