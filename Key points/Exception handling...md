Throwable class-- Under throwable there is exception and the error 

Exception --> Is recovarable. Checked and Unchecked exceptions.
Error --> Is not recoverable (OutOfMemoryError, StackOverflowError)

## **Exception Concepts in Java**

|Concept|Description|
|---|---|
|**Throwable**|Superclass of all errors and exceptions in Java|
|**Error**|Serious issues beyond application control (e.g., `OutOfMemoryError`)|
|**Exception**|Recoverable issues — business/application logic can handle|
|**Checked Exception**|Must be handled with `try-catch` or declared using `throws`|
|**Unchecked Exception**|Runtime exceptions — typically result from programming bugs|

Java Exception Hierarchy Diagram

                      ┌───────────────┐
                      │  Throwable   │
                      └──────┬────────┘
                             │
            ┌──────────────────────────────┐
            │                              │
        ┌───────┐                    ┌────────┐
        │ Error │                    │Exception│
        └───────┘                    └────┬─────┘
                                         │
            ┌─────────────────────────────────────────────┐
            │                                             │
    ┌──────────────┐                         ┌────────────────────┐
    │ Checked(compile)                       │ Unchecked (Runtime)│
    │ Exception     │                        │Exception           │
    └──────────────┘                         └────────────────────┘
		    │                                             │
┌────────────┐                        ┌────────────────────────┐
		│IOException │                                                          │NullPointerException     │
		│SQLException│                                                   │ArrayIndexOutOfBounds    │
		│ParseException│                                                  │IllegalArgumentException │
└────────────┘                        └────────────────────────┘

### Checked Exception Example (Compile-Time)

```
import java.io.*;

public class CheckedDemo {
    public static void main(String[] args) throws IOException {
        BufferedReader reader = new BufferedReader(new FileReader("file.txt"));
        System.out.println(reader.readLine());
        reader.close();
    }
}

```

**`FileNotFoundException`** and **`IOException`** must be handled or declared.

Unchecked Exception (Runtime) 
It's developers issue so it must be fix not handled....

```
public class UncheckedDemo {
    public static void main(String[] args) {
        String text = null;
        System.out.println(text.length());  // NullPointerException
    }
}

```

### What is the difference between Checked and Unchecked Exception?


- Checked: Checked at **compile-time**,  Should be handled e.g., `IOException, SQLException`.
    
- Unchecked: Occur at **runtime**, It's developers mistake so should be fixed not handled e.g., `NullPointerException, IndexOutOfException`.

### Why are RuntimeExceptions unchecked?

**A:** Because they usually represent **programming errors** that should be fixed in code (e.g., null checks), not caught in `try-catch`.

### is it good to catch `RuntimeException`?

Generally no, unless you want to log/report unexpected bugs without crashing the app.


Errors -->  subclass of Throwable that indicates serious problems a Java appliccation which can not be handled within the program.

They are:

- **Not intended to be caught**
    
- **Unchecked**
    
- Related to **system failures** or **JVM-level issues**
    

**🔍 Example:** `OutOfMemoryError`, `StackOverflowError`, `NoClassDefFoundError`

## **Types of Errors in Java (With Examples)**
The issues that cannot be handled within the application is called error

| Error Type                    | Description                                                  | Example Scenario                                   |
| ----------------------------- | ------------------------------------------------------------ | -------------------------------------------------- |
| `OutOfMemoryError`            | JVM ran out of memory (heap or metaspace)                    | Creating millions of objects, Creating the threads |
| `StackOverflowError`          | Too deep recursion exceeded the call stack                   | Recursive call with no base case                   |
| `NoClassDefFoundError`        | Class was present during compile-time but missing at runtime | Library JAR not in classpath                       |
| `UnsatisfiedLinkError`        | Native library (JNI) required but not found                  | Missing `.dll` or `.so` file                       |
| `ExceptionInInitializerError` | Exception thrown during static initialization                | Static block throws unchecked exception            |
| `InternalError`               | Unexpected JVM-level issue                                   | JVM bug or corrupted native code                   |
| `VirtualMachineError`         | Base class for severe JVM errors                             | `OutOfMemoryError`, `StackOverflowError`, etc.     |

### `OutOfMemoryError`

```
public class OOMExample {
    public static void main(String[] args) {
        int[] arr = new int[Integer.MAX_VALUE];  // Huge array → Heap crash
    }
}

```

StackOverFlowError

```
public class StackOverflowDemo {
    public static void recurse() {
        recurse();  // No termination
    }
    public static void main(String[] args) {
        recurse();
    }
}

```

| Question                                                                     | Sample Answer                                                                                                                                                                                                                                                                                |
| ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 🔹 **Q1: What is the difference between Exception and Error in Java?**       | Exception: Application-level issues that can be handled (like `IOException`, `SQLException`) Error: JVM-level issues that should not be caught (like `OutOfMemoryError`, `StackOverflowError`)                                                                                               |
| 🔹 **Q2: Have you ever encountered an `OutOfMemoryError`? What did you do?** | Yes, it occurs when heap/memory is insufficient. We analyzed memory leaks with tools like VisualVM/YourKit and optimized data structures or added GC tuning parameters.                                                                                                                      |
| 🔹 **Q3: Can you catch Errors? Should you?**                                 | Technically you can (via `catch(Error e)`), but it's **strongly discouraged**. Errors indicate conditions that the application cannot reasonably recover from.                                                                                                                               |
| 🔹 **Q4: What is `NoClassDefFoundError` vs `ClassNotFoundException`?**       | `NoClassDefFoundError`: Class was available during compile-time but missing at runtime. <br>`ClassNotFoundException`: Checked exception when class is **dynamically** loaded using `Class.forName()` and not found. and the path you pass is not valid that means the class does not exist.. |
| 🔹 **Q5: How to debug `StackOverflowError`?**                                | Use stack trace analysis tools or JVM options like `-XX:+ShowCodeDetailsInExceptionMessages` and optimize recursion logic.                                                                                                                                                                   |
|                                                                              |                                                                                                                                                                                                                                                                                              |


### **Enable Detailed Stack Trace**

Use the following JVM option for more descriptive error output:

```
java -XX:+ShowCodeDetailsInExceptionMessages -cp . StackOverflowDemo
```

If you're using **Java 14+**, this option may already be enabled. It enhances messages for exceptions like `NullPointerException`, and in newer JVMs, it may help with `StackOverflowError` too.

### What's the difference between `ClassNotFoundException` and `NoClassDefFoundError`?

| Aspect          | `ClassNotFoundException`                   | `NoClassDefFoundError`                        |
| --------------- | ------------------------------------------ | --------------------------------------------- |
| Type            | Checked Exception                          | Unchecked Error                               |
| Occurs when?    | Dynamically loading a class via reflection | Class missing during runtime loading          |
| Common scenario | `Class.forName("com.XYZ")`                 | Referencing class in code, but missing .class |
