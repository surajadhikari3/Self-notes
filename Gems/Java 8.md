
Functional interface

is special type of interface that have the SAM(Single abstract method ) that can have n number of default and the static methods. There are several pre defined functional interface in java.util.function package which are

### **Built-in Functional Interfaces (Java 8+ in `java.util.function`)**

| **Interface**       | **Method Signature** | **Use Case**                    |
| ------------------- | -------------------- | ------------------------------- |
| `Function<T,R>`     | `R apply(T t)`       | Transform input to output       |
| `Consumer<T>`       | `void accept(T t)`   | Process input with side-effects |
| `Supplier<T>`       | `T get()`            | Provides a value, no input      |
| `Predicate<T>`      | `boolean test(T t)`  | Test condition                  |
| `BiFunction<T,U,R>` | `R apply(T t, U u)`  | Function with two inputs        |



- ✅ A **Functional Interface (FI)** can have:
    
    - **Only one abstract method** (this is mandatory)
        
    - ✅ **Any number of `default` methods**
        
    - ✅ **Any number of `static` methods**
        

> 💡 The `@FunctionalInterface` annotation ensures this constraint — **only one abstract method** — but **allows multiple `default` and `static` methods**.
### Summary Table

| **Type** | **Allowed in FI?** | **Count Limit** | **Purpose**                                 |
| -------- | ------------------ | --------------- | ------------------------------------------- |
| Abstract | ✅ Yes              | **Exactly 1**   | Enables functional-style behavior           |
| Default  | ✅ Yes              | **Unlimited**   | Optional reusable behavior for implementers |
| Static   | ✅ Yes              | **Unlimited**   | Utility methods tied to the interface       |


🧠 **Pro Trick:**  
Mention that functional interfaces are **used heavily in Streams API**, `CompletableFuture`, and reactive frameworks like **Project Reactor**, **RxJava**, and **Spring WebFlux** — this shows you're thinking at a **framework level**, not just Java syntax.