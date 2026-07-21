# JavaScript — Scope Chain, Lexical Scope & Hoisting

> **Phase 1 | Week 1 | Topic 2 of 30**
> **Difficulty:** Foundation
> **Time to complete:** 3 hours
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

**Scope** answers one question: *which variables can this piece of code access?*

**Lexical Scope** means: scope is decided by **where you write the code**, not where you run it.

**Scope Chain** is how JavaScript finds variables — it looks in the current scope first, then works outward through each parent scope until it finds the variable or hits the global scope.

**Hoisting** is what happens before any code runs — JavaScript registers all variable and function declarations in advance, with different behaviour for `var`, `let`/`const`, and function declarations.

> **Analogy for Scope Chain:** Think of Russian nesting dolls. The innermost doll can see everything outside it. The outermost doll can only see itself. Variables work the same way — inner functions see outer variables, but not vice versa.

### Why should a senior frontend engineer care?

Every question about closures, module patterns, variable shadowing, Angular service scope, RxJS operator behaviour, and memory leaks connects back to scope. Understanding scope chain is what separates engineers who debug by guessing from those who debug by reasoning.

---

## 2. Why It Exists

### The problem without scope

Without scope, every variable would be global. Every function could accidentally overwrite every other function's variables. Code would be impossible to maintain at scale.

```javascript
// Without scope — everything global, total chaos
var i = 0;          // loop counter
var name = 'app';   // app name

function loop() {
  for (i = 0; i < 10; i++) { // accidentally overwrites global i
    name = i;                 // accidentally overwrites global name
  }
}
```

Scope gives each function its own **private space** for variables.

### Why LEXICAL scope (not dynamic scope)?

JavaScript chose lexical scope — scope based on where you write code.

The alternative is **dynamic scope** — scope based on where you call code. Some older languages use this. JavaScript does not.

```javascript
const name = 'Global';

function printName() {
  console.log(name); // lexical: sees 'Global' (where it was written)
}

function outer() {
  const name = 'Outer';
  printName();       // dynamic would see 'Outer', lexical sees 'Global'
}

outer(); // prints 'Global' — lexical scope
```

Lexical scope is more predictable. You can reason about what a function can access just by reading the code — without needing to know the call history.

### Engineering Perspective

> **Why was hoisting designed this way?**
> `var` hoisting with `undefined` was designed for forgiveness — Brendan Eich wanted JavaScript to be beginner-friendly.
> `let`/`const` TDZ was designed for correctness — ES6 fixed the silent failure problem.

> **What trade-offs does lexical scope introduce?**
> Closures. A function can hold references to outer variables long after the outer function returns. This is powerful (module pattern, memoization) but can cause memory leaks if not managed carefully.

---

## 3. Fundamentals

### Types of Scope

| Scope Type | Created by | Example |
|---|---|---|
| **Global Scope** | Automatically | Variables outside any function |
| **Function Scope** | Every function call | `function foo() { var x }` |
| **Block Scope** | `{}` blocks (only `let`/`const`) | `if`, `for`, `{}` |
| **Module Scope** | ES modules | Variables in `.mjs` files |

### Key Definitions

**Lexical Scope:** Scope is determined at write time by the physical location of code in the source file. A function's scope is always the scope where it was defined — not where it is called.

**Scope Chain:** The linked list of scopes from current → outer → ... → global. Variable lookup traverses this chain.

**Variable Shadowing:** When an inner scope declares a variable with the same name as an outer scope, the inner one shadows (hides) the outer one within that inner scope.

**Hoisting:** The process during the creation phase of execution context where JavaScript registers all declarations before running any code.

### Hoisting Rules Summary

