We only have to provide the spring-context in pom.xml as it has the transitive dependency in the spring-core and spring-bean..

# List of Core Spring Modules (organized Layer-wise)

| Layer                 | Modules                                                                                                                                                                 | Purpose                                  |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| **Core Layer**        | Core, Beans, Context, Expression Language (SpEL)<br><br>-> Now we only have to provide the Context in pom and it<br>pulls the core and bean as transitive dependencies. | Base functionalities                     |
| **Data Access Layer** | JDBC, ORM, Transaction, DAO                                                                                                                                             | Database operations                      |
| **Web Layer**         | Web, Web MVC, Web Websocket, WebFlux for reactive (non blocking web app )                                                                                               | Web application support                  |
| **AOP Layer**         | AOP, Aspects, Instrumentation                                                                                                                                           | Aspect-Oriented Programming              |
| **Messaging Layer**   | Messaging, Websocket                                                                                                                                                    | Asynchronous message communication       |
| **Testing Layer**     | Test                                                                                                                                                                    | Support for unit and integration testing |


### Core Container Modules:

| Module                                | Purpose                                                                                                                    |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Spring Core**                       | Provides basic features like dependency injection (DI) and IoC container.                                                  |
| **Spring Beans**                      | Manages configuration and lifecycle of application objects (beans).                                                        |
| **Spring Context**                    | Provides a way to access beans (ApplicationContext).                                                                       |
| **Spring Expression Language (SpEL)** | Powerful language for querying and manipulating object graphs at runtime (e.g., `#{user.name}` inside XML or Annotations). |

---

### 💾 Data Access/Integration Modules:

| Module                            | Purpose                                                  |
| --------------------------------- | -------------------------------------------------------- |
| **Spring JDBC**                   | Simplifies working with relational databases via JDBC.   |
| **Spring ORM**                    | Integrates with ORM frameworks like Hibernate, JPA, etc. |
| **Spring Transaction Management** | Ensures consistency in transactions (commit, rollback).  |
| **Spring DAO**                    | Abstraction layer for data access technologies.          |

---

### 🌐 Web Layer Modules:

| Module                   | Purpose                                                                          |
| ------------------------ | -------------------------------------------------------------------------------- |
| **Spring Web**           | Provides basic web-oriented features (multipart file upload, etc.).              |
| **Spring Web MVC**       | Implements Model-View-Controller (MVC) pattern for web apps (DispatcherServlet). |
| **Spring Web WebSocket** | Supports real-time communication over WebSocket protocol.                        |

---

### 🧩 AOP and Instrumentation Modules:

| Module                     | Purpose                                                                 |
| -------------------------- | ----------------------------------------------------------------------- |
| **Spring AOP**             | Aspect-Oriented Programming features (logging, security, transactions). |
| **Spring Aspects**         | Integration with AspectJ for advanced AOP features.                     |
| **Spring Instrumentation** | Class instrumentation support for servers and JPA enhancement.          |

---

### 💬 Messaging and Event-Driven Modules:

|Module|Purpose|
|---|---|
|**Spring Messaging**|Provides messaging support for STOMP, WebSocket, etc.|
|**Spring WebSocket**|Full-duplex communication between client and server.|

---

### 🧪 Testing Module:

|Module|Purpose|
|---|---|
|**Spring Test**|Provides utilities for unit testing Spring components (mock beans, test contexts).|


---

# 5. 🧠 Easy Way to Remember

Use this mnemonic:

> "**Core Data Web AOP Messaging Test**"

- Core → Core, Beans, Context, SpEL
    
- Data → JDBC, ORM, Transactions
    
- Web → Web, Web MVC, WebSocket
    
- AOP → AOP and Aspects
    
- Messaging → Messaging module
    
- Test → Testing module
    

✅ **Bottom to top** thinking = First Core (base), then Data access, then Web, then Cross-cutting concerns like AOP and Messaging, then finally Testing!

---

# 6. 🎯 Important Interview Questions (and Quick Answers)

| Question                            | Quick Answer                                                                  |
| ----------------------------------- | ----------------------------------------------------------------------------- |
| What is Spring Core?                | Provides basic Dependency Injection, IoC container features.                  |
| What is Spring ORM?                 | Provides integration with ORM frameworks like Hibernate.                      |
| What is Spring AOP used for?        | Cross-cutting concerns like logging, transactions, security.                  |
| What is the role of Spring Web MVC? | Implements MVC pattern using DispatcherServlet.                               |
| How does Spring support testing?    | With `spring-test` module that provides integration and unit testing support. |

## **What is Dependency Injection (DI)? How does Spring implement it?**

**Answer:**  
DI is a design pattern that allows objects to receive their dependencies from an external source (like Spring) instead of creating them internally.

