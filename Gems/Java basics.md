
👉 “OOP in Java is based on four core principles: **Encapsulation, Inheritance, Polymorphism, and Abstraction**.

- **Encapsulation** means bundling fields and methods together and restricting direct access using access modifiers.
    
- **Inheritance** allows code reuse by letting a subclass acquire properties and behaviors of a parent class.
    
- **Polymorphism** provides flexibility: compile-time via method overloading and runtime via method overriding.
    
- **Abstraction** hides implementation details and exposes only the necessary behavior, implemented using abstract classes and interfaces in Java.  
    Together, these principles make code more modular, reusable, flexible, and maintainable.”


Contract Between equals() and hashCode()

- If two objects are equal (`a.equals(b) == true`), then `a.hashCode() == b.hashCode()` **must** hold.
    
- If two objects are not equal, their `hashCode()` may still be the same (collision is allowed, but should be minimized).
    
- Collections like `HashMap` and `HashSet` rely on **both** methods:
    
    - `hashCode()` decides the **bucket**.
        
    - `equals()` decides if two objects are **exact matches** inside the bucket.

---

# 🔹 Java Memory Model (JMM) Diagram

```
                 ┌────────────────────────────┐
                 │         Main Memory         │
                 │ (Shared heap, variables)    │
                 └───────────▲────────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
 ┌────────┴───────┐   ┌──────┴────────┐  ┌──────┴────────┐
 │ Thread A        │   │ Thread B       │  │ Thread C       │
 │ (Working Memory │   │ (Working Memory│  │ (Working Memory│
 │   / CPU cache)  │   │   / CPU cache) │  │   / CPU cache) │
 └───────▲─────────┘   └──────▲────────┘  └──────▲────────┘
         │                     │                  │
   Reads/Writes           Reads/Writes      Reads/Writes
 (may not sync unless     (may not sync     (may not sync
 volatile/sync used)       immediately)      immediately)

```

---

The **Java Memory Model (JMM)** defines how threads interact with memory, what changes are visible between threads, and how instruction reordering is handled
# 🔹 How to Explain It in Interview

👉 “In the Java Memory Model:

- Variables live in **main memory** (shared).
    
- Each thread works on a **local copy** in its own working memory (CPU cache).
    
- A thread’s updates may not be immediately visible to others, unless we use JMM rules.
    

The JMM defines **happens-before relationships**, ensuring visibility and ordering:

- A write to a `volatile` variable happens-before a subsequent read.
    
- Releasing a lock happens-before acquiring the same lock.
    
- Starting a thread happens-before its first action.
    

This prevents **visibility issues** and ensures consistency.”

---

# 🔹 Bonus Quick Example for Whiteboard

```
Thread A: flag = true;   // write (local cache)
Thread B: if(flag) ...   // may still see false!

✔ Fix: declare flag as volatile
```

---
**GC Algorithms:**

- **Mark and Sweep:** Mark live objects, sweep unused ones.
    
- **Copying:** Copy live objects to another space (used in Young Gen).
    
- **Compacting:** Rearrange objects to avoid fragmentation.


Copying is used to copy the live objects from eadon space to survior space . 
mark and sweep to mark the live objects and sweep unused objects. and used the compacting to prevent the fragmentation..............


# 🔹 Polished Interview Answer

👉 “`Comparable` is used to define the **natural ordering** of objects by implementing `compareTo()` inside the class itself. For example, an `Employee` class can implement `Comparable<Employee>` to sort by ID.

`Comparator`, on the other hand, is used to define **custom orderings** externally, using `compare(o1, o2)`. You can create multiple comparators, for example one to sort employees by name and another by salary.

In short: `Comparable` is for a single, natural order, while `Comparator` gives flexibility to define multiple different sorting strategies.”


👉 “In my projects, I’ve used several concurrent collections from `java.util.concurrent`.

- For thread-safe maps, I’ve used **ConcurrentHashMap**, which allows concurrent reads and writes and provides weakly consistent iterators.
    
- For read-heavy scenarios like configuration and event listeners, I’ve used **CopyOnWriteArrayList**.
    
- For producer-consumer pipelines, I’ve used **BlockingQueue** implementations like `LinkedBlockingQueue` and `ArrayBlockingQueue`.
    
- I’ve also worked with **ConcurrentLinkedQueue** for lock-free message passing.
    

These collections are **fail-safe**, meaning their iterators don’t throw `ConcurrentModificationException`; they either work on a snapshot (like CopyOnWriteArrayList) or provide weakly consistent iteration (like ConcurrentHashMap). In contrast, normal collections like `ArrayList` or `HashMap` are **fail-fast** — they throw an exception if modified while iterating.”

