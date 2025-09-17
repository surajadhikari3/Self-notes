Module that separates the cross cutting concerns like Logging, Authentication/Authorization / Transaction management from the   Business Logic..

It allows to define the separate logic and inject it to the service…As service layer holds the business logic..

Commonly used annotations 

@Aspect —>   Makes a class as an Aspect 

@Before , @After, @AfterReturning, @AfterThrowing, @Around —> Types of advice

@Pointcut. ——> Define the reusable point cuts…

- **JoinPoint** = Any events where the advice can be hooked. For example method call
- **Pointcut** = **Filter** that says "only calls related to _technical support_ are important for me."

  

✅ **JoinPoint** = **Actual happening**  
✅ **Pointcut** = **Selection criteria**

  
class UserService {

    public void createUser() {...}  // JoinPoint

    public void deleteUser() {...}  // JoinPoint

}

  
```
@Before("execution(* com.example.UserService.createUser(..))")

public void logBefore() {

    System.out.println("Logging before createUser");

}

```


createUser() method call is **JoinPoint**.

"execution(* com.example.UserService.createUser(..))" is **Pointcut**.



# Spring AOP vs AspectJ – What's the Difference?

| Feature            | **Spring AOP**                 | **AspectJ**                                       |
| ------------------ | ------------------------------ | ------------------------------------------------- |
| Weaving Type       | **Runtime** (Proxy-based)      | **Compile-time**, **Load-time**, or **Runtime**   |
| Scope              | **Method execution only**      | Method, Constructor, Field access, etc.           |
| Performance        | Slower (proxy overhead)        | Faster (compiled into bytecode)                   |
| Simplicity         | Very simple (uses annotations) | More complex (requires special compiler or agent) |
| Default in Spring? | ✅ Yes                          | ❌ No (must be explicitly configured)              |

  

**Types of Advice (Spring AOP)**

For interview 

AspectJ is a full AOP framework for Java that offers more powerful and flexible AOP capabilities than Spring AOP. Spring AOP uses AspectJ’s annotations and pointcut expressions but is limited to method-level join points using proxies. For more advanced AOP (like field access or constructor interception), we can integrate full AspectJ into Spring using the AspectJ Weaver.

|                     |                                          |                            |
| ------------------- | ---------------------------------------- | -------------------------- |
| **Type**            | **When it runs**                         | **Example**                |
| **Before**          | Before method executes                   | Validate user before order |
| **After Returning** | After method successfully returns        | Log success                |
| **After Throwing**  | After method throws an exception         | Log error                  |
| **After (Finally)** | After method (either success or failure) | Cleanup                    |
| **Around**          | Before and after method both             | Time method execution      |

  

Quick interview Questions…


|                                                        |                                                                                    |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| **Question**                                           | **Quick Answer**                                                                   |
| Difference between AOP and OOP?                        | AOP = modularizing cross-cutting concerns; OOP = modularizing data and behavior.   |
| Is Spring AOP runtime or compile-time weaving?         | **Spring AOP** = runtime weaving using **dynamic proxies**.                        |
| Which proxies does Spring AOP use?                     | JDK Dynamic Proxy (interface-based) OR CGLIB Proxy (subclass-based)                |
| Can we use AOP on private methods?                     | No, only **public** methods are advised by Spring AOP.                             |
| What is the difference between AspectJ and Spring AOP? | AspectJ = compile-time weaving. Spring AOP = proxy-based runtime weaving.          |
| Can AOP be used in REST Controllers?                   | Yes, but common practice is to use AOP for services, not directly for controllers. |

**Proxy in Spring AOP**

In Spring AOP, Proxy is an intermediate object created around the real bean. It intercepts method calls to inject additional behavior (like logging, transactions) without modifying the original business logic. Spring uses JDK Dynamic Proxy or CGLIB Proxy depending on whether the target bean implements an interface.


##### **Proxy Mechanism in Spring AOP**


- If the target object to be proxied implements **at least one interface**, a **JDK dynamic proxy** will be used. All interfaces implemented by the target type will be proxied.
    
- If the target object does **not implement any interfaces**, a **CGLIB proxy** will be created.Forcing the Use of CGLIB Proxying**
    

If you want to explicitly use CGLIB proxying, there are a few considerations to keep in mind:

1. **Final Methods Cannot Be Advised:**
    
    - Final methods cannot be overridden, so they cannot be proxied or advised using CGLIB.
2. **CGLIB Dependency:**
    
    - You will need the **CGLIB 2** binaries on your classpath, whereas JDK dynamic proxies are available out-of-the-box with the JDK.
3. **Spring Warnings:**
    
    - If Spring requires CGLIB and the **CGLIB library classes** are not found on the classpath, Spring will automatically issue a warning.



#### **Choosing which AOP declarations**

##### Spring AOP or Full AspectJ?

- If you only need to advise the execution of operations on **Spring beans**, then **Spring AOP** is the right choice.
- If you need to advise **objects not managed by the Spring container** (such as domain objects), then you will need to use **AspectJ**.
- Additionally, you should use **AspectJ** if you need to advise join points beyond simple method executions, such as field `get` or `set` join points.