### Types of DI:

- **Constructor Injection**. --> Use the constructor injection for the required dependencies
    
- **Setter Injection** ---> Use the setter injection for the optional ones
    
- **Field Injection (@Autowired)** --> Avoid  the field injection in the business logic or unit tests..

@Autowired is required only for the field dependency injection and after the spring 4.3+ it is optional in the constructor too.



```   
@Autowired  // optional in Spring 4.3+ if single constructor
    public Car(Engine engine) {
        this.engine = engine;
    }
```


**Example:**  
If a car needs an engine, instead of creating it inside the `Car` class, Spring injects an `Engine` into `Car`.

```java
@Component 
public class Engine {} 

@Component
public class Car { 
private final Engine engine;  

@Autowired  
public Car(Engine engine) {   
this.engine = engine;   }
}`

**Diagram:**
[Spring Container]
       |
[Engine Bean] ---> injected into ---> [Car Bean]

```


## **What are Spring Beans?**

Beans are objects managed by the Spring IoC container.


`@Component 
public class MyService {}`

Beans are defined using:

- `@Component`, `@Service`, `@Repository`, `@Controller`
    
- XML configuration

In **Spring Framework**, you can create objects (beans) in different scopes using various **annotations**, and each scope controls **how and when a new instance of the object is created**.

| Scope         | Description                       | When New Object is Created       | Use Case Example                 |
| ------------- | --------------------------------- | -------------------------------- | -------------------------------- |
| `singleton`   | Default, one per Spring container | Only once                        | Shared services                  |
| `prototype`   | New object every time             | Every time `getBean()` is called | Multi-instance workers           |
| `request`     | One per HTTP request              | Every HTTP request               | Request-scoped user data         |
| `session`     | One per HTTP session              | One per user session             | User preferences                 |
| `application` | One per ServletContext            | Once per web application         | Shared caches/configs            |
| `websocket`   | One per WebSocket session         | Per WebSocket client             | Real-time communication features |

## **What are the common Spring annotations?**

| Annotation    | Purpose                              |
| ------------- | ------------------------------------ |
| `@Component`  | Generic bean                         |
| `@Service`    | Service layer                        |
| `@Repository` | DAO layer with exception translation |
| `@Controller` | MVC controller                       |
| `@Autowired`  | Dependency injection                 |
| `@Qualifier`  | Resolve multiple bean conflicts      |
| `@Value`      | Inject property values               |

## **What is the Spring Bean Lifecycle?**


`Instantiation → Populate properties → BeanNameAware → BeanFactoryAware →  @PostConstruct → afterPropertiesSet() → Bean is ready → @PreDestroy → destroy()`

#Imp ---Use `@PostConstruct` and `@PreDestroy` to hook into lifecycle.

## **What is ApplicationContext? How is it different from BeanFactory?**

| Feature              | BeanFactory | ApplicationContext |
| -------------------- | ----------- | ------------------ |
| Lazy initialization  | Yes         | No (by default)    |
| AOP Support          | No          | Yes                |
| Internationalization | No          | Yes                |

**ApplicationContext** is the preferred container.

## **What is Aspect-Oriented Programming (AOP)?**

**Answer:**  
AOP separates cross-cutting concerns (like logging, security, transactions) from business logic.

### Key Concepts:

- **Aspect**: Class with cross-cutting concern
    
- **Advice**: Code to run (before, after, etc.)
    
- **JoinPoint**: Where to apply
    
- **Pointcut**: Expression to select joinpoints
    

``@Aspect 
@Component
public class LoggingAspect { 

@Before("execution(* com.example.service.*.*(..))")  
public void logBefore() {   
	System.out.println("Method execution started");   
		} 
	}`


## **How does Spring handle transactions?**

**Answer:**  
Spring provides declarative transaction management using `@Transactional`.


```
@Service 
public class PaymentService { 

@Transactional 
public void processPayment() { 
// db operations   
} 
}
```

**Propagation Levels**:

- REQUIRED, REQUIRES_NEW, NESTED, SUPPORTS, MANDATORY, NOT_SUPPORTED, NEVER


## **What are Profiles in Spring?**

**Answer:**  
Profiles allow loading beans conditionally based on the environment (dev, test, prod).


```
@Profile("dev") 
@Bean 
public DataSource devDataSource() {}
```


Set with: `-Dspring.profiles.active=dev`


## **What is Spring’s @Configuration and @Bean?**


```
@Configuration 
public class AppConfig {

@Bean  
public MyService myService() {   
return new MyService();  
} }`

