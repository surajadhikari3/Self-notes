Life cycle of Thread 


1. **New:** Thread is created. 
    
2. **Runnable:** Thread is ready to run.
    
3. **Running:** Thread is executing.
    
4. **Blocked/Waiting:** Thread is waiting for a resource.
    
5. **Terminated:** Thread has finished execution.


  
Both wait() and join() are used in Java for thread synchronization, but they serve different purposes. Here's a breakdown of their differences:


**1. Purpose**

- **wait()**: Causes the current thread to release the lock and enter the waiting state until another thread notifies it (notify() or notifyAll()).
- **join()**: Causes the current thread to wait until the thread on which join() was called has finished execution.

**2. Usage**

- **wait()** is used for inter-thread communication (inside synchronized blocks/methods).
- **join()** is used to wait for a thread to complete its execution.

**3. Lock Handling**

- **wait()** must be called inside a synchronized block or method, as it requires the owning object's monitor lock.
  
- **join()** does not require a synchronized block since it simply waits for another thread to die.

**4. How They Work**

- **wait()** releases the object's lock and allows other threads to acquire it. The waiting thread resumes only when it gets notified.
- **join()** does not release any lock; it just pauses the current thread until the target thread completes.

**5. Where They Are Defined**

- **wait()** is a method of Object class.
- **join()** is a method of Thread class.


  
Key Takeaways**

|                        |                                                       |                                           |
| ---------------------- | ----------------------------------------------------- | ----------------------------------------- |
| **Feature**            | **wait()**                                            | **join()**                                |
| Purpose                | Releases the lock and waits for notify or notifyAll() | Thread waits for another thread to finish |
| Belongs To             | Object class                                          | Thread class                              |
| Requires synchronized? | Yes                                                   | No                                        |
| Releases Lock?         | Yes                                                   | No                                        |
| Wakes Up By            | notify() / notifyAll()                                | When the thread finishes execution        |
|                        |                                                       |                                           |


---

**Thread Dump**

Thread dump is a snapshot of all threads running in the JVM at a particular moment.

### **How to Generate a Thread Dump**

✅ On **Linux/macOS**:


```
kill -3 <pid> --> It logs to the tomcat application terminal even you fire the  cmd from outer terminal not in the terminal iteself
```

➡ Outputs to `stderr` (often visible in `catalina.out`, if using Tomcat)

✅ With **jstack** (tool from JDK):


`jstack <pid>`

✅ On **Windows**: Use `CTRL+Break` in the command window running the app  
Or use **JVisualVM** or **JConsole** (GUI tools) --> Visual Tools


Dynatrace --> Monitoring tools...

say :

kill -3 pid --> For linux
Jconsole, JVisualVM --> Visual tool
Dynatrace. --> Monitoring tools....



# 🛠 Exception Handling: `execute()` vs `submit()`

| Feature              | `execute()`                | `submit()`                 |
| :------------------- | :------------------------- | :------------------------- |
| Exception Handling   | Terminates thread abruptly | Captured inside Future     |
| Supports Callable<T> | ❌ No                       | ✅ Yes                      |
| Can Retrieve Result  | ❌ No                       | ✅ Yes (via Future.get())   |
| Exception Handling   | No                         | Yes                        |
| Takes                | Runnable                   | Both Runnable and Callable |

> **Use `execute()`** when you **don’t need** a return value or exception handling.  
> **Use `submit()`** when you **need a result** or **want to handle exceptions** properly. 🚀

---

# 📤 Conveying Data to Threads

There are **four main ways**:

| Method                             | Description                                           | Example Scenario                |
| :--------------------------------- | :---------------------------------------------------- | :------------------------------ |
| **Constructor Injection**          | Pass data via constructor                             | Custom processing task          |
| **Setter Methods / Public Fields** | Set data after object creation before starting thread | Multiple values or late-binding |
| **ThreadLocal Variables**          | Store thread-specific data without passing            | User session ID, request ID     |
| **Lambda Expressions (Java 8+)**   | Capture local variables into Runnable (closures)      | Very short tasks                |

