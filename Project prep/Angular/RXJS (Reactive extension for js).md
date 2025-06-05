In real-world Angular applications, **RxJS (Reactive Extensions for JavaScript)** is heavily used for **asynchronous programming, state management, event handling, and API composition**. Below is a curated list of the **most widely used RxJS operators** (with real-world examples), grouped by purpose.

---

## 🧠 1. **Creation Operators**

Used to create observables

|Operator|Description|Example|
|---|---|---|
|`of()`|Emits the arguments|`of(1, 2, 3)`|
|`from()`|Converts array, Promise, etc. into Observable|`from(fetch('/api/data'))`|
|`interval()`|Emits sequential numbers every N ms|`interval(1000)`|
|`timer()`|Emits after a delay or repeatedly|`timer(1000, 2000)`|
|`Subject()` / `BehaviorSubject()`|Emits values to subscribers manually|`this.search$ = new BehaviorSubject('')`|

---

## 🔄 2. **Transformation Operators**

Used to manipulate or change the emitted values

|Operator|Description|Real-World Use|
|---|---|---|
|`map()`|Transforms emitted values|Change API response format|
|`pluck()`|Extracts property from objects|`pluck('user', 'name')`|
|`switchMap()`|Cancels previous inner observable|Search-as-you-type|
|`mergeMap()`|Flattens and merges observables|Parallel HTTP requests|
|`concatMap()`|Sequentially queues requests|Uploading files one by one|
|`exhaustMap()`|Ignores new emissions if one is in progress|Button spam prevention|

🔁 **switchMap vs mergeMap vs concatMap vs exhaustMap** is critical in real-world apps!

---

## 📦 3. **Filtering Operators**

Used to filter or control emissions

|Operator|Description|Use Case|
|---|---|---|
|`filter()`|Emits only values that pass predicate|`filter(user => user.isActive)`|
|`debounceTime()`|Waits before emitting|Search box throttling|
|`distinctUntilChanged()`|Prevents emitting same value twice|Input change detection|
|`take(n)`|Emits first `n` values|One-time calls|
|`takeUntil()`|Unsubscribe based on another observable|Component destruction cleanup|

---

## 🧪 4. **Combination Operators**

Used to combine multiple observables

|Operator|Description|Use Case|
|---|---|---|
|`combineLatest()`|Combines latest values from multiple sources|Form control values|
|`forkJoin()`|Waits for all to complete|Parallel API calls at init|
|`zip()`|Emits pairs of values|Combine related observables|
|`withLatestFrom()`|Combine with most recent value from another stream|Click + current state|

---

## ⏱️ 5. **Utility Operators**

|Operator|Description|Use Case|
|---|---|---|
|`tap()`|Side effects (logging/debug)|`tap(console.log)`|
|`catchError()`|Error handling and recovery|Fallback values or redirection|
|`finalize()`|Cleanup logic when observable ends|Hide loading spinner|

---

## 🛠️ 6. **Real-World Example: Search-as-you-type**

```ts
searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => this.apiService.search(query))
).subscribe(results => {
  this.results = results;
});
```

---

## 🧹 7. **Clean-up in `ngOnDestroy()` Using `takeUntil`**

