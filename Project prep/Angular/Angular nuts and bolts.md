
 Use of websockets for analytics dashboard with spring boot backend and mongo dB backend
 Get really good with angular - all stuff Including writing tests
 Bidirectional streaming of updates

| Component Relation            | Best Method                           |
| ----------------------------- | ------------------------------------- |
| Parent ➡ Child                | `@Input()`                            |
| Child ➡ Parent                | `@Output()`                           |
| Parent ↔ Child methods        | `ViewChild()`                         |
| Sibling ↔ Sibling / Unrelated | Shared service (with or without RxJS) |
| Across app, persistent        | `localStorage`, `NgRx`                |
| Navigation-based              | `ActivatedRoute`                      |

### ✅ **Angular Component Communication – With Code & Decorators**

| Method                          | Use Case                                                             | Decorators / Mechanism                      | Code Example                                                                                                                                                                                                               |
| ------------------------------- | -------------------------------------------------------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **@Input()**                    | Pass data from parent ➡ child                                        | `@Input()`                                  | **Parent:**`<app-child [title]="pageTitle"></app-child>`**Child:**`@Input() title: string;`                                                                                                                                |
| **@Output() + EventEmitter**    | Send event from child ➡ parent                                       | `@Output()` + `EventEmitter`                | **Child:**`<br>@Output() courseSelected = new EventEmitter<string>();<br>``selectCourse(id: string) { this.courseSelected.emit(id); }<br>`**Parent:**`<app-child (courseSelected)="onCourseSelected($event)"></app-child>` |
| **ViewChild / ContentChild**    | Access child  component instances programmatically /method in parent | `@ViewChild()` / `@ContentChild()`          | **Parent:**`@ViewChild(ChildComponent) child!: ChildComponent;``ngAfterViewInit() { this.child.doSomething(); }`                                                                                                           |
| **Service with RxJS Subject**   | Communicate between siblings or unrelated components                 | `Subject`, `BehaviorSubject` in service     | **Shared Service:**`course$ = new Subject<string>();``this.course$.next('Java')`**Subscriber Component:**`this.service.course$.subscribe(course => { ... });`                                                              |
| **ngOnChanges**                 | Respond to changes on `@Input()` properties                          | `ngOnChanges` lifecycle hook                | **Child Component:**`@Input() data: any;<br>``ngOnChanges(changes: SimpleChanges) { console.log(changes); }`                                                                                                               |
| **Shared Service (No RxJS)**    | Share state across multiple components                               | Service with plain property/methods         | **Service:**`selectedColor = 'blue';`<br>**Components:**`this.color = themeService.selectedColor;`                                                                                                                         |
| **Route Parameters**            | Share data via URL route                                             | Angular Router + `ActivatedRoute`           | **URL:** `/user/12`<br>**Component:**`id = this.route.snapshot.paramMap.get('id');`                                                                                                                                        |
| **Template Reference Variable** | Access child DOM or component in template                            | Template ref + `@ViewChild()`               | **Template:**`<input #userInput>`<br>**TS:**`@ViewChild('userInput') input!: ElementRef;`                                                                                                                                  |
| **Local/Session Storage**       | Cross-component or persistent state                                  | Web APIs (`localStorage`, `sessionStorage`) | **Save:**<br>`localStorage.setItem('cart', JSON.stringify(cartItems));<br>`**Retrieve:**`const items = JSON.parse(localStorage.getItem('cart'));`                                                                          |
| **NgRx / Signal Store**         | Centralized global app state                                         | `Store`, `@select()`, `actions`, etc.       | **Store Setup:**`store.dispatch(addToCart({item}));<br>``store.select('cart').subscribe(...)`                                                                                                                              |
|                                 |                                                                      |                                             |                                                                                                                                                                                                                            |


### ✅ **Angular Binding Concepts Comparison Table**

| Binding Type                       | Concept                                                        | When to Use                                                   | Syntax                                               | Code Example                                           |
| ---------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------ |
| **Property Binding**               | One-way binding from component ➡ template                      | Display dynamic values in DOM                                 | `[property]="expression"`                            | `<img [src]="profileImageUrl">`                        |
| **Event Binding (Method Binding)** | One-way binding from template ➡ component                      | Handle user interactions like clicks, input, etc.             | `(event)="method()"`                                 | `<button (click)="onSubmit()">Submit</button>`         |
| **Two-Way Binding**                | Two-way sync between component and view (template ⇄ component) | When user input needs to be reflected in model and vice versa | `[(ngModel)]="property"`  <br>Requires `FormsModule` | `<input [(ngModel)]="username">`  <br>`{{ username }}` |
| **Interpolation**                  | Embed dynamic values in template text                          | Display values in HTML text nodes                             | `{{ expression }}`                                   | `<h1>Hello, {{ user.name }}!</h1>`                     |

### 🧠 **Concept Simplified**

| Type             | Flow                      | Use Case Example                           |
| ---------------- | ------------------------- | ------------------------------------------ |
| Property Binding | Component ➡ HTML          | Set image source, class, style dynamically |
| Event Binding    | HTML ➡ Component (Method) | Button click triggers function             |
| Two-Way Binding  | HTML ⇄ Component (Sync)   | Form input bound to component variable     |
| Interpolation    | Component ➡ Inline Text   | Greet user using `{{ user.name }}`         |

