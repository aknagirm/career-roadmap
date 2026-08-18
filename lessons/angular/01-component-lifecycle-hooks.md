# Angular — Component Lifecycle Hooks Deep Dive

> **Phase 3 | Angular Week 1 | Topic 1**
> **Difficulty:** Intermediate
> **Time to complete:** 2–3 hours
> **Status:** ⬜ Not Started

---

## Table of Contents

**Core**
1. [Introduction](#1-introduction)
2. [All 8 Hooks — Quick Reference](#2-all-8-hooks--quick-reference)
3. [Each Hook in Depth](#3-each-hook-in-depth)
4. [Execution Order](#4-execution-order)
5. [Common Patterns](#5-common-patterns)
6. [Common Mistakes](#6-common-mistakes)
7. [Interview Questions](#7-interview-questions)
8. [Hands-on Exercises](#8-hands-on-exercises)
9. [Revision & Cheat Sheet](#9-revision--cheat-sheet)

**Good to Know**
10. [Lifecycle in Parent-Child Components](#10-lifecycle-in-parent-child-components)
11. [Lifecycle with OnPush Change Detection](#11-lifecycle-with-onpush-change-detection)
12. [Lifecycle Hooks in Angular 17+ (Signals era)](#12-lifecycle-hooks-in-angular-17-signals-era)

---

## 1. Introduction

### What are lifecycle hooks?

An Angular component goes through a defined sequence of phases from creation to
destruction. Angular provides **lifecycle hooks** — interfaces with callback methods
that let you tap into these phases and run your own code.

```
Component created
    ↓ constructor()          ← not a hook, but runs first
    ↓ ngOnChanges()          ← input values set/changed
    ↓ ngOnInit()             ← component fully initialised
    ↓ ngDoCheck()            ← every change detection run
    ↓ ngAfterContentInit()   ← projected content (ng-content) initialised
    ↓ ngAfterContentChecked() ← projected content checked
    ↓ ngAfterViewInit()      ← component's own view + child views initialised
    ↓ ngAfterViewChecked()   ← component's own view + child views checked
    ↓ ngOnDestroy()          ← component about to be destroyed
Component destroyed
```

### Why should a senior engineer care?

Almost every Angular interview includes lifecycle hook questions. Beyond interviews,
the hooks are where you:

- Start and clean up subscriptions
- Access the DOM safely (after view initialisation)
- React to input changes
- Avoid memory leaks (ngOnDestroy)
- Optimise expensive operations (understanding when hooks fire)

---

## 2. All 8 Hooks — Quick Reference

| Hook | Interface | When it fires | How often |
|---|---|---|---|
| `ngOnChanges` | `OnChanges` | Before `ngOnInit`, whenever an `@Input` changes | Multiple times |
| `ngOnInit` | `OnInit` | Once, after first `ngOnChanges` | Once |
| `ngDoCheck` | `DoCheck` | Every change detection cycle | Many times |
| `ngAfterContentInit` | `AfterContentInit` | Once, after `ng-content` is projected | Once |
| `ngAfterContentChecked` | `AfterContentChecked` | After every content check | Many times |
| `ngAfterViewInit` | `AfterViewInit` | Once, after component + child views initialise | Once |
| `ngAfterViewChecked` | `AfterViewChecked` | After every view check | Many times |
| `ngOnDestroy` | `OnDestroy` | Once, just before component is removed | Once |

> **The ones you'll use 90% of the time:** `ngOnInit`, `ngOnDestroy`, `ngOnChanges`, `ngAfterViewInit`

---

## 3. Each Hook in Depth

### `constructor`

Not an Angular hook — it's a TypeScript class constructor. Angular calls it when
creating the component instance.

```typescript
@Component({ selector: 'app-user', template: '' })
export class UserComponent {
  constructor(private userService: UserService) {
    // ✅ Inject dependencies
    // ❌ Do NOT access @Input() values — not set yet
    // ❌ Do NOT access child components or DOM — not available yet
    // ❌ Do NOT make HTTP calls here
  }
}
```

**Rule:** Only use the constructor for dependency injection. Nothing else.

---

### `ngOnChanges(changes: SimpleChanges)`

Fires **before `ngOnInit`** on the first run, then every time an `@Input` property
changes. Receives a `SimpleChanges` object describing what changed.

```typescript
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({ selector: 'app-user', template: '' })
export class UserComponent implements OnChanges {

  @Input() userId!: string;
  @Input() role!: string;

  ngOnChanges(changes: SimpleChanges) {
    // fires when userId or role changes

    if (changes['userId']) {
      const prev = changes['userId'].previousValue;
      const curr = changes['userId'].currentValue;
      const isFirst = changes['userId'].firstChange;

      console.log(`userId: ${prev} → ${curr} (first: ${isFirst})`);

      if (!isFirst) {
        // reload data only on subsequent changes, not the initial set
        this.loadUser(curr);
      }
    }
  }

  private loadUser(id: string) { /* ... */ }
}
```

**Key points:**
- Only fires for `@Input()` properties — NOT for changes to internal component state
- `firstChange` is `true` the very first time — useful to skip redundant work on init
- Does NOT fire if you mutate an object/array — Angular uses reference equality

```typescript
// This does NOT trigger ngOnChanges — same object reference
this.user.name = 'new name'; // ❌ mutation — no hook

// This DOES trigger ngOnChanges — new reference
this.user = { ...this.user, name: 'new name' }; // ✅ new object
```

---

### `ngOnInit()`

Fires **once**, after the first `ngOnChanges`. All `@Input()` values are set by
this point. The component's own template is not yet rendered (that's `ngAfterViewInit`).

```typescript
@Component({ selector: 'app-user', template: '' })
export class UserComponent implements OnInit {

  @Input() userId!: string;
  user: User | null = null;

  constructor(private userService: UserService) {}

  ngOnInit() {
    // ✅ @Input values are available here
    // ✅ Safe to make HTTP calls, set up subscriptions
    // ❌ Do NOT access @ViewChild — view not rendered yet
    this.userService.getUser(this.userId).subscribe(u => this.user = u);
  }
}
```

**constructor vs ngOnInit — the interview question:**

| | constructor | ngOnInit |
|---|---|---|
| `@Input()` available | ❌ No | ✅ Yes |
| DI / service injection | ✅ Yes | ✅ Yes (but already done) |
| HTTP calls | ❌ Avoid | ✅ Correct place |
| `@ViewChild` access | ❌ No | ❌ No (use `ngAfterViewInit`) |
| Runs | Once | Once |

---

### `ngDoCheck()`

Fires on **every change detection cycle** — even when nothing in this component
changed. Extremely frequent. Used when you need custom change detection logic that
Angular's default equality check misses.

```typescript
@Component({ selector: 'app-list', template: '' })
export class ListComponent implements DoCheck {

  @Input() items: string[] = [];
  private previousLength = 0;

  ngDoCheck() {
    // Angular won't detect array mutation via ngOnChanges
    // Use ngDoCheck for that
    if (this.items.length !== this.previousLength) {
      console.log('items array length changed');
      this.previousLength = this.items.length;
    }
  }
}
```

**Warning:** `ngDoCheck` runs very frequently. Keep it extremely fast — no HTTP
calls, no heavy computation. A slow `ngDoCheck` directly degrades app performance.

**When to use:** Rarely. Prefer immutable data patterns (new references) so
`ngOnChanges` fires instead.

---

### `ngAfterContentInit()`

Fires **once** after Angular projects external content (via `<ng-content>`) into
the component. This is where `@ContentChild` / `@ContentChildren` are first available.

```typescript
import { Component, AfterContentInit, ContentChild, ElementRef } from '@angular/core';

@Component({
  selector: 'app-card',
  template: `<ng-content></ng-content>`
})
export class CardComponent implements AfterContentInit {

  @ContentChild('header') header!: ElementRef;

  ngAfterContentInit() {
    // ✅ @ContentChild is available here
    console.log('projected header:', this.header?.nativeElement);
  }
}
```

**`ng-content` recap:**
```html
<!-- parent template -->
<app-card>
  <h2 #header>I am projected content</h2>
</app-card>
```

---

### `ngAfterContentChecked()`

Fires after every check of the projected content — after `ngAfterContentInit`
and after every subsequent `ngDoCheck`. Very frequent.

```typescript
ngAfterContentChecked() {
  // Runs after every CD cycle that checks projected content
  // Keep this extremely lightweight
}
```

Rarely needed directly. Most use cases are better served by `ngAfterContentInit`.

---

### `ngAfterViewInit()`

Fires **once** after Angular fully initialises the component's own view and all
child component views. This is where `@ViewChild` and `@ViewChildren` are first
available and safe to use.

```typescript
import { Component, AfterViewInit, ViewChild, ElementRef } from '@angular/core';

@Component({
  selector: 'app-chart',
  template: `<canvas #chartCanvas></canvas>`
})
export class ChartComponent implements AfterViewInit {

  @ViewChild('chartCanvas') canvas!: ElementRef<HTMLCanvasElement>;

  ngAfterViewInit() {
    // ✅ @ViewChild is now available — DOM element exists
    const ctx = this.canvas.nativeElement.getContext('2d');
    this.renderChart(ctx);
  }

  private renderChart(ctx: CanvasRenderingContext2D | null) { /* ... */ }
}
```

**Common use cases:**
- Initialising third-party DOM libraries (charts, editors, maps)
- Measuring DOM element dimensions
- Setting focus on an input element
- Accessing child component methods

```typescript
@ViewChild(ChildComponent) child!: ChildComponent;

ngAfterViewInit() {
  this.child.doSomething(); // ✅ child component instance available
}
```

---

### `ngAfterViewChecked()`

Fires after every check of the component's view and child views. Very frequent —
after `ngAfterViewInit` and after every `ngDoCheck`. Keep it fast.

```typescript
ngAfterViewChecked() {
  // Runs after every CD cycle
  // ⚠️ Modifying component state here can cause ExpressionChangedAfterItHasBeenCheckedError
}
```

**The dreaded `ExpressionChangedAfterItHasBeenCheckedError`:**
If you update a bound property in `ngAfterViewChecked`, Angular detects the change
*after* it already recorded the value for the current cycle, and throws this error
in dev mode.

```typescript
// ❌ Causes ExpressionChangedAfterItHasBeenCheckedError in dev
ngAfterViewChecked() {
  this.title = 'updated'; // bound in template — throws error
}

// ✅ Fix — defer with microtask or use ChangeDetectorRef
ngAfterViewChecked() {
  Promise.resolve().then(() => this.title = 'updated');
  // or
  this.cdr.detectChanges();
}
```

---

### `ngOnDestroy()`

Fires **once**, just before Angular destroys the component. This is where you
clean up — unsubscribe, clear timers, disconnect observers.

```typescript
import { Component, OnDestroy, OnInit } from '@angular/core';
import { Subscription } from 'rxjs';

@Component({ selector: 'app-user', template: '' })
export class UserComponent implements OnInit, OnDestroy {

  private sub = new Subscription();

  ngOnInit() {
    this.sub.add(
      this.userService.getUpdates().subscribe(u => this.user = u)
    );
    this.sub.add(
      this.router.events.subscribe(e => this.handleRoute(e))
    );
  }

  ngOnDestroy() {
    this.sub.unsubscribe(); // ✅ clean up all subscriptions at once
  }
}
```

**Modern alternative — `takeUntilDestroyed` (Angular 16+):**

```typescript
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ selector: 'app-user', template: '' })
export class UserComponent {

  constructor(private userService: UserService) {
    // No ngOnDestroy needed — Angular handles it automatically
    this.userService.getUpdates()
      .pipe(takeUntilDestroyed())
      .subscribe(u => this.user = u);
  }
}
```

**What to clean up in `ngOnDestroy`:**

```typescript
ngOnDestroy() {
  // RxJS subscriptions
  this.subscription.unsubscribe();

  // Timers
  clearInterval(this.intervalId);
  clearTimeout(this.timeoutId);

  // DOM event listeners added manually
  this.renderer.unlisten(this.el, 'click', this.handler);

  // Third-party library instances
  this.chartInstance.destroy();
  this.mapInstance.remove();

  // Subjects (complete them so subscribers don't hang)
  this.destroy$.next();
  this.destroy$.complete();
}
```

---

## 4. Execution Order

### Single component — full sequence

```
new UserComponent()              ← constructor
ngOnChanges({ userId: ... })     ← inputs set by parent
ngOnInit()                       ← initialise
ngDoCheck()                      ← CD runs
ngAfterContentInit()             ← ng-content projected
ngAfterContentChecked()          ← content checked
ngAfterViewInit()                ← view rendered
ngAfterViewChecked()             ← view checked

--- subsequent CD cycles ---
ngOnChanges()     ← only if @Input changed
ngDoCheck()
ngAfterContentChecked()
ngAfterViewChecked()

--- component removed ---
ngOnDestroy()
```

### What fires once vs many times

```
Once:                    Many times (every CD cycle):
  constructor              ngDoCheck
  ngOnInit                 ngAfterContentChecked
  ngAfterContentInit       ngAfterViewChecked
  ngAfterViewInit          ngOnChanges (if inputs change)
  ngOnDestroy
```

---

## 5. Common Patterns

### Pattern 1 — HTTP call on init, clean up on destroy

```typescript
@Component({ selector: 'app-user', template: '' })
export class UserComponent implements OnInit, OnDestroy {
  @Input() userId!: string;
  user: User | null = null;
  private sub!: Subscription;

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.sub = this.userService.getUser(this.userId)
      .subscribe(u => this.user = u);
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();
  }
}
```

### Pattern 2 — Reload data when input changes

```typescript
ngOnChanges(changes: SimpleChanges) {
  if (changes['userId'] && !changes['userId'].firstChange) {
    this.loadUser(changes['userId'].currentValue);
  }
}
```

### Pattern 3 — Access DOM after view init

```typescript
@ViewChild('inputEl') inputEl!: ElementRef<HTMLInputElement>;

ngAfterViewInit() {
  this.inputEl.nativeElement.focus();
}
```

### Pattern 4 — destroy$ Subject pattern (pre Angular 16)

```typescript
private destroy$ = new Subject<void>();

ngOnInit() {
  this.service.getData()
    .pipe(takeUntil(this.destroy$))
    .subscribe(data => this.data = data);
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

### Pattern 5 — `takeUntilDestroyed` (Angular 16+, preferred)

```typescript
// In constructor (inject DestroyRef implicitly)
constructor(private service: DataService) {
  this.service.getData()
    .pipe(takeUntilDestroyed())
    .subscribe(data => this.data = data);
}
```

---

## 6. Common Mistakes

### 1 — Accessing `@ViewChild` in `ngOnInit`

```typescript
@ViewChild('canvas') canvas!: ElementRef;

ngOnInit() {
  console.log(this.canvas); // ❌ undefined — view not rendered yet
}

ngAfterViewInit() {
  console.log(this.canvas); // ✅ available here
}
```

### 2 — Accessing `@Input` in `constructor`

```typescript
@Input() userId!: string;

constructor() {
  console.log(this.userId); // ❌ undefined — inputs not set yet
}

ngOnInit() {
  console.log(this.userId); // ✅ set by parent by this point
}
```

### 3 — Making HTTP calls in `constructor`

```typescript
constructor(private service: UserService) {
  this.service.getUser('1').subscribe(); // ❌ too early, avoid
}

ngOnInit() {
  this.service.getUser('1').subscribe(); // ✅ correct
}
```

### 4 — Forgetting to unsubscribe in `ngOnDestroy`

```typescript
ngOnInit() {
  // ❌ memory leak — subscription lives forever even after component is gone
  this.service.getUpdates().subscribe(data => this.data = data);
}

// Fix: unsubscribe in ngOnDestroy, or use takeUntilDestroyed
```

### 5 — Heavy logic in `ngDoCheck` / `ngAfterViewChecked`

```typescript
ngDoCheck() {
  this.http.get('/api/check').subscribe(); // ❌ HTTP call on every CD cycle
}
```

These fire constantly. Only put ultra-cheap synchronous checks in them.

### 6 — Modifying bound state in `ngAfterViewChecked`

```typescript
ngAfterViewChecked() {
  this.title = 'new'; // ❌ ExpressionChangedAfterItHasBeenCheckedError in dev
}
```

---

## 7. Interview Questions

### Beginner

**Q: What is the correct order of Angular lifecycle hooks?**
> `ngOnChanges → ngOnInit → ngDoCheck → ngAfterContentInit →
> ngAfterContentChecked → ngAfterViewInit → ngAfterViewChecked → ngOnDestroy`

**Q: What is the difference between `constructor` and `ngOnInit`?**
> `constructor` is a TypeScript class constructor — use it only for DI. At that
> point, `@Input()` values are not yet set. `ngOnInit` fires after all inputs are
> set by the parent. HTTP calls, subscriptions, and any initialisation that depends
> on inputs belong in `ngOnInit`.

**Q: When is `@ViewChild` first available?**
> In `ngAfterViewInit`. The DOM and child components are not yet rendered during
> `ngOnInit`, so `@ViewChild` is `undefined` there.

**Q: Where should you unsubscribe from RxJS subscriptions?**
> In `ngOnDestroy`. This fires once just before the component is removed from the
> DOM. Failing to unsubscribe causes memory leaks. In Angular 16+, use
> `takeUntilDestroyed()` to handle this automatically.

---

### Intermediate

**Q: What is the difference between `ngOnInit` and `ngOnChanges`?**
> `ngOnInit` fires once after the component initialises. `ngOnChanges` fires before
> `ngOnInit` on the first run, and then every time an `@Input()` property changes.
> Use `ngOnChanges` when you need to react to subsequent input changes; use
> `ngOnInit` for one-time initialisation.

**Q: Why doesn't `ngOnChanges` fire when you mutate an array or object?**
> Angular uses reference equality (`===`) to detect input changes. Mutating an
> array or object keeps the same reference — Angular sees no change. You need to
> pass a new reference (e.g., spread operator) to trigger `ngOnChanges`.

**Q: What is `ExpressionChangedAfterItHasBeenCheckedError` and how do you fix it?**
> It occurs in dev mode when you modify a template-bound property in
> `ngAfterViewChecked` or `ngAfterViewInit` — after Angular has already recorded
> the value for the current change detection cycle. Fix: defer the update with
> `Promise.resolve().then(...)` (microtask) or call `ChangeDetectorRef.detectChanges()`
> to trigger an extra cycle explicitly.

**Q: When would you use `ngDoCheck`?**
> When you need to detect changes that Angular's default change detection misses —
> such as mutations inside arrays or objects passed as `@Input`. However, it fires
> on every CD cycle so it must be extremely fast. Prefer immutable patterns so
> `ngOnChanges` fires instead.

---

### Advanced

**Q: What is the difference between `ngAfterContentInit` and `ngAfterViewInit`?**
> `ngAfterContentInit` fires after external content projected via `<ng-content>` is
> initialised — this is where `@ContentChild` is first available. `ngAfterViewInit`
> fires after the component's own template and child component views are initialised
> — this is where `@ViewChild` is first available.

**Q: How does `takeUntilDestroyed` work in Angular 16+?**
> It uses `DestroyRef` — an injectable token that Angular provides per component
> instance. It registers a callback on `DestroyRef` that calls `complete()` on an
> internal Subject when the component is destroyed. The `takeUntil` operator then
> completes the observable automatically. No need to implement `OnDestroy` manually.

**Q: In what order do lifecycle hooks fire in a parent-child component tree?**
> Parent `ngOnInit` → Child `ngOnInit` → Child `ngAfterViewInit` →
> Parent `ngAfterViewInit`. The parent's view isn't fully initialised until all
> child views are initialised — so child `ngAfterViewInit` fires before parent's.

---

## 8. Hands-on Exercises

### Exercise 1 — What is the output?

```typescript
@Component({ selector: 'app-demo', template: '' })
export class DemoComponent implements OnInit, AfterViewInit, OnDestroy {

  @Input() value!: string;

  constructor() { console.log('constructor'); }
  ngOnInit()      { console.log('ngOnInit, value =', this.value); }
  ngAfterViewInit() { console.log('ngAfterViewInit'); }
  ngOnDestroy()   { console.log('ngOnDestroy'); }
}
```

Parent template: `<app-demo value="hello" />`

<details>
<summary>Answer</summary>

```
constructor
ngOnInit, value = hello
ngAfterViewInit
--- when component removed ---
ngOnDestroy
```

`@Input()` is set before `ngOnInit` but after `constructor`.
</details>

---

### Exercise 2 — Fix the bug

```typescript
@Component({
  selector: 'app-chart',
  template: `<canvas #myCanvas></canvas>`
})
export class ChartComponent implements OnInit {

  @ViewChild('myCanvas') canvas!: ElementRef;

  ngOnInit() {
    // Bug: this.canvas is undefined here
    this.canvas.nativeElement.getContext('2d');
  }
}
```

<details>
<summary>Solution</summary>

```typescript
export class ChartComponent implements AfterViewInit {

  @ViewChild('myCanvas') canvas!: ElementRef;

  ngAfterViewInit() {
    // ✅ canvas is available here
    this.canvas.nativeElement.getContext('2d');
  }
}
```
</details>

---

### Exercise 3 — Fix the memory leak

```typescript
@Component({ selector: 'app-feed', template: '' })
export class FeedComponent implements OnInit {

  feed: Post[] = [];

  constructor(private feedService: FeedService) {}

  ngOnInit() {
    this.feedService.getLiveFeed().subscribe(posts => this.feed = posts);
  }
}
```

<details>
<summary>Solution — two approaches</summary>

**Option 1 — ngOnDestroy:**
```typescript
export class FeedComponent implements OnInit, OnDestroy {
  feed: Post[] = [];
  private sub!: Subscription;

  ngOnInit() {
    this.sub = this.feedService.getLiveFeed()
      .subscribe(posts => this.feed = posts);
  }

  ngOnDestroy() {
    this.sub.unsubscribe();
  }
}
```

**Option 2 — takeUntilDestroyed (Angular 16+):**
```typescript
export class FeedComponent {
  feed: Post[] = [];

  constructor(private feedService: FeedService) {
    this.feedService.getLiveFeed()
      .pipe(takeUntilDestroyed())
      .subscribe(posts => this.feed = posts);
  }
}
```
</details>

---

## 9. Revision & Cheat Sheet

### 5-Minute Summary

- **constructor** — DI only, no inputs, no DOM
- **ngOnChanges** — fires before init and on every `@Input` change; use `SimpleChanges` to see what changed; does NOT fire on object mutation
- **ngOnInit** — one-time init; `@Input` values available; correct place for HTTP calls
- **ngDoCheck** — every CD cycle; use sparingly; keep fast
- **ngAfterContentInit** — `@ContentChild` first available; `ng-content` projected
- **ngAfterViewInit** — `@ViewChild` first available; DOM rendered; init third-party libs
- **ngAfterViewChecked** — after every view check; modifying bound state here causes `ExpressionChangedAfterItHasBeenCheckedError`
- **ngOnDestroy** — clean up subscriptions, timers, listeners

### Hook Decision Guide

```
Need to react to @Input changes?          → ngOnChanges
One-time init, HTTP call?                 → ngOnInit
Need @ViewChild / DOM access?             → ngAfterViewInit
Need @ContentChild?                       → ngAfterContentInit
Clean up subscriptions / timers?          → ngOnDestroy (or takeUntilDestroyed)
Detect array/object mutations as inputs?  → ngDoCheck (last resort)
```

### Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│           ANGULAR LIFECYCLE HOOKS — CHEAT SHEET              │
├────────────────────┬─────────┬───────────────────────────────┤
│ Hook               │ Times   │ Use for                       │
├────────────────────┼─────────┼───────────────────────────────┤
│ constructor        │ once    │ DI only                       │
│ ngOnChanges        │ many    │ React to @Input changes       │
│ ngOnInit           │ once    │ Init, HTTP, subscriptions     │
│ ngDoCheck          │ many    │ Custom change detection       │
│ ngAfterContentInit │ once    │ @ContentChild access          │
│ ngAfterViewInit    │ once    │ @ViewChild, DOM, 3rd-party    │
│ ngAfterViewChecked │ many    │ (avoid modifying state here)  │
│ ngOnDestroy        │ once    │ Cleanup — unsub, clear timers │
├────────────────────┴─────────┴───────────────────────────────┤
│ KEY RULES                                                    │
│  @Input available from    → ngOnChanges / ngOnInit           │
│  @ViewChild available from → ngAfterViewInit                 │
│  @ContentChild available from → ngAfterContentInit           │
│  ngOnChanges fires on mutation? → NO (needs new reference)   │
│  Best cleanup pattern (16+) → takeUntilDestroyed()           │
└──────────────────────────────────────────────────────────────┘
```

---

---

# Good to Know

---

## 10. Lifecycle in Parent-Child Components

When a parent contains a child component, the lifecycle interleaves:

```
Parent constructor
Parent ngOnChanges
Parent ngOnInit
Parent ngDoCheck
Parent ngAfterContentInit
Parent ngAfterContentChecked
  Child constructor
  Child ngOnChanges
  Child ngOnInit
  Child ngDoCheck
  Child ngAfterContentInit
  Child ngAfterContentChecked
  Child ngAfterViewInit
  Child ngAfterViewChecked
Parent ngAfterViewInit      ← fires AFTER all children are ready
Parent ngAfterViewChecked
```

**Key insight:** Parent's `ngAfterViewInit` waits for all children's `ngAfterViewInit`
to complete first. The parent's view isn't considered initialised until all its
child views are too.

---

## 11. Lifecycle with OnPush Change Detection

With `ChangeDetectionStrategy.OnPush`, Angular skips change detection for the
component unless:
- An `@Input` reference changes
- An event originates from the component or its children
- An `async` pipe resolves
- `ChangeDetectorRef.markForCheck()` is called

This affects how often `ngDoCheck`, `ngAfterContentChecked`, and
`ngAfterViewChecked` fire — they fire far less frequently, which is the
performance benefit of OnPush.

```typescript
@Component({
  selector: 'app-user',
  template: '',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserComponent implements OnInit {
  @Input() user!: User;

  ngOnChanges() {
    // Only fires when user reference changes — not on mutation
  }
}
```

---

## 12. Lifecycle Hooks in Angular 17+ (Signals era)

With Angular Signals, some use cases for lifecycle hooks are replaced by reactive
primitives:

```typescript
// Old — ngOnChanges to react to input changes
@Input() userId!: string;
ngOnChanges() { this.loadUser(this.userId); }

// New — signal input + effect (Angular 17+)
userId = input<string>();

constructor() {
  effect(() => {
    this.loadUser(this.userId()); // reactive — runs when userId signal changes
  });
}
```

Lifecycle hooks (`ngOnInit`, `ngOnDestroy`, etc.) still exist and work the same
in signal-based components. The difference is that `ngOnChanges` becomes less
necessary when using signal inputs, since `effect()` and `computed()` provide
reactive alternatives.

`takeUntilDestroyed()` works seamlessly with signal-based components too.