| Declaration | Hoisted? | Initial Value | Safe before declaration? |
|---|---|---|---|
| `var` | ✅ Yes | `undefined` | ✅ Yes (gets `undefined`) |
| `let` | ✅ Yes | TDZ | ❌ No (ReferenceError) |
| `const` | ✅ Yes | TDZ | ❌ No (ReferenceError) |
| `function` declaration | ✅ Yes | Full body | ✅ Yes |
| `function` expression (`var`) | ✅ Yes | `undefined` | ❌ No (TypeError) |
| Arrow function (`var`) | ✅ Yes | `undefined` | ❌ No (TypeError) |
| `class` | ✅ Yes | TDZ | ❌ No (ReferenceError) |

---

## 4. Internal Working

### How the engine resolves a variable

When the engine encounters a variable name, it does this:

```
1. Look in current Lexical Environment (local scope)
2. Not found? Follow outer reference to parent scope
3. Not found? Follow outer reference again
4. Repeat until Global scope
5. Not found in Global? → ReferenceError
```

This traversal is determined at **compile time** (when V8 parses the code), not at runtime. V8 knows exactly which scope each variable belongs to before running a single line.

### How hoisting works internally

The engine processes code in two passes:

**Pass 1 — Creation Phase:**
```
Scans the entire scope
  → var declarations    : registered as undefined
  → function declarations: registered with full body
  → let/const           : registered in TDZ (uninitialised)
  → class               : registered in TDZ
```

**Pass 2 — Execution Phase:**
```
Runs line by line
  → var/let/const assignments: value assigned when line runs
  → function calls            : new EC created
```

### Block scope vs Function scope internally

`var` is stored in the nearest **function**'s Variable Environment.
`let`/`const` are stored in the nearest **block**'s Lexical Environment.

```javascript
function test() {
  var a = 1;        // stored in test's Variable Environment
  if (true) {
    var b = 2;      // ALSO stored in test's Variable Environment (not block)
    let c = 3;      // stored in if-block's Lexical Environment
  }
  console.log(a); // 1 ✅
  console.log(b); // 2 ✅ — var leaks out of block
  console.log(c); // ReferenceError — let stays in block
}
```

---

## 5. Visual Architecture

### Scope Chain — Nested Dolls

```
┌─────────────────────────────────────────┐
│  GLOBAL SCOPE                           │
│  const a = 'global'                     │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  level1() SCOPE                   │  │
│  │  const b = 'level1'               │  │
│  │                                   │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │  level2() SCOPE             │  │  │
│  │  │  const c = 'level2'         │  │  │
│  │  │                             │  │  │
│  │  │  ┌───────────────────────┐  │  │  │
│  │  │  │  level3() SCOPE       │  │  │  │
│  │  │  │  (no own variables)   │  │  │  │
│  │  │  │                       │  │  │  │
│  │  │  │  console.log(c) ──────┼──┘  │  │
│  │  │  │  console.log(b) ──────┼─────┘  │
│  │  │  │  console.log(a) ──────┼────────┘
│  │  │  └───────────────────────┘  │  │
│  │  └─────────────────────────────┘  │
│  └───────────────────────────────────┘
└─────────────────────────────────────────┘
```

### Hoisting Timeline

```
Your Code:                    What JS sees internally:
─────────────────────         ──────────────────────────────
console.log(a);               var a = undefined;  ← hoisted
console.log(b);               [b in TDZ]          ← hoisted but dead
console.log(fn);              function fn() {}    ← hoisted fully

var a = 1;                    // execution phase: a = 1
let b = 2;                    // execution phase: b = 2 (TDZ ends)
function fn() { return 3; }   // already hoisted, skipped
```

### Lexical Scope — write location determines scope

```
Source code physical location:
─────────────────────────────
GLOBAL
  └── outer()          ← outer written here → sees global
        └── inner()    ← inner written here → sees global + outer

Call location (irrelevant for scope):
─────────────────────────────
main()
  └── calls outer()
        └── inner() called from outer or anywhere else
              → scope is STILL what was determined at write time
```

---

## 6. Syntax & API

### Function Scope vs Block Scope

