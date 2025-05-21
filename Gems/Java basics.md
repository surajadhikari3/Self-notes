
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

| Collection Type              | Data Access (LIFO/FIFO/Key-Value) | Maintains Order | Sorted                              | Allows Nulls                     | Duplicates Allowed | Thread Safe | Bidirectional Access     | Fail-Fast / Fail-Safe |
| ---------------------------- | --------------------------------- | --------------- | ----------------------------------- | -------------------------------- | ------------------ | ----------- | ------------------------ | --------------------- |
| **ArrayList**                | Index-based (List)                | ✅ Yes           | ❌ No                                | ✅ Yes                            | ✅ Yes              | ❌ No        | ✅ Yes (via ListIterator) | ✅ Fail-Fast           |
| **LinkedList**               | FIFO / Deque                      | ✅ Yes           | ❌ No                                | ✅ Yes                            | ✅ Yes              | ❌ No        | ✅ Yes                    | ✅ Fail-Fast           |
| **HashSet**                  | Hash-based                        | ❌ No            | ❌ No                                | ✅ One null                       | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **LinkedHashSet**            | Hash-based                        | ✅ Yes           | ❌ No                                | ✅ One null                       | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **TreeSet**                  | Sorted                            | ❌ No            | ✅ Yes                               | ❌ Null not allowed               | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **HashMap**                  | Key-Value                         | ❌ No            | ❌ No                                | ✅ One null key, many null values | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **LinkedHashMap**            | Key-Value                         | ✅ Yes           | ❌ No                                | ✅ Yes                            | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **TreeMap**                  | Key-Value (Sorted by keys)        | ❌ No            | ✅ Yes                               | ❌ Null key not allowed           | ❌ No               | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **ConcurrentHashMap**        | Key-Value                         | ❌ No            | ❌ No                                | ❌ Null not allowed               | ❌ No               | ✅ Yes       | ❌ No                     | ✅ Fail-Safe           |
| **CopyOnWriteArrayList**     | Index-based                       | ✅ Yes           | ❌ No                                | ✅ Yes                            | ✅ Yes              | ✅ Yes       | ✅ Yes                    | ✅ Fail-Safe           |
| **Vector**                   | Index-based (Legacy)              | ✅ Yes           | ❌ No                                | ✅ Yes                            | ✅ Yes              | ✅ Yes       | ✅ Yes                    | ✅ Fail-Fast           |
| **Stack** (extends Vector)   | LIFO                              | ✅ Yes           | ❌ No                                | ✅ Yes                            | ✅ Yes              | ✅ Yes       | ✅ Yes                    | ✅ Fail-Fast           |
| **PriorityQueue**            | Queue (Heap)                      | ❌ No            | ✅ Yes (natural order or comparator) | ❌ No                             | ✅ Yes              | ❌ No        | ❌ No                     | ✅ Fail-Fast           |
| **Deque (e.g., ArrayDeque)** | LIFO / FIFO                       | ✅ Yes           | ❌ No                                | ✅ Yes                            | ✅ Yes              | ❌ No        | ✅ Yes                    | ✅ Fail-Fast           |


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