---

## 📦 Example: Passing Data to Threads

```java
class Task implements Runnable {
    private final int data;

    public Task(int data) {
        this.data = data;
    }

    @Override
    public void run() {
        System.out.println("Processing data: " + data);
    }
}
```

---

# 🔗 Using BlockingQueue (Thread Communication)

Useful for producer-consumer pattern.

```java
import java.util.concurrent.*;

public class Main {
    public static void main(String[] args) {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

        Runnable producer = () -> {
            try {
                queue.put(1);
                System.out.println("Produced: 1");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };

        Runnable consumer = () -> {
            try {
                System.out.println("Consumed: " + queue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };

        new Thread(producer).start();
        new Thread(consumer).start();
    }
}
```

---

# 📝 Summary: Data Sharing Strategies

| Approach                         | Use Case                               |
| :------------------------------- | :------------------------------------- |
| **Constructor Arguments**        | Simple immutable data                  |
| **ThreadLocal**                  | Per-thread unique data (user sessions) |
| **Callable & Future**            | Return values from threads             |
| **Shared Object (Synchronized)** | Shared mutable data                    |
| **BlockingQueue**                | Thread communication                   |

---

# 🛠 Java NIO Core Components

| Component     | Description                                             |
| :------------ | :------------------------------------------------------ |
| **Buffers**   | Holds data for reading/writing (ByteBuffer, etc.)       |
| **Channels**  | Represents I/O connections (FileChannel, SocketChannel) |
| **Selectors** | Monitor multiple channels with a single thread          |
| **Charsets**  | Handle character encoding/decoding                      |

---

# ⚡ Advantages of Java NIO over Traditional I/O

- ✔ Better performance (especially for large files/network operations)
    
- ✔ Non-blocking I/O
    
- ✔ Memory-mapped files (fast file access)
    
- ✔ Single-threaded multiplexing with Selectors
    

---

# 🔒 Transaction Isolation Levels

| Level            | Description                             |
| :--------------- | :-------------------------------------- |
| READ_UNCOMMITTED | Can read uncommitted (dirty) data       |
| READ_COMMITTED   | Only reads committed data               |
| REPEATABLE_READ  | Prevents non-repeatable reads           |
| SERIALIZABLE     | Highest isolation, locks entire dataset |

  
Isolation level understanding from the below video...