```ts
private destroy$ = new Subject<void>();

ngOnInit() {
  this.apiService.getData().pipe(
    takeUntil(this.destroy$)
  ).subscribe(...);
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

---

## 🧰 8. **Subjects in Real-World State Management**

|Type|Description|Example|
|---|---|---|
|`Subject`|Multicast, no initial value|Emits events|
|`BehaviorSubject`|Requires initial value, replays last|Ideal for UI state|
|`ReplaySubject`|Replays N values|Analytics buffering|
|`AsyncSubject`|Emits last value on complete|Single value after async call|

---

## 🎯 Summary (Memory Aid: **MAP FIT CUBE**)

|Category|Key Operators|
|---|---|
|**M**apping|`map`, `switchMap`, `mergeMap`, `concatMap`|
|**A**ggregating|`combineLatest`, `forkJoin`, `zip`|
|**P**reventing|`debounceTime`, `distinctUntilChanged`|
|**F**iltering|`filter`, `take`, `takeUntil`|
|**I**nitiating|`of`, `from`, `interval`, `timer`|
|**T**erminating|`finalize`, `catchError`|
|**C**leaning|`tap`, `catchError`, `takeUntil`|
|**U**tilities|`withLatestFrom`, `pluck`, `shareReplay`|
|**B**ehaviors|`BehaviorSubject`, `ReplaySubject`|
|**E**vent control|`Subject`, `AsyncSubject`|

---

Sure! Here's the **RxJS Operators Table** in plain text (Markdown-style) format — easy to **copy-paste into any document, Confluence page, Notion, or README**:

---

### ✅ RxJS Operators with Descriptions and Examples

|Category|Operator|What It Does / When to Use|Example|
|---|---|---|---|
|Creation|`of`|Create observable from arguments|`of(1, 2, 3)`|
|Creation|`from`|Convert array/promise to observable|`from([10, 20, 30])`|
|Creation|`interval`|Emit values at regular intervals|`interval(1000)`|
|Creation|`timer`|Emit after delay (once or repeatedly)|`timer(2000, 1000)`|
|Transformation|`map`|Transform each emitted value|`map(x => x * 2)`|
|Transformation|`pluck`|Extract property from object|`pluck('user', 'name')`|
|Transformation|`switchMap`|Cancel previous & switch to new observable (e.g., search box)|`switchMap(q => api.search(q))`|
|Transformation|`mergeMap`|Merge multiple inner observables (parallel calls)|`mergeMap(id => http.get('/user/' + id))`|
|Transformation|`concatMap`|Queue requests sequentially|`concatMap(val => http.post('/log', val))`|
|Transformation|`exhaustMap`|Ignore new emissions if one is still processing|`exhaustMap(() => loginClick$)`|
|Filtering|`filter`|Emit only matching values|`filter(x => x > 5)`|
|Filtering|`debounceTime`|Delay emissions (used for inputs/search)|`debounceTime(300)`|
|Filtering|`distinctUntilChanged`|Only emit when value changes|`distinctUntilChanged()`|
|Filtering|`take`|Take first N values then complete|`take(3)`|
|Filtering|`takeUntil`|Complete when notifier emits (cleanup)|`takeUntil(this.destroy$)`|
|Combination|`combineLatest`|Emit latest values from all sources|`combineLatest([a$, b$])`|
|Combination|`forkJoin`|Wait for all observables to complete then emit one final result|`forkJoin([http1$, http2$])`|
|Combination|`zip`|Pair values from multiple observables|`zip([interval1$, interval2$])`|
|Combination|`withLatestFrom`|Combine current with latest from another observable|`withLatestFrom(this.form$)`|
|Utility|`tap`|Perform side effects like logging, analytics, debugging|`tap(console.log)`|
|Utility|`catchError`|Catch and handle errors|`catchError(err => of([]))`|
|Utility|`finalize`|Cleanup logic on observable complete or error|`finalize(() => stopSpinner())`|
|State|`Subject`|Manual multicasting, no initial value|`const s$ = new Subject()`|
|State|`BehaviorSubject`|Holds latest value, emits on subscribe|`const s$ = new BehaviorSubject('init')`|
|State|`ReplaySubject`|Replays multiple previous values|`const s$ = new ReplaySubject(2)`|
|State|`AsyncSubject`|Emits last value only after completion|`const s$ = new AsyncSubject()`|

---

Here are **most frequently used RxJS operator combinations** in real-world Angular apps — especially for API calls, reactive forms, auto-suggestions, authentication flows, and lifecycle management.

---

## ✅ 1. **Search-as-You-Type**

Used in inputs (autocomplete, filters)

```ts
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => this.api.searchUsers(query))
).subscribe(results => {
  this.users = results;
});
```

### 🔧 Combo:

- `debounceTime(300)`: Waits 300ms before emitting
    
- `distinctUntilChanged()`: Emits only if the input has changed
    
- `switchMap()`: Cancels previous request if a new input comes in
    

---

## ✅ 2. **Form Save with Delay (Debounced Auto-Save)**

```ts
this.form.valueChanges.pipe(
  debounceTime(1000),
  distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b)),
  switchMap(updatedForm => this.api.saveForm(updatedForm))
).subscribe();
```

---

## ✅ 3. **Chained API Calls (e.g., login → fetch profile)**

```ts
this.authService.login(credentials).pipe(
  switchMap(() => this.api.getUserProfile())
).subscribe(profile => this.profile = profile);
```

---

## ✅ 4. **Parallel API Calls (forkJoin)**

```ts
forkJoin([
  this.api.getUserInfo(),
  this.api.getUserOrders()
]).subscribe(([info, orders]) => {
  this.info = info;
  this.orders = orders;
});
```

---

## ✅ 5. **Cancel Request on Component Destroy (takeUntil pattern)**

```ts
private destroy$ = new Subject<void>();

ngOnInit() {
  this.api.fetchData().pipe(
    takeUntil(this.destroy$)
  ).subscribe(data => this.data = data);
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

---

## ✅ 6. **With Latest Form Value When Button Clicked**

```ts
this.saveButtonClicked$.pipe(
  withLatestFrom(this.form.valueChanges),
  switchMap(([click, formData]) => this.api.save(formData))
).subscribe();
```

---

## ✅ 7. **Sequential Requests (concatMap)**

```ts
from([1, 2, 3]).pipe(
  concatMap(id => this.api.getDetails(id))
).subscribe(console.log);
```

---

## ✅ 8. **Prevent Button Spamming (exhaustMap)**

```ts
this.saveClick$.pipe(
  exhaustMap(() => this.api.saveForm(this.form.value))
).subscribe();
```

---

## 🧠 Tip: When to Use Which Mapping Operator?

|Operator|Behavior|Use Case Example|
|---|---|---|
|`switchMap`|Cancels previous, switches to new|Autocomplete, real-time input|
|`mergeMap`|Parallel execution|Parallel uploads, multiple APIs|
|`concatMap`|Sequentially queues|Retry queues, step-by-step workflows|
|`exhaustMap`|Ignores new until current finishes|Prevent duplicate save/login calls|

---

> The `pipe()` function lets you chain **RxJS operators** to modify the stream before it reaches your subscriber.