```javascript
// var — function scoped, leaks out of blocks
function demo() {
  if (true) {
    var x = 10;   // belongs to demo(), not the if block
  }
  console.log(x); // 10 ✅ — var leaked out
}

// let — block scoped, stays inside {}
function demo2() {
  if (true) {
    let y = 20;   // belongs to the if block
  }
  console.log(y); // ReferenceError — let stayed in block
}
```

### Variable Shadowing

```javascript
const name = 'Global';

function outer() {
  const name = 'Outer';  // shadows global name

  function inner() {
    const name = 'Inner'; // shadows outer name
    console.log(name);    // 'Inner'
  }

  inner();
  console.log(name); // 'Outer'
}

outer();
console.log(name); // 'Global'
```

### Hoisting — all 6 cases

```javascript
// 1. var — undefined
console.log(a); // undefined
var a = 5;

// 2. let — TDZ
console.log(b); // ReferenceError
let b = 5;

// 3. const — TDZ
console.log(c); // ReferenceError
const c = 5;

// 4. function declaration — fully hoisted
fn();           // 'hello' ✅
function fn() { console.log('hello'); }

// 5. function expression (var) — var hoisted as undefined
fe();           // TypeError: fe is not a function
var fe = function() { console.log('hello'); };

// 6. arrow function (var) — same as function expression
ae();           // TypeError: ae is not a function
var ae = () => console.log('hello');
```

---

## 7. Practical Examples

### Simple — scope chain lookup

```javascript
const x = 'global';

function outer() {
  const y = 'outer';

  function inner() {
    const z = 'inner';
    console.log(z); // 'inner' — found locally
    console.log(y); // 'outer' — found in outer's scope
    console.log(x); // 'global' — found in global scope
  }

  inner();
}

outer();
```

### Intermediate — shadowing trap

```javascript
var count = 0;

function increment() {
  var count = 0;  // shadows global count
  count++;
  console.log(count); // 1 — local count
}

increment();
increment();
console.log(count); // 0 — global count never changed
```

### Advanced — TDZ shadowing outer scope

```javascript
let x = 'global';

function test() {
  console.log(x); // ReferenceError — local x is in TDZ, shadows global
  let x = 'local';
  console.log(x); // 'local'
}

test();
```

### Real Interview Example — for loop with var

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 3, 3, 3
// Why? var is function-scoped, all 3 callbacks share the SAME i
// By the time callbacks run, loop is done, i = 3

for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2
// Why? let is block-scoped, each iteration gets its OWN i
```

This is one of the most common JavaScript interview questions. The answer is entirely about `var` function scope vs `let` block scope.

---

## 8. Real-world Usage

### In Angular — service scope

```typescript
// Provided in root — one instance, global scope
@Injectable({ providedIn: 'root' })
export class UserService { }

// Provided in component — scoped to that component's subtree
@Injectable()
export class CartService { }

@Component({
  providers: [CartService] // new instance, scoped here
})
```

Angular's DI system mirrors JavaScript scope — services can be scoped globally or locally, just like variables.

### In Angular — for loop + async (the classic bug)

```typescript
// Bug in template — setTimeout inside *ngFor
ngOnInit() {
  for (var i = 0; i < this.items.length; i++) {
    setTimeout(() => {
      console.log(this.items[i]); // undefined — i is already at max
    }, i * 1000);
  }
}

// Fix — let gives each iteration its own i
ngOnInit() {
  for (let i = 0; i < this.items.length; i++) {
    setTimeout(() => {
      console.log(this.items[i]); // ✅ correct item
    }, i * 1000);
  }
}
```

### In RxJS — closure over outer variable

```typescript
ngOnInit() {
  const userId = this.route.snapshot.params['id']; // captured in closure

  this.userService.getUser(userId).subscribe(user => {
    // userId accessed here via scope chain — closure
    console.log(`Loaded user ${userId}:`, user);
  });
}
```

The RxJS callback accesses `userId` via scope chain. This is a closure — and it's why RxJS subscriptions can access component variables without passing them explicitly.

---

## 9. Performance

### Scope chain traversal cost

Variable lookup traverses the scope chain — a linked list. Deeper nesting = more traversal.

```javascript
const config = { timeout: 3000 };

