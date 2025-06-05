

## ✅ 1. **What is Karma?**

### 🔹 Karma is a **test runner**, not a testing framework.

It runs your tests in real browsers and watches for file changes.

### 🎯 Karma's Role:

|Feature|What It Does|
|---|---|
|🧪 Test Runner|Launches browsers like Chrome, Firefox|
|🔁 Auto Watch|Reruns tests when code changes|
|📊 Reporting|Shows test results, errors, and code coverage|
|🔗 Integrates with|Jasmine (by default), or Mocha, QUnit, etc.|

### 📦 Used with:

```bash
ng test         # Under the hood: runs "karma start karma.conf.js"
```

---

## ✅ 2. **What is Jasmine?**

### 🔹 Jasmine is a **testing framework** for writing test cases.

It's used **with Karma** by default in Angular projects.

### 🎯 Jasmine Features:

- BDD-style syntax (`describe`, `it`, `expect`)
    
- Spies for mocking
    
- Built-in matchers (`toBe`, `toEqual`, `toContain`, etc.)
    
- No dependencies (zero-config)
    

### 🔸 Example Jasmine test:

```ts
describe('Math Utils', () => {
  it('should add numbers correctly', () => {
    expect(2 + 2).toBe(4);
  });
});
```

---

## ✅ 3. **What is Jest?**

### 🔹 Jest is an **all-in-one test runner + assertion + mocking library** created by Facebook.

> Jest = Karma + Jasmine + Sinon (combined, faster, easier)

### 🎯 Why use Jest?

|Feature|Benefit|
|---|---|
|🧪 Runs without browser|Headless + fast|
|🚫 No Karma|No need for karma.conf.js|
|📈 Built-in coverage|No extra setup|
|💬 Snapshot testing|Great for React, usable in Angular|
|⚡ Blazing fast|Runs in Node.js, parallel tests|

### ✅ Use in Angular?

```bash
ng add @briebug/jest-schematic
```

---

## ✅ 4. **How to Mock a Service in Angular Unit Tests**

Here’s how you mock a service in **Jasmine** (same concept applies for Jest):

---

### 🧪 Service: `user.service.ts`

```ts
@Injectable({ providedIn: 'root' })
export class UserService {
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
}
```

---

### 👩‍🔬 Component: `user-list.component.ts`

```ts
@Component({...})
export class UserListComponent {
  users: User[] = [];

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.userService.getUsers().subscribe(data => this.users = data);
  }
}
```

---

### ✅ Mocking in Test: `user-list.component.spec.ts`

```ts
describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let userServiceSpy: jasmine.SpyObj<UserService>;

  beforeEach(() => {
    const mockService = jasmine.createSpyObj('UserService', ['getUsers']);

    TestBed.configureTestingModule({
      declarations: [UserListComponent],
      providers: [
        { provide: UserService, useValue: mockService }
      ]
    });

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
    userServiceSpy = TestBed.inject(UserService) as jasmine.SpyObj<UserService>;
  });

  it('should fetch and show users', () => {
    const mockUsers = [{ id: 1, name: 'Alice' }];
    userServiceSpy.getUsers.and.returnValue(of(mockUsers)); // 👈 Mocking service response

    fixture.detectChanges(); // Triggers ngOnInit()

    expect(component.users.length).toBe(1);
    expect(component.users[0].name).toBe('Alice');
  });
});
```

---

### ✅ Mocking in Jest:

If you're using **Jest**, replace Jasmine spy with `jest.fn()`:

```ts
const mockUserService = {
  getUsers: jest.fn().mockReturnValue(of([{ name: 'John' }]))
};
```

---

## 🔁 Summary Chart

| Tool    | Role                        | Real-World Equivalent       |
| ------- | --------------------------- | --------------------------- |
| Karma   | Test runner (browser-based) | Mocha, CLI test launcher    |
| Jasmine | Testing framework           | JUnit, NUnit                |
| Jest    | All-in-one modern test tool | Best for speed + simplicity |
| SpyObj  | Mocking with Jasmine        | Mockito (Java), Sinon (JS)  |