explain client side load-balancing/ service-discovery and server side load balancing/ service discovery

# 🔹 Polished Interview Answer

👉 “In **client-side load balancing**, the client queries the service registry to get a list of available service instances and then applies a load-balancing algorithm (like round-robin or random) to pick one. This avoids an extra hop but puts complexity on the client. Netflix Ribbon with Eureka in Spring Cloud is a classic example.

In **server-side load balancing**, the client just calls a single endpoint (like a load balancer or API Gateway). The load balancer queries the registry (or monitors health itself) and forwards the request to a service instance. This keeps clients simple but adds an extra hop. Examples include AWS ALB, Nginx, HAProxy, or Kubernetes Service.”

















----------------------------------------------------------
OLD


String immutability --> 

```
String s = "abc";
s.concat("def");   // New string is created, but not stored
System.out.println(s);  // Still prints "abc"
```

### Why?

- **Immutability** means the **original object (`"abc"`) cannot be changed**.
    
- So when you call `s.concat("def")`, it creates a **new string object `"abcdef"`**, but it **doesn't modify `s`**.
    
- Since you didn’t assign the result of `.concat()` to a variable, the new string is **lost** (eligible for garbage collection

### **Exception Chaining in Java**

**Exception chaining** is a technique where one exception (often a high-level or custom exception) is caused by another (a lower-level exception). This helps preserve the **original cause** of an error while adding **context** at a higher level in your application.

For this while defining the custom exception class in the constructore we can pass the Throwable cause which preserves the low level exception..


```
class DataAccessException extends Exception {
    public DataAccessException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

```
Here we are getting the both high level and low level exceptions..

DataAccessException: Unable to read from DB
    at ...
Caused by: java.lang.NullPointerException: Database connection is null
    at ...

`````

Closeable vs auto closable....

| Feature                          | `Closeable`                                           | `AutoCloseable`                                        |
| -------------------------------- | ----------------------------------------------------- | ------------------------------------------------------ |
| **Package**                      | `java.io`                                             | `java.lang`                                            |
| **Introduced In**                | Java 5                                                | Java 7                                                 |
| **Method**                       | `void close() throws IOException`                     | `void close() throws Exception`                        |
| **Checked Exception**            | Only `IOException` can be thrown                      | Any `Exception` can be thrown                          |
| **Typical Use Case**             | IO streams (e.g. `FileInputStream`, `BufferedReader`) | General resources (JDBC connections, etc.)             |
| **Supports try-with-resources?** | ✅ Yes                                                 | ✅ Yes                                                  |
| **Can be used for**              | File operations, network streams                      | Anything that needs cleanup (e.g., custom classes, DB) |
| **Example Classes**              | `FileReader`, `BufferedWriter`, `InputStream`         | `Connection`, `Statement`, `ReentrantLock` (custom)    |

Closeable interface extends AutoCloasable interface so all. the Closeable supports the AutoCloaseable

Marker interface 

A **marker interface** is an interface that have empty body. It is used to "mark" a class to convey metadata to the JVM or frameworks about how the class should be treated.

**Examples:**

- `java.io.Serializable` – Marks a class as serializable.
    
- `java.lang.Cloneable` – Marks a class as cloneable (can use `Object.clone()`).
    
- `java.util.RandomAccess` – Marks that a list supports fast (constant time) random access.


Serialization

Serialization is the process of converting the objects into the byte stream so that it can be saved in file or sent over the network.

```
import java.io.*;

class Student implements Serializable {
    int id;
    String name;

    public Student(int id, String name) {
        this.id = id;
        this.name = name;
    }
}

```

### **Fail-Fast vs Fail-Safe vs Thread-Safe**

| Feature           | **Fail-Fast**                                | **Fail-Safe**                                      | **Thread-Safe**                              |
| ----------------- | -------------------------------------------- | -------------------------------------------------- | -------------------------------------------- |
| Definition        | Fails immediately on concurrent modification | Safe iteration even during concurrent modification | Safe to access/modify from multiple threads  |
| Example Classes   | `ArrayList`, `HashMap`, `HashSet`            | `CopyOnWriteArrayList`, `ConcurrentHashMap`        | `Vector`, `Hashtable`, synchronized wrappers |
| Throws Exception? | Yes – `ConcurrentModificationException`      | No                                                 | No                                           |
| Performance       | Fast but risky in multi-threaded context     | Slower due to copying mechanism                    | Slower due to synchronization                |

Remeber --> ConcurrentHashMap, CopyOnWriteArrayList are fail-safe and rest are fail-fast

Collections Evaluation Matrix

| Collection Type              | Data Access (LIFO/FIFO/Key-Value) | Maintains Order | Sorted                              | Allows Nulls                                                                                                        | Duplicates Allowed | Thread Safe | Bidirectional Access     | Fail-Fast / Fail-Safe |
| ---------------------------- | --------------------------------- | --------------- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ------------------ | ----------- | ------------------------ | --------------------- |
| **ArrayList**                | Index-based (List)                | ✅ Yes           | ❌ No                                | ✅ Yes                                                                                                               | ✅ Yes              | ❌ No        | ✅ Yes (via ListIterator) | ✅ Fail-Fast           |
| **LinkedList**               | FIFO / Deque                      | ✅ Yes           | ❌ No                                | ✅ Yes                                                                                                               | ✅ Yes              | ❌ No        | ✅ Yes                    | ✅ Fail-Fast           |
| **HashSet**                  | Hash-based                        | ❌ No            | ❌ No                                | ✅ One null                                                                                                          | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **LinkedHashSet**            | Hash-based                        | ✅ Yes           | ❌ No                                | ✅ One null                                                                                                          | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **TreeSet**                  | Sorted                            | ❌ No            | ✅ Yes                               | ❌ Null not allowed                                                                                                  | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **HashMap**                  | Key-Value                         | ❌ No            | ❌ No                                | ✅ One null key, many null values                                                                                    | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **LinkedHashMap**            | Key-Value                         | ✅ Yes           | ❌ No                                | ✅ One null key and multiple null value--> as the key should be unique and doesn't have the constraints  on value... | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **TreeMap**                  | Key-Value (Sorted by keys)        | ❌ No            | ✅ Yes                               | ❌ Null key not allowed                                                                                              | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **ConcurrentHashMap**        | Key-Value                         | ❌ No            | ❌ No                                | ❌ Null not allowed                                                                                                  | ❌ No               | ✅ Yes       | ❌ No                     | ✅ Fail-Safe           |
| **CopyOnWriteArrayList**     | Index-based                       | ✅ Yes           | ❌ No                                | ✅ Yes                                                                                                               | ✅ Yes              | ✅ Yes       | ✅ Yes                    | ✅ Fail-Safe           |
| **Vector**                   | Index-based (Legacy)              | ✅ Yes           | ❌ No                                | ✅ Yes                                                                                                               | ✅ Yes              | ✅ Yes       | ✅ Yes                    | ✅ Fail-Fast           |
| **Stack** (extends Vector)   | LIFO                              | ✅ Yes           | ❌ No                                | ✅ Yes                                                                                                               | ✅ Yes              | ✅ Yes       | ✅ Yes                    | ✅ Fail-Fast           |
| **PriorityQueue**            | Queue (Heap)                      | ❌ No            | ✅ Yes (natural order or comparator) | ❌ No                                                                                                                | ✅ Yes              | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **Deque (e.g., ArrayDeque)** | LIFO / FIFO                       | ✅ Yes           | ❌ No                                | ✅ Yes                                                                                                               | ✅ Yes              | ❌ No        | ✅ Yes                    | ✅ Fail-Fast           |
Remember --> Tree based collection are sorted and doesn't allow null value as null value doesn't have natural ordering and cannot maintain the order if null is allowed...........................

Common path

$JAVA_HOME --> it gives the path that is set in the shell for me ~./zshrc --> which is /usr/libexec/java_home

--> To set the $JAVA_HOME in the shell add the export = $(/usr/libexec/java_home) --> This actually returns the JDK location -->/Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home



Flow usal  --> /Library/Java/JavaVirtualMachines/jdk-17.0.1/Contents/Home it can be pointed with the /usr/libexec/java_home which we keep in the global variable called JAVA_VERSION which is kept in the zshrc or the prefered terminal you used...

```
	$JAVA_HOME --> This is the shell based variable pointing to the next high level variable... ß
		|
	/usr/libexec/java_home --> Point to the root
		|
	/Library/Java/JavaVirtualMachine/jdk-17.0.1/Contents/Home --> Root level is this
```


Can we instantiate the interface --> No

In Java, **you cannot instantiate an interface directly** because an interface is an abstract type that defines a contract but provides no implementation (unless using default or static methods, which still don't make the interface instantiable on its own).


Important questions.............
### **Can You Use `finally` with `try-with-resources`?**

**Yes**, but it's rarely needed.

**Example:**

```
try (MyResource r = new MyResource()) {
    r.use();
} catch (Exception e) {
    e.printStackTrace();
} finally {
    System.out.println("Finally block executed.");
}
```

🧠 **Best Practice:** Use `finally` only for **non-resource cleanup** — like logging, counter reset, UI update, etc.  
➡️ Resources are already closed **before** `finally` runs.

For using in the try-with-resource it has to be auto-closeable
note All closeable is auto-cloaseable  as Cloaseable extends AutoCloseable

