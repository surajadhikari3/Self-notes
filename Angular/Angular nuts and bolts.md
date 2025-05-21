
Communication..

| Component Relation            | Best Method                           |
| ----------------------------- | ------------------------------------- |
| Parent ➡ Child                | `@Input()`                            |
| Child ➡ Parent                | `@Output()`                           |
| Parent ↔ Child methods        | `ViewChild()`                         |
| Sibling ↔ Sibling / Unrelated | Shared service (with or without RxJS) |
| Across app, persistent        | `localStorage`, `NgRx`                |
| Navigation-based              | `ActivatedRoute`                      |

### ✅ **Angular Component Communication – With Code & Decorators**

| Method                          | Use Case                                             | Decorators / Mechanism                      | Code Example                                                                                                                                                                                                   |
| ------------------------------- | ---------------------------------------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **@Input()**                    | Pass data from parent ➡ child                        | `@Input()`                                  | **Parent:**`<app-child [title]="pageTitle"></app-child>`**Child:**`@Input() title: string;`                                                                                                                    |
| **@Output() + EventEmitter**    | Send event from child ➡ parent                       | `@Output()` + `EventEmitter`                | **Child:**`@Output() courseSelected = new EventEmitter<string>();``selectCourse(id: string) { this.courseSelected.emit(id); }`**Parent:**`<app-child (courseSelected)="onCourseSelected($event)"></app-child>` |
| **ViewChild / ContentChild**    | Access child component/method in parent              | `@ViewChild()` / `@ContentChild()`          | **Parent:**`@ViewChild(ChildComponent) child!: ChildComponent;``ngAfterViewInit() { this.child.doSomething(); }`                                                                                               |
| **Service with RxJS Subject**   | Communicate between siblings or unrelated components | `Subject`, `BehaviorSubject` in service     | **Shared Service:**`course$ = new Subject<string>();``this.course$.next('Java')`**Subscriber Component:**`this.service.course$.subscribe(course => { ... });`                                                  |
| **ngOnChanges**                 | Respond to changes on `@Input()` properties          | `ngOnChanges` lifecycle hook                | **Child Component:**`@Input() data: any;``ngOnChanges(changes: SimpleChanges) { console.log(changes); }`                                                                                                       |
| **Shared Service (No RxJS)**    | Share state across multiple components               | Service with plain property/methods         | **Service:**`selectedColor = 'blue';`**Components:**`this.color = themeService.selectedColor;`                                                                                                                 |
| **Route Parameters**            | Share data via URL route                             | Angular Router + `ActivatedRoute`           | **URL:** `/user/12`**Component:**`id = this.route.snapshot.paramMap.get('id');`                                                                                                                                |
| **Template Reference Variable** | Access child DOM or component in template            | Template ref + `@ViewChild()`               | **Template:**`<input #userInput>`**TS:**`@ViewChild('userInput') input!: ElementRef;`                                                                                                                          |
| **Local/Session Storage**       | Cross-component or persistent state                  | Web APIs (`localStorage`, `sessionStorage`) | **Save:**`localStorage.setItem('cart', JSON.stringify(cartItems));`**Retrieve:**`const items = JSON.parse(localStorage.getItem('cart'));`                                                                      |
| **NgRx / Signal Store**         | Centralized global app state                         | `Store`, `@select()`, `actions`, etc.       | **Store Setup:**`store.dispatch(addToCart({item}));``store.select('cart').subscribe(...)`                                                                                                                      |


### ✅ **Angular Binding Concepts Comparison Table**

| Binding Type                       | Concept                                                        | When to Use                                                   | Syntax                                               | Code Example                                           |
| ---------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------ |
| **Property Binding**               | One-way binding from component ➡ template                      | Display dynamic values in DOM                                 | `[property]="expression"`                            | `<img [src]="profileImageUrl">`                        |
| **Event Binding (Method Binding)** | One-way binding from template ➡ component                      | Handle user interactions like clicks, input, etc.             | `(event)="method()"`                                 | `<button (click)="onSubmit()">Submit</button>`         |
| **Two-Way Binding**                | Two-way sync between component and view (template ⇄ component) | When user input needs to be reflected in model and vice versa | `[(ngModel)]="property"`  <br>Requires `FormsModule` | `<input [(ngModel)]="username">`  <br>`{{ username }}` |
| **Interpolation**                  | Embed dynamic values in template text                          | Display values in HTML text nodes                             | `{{ expression }}`                                   | `<h1>Hello, {{ user.name }}!</h1>`                     |

### 🧠 **Concept Simplified**

|Type|Flow|Use Case Example|
|---|---|---|
|Property Binding|Component ➡ HTML|Set image source, class, style dynamically|
|Event Binding|HTML ➡ Component (Method)|Button click triggers function|
|Two-Way Binding|HTML ⇄ Component (Sync)|Form input bound to component variable|
|Interpolation|Component ➡ Inline Text|Greet user using `{{ user.name }}`|

AG grid 

## ✅ Community vs Enterprise

| Feature Group                 | Community (Free) ✅ | Enterprise (Paid) 💼 |
| ----------------------------- | ------------------ | -------------------- |
| Basic Sorting, Filter, Paging | ✅                  | ✅                    |
| Row Grouping & Aggregation    | ✅ (programmatic)   | 💼 (drag/drop UI)    |
| Pivot Tables, Range Selection | ❌                  | 💼                   |
| Export to CSV                 | ✅                  | ✅                    |
| Export to Excel               | ❌                  | 💼                   |
| Charting, Clipboard Range     | ❌                  | 💼                   |