function level1() {
  function level2() {
    function level3() {
      // Bad — config looked up via 3-level chain on every call
      for (let i = 0; i < 1000000; i++) {
        if (i > config.timeout) break;
      }

      // Better — cache in local variable, 1 lookup
      const timeout = config.timeout;
      for (let i = 0; i < 1000000; i++) {
        if (i > timeout) break;
      }
    }
  }
}
```

In practice this only matters in extremely hot loops. V8 optimises most scope lookups. But it's good to know for senior interviews.

### `var` leaking into function scope — memory

```javascript
function process() {
  for (var i = 0; i < 1000; i++) {
    var temp = heavyObject(); // var temp lives until process() returns
  }
  // temp still in memory here even though loop is done
}

function processBetter() {
  for (let i = 0; i < 1000; i++) {
    let temp = heavyObject(); // let temp freed at end of each iteration
  }
  // no temp here
}
```

`let` in blocks allows GC to collect variables sooner than `var`.

---

## 10. Common Mistakes

### Mistake 1 — `var` in for loop with async

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 3, 3, 3
}
// Fix: use let
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 0, 1, 2
}
```

### Mistake 2 — Accidentally creating global variables

```javascript
function oops() {
  x = 10; // no var/let/const — creates global variable silently
}
oops();
console.log(x); // 10 — polluted global scope
// Fix: always use let/const/var. Use strict mode to catch this.
```

### Mistake 3 — Hoisting function expression

```javascript
sayHi(); // TypeError: sayHi is not a function
var sayHi = function() { console.log('hi'); };
// Fix: use function declaration, or call after the expression
```

### Mistake 4 — TDZ shadowing outer variable (covered earlier)

```javascript
let x = 'outer';
function test() {
  console.log(x); // ReferenceError — not 'outer'
  let x = 'inner';
}
```

### Mistake 5 — Assuming block creates function scope for `var`

```javascript
if (true) {
  var secret = 'exposed'; // leaks to function/global scope
}
console.log(secret); // 'exposed' — surprise!
// Fix: use let
```

---

## 11. Best Practices

1. **Always use `const` by default, `let` when reassignment needed, never `var`**
2. **Declare variables at the top of their scope** — makes hoisting explicit
3. **Avoid variable shadowing** — use distinct names to prevent confusion
4. **Use strict mode** — catches accidental global variable creation
5. **Prefer `let` over `var` in loops** — especially with async operations
6. **Keep functions shallow** — deep nesting increases scope chain length and reduces readability
7. **Name variables clearly** — when scopes nest, ambiguous names cause bugs

---

## 12. Debugging

### Chrome DevTools — Scope panel

1. Set a breakpoint inside a nested function
2. When paused, look at the **Scope** panel on the right
3. It shows: **Local → Closure → Script → Global**
4. Each level is one step in the scope chain
5. You can see exactly what each scope contains

### Finding variable shadowing bugs

```javascript
// Add debugger to inspect what value a variable has
function test() {
  const name = 'local';
  debugger; // pause here — check Scope panel
  console.log(name);
}
```

### Checking for accidental globals

```javascript
'use strict'; // catches x = 10 without declaration → ReferenceError
```

Or in DevTools Console:
```javascript
Object.keys(window).filter(k => !['...built-ins'].includes(k))
// Shows all variables you accidentally added to window
```

---

## 13. Advanced Concepts

### Module Scope (ES Modules)

In ES modules, every file has its own module scope. Variables are not global even if declared at the top level.

```javascript
// module-a.js
const secret = 'only in module-a'; // module scope, not global

export const publicValue = 42; // only this is accessible outside
```

### The IIFE Pattern (old module pattern using scope)

Before ES modules, developers used IIFEs to create private scope:

```javascript
const counter = (function() {
  let count = 0; // private — not accessible outside
  return {
    increment: () => ++count,
    get: () => count,
  };
})();

counter.increment();
console.log(counter.get()); // 1
console.log(count);         // ReferenceError — private
```