[https://www.youtube.com/watch?v=-gxyut1VLcs](https://www.youtube.com/watch?v=-gxyut1VLcs)

  
understand 3 things....


1. Dirty Read --> Even uncommitted read is visible to other transaction.
2. Non repetable read --> T1 reads one. value later when it reads the same value the value is changed which is non repetable meaning it is being updated by another T2
3. Phantom read --> T1 performs query based on the certain condn get specific rows from table later T2 performs the update query based on the same condition and when T1 performs the same query again then it will see the new row which is phantom read.

  

In spring we can use the @Transactional annotation to maintain the isolation level..

```
@Transactional(isolation = Isolation.READ_COMMITTED)

public void processTransaction() {

    // Business logic

} 

```

  
**![[Isolation level.png]]**


---

# 📚 Transaction Concepts:

Video Reference: [Transaction Isolation Explained](https://www.youtube.com/watch?v=-gxyut1VLcs)

Understand these:

1. **Dirty Read**: Reading uncommitted data.
    
2. **Non-repeatable Read**: Re-reading the same row gives different values.
    
3. **Phantom Read**: Re-querying yields new unexpected rows.
    

---

## 🌱 Spring Transaction Example

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void processTransaction() {
    // Business logic
}
```

---

# 🔄 Transaction Propagation in Spring Boot

**Propagation** defines what happens if a method is called inside another transaction.

---

## 🚀 REQUIRES_NEW vs NESTED

| Feature                          | REQUIRES_NEW                                | NESTED                           |
| :------------------------------- | :------------------------------------------ | :------------------------------- |
| Transaction Behavior             | Always starts a new independent transaction | Nested inside parent transaction |
| Parent-Child Relationship        | Completely separate                         | Logical child inside parent      |
| Rollback Behavior                | Inner rollback doesn't affect parent        | Parent rollback undoes child too |
| Suspension of Parent Transaction | Yes                                         | No                               |
| Use Case                         | Independent ops like audit logs             | Dependent but isolated sub-ops   |

---

# 🛠 JDBC vs Spring Transaction Handling

| Feature     | JDBC                                        |
| :---------- | :------------------------------------------ |
| Isolation   | `connection.setTransactionIsolation(level)` |
| Propagation | Manual handling (multiple connections)      |
| Rollback    | `connection.rollback()` manually            |
| Commit      | `connection.commit()` manually              |

---

# 🔥 Executors Overview

| Executors Method                 | Best Use Case                                                                   |
| :------------------------------- | :------------------------------------------------------------------------------ |
| newFixedThreadPool               | Predictable fixed-size thread use                                               |
| newCachedThreadPool              | Many short tasks, create threads as needed (Removed after 60 sec of inactivity) |
| newSingleThreadExecutor          | One thread at a time tasks                                                      |
| newScheduledThreadPool           | Scheduled repeated tasks                                                        |
| newSingleThreadScheduledExecutor | Single-threaded scheduled task                                                  |
| newWorkStealingPool              | CPU-heavy parallel tasks (Java 8+)                                              |

---

### `CompletableFuture`

`CompletableFuture` is part of Java 8’s `java.util.concurrent` package.  --> come in java  5.0
It represents **a future result of an asynchronous computation** and allows you to **write non-blocking, callback-style code** with the callback chaining functionality that is easier to read and compose.

---

### 🧠 Layman Analogy:

> Think of `CompletableFuture` like **ordering food in a restaurant**. You place your order (start an async task) and continue chatting (do other work). When your food is ready (task completed), the waiter **notifies you** (callback executed) — you don't block or wait.

---

### 🧩 Key Concepts

| Concept            | Description                                                                       |
| ------------------ | --------------------------------------------------------------------------------- |
| `supplyAsync()`    | Runs a task asynchronously in the **ForkJoinPool** (or custom executor)           |
| `thenApply()`      | Transforms result when it's available (takes the function)                        |
| `thenAccept()`     | Consumes result without returning a value (consumer)                              |
| `thenCombine()`    | Combine two independent futures                                                   |
| `exceptionally()`  | Handles exception in async computation                                            |
| `join()` / `get()` | Blocks and waits for result (get throws checked exception; join throws unchecked) |
| thenCompose()      | Chains a **dependent async task**, flattens nested futures                        |
| exceptionally()    | Handles **exceptions only**, lets you recover or default                          |

---


### ✅ Code Example

```
`CompletableFuture.supplyAsync(() -> {  
	return "Hello"; 
}).thenApply(result -> {    
	return result + " World"; 
}).thenAccept(finalResult -> {    
System.out.println(finalResult); // Hello World });`
```
---

### ⚙️ Advanced Use: Combining Multiple Futures


```CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Java"); 
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "Rocks");  
CompletableFuture<String> combined = future1.thenCombine(future2, (a, b) -> a + " " + b); 
combined.thenAccept(System.out::println);  // Java Rocks`
```


---

### ⚠️ Exception Handling


```
`CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {    
if (true) throw new RuntimeException("Oops");     
	return "Success"; 
}).exceptionally(ex -> {     
	return "Fallback Value: " + ex.getMessage(); 
});  
System.out.println(future.join());  // Fallback Value: Oops`
```

| Question                                                     | Sample Answer                                                                                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **1. What is `CompletableFuture`?**                          | It’s a class that represents an asynchronous computation result. It lets to run asynchronously with callback chaining.                     |
| **2. How is it different from `Future`?**                    | `CompletableFuture` is non-blocking and supports chaining callbacks, unlike `Future` which requires blocking `.get()` to retrieve results. |
| **3. What are `supplyAsync()` and `runAsync()`?**            | `supplyAsync` returns a result; `runAsync` runs a void task.                                                                               |
| **4. How do you handle errors in `CompletableFuture`?**      | Using `exceptionally()`, `handle()`, or `whenComplete()` to define fallback logic.                                                         |
| **5. How do you combine two futures?**                       | Using `thenCombine()` or `thenCompose()` depending on dependency.                                                                          |
| **6. How to make multiple parallel calls and wait for all?** | Use `CompletableFuture.allOf(f1, f2, ...)`.                                                                                                |
| **7. Difference between `thenApply` and `thenCompose`?**     | `thenApply` transforms result; `thenCompose` flattens nested futures.                                                                      |

Daemon Threads -> Special type of threads that run in the background and performs the tasks like gc, cleanup or monitoring.

new Thread(() -> {}).setDemon(true)..
 
Note: -> It does not prevent the JVM from stoping since it is low priority background thread..

Mutex (Mutual Exclusion), Semaphore, Cyclic barrier, lock, latch

Lock ReentrantLock
  fairness --> Enable by passing true in the constructor
  new ReentrantLock(true); It executes the thread priority in the FIFO (Compare and swap)
  lock.lock()
  lock.unlock()
  lock.trylock() --> return false if it is locked. It is non blocking allows to check whether the resource is being locked by other thread.


Semaphore
	does the permit management allowing the limited number of threads to allow on the shared resource.
	example 10  connection for the database source...
	semaphore.acquire() , semaphore.release()
	 it is reusable
	
Cyclic Barrier
	Is used for all thread to wait at the barrier point ensure by the barrier.await() then it can do some batching task saving the multiple network call 
	It is reusable too . 


### Java Concurrency Primitives Comparison (with Reusability)

| Concept                    | Description                                                   | Key Methods                       | Thread Control                      | Fairness Support        | Reusable? |
| -------------------------- | ------------------------------------------------------------- | --------------------------------- | ----------------------------------- | ----------------------- | --------- |
| **Mutex**                  | Allows only one thread to access a section at a time          | `lock()`, `unlock()`              | 1 thread at a time                  | ❌ No                    | ✅ Yes     |
| **Semaphore**              | Allows limited threads to access shared resources (N permits) | `acquire()`, `release()`          | Up to N threads                     | ✅ Yes (via constructor) | ✅ Yes     |
| **Latch (CountDownLatch)** | Waits until a count reaches zero, then all threads proceed    | `countDown()`, `await()`          | All threads wait for count to hit 0 | ❌ No                    | ❌ No      |
| **CyclicBarrier**          | All threads wait at a barrier point, then proceed together    | `await()`                         | All threads wait for each other     | ❌ No                    | ✅ Yes     |
| **Lock (ReentrantLock)**   | Advanced locking with features like `tryLock()`, fairness     | `lock()`, `unlock()`, `tryLock()` | 1 thread, with better control       | ✅ Yes (via constructor) | ✅ Yes     |

Interview Based question 

### What is Thread Priority?

**Answer:**  
Thread priority in Java is a number between 1 and 10, with higher numbers indicating higher priority. The thread scheduler uses this priority to decide when each thread should run.

### What is the Use of `ThreadLocal`?

**Answer:**  
`ThreadLocal` provides thread-local variables. Each thread accessing such a variable has its own, independently initialized copy of the variable.

### What is the Difference Between `synchronized` and `Lock`?

**Answer:**

- **`synchronized`:** Acquires the lock automatically and releases it when the block is exited.
    
- **`Lock`:** Requires explicit lock and unlock calls, offering more flexibility, granularity. and also there is fairness policy. tryLock() to check the lock status preventing the thread from being blocked.
  
  Fork/Join --> uses the work stealing algorithm performs the divide and conquer and joins back the result. 
				ParallelStream() is one of the example of the fork/join framework.


How can you ensure the thread safety?
	 Avoiding the shared mutable state.
	(Mutual exclusion ) synchronization, reentrant lock.. for making the shared mutable state
	 use the concurrent, blocking (ArrayBlockingQueue, LinkedBlockingDeque) thread safe data structure                                   			 
	