**@Configuration**: Marks class as source of bean definitions  
**@Bean**: Declares a bean
```


## **What is Spring MVC Architecture?**

**Answer:**  
Model-View-Controller pattern for web apps.

### Flow Diagram:


`Client -> 
DispatcherServlet ->
HandlerMapping        -> 
Controller ->
Service -> Repository -> 
ModelAndView -> 
ViewResolver ->
View (HTML/JSON)`


## **How does Spring handle exceptions globally?**


```
@ControllerAdvice 
public class GlobalExceptionHandler {

@ExceptionHandler(Exception.class) 
public ResponseEntity<String> handleAll() { 
	return ResponseEntity.status(500).body("Error occurred"); 

	} 
}
```


## **What is the difference between Singleton and Prototype scope?**

| Scope     | Description                   |
| --------- | ----------------------------- |
| Singleton | One shared instance (default) |
| Prototype | New instance every time       |


`@Scope("prototype") @Component public class MyPrototypeBean {}`

## Spring Bean Lifecycle: Step-by-Step

Sequence of steps by which the spring create, configure, manage and destroy the bean

| Step # | Stage                                                                                  | Description                                                                             |
| ------ | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| 1️⃣    | **Instantiation**                                                                      | Bean is created (via constructor)(Uses reflection by spring)                            |
| 2️⃣    | **Populate Properties**                                                                | Dependencies are injected (DI)                                                          |
| 3️⃣    | Aware Interfaces like **BeanNameAware, BeanFactoryAware, <br>ApplicationContextAware** | Spring calls special "aware" methods<br>to provide. bean specific context info          |
| 4️⃣    | **Post-Processing (@PostConstruct,  afterPropertiesSet())**                            | Custom initialization logic runs(Like loading configuration or <br>creating connection) |
| 5️⃣    | **Ready to Use**                                                                       | Bean is fully initialized and in use                                                    |
| 6️⃣    | **@PreDestroy / destroy()**                                                            | Bean is cleaned up before shutdown                                                      |

|Lifecycle Phase|Trigger Method|
|---|---|
|Instantiation|Constructor|
|Property Injection|`@Autowired`, `setX()`|
|Aware Phase|`setBeanName()`, `setBeanFactory()`|
|Initialization|`@PostConstruct`, `afterPropertiesSet()`|
|Destruction|`@PreDestroy`, `destroy()`|
Example for the postconstruct..

```
@Service
public class CountryCache {

    private final CountryRepository countryRepository;
    private List<String> countries;

    public CountryCache(CountryRepository countryRepository) {
        this.countryRepository = countryRepository;
    }

    @PostConstruct
    public void loadCountriesIntoCache() {
        this.countries = countryRepository.findAllCountryNames();
        System.out.println("Loaded countries into cache: " + countries);
    }

    public List<String> getCountries() {
        return countries;
    }
}

```

### Benefits:

- Loaded **once at startup**
    
- No repeated DB calls
    
- Easy to test and maintain
## 
Rules for writing the @PostConstruct..

|Rule|Detail|
|---|---|
|Must be `void` method|No return values|
|No arguments allowed|Should take 0 parameters|
|Called **once per bean**|Right after bean is initialized|
|Works on **Spring beans only**|Managed by container with `@Component`, `@Service`, etc.|

## `@Primary` Annotation

> Tells Spring: **“Use this bean by default”** when multiple candidates exist.

### 🔧 Example:


`@Component 
@Primary
public class MySQLDatabase implements Database { 
public String connect() {   
return "Connected to MySQL";   
} }

@Component 
public class PostgreSQLDatabase implements Database { 
public String connect() {   
return "Connected to PostgreSQL";    
} } 

@Component 
public class DataService {  
@Autowired   
private Database database;    
public void showConnection() {   
System.out.println(database.connect());   
} }`

## 2. `@Qualifier` Annotation

> Tells Spring: **“Use this specific bean”** explicitly by name.

### 🔧 Example (continued from above):

java

CopyEdit

`@Component public class DataService {  
@Autowired    
@Qualifier("postgreSQLDatabase") 
private Database database;    
public void showConnection() { 
System.out.println(database.connect());
} }`

### ✅ Output:

`Connected to PostgreSQL`

> `@Qualifier` overrides the `@Primary` preference when explicitly used.

---

## ⚖️ Comparison Table

|Feature|`@Primary`|`@Qualifier`|
|---|---|---|
|Purpose|Default bean for a type|Explicitly choose one among many|
|Scope|Class-level|Field/parameter level|
|Override|Can be overridden by `@Qualifier`|Overrides `@Primary`|
|Use Case|You want a default bean unless specified|You want full control over which bean|

---

## 🧠 Analogy:

- `@Primary` is like the default coffee in a vending machine.
    
- `@Qualifier` is like pressing the exact button for espresso even if the default is cappuccino.
    

---