This is the **Module Pattern** — entirely based on function scope and closures.

### Hoisting in `class`

```javascript
const obj = new MyClass(); // ReferenceError — class is in TDZ
class MyClass {}
```

Classes behave like `let`/`const` — hoisted but in TDZ. Always define before using.

### `var` in `switch` statements

```javascript
switch (x) {
  case 1:
    var result = 'one'; // var — belongs to surrounding function, not case block
    break;
  case 2:
    var result = 'two'; // same var! — redeclaration, no error with var
    break;
}
// Fix: use let, or wrap cases in {}
```

---

## 14. Interview Questions

### Beginner

**Q: What is scope in JavaScript?**
> Scope determines which variables a piece of code can access. JavaScript uses lexical scope — scope is determined by where code is written, not where it is called.

**Q: What is the difference between `var`, `let`, and `const` scope?**
> `var` is function-scoped — it belongs to the nearest enclosing function. `let` and `const` are block-scoped — they belong to the nearest `{}` block. All three are hoisted, but `var` initialises to `undefined` while `let`/`const` stay in TDZ.

**Q: What is hoisting?**
> During the creation phase of execution context, JavaScript registers all declarations before running any code. `var` is initialised to `undefined`, function declarations are stored with full body, `let`/`const` are in TDZ.

---

### Intermediate

**Q: What is the output? Why?**
```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```
> Output: `3, 3, 3`. `var` is function-scoped — all three callbacks share the same `i`. By the time the callbacks run, the loop has completed and `i` is 3. Fix: use `let` — each iteration gets its own block-scoped `i`.

**Q: What is the Temporal Dead Zone?**
> The period between when `let`/`const` is hoisted (start of its scope) and when its declaration line is executed. Accessing a variable in TDZ throws `ReferenceError`. It exists to catch bugs that `var`'s silent `undefined` would hide.

---

### Advanced

**Q: Explain lexical scope vs dynamic scope.**
> Lexical scope: variable access is determined at write time by the physical location of code. JavaScript uses lexical scope — a function always sees the scope where it was defined, regardless of where it's called. Dynamic scope would mean a function sees the scope of its caller. JavaScript does not use dynamic scope (except `this`, which IS dynamic).

**Q: Why does this throw a ReferenceError?**
```javascript
let x = 'global';
function test() {
  console.log(x);
  let x = 'local';
}
test();
```
> The `let x` inside `test()` is hoisted to TDZ for the entire function scope from line 1. It shadows the global `x` immediately — even before its declaration line. So `console.log(x)` finds the local `x` (in TDZ) rather than falling back to global `x`. TDZ access → ReferenceError.

---

### Staff Level

**Q: How does JavaScript's scope model enable the Module Pattern and closures?**
> Lexical scope means inner functions always retain access to their outer function's variables — even after the outer function returns. This is what closures are. The Module Pattern exploits this: wrap everything in a function (creating private scope), return only the public API. The returned functions close over the private variables via lexical scope. ES modules formalise this with module scope, but the underlying mechanism is the same — lexical scoping guarantees the inner functions always see the variables from where they were written.

---

## 15. Hands-on Exercises

### Exercise 1 — Predict the output

```javascript
var x = 1;
function outer() {
  var x = 2;
  function inner() {
    var x = 3;
    console.log(x); // A
  }
  inner();
  console.log(x); // B
}
outer();
console.log(x); // C
```

<details>
<summary>Answer</summary>
A: 3, B: 2, C: 1 — each function has its own `x`, scope chain lookup finds the nearest one.
</details>

---

### Exercise 2 — Fix the for loop bug

```javascript
const fns = [];
for (var i = 0; i < 5; i++) {
  fns.push(function() { return i; });
}
console.log(fns[0]()); // Expected 0, gets 5
console.log(fns[3]()); // Expected 3, gets 5
```

Fix it two ways: using `let`, and using an IIFE.

<details>
<summary>Solution</summary>