AG grid 

## ✅ Community vs Enterprise

| Feature Group                 | Community (Free) ✅ | Enterprise (Paid) 💼 |
| ----------------------------- | ------------------ | -------------------- |
| Basic Sorting, Filter, Paging | ✅                  | ✅                    |
| Row Grouping & Aggregation    | ✅ (programmatic)   | 💼 (drag/drop UI)    |
| Pivot Tables, Range Selection | ❌                  | 💼                   |
| Export to CSV                 | ✅                  | ✅                    |
| Export to Excel               | ❌                  | 💼                   |
| Charts, Clipboard Range       | ❌                  | 💼                   |

https://chatgpt.com/g/g-p-6810e0b86290819181d84fc8f0e86c94-miscellenous/project

Read about infinite scrolling and virtual scrolling.....
Pivoting......... high level --> summarize the data make the raw data more insightful.....

Ag grid enterprise can be used locally but for the production we need the liscence. It shows the watermark of the enterprise.. 
## 🧠 Summary (TL;DR)

- **Pivoting** = rotating your data → rows become columns
    
- **Used for**: Data summarization, comparison, trend analysis
    
- **AG Grid** supports pivoting with: 
    
    - `pivot: true` (In the column defs)
        
    - `pivotMode: true` (in the grid options)
        
    - `aggFunc: 'sum, avg, count'.
        
- **Enterprise feature** in AG Grid
    
- Very similar in concept to Excel pivot tables, but more dynamic in code

Code example......

```
const columnDefs = [
  { field: 'branch', rowGroup: true },
  { field: 'month', pivot: true },
  { field: 'orders', aggFunc: 'sum' }
];

const rowData = [
  { branch: 'New York', month: 'Jan', orders: 210 },
  { branch: 'New York', month: 'Feb', orders: 150 },
  { branch: 'LA', month: 'Jan', orders: 190 },
  { branch: 'LA', month: 'Feb', orders: 130 }
];

const gridOptions = {
  columnDefs,
  rowData,
  pivotMode: true,   // Important!
  animateRows: true,
  groupDefaultExpanded: 1
};

new agGrid.Grid(document.getElementById('myGrid'), gridOptions);

```
Testing...


## 🛠 Tooling Stack

|Purpose|Tool|
|---|---|
|Unit Testing|Jest or Jasmine + Karma|
|E2E Testing|Cypress or Playwright|
|HTTP Mocking|HttpClientTestingModule|
|Code Coverage|`ng test --code-coverage`|
|Linting|ESLint|
|Formatter|Prettier|
|Type Checking|TypeScript (strict mode recommended)|

---

## 📌 Summary (Memory Trick)

> **“SCARF”** — for real-world Angular test setup:

- **S**ervices → Test with `HttpClientTestingModule`
    
- **C**omponents → Use `TestBed` + `detectChanges()`
    
- **A**sserts → Keep focused, small, and isolated
    
- **R**outing → Use `RouterTestingModule`
    
- **F**ront-to-back → Add Cypress for full E2E


---

## 🏗️ High-Level Architecture of a Real-World Angular App

```
                                +------------------------+
                                |   Backend REST API     |
                                |   (e.g., Spring Boot)   |
                                +------------------------+
                                          ▲
                                          │ HTTP (JSON)
                                          ▼
+-------------------------+     +-------------------------+     +-------------------------+
|   Core Module           |<--->| Feature Module (Users)  |<--->| Shared Module           |
| - AuthGuard             |     | - user.service.ts       |     | - UI components         |
| - Global Interceptors   |     | - user.component.ts     |     | - Pipes, Directives     |
| - Http Interceptor      |     | - user-list.component.ts|     +-------------------------+
| - Error Handler Service |     +-------------------------+
+-------------------------+
            ▲
            │
     +------+-------+
     |   App Module  |
     | - Routing     |
     | - AppComponent|
     +---------------+

Test Architecture
-----------------
Unit Tests   → Jasmine/Jest (Component/Service level)
E2E Tests    → Cypress or Playwright (Full page flows)
Mock Server  → JSON Server / MSW (Mock HTTP backend)
```

---

## 📁 Folder Structure

```
src/
├── app/
│   ├── core/                # Singleton services, guards, interceptors
│   │   └── auth/
│   │       ├── auth.guard.ts
│   │       ├── auth.service.ts
│   │       └── token.interceptor.ts
│   ├── shared/              # Reusable UI components, pipes
│   │   ├── components/
│   │   └── pipes/
│   ├── features/
│   │   └── users/
│   │       ├── user.service.ts
│   │       ├── user-list.component.ts
│   │       ├── user-card.component.ts
│   │       └── user-list.component.spec.ts
│   ├── app-routing.module.ts
│   ├── app.component.ts
│   └── app.module.ts
├── assets/
├── environments/
├── styles/
│   └── _variables.scss
├── tests/
│   └── e2e/                 # Cypress or Playwright tests
├── main.ts
└── index.html

