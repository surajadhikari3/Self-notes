Async/await. Promises, Callbacks
Event Loop
Async nature of js
react (react hooks (useState, useEffect, useMemo), state management(redux), Context api
How to pass state from parent to child and vice versa.
class based vs functional based components...
virtual DOM vs Real DOM..

//as well 
microservice architectures
api development methodology 
devops work
kafka realted --> messaging discussion


//For standup
Agile /Scrum methodology / Team structure
Sprint May 1 to May 15
Sprint Planning - April 25th ,26th or 27th (should be before sprint) --> Story creation and rough estimation
Daily Standup
Sprint retro --> 




Event loop

Event loop is the mechanism that enables non-blocking, asynchronous behaviour in js event though the js is single threaded...




![[Screenshot 2025-05-06 at 10.36.07 PM.png]]

## Comparison Table

| Component           | Type         | Examples                                                                                                                                                                          | Priority Order |
| ------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| **Call Stack**      | Synchronous  | Function calls, expressions                                                                                                                                                       | Highest        |
| **Microtask Queue** | Asynchronous | `Promise.then()`, `queueMicrotask(), mutableObservable(), inside async await implementation                                                                                       | High           |
| **Task Queue**      | Asynchronous | `setTimeout`, `setInterval`, `fetch`                                                                                                                                              | Lower          |
| **Web APIs**        | Environment  | Timers, AJAX, DOM events --> It registers the callback, delay for setTimeout                                                                                                      | N/A            |
| **Event Loop**      | Mechanism    | Orchestrates execution (It constantly checks the call stack and pushes the task from microtask queue <br>and once the microtask queue is empty then task queue to the Call Stack) | N/A            |


Microtask queue has the highest priority over the task queue after the call stack --> the event loop pushes the microtask queue to the call stack.



Interview Summary

Javascript uses a single threaded event loop model for code execution, handling events and managing asynchronous operations.

However Node.js uses multithreading internally (via libuv) to handle file I/O, networking , and timers efficiently, making it appear asynchronous to the developer.

## **`sync/await` Example and Explanation**

### 🔧 What is `async/await`?

`async/await` is **syntactic sugar** over Promises. It allows you to write **asynchronous code** that looks **synchronous**, making it easier to read and maintain.

### ✅ Example:

```
`function delay(ms) { 
return new Promise(resolve => setTimeout(resolve, ms));
} 
async function fetchData() { 
console.log('Fetching data...'); 
await delay(2000); // waits 2 seconds  
console.log('Data received!');
}  

fetchData();`
```

### 🔄 Output:

bash

`Fetching data... 
(wait 2 seconds)
Data received!`

---

## 2️⃣ **`Promise.all()` – Run Tasks in Parallel**

### 🔧 What is `Promise.all()`?

It takes an **array of Promises** and returns a **single Promise** that resolves when **all promises** resolve, or **rejects if any one fails**.

### ✅ Example:


`const p1 = new Promise(resolve => setTimeout(() => resolve('A'), 1000)); 
const p2 = new Promise(resolve => setTimeout(() => resolve('B'), 2000)); 
const p3 = new Promise(resolve => setTimeout(() => resolve('C'), 1500));  
Promise.all([p1, p2, p3]) .then(values => console.log(values))   .catch(err => console.error(err));`

### 🔄 Output:


CopyEdit

`['A', 'B', 'C']  // after 2 seconds (longest one)`


## 📊 Comparison: `CompletableFuture` (Java) vs `Promise` (JavaScript)

| Feature                    | `CompletableFuture` (Java)                | `Promise` (JavaScript)                        |
| -------------------------- | ----------------------------------------- | --------------------------------------------- |
| Language                   | Java                                      | JavaScript                                    |
| Async Execution            | Yes (via `supplyAsync`, `runAsync`, etc.) | Yes (native support with `.then()`, `await`)  |
| Callback Chaining          | Yes (`thenApply`, `thenCompose`)          | Yes (`then`, `catch`, `finally`)              |
| Exception Handling         | `exceptionally`, `handle`, `whenComplete` | `catch`, `try/catch` with `async/await`       |
| Combining Tasks            | `thenCombine`, `allOf`, `anyOf`           | `Promise.all`, `Promise.race`, `Promise.any`  |
| Blocking Option            | `get()`, `join()` (can block)             | Usually non-blocking, no true blocking method |
| Custom Thread Pool Support | Yes (`ExecutorService`)                   | No (uses event loop)                          |
| Parallel Task Execution    | Yes (via thread pool)                     | Yes (via `Promise.all`)                       |