```javascript
// Fix 1: let
for (let i = 0; i < 5; i++) {
  fns.push(function() { return i; });
}

// Fix 2: IIFE closure
for (var i = 0; i < 5; i++) {
  fns.push((function(j) {
    return function() { return j; };
  })(i));
}
```
</details>

---

### Exercise 3 — Write from scratch

Create a `makeCounter` function that:
- Has a private `count` variable (not accessible outside)
- Returns `{ increment, decrement, reset, getCount }`
- Uses closure + scope chain (no classes)

<details>
<summary>Solution</summary>

```javascript
function makeCounter(initial = 0) {
  let count = initial; // private — closure variable

  return {
    increment: () => ++count,
    decrement: () => --count,
    reset:     () => { count = initial; },
    getCount:  () => count,
  };
}

const c = makeCounter(10);
c.increment(); // 11
c.increment(); // 12
c.decrement(); // 11
console.log(c.getCount()); // 11
console.log(count);        // ReferenceError — private
```
</details>

---

## 16. Mini Project

### Build: Scope Visualiser

A Node.js script that demonstrates scope chain, hoisting, and TDZ in action:

```javascript
// scope-demo.js

console.log('=== HOISTING ===');
console.log(typeof varDemo);   // undefined — safe
// console.log(typeof letDemo); // ReferenceError — TDZ
console.log(typeof fnDemo);    // function — fully hoisted

var varDemo = 'assigned';
let letDemo = 'let';
function fnDemo() {}

console.log('\n=== SCOPE CHAIN ===');
const global = 'I am global';

function level1() {
  const l1 = 'I am level1';
  function level2() {
    const l2 = 'I am level2';
    function level3() {
      console.log(global); // scope chain: level3 → level2 → level1 → global
      console.log(l1);
      console.log(l2);
    }
    level3();
  }
  level2();
}
level1();

console.log('\n=== VAR vs LET IN LOOP ===');
const varFns = [], letFns = [];

for (var i = 0; i < 3; i++) varFns.push(() => i);
for (let j = 0; j < 3; j++) letFns.push(() => j);

console.log('var:', varFns.map(f => f())); // [3, 3, 3]
console.log('let:', letFns.map(f => f())); // [0, 1, 2]

console.log('\n=== PRIVATE SCOPE (Module Pattern) ===');
const bank = (function() {
  let balance = 1000; // private
  return {
    deposit:  (n) => balance += n,
    withdraw: (n) => balance -= n,
    getBalance: () => balance,
  };
})();

bank.deposit(500);
bank.withdraw(200);
console.log('Balance:', bank.getBalance()); // 1300
// console.log(balance); // ReferenceError — private
```

---

## 17. Revision

### 5-Minute Summary

- **Scope** = what variables can this code access?
- **Lexical scope** = determined by WHERE you write the code (not where you call it)
- **Scope chain** = inner → outer → ... → global → null. Lookup traverses this chain.
- **Variable shadowing** = inner scope variable hides outer one with same name
- **`var`** = function-scoped, hoisted as `undefined`
- **`let`/`const`** = block-scoped, hoisted in TDZ → ReferenceError if accessed early
- **function declaration** = hoisted fully, callable before declaration line
- **function expression/arrow** = variable hoisted as `undefined`, not the function
- **for loop + var + async** = all callbacks share one `i`. Fix: use `let`.

### Mind Map

```
Scope
├── Types
│   ├── Global
│   ├── Function (var)
│   ├── Block (let/const)
│   └── Module (ES modules)
├── Lexical Scope
│   └── Determined at WRITE time, not call time
├── Scope Chain
│   └── Inner → Outer → Global → null
│   └── Variable shadowing
└── Hoisting
    ├── var → undefined
    ├── let/const → TDZ → ReferenceError
    └── function declaration → full body
```

### Top 10 Flash Cards

| Q | A |
|---|---|
| What determines scope in JavaScript? | Where the code is WRITTEN (lexical) |
| What is the scope chain? | Linked list from current scope to global |
| What does `var` hoist to? | `undefined` |
| What does `let` hoist to? | TDZ (ReferenceError if accessed) |
| What is fully hoisted? | Function declarations |
| What is variable shadowing? | Inner scope variable hiding outer same-named variable |
| `var` in a for loop + setTimeout prints what? | Last value of i (all callbacks share one i) |
| `let` in a for loop + setTimeout prints what? | 0, 1, 2... (each iteration has own i) |
| What scope does `var` belong to? | Nearest function (or global) |
| What scope does `let` belong to? | Nearest block `{}` |

---

## 18. Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│        SCOPE CHAIN, LEXICAL SCOPE & HOISTING — CHEAT SHEET       │
├──────────────────────────────────────────────────────────────────┤
│ SCOPE TYPES                                                      │
│   Global    → outside everything                                 │
│   Function  → inside function (var lives here)                   │
│   Block     → inside {} (let/const live here)                    │
│   Module    → ES module file scope                               │
│                                                                  │
│ LEXICAL SCOPE                                                    │
│   Scope = where you WRITE the function                           │
│   Call location is irrelevant                                    │
│                                                                  │
│ SCOPE CHAIN                                                      │
│   current → outer → ... → global → null                         │
│   Not found anywhere → ReferenceError                            │
│                                                                  │
│ HOISTING                                                         │
│   var            → undefined (safe, but surprising)              │
│   let / const    → TDZ (ReferenceError if accessed early)        │
│   function decl  → full body (call before declaration ✅)        │
│   function expr  → undefined (TypeError if called early)         │
│   class          → TDZ (ReferenceError if accessed early)        │
│                                                                  │
│ VAR vs LET KEY DIFFERENCES                                       │
│   var  → function scope, hoists as undefined, no block scope     │
│   let  → block scope, TDZ, can reassign                          │
│   const→ block scope, TDZ, cannot reassign binding               │
│                                                                  │
│ CLASSIC INTERVIEW TRAP                                           │
│   for (var i...) + setTimeout → all print last value of i        │
│   for (let i...) + setTimeout → prints 0, 1, 2... ✅             │
│                                                                  │
│ TDZ SHADOW TRAP                                                  │
│   let x = 'global'                                               │
│   function f() { console.log(x); let x = 'local'; }             │
│   → ReferenceError (local x in TDZ shadows global x)            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 19. References

### Official Documentation
- [MDN — Scope](https://developer.mozilla.org/en-US/docs/Glossary/Scope)
- [MDN — Hoisting](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting)
- [MDN — let — Temporal Dead Zone](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let#temporal_dead_zone_tdz)
- [ECMAScript Spec — Lexical Environments](https://tc39.es/ecma262/#sec-lexical-environments)

### Best Articles
- [You Don't Know JS — Scope & Closures (free online)](https://github.com/getify/You-Dont-Know-JS/tree/2nd-ed/scope-closures)
- [JavaScript Scoping and Hoisting — Ben Cherry](http://www.adequatelygood.com/JavaScript-Scoping-and-Hoisting.html)
- [The Temporal Dead Zone — FreeCodeCamp](https://www.freecodecamp.org/news/what-is-the-temporal-dead-zone/)

### Best Videos
- [JavaScript Scope — Akshay Saini (Namaste JS)](https://www.youtube.com/watch?v=uH-tVP8MUs8)
- [JavaScript Hoisting — Akshay Saini](https://www.youtube.com/watch?v=Fnlnw8uY6jo)
- [var vs let vs const — Fireship](https://www.youtube.com/watch?v=9WIJQDvt4Us)

### Best GitHub Repository
- [You-Dont-Know-JS — Scope & Closures](https://github.com/getify/You-Dont-Know-JS/tree/2nd-ed/scope-closures)

### RFC / Specification
- [TC39 — let and const declarations](https://tc39.es/ecma262/#sec-let-and-const-declarations)

---

> **Next topic:** `03-closures.md`
> Say **"next topic"** when ready.
