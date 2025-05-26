Spring boot over spring

	1. Version Mangagement
	2. Spring boot actuator --> For monitoring, threaddump and so on 
	3. Embedded server(Tomcat, jetty)
	4. Slice based testing (@MockMVC, )
	5. Auto Configuration --> automatically configures bean based on the classpath and application properties.
							If `spring-boot-starter-web` is added, Spring Boot configures an embedded Tomcat server,                                                                    a DispatcherServlet, and JSON support automatically.
							Can be customized using:
						    Exclusion: `@EnableAutoConfiguration(exclude ={DataSourceAutoConfiguration.class})`
		

Starter Dependencies

| Starter Name                   | Purpose                                  |
| ------------------------------ | ---------------------------------------- |
| `spring-boot-starter-web`      | Spring MVC, Jackson, and embedded Tomcat |
| `spring-boot-starter-data-jpa` | Spring Data JPA + Hibernate              |
| `spring-boot-starter-security` | Spring Security                          |
| `spring-boot-starter-test`     | JUnit, Mockito, Spring Test support      |


REST (Representational State Transfer) --> Architectural style for designing web services for synchronous communication using HTTP protocol.

HTTP Status Code

- `200 OK`: Successful request.
- `201 Created`: Resource created successfully.
- `400 Bad Request`: Client sent an invalid request.
- `401 -> UnAuthenticated.
- 403 --> Unauthorized
- `404 Not Found`: Resource not found.
- `500 Internal Server Error`: Server-side issue.


Conversion concepts

| Concern              | Preferred Tool      | Alternatives    |
| -------------------- | ------------------- | --------------- |
| ORM                  | Hibernate (JPA)     | MyBatis, JOOQ   |
| JSON Serialization   | Jackson             | Gson, JSON-B    |
| Dependency Injection | Spring IoC          | —               |
| Reactive Programming | Spring WebFlux      | RxJava          |
| Security             | Spring Security     | Apache Shiro    |
| HTML Templating      | Thymeleaf           | FreeMarker, JSP |
| Config Management    | Spring Boot Config  | —               |
| API Documentation    | SpringDoc OpenAPI   | Swagger         |
| Testing              | JUnit + Spring Test | TestNG          |

### . **@Query for Custom Queries** in the Repository interface where we extends the JPA  repoository..

If the query cannot be derived or you need a custom one, you can use the `@Query` annotation:

```java
@Query("SELECT u FROM User u WHERE u.age > :age")
List<User> findUsersOlderThan(@Param("age") int age);
```

You can also use native SQL queries:

```java
@Query(value = "SELECT * FROM users WHERE age > :age", nativeQuery = true)
List<User> findUsersOlderThan(@Param("age") int age);
```

### **Paging and Sorting**


For pageable and sortable queries, Spring generates the appropriate SQL using the `Pageable` and `Sort` parameters. For example:

```java
Page<User> findByLastName(String lastName, Pageable pageable);
```

This translates to a paginated query like:

```sql
SELECT * FROM users WHERE last_name = ? LIMIT ? OFFSET ?
```

### Advantages of Spring Data JPA in CRUD and Finder Methods:

1. **Reduced Boilerplate Code**: You don’t need to write repetitive DAO implementations.(Provides CRUD, Pagination, sorting)
2. **Custom Queries**: Easily extend functionality with `@Query`.
3. **Seamless Integration**: Works directly with JPA providers like Hibernate.
4. **Transaction Management**: Automatically managed transactions simplify application logic.
5. **Dynamic Proxies**: Enables runtime flexibility and faster development cycles.

By following these conventions, Spring Boot allows developers to focus on business logic while handling the database layer efficiently.

Global Exception Handler --> For the class level we use the @RestControllerAdvice (For rest controller) or @ControllerAdvice(For non-rest controller)  and method level @ExceptionHandler...

```
@RestControllerAdvice   // Use @ControllerAdvice for non-REST controllers
public class GlobalExceptionHandler {

   @RestControllerAdvice // Use @ControllerAdvice for non-REST controllers
public class GlobalExceptionHandler {

    // Handle a specific exception (e.g., ResourceNotFoundException)
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleResourceNotFoundException(
            ResourceNotFoundException ex, WebRequest request) {
        Map<String, Object> body = new HashMap<>();
        body.put("timestamp", LocalDateTime.now());
        body.put("status", HttpStatus.NOT_FOUND.value());
        body.put("error", "Not Found");
        body.put("message", ex.getMessage());
        body.put("path", request.getDescription(false).replace("uri=", ""));
        return new ResponseEntity<>(body, HttpStatus.NOT_FOUND);
    }

    // Handle general exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleGlobalException(
            Exception ex, WebRequest request) {
        Map<String, Object> body = new HashMap<>();
        body.put("timestamp", LocalDateTime.now());
        body.put("status", HttpStatus.INTERNAL_SERVER_ERROR.value());
        body.put("error", "Internal Server Error");
        body.put("message", ex.getMessage());
        body.put("path", request.getDescription(false).replace("uri=", ""));
        return new ResponseEntity<>(body, HttpStatus.INTERNAL_SERVER_ERROR);
    }

    // Handle unsupported media type exceptions
    @ExceptionHandler(HttpMediaTypeNotSupportedException.class)
    public ResponseEntity<Map<String, Object>> handleHttpMediaTypeNotSupported(
            HttpMediaTypeNotSupportedException ex, WebRequest request) {
        Map<String, Object> body = new HashMap<>();
        body.put("timestamp", LocalDateTime.now());
        body.put("status", HttpStatus.UNSUPPORTED_MEDIA_TYPE.value());
        body.put("error", "Unsupported Media Type");
        body.put("message", ex.getMessage());
        body.put("path", request.getDescription(false).replace("uri=", ""));
        return new ResponseEntity<>(body, HttpStatus.UNSUPPORTED_MEDIA_TYPE);
    }
}
```

### **Global Exception Handling**

- Use `@RestControllerAdvice` or `@ControllerAdvice` to handle exceptions for all controllers in the application.
- Covers exceptions not handled by method-level or controller-level handlers.

**Key Annotations:**

- `@ExceptionHandler`: Marks methods that handle specific exceptions.
- `@RestControllerAdvice`: Combines `@ControllerAdvice` with `@ResponseBody` for REST APIs.
- `@ControllerAdvice`: Global handler for traditional MVC controllers.


Check Sir note once for this

Testing...(Slice based testing)

@WebMvcTest --> Controller test..........

@DataJpaTest -->  focuses on testing only the repository layer. It uses an in-memory database like H2 for testing.

### **Best Practices for Slice-Based Testing**

1. **Mock Dependencies:** Use `@MockBean` to mock service/repository dependencies.
2. **Focus on One Layer:** Avoid testing multiple layers in slice-based tests.
3. **Use In-Memory Databases:** For `@DataJpaTest`, use H2 or an embedded database to test persistence logic.
4. **Test Edge Cases:** Verify the behavior for invalid inputs, exceptions, and boundary conditions.
5. **Leverage Utility Libraries:** Use tools like `ObjectMapper` for JSON serialization/deserialization.


| Slice Annotation | Loads                    | Doesn’t Load                     | Use Case                    |
| ---------------- | ------------------------ | -------------------------------- | --------------------------- |
| `@WebMvcTest`    | MVC Controllers, Filters | Service, Repository, DB          | Test REST endpoints         |
| `@DataJpaTest`   | JPA, H2, Repositories    | Controller, Service, Web context | Test DB logic, queries      |
| `@JsonTest`      | Jackson                  | All Web, DB, Services            | Test JSON mapping logic     |
| `@JdbcTest`      | JdbcTemplate, DB         | Web, Controllers, Services       | Test raw JDBC DAO           |
| `@WebFluxTest`   | Reactive Controllers     | Services, DB                     | Test WebFlux REST endpoints |

@Valid is from jakatra with JSR 303 that have the general validation logic at field and method level
@Validated is from the spring framework which can be applied at class, method and field level.

|Feature|`@Valid`|`@Validated`|
|---|---|---|
|**Package Origin**|`javax.validation.Valid` / `jakarta.validation.Valid`|`org.springframework.validation.annotation.Validated`|
|**Belongs To**|JSR-303 / Bean Validation API|Spring Framework|
|**Supports Validation Groups**|❌ No|✅ Yes|
|**Used For**|Basic bean validation|Advanced use-cases like group validation or method-level validation|
|**Target Level**|Method parameters and fields|Method parameters and **class-level** (AOP validation)|
|**Nested Object Validation**|✅ Yes|✅ Yes|
|**Service Method Validation**|❌ Not supported|✅ Yes (with `@Validated` on service class)|
|**Common Use-case**|Simple input validation in controllers|Group-based validation in controller or service layer|
|**Integrated With Spring Boot?**|✅ Yes|✅ Yes|
|**Exception Thrown On Failure**|`MethodArgumentNotValidException` (in controllers)|`ConstraintViolationException` (in services/methods)|

Interview questions for spring

#### 1. What is Spring Boot and how is it different from Spring Framework?

**Answer:** Spring Boot is a framework built on top of Spring that simplifies application setup by 
auto-configuring components,
embedding servers, 
version management and 
using convention over configuration.

**Example:** In Spring, you configure DispatcherServlet and web.xml manually. In Spring Boot, this is auto-configured.

#### What does `@SpringBootApplication` do?

**Answer:** Combines:

- `@Configuration`
    
- `@ComponentScan`
    
- `@EnableAutoConfiguration`
    

**Example:**


```
@SpringBootApplication --> Combines @Configuration, @EnableAutoConfiguration, @ComponentScan...
public class MyAPP {

public static void main(String[] args){
	SpringApplication.run(MyApp.class, args)
		}
	}
```

#### 5. What are Spring Boot starters?

**Answer:** Starters are pre-configured dependency bundles for specific functionalities.

**Examples:**

- `spring-boot-starter-web` --> MVC, Jackson, Tomcat
    
- `spring-boot-starter-data-jpa` --> Spring data jpa + hibernate
     
- `spring-boot-starter-security` --> Spring security

#### 6. What is Spring Boot DevTools?

**Answer:** DevTools provides auto-reload, enhanced logging, and live reload features for faster development.

#### 7. What is the role of `application.properties` or `application.yml`?

**Answer:** They are used for external configuration, overriding default values in your app.

**Example:** spring.datasource.url, 
		spring.jpa.hibernate.ddl-auto

#### 8. How do Spring Profiles work?

**Answer:** Allows you to define separate configurations for different environments (dev, test, prod).

Let's say --> Environment specific configuration...

**Example:** `application-dev.properties`, `@Profile("dev")`

#### 9. What is Spring Boot Actuator?

**Answer:** Provides production-ready endpoints for monitoring, like 
`/actuator/health`, `/actuator/metrics`.

for enabling the spring boot actuator we add the following 

```<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

| **Endpoint**            | **Description**                               | **Default Access** |
| ----------------------- | --------------------------------------------- | ------------------ |
| `/actuator/health`      | Reports application health.                   | Public             |
| `/actuator/info`        | Displays application-specific information.    | Public             |
| `/actuator/metrics`     | Displays application metrics.                 | Secured            |
| `/actuator/env`         | Shows application environment properties.     | Secured            |
| `/actuator/threaddump`  | Provides a thread dump of the application.    | Secured            |
| `/actuator/loggers`     | Displays and modifies application log levels. | Secured            |
| `/actuator/httptrace`   | Shows HTTP request/response traces.           | Secured            |
| `/actuator/beans`       | Lists all Spring beans in the application.    | Secured            |
| `/actuator/configprops` | Displays configuration properties.            | Secured            |

10. How do you secure Actuator endpoints?
		Configure -->  `management.endpoints.web.exposure.include`
		 Add Spring security and configure users/roles

#### 11. How does Spring Boot simplify database integration?

**Answer:** Auto-configures DataSource, TransactionManager, JPA EntityManager using starters.

#### 12. What is the use of Spring Data JPA?

**Answer:** Simplifies database operations using JpaRepository with methods like `findById`, `save`, `delete`

it supports by default CRUD, pagination and sorting

```
interface UserRepository extends JpaRepository<User, Long> {}
```

#### 13. How do you handle exceptions globally?

**Answer:** Use `@ControllerAdvice` and `@ExceptionHandler` for centralized exception handling.

#### 15. How do you implement pagination in Spring Boot?
	Use Spring Data’s `Pageable` and `Page<T>` types.

```
Page<User> users = userRepository.findAll(PageRequest.of(0,10), Sort.by("lastName").ascending())
```

#### 18. What is CORS and how do you enable it in Spring Boot?

**Answer:** CORS (Cross-Origin Resource Sharing) enables APIs to be called from different domains.

```
@CrossOrigin(origins = "http://frontend.com")
```

Spring Boot CORS Configuration Methods – Comparison Table

| Configuration Method                                              | Scope                | Where to Define                                     | Best For                           | Supports Credentials | Pros                                    | Cons                                               |
| ----------------------------------------------------------------- | -------------------- | --------------------------------------------------- | ---------------------------------- | -------------------- | --------------------------------------- | -------------------------------------------------- |
| `@CrossOrigin` (Annotation)                                       | Method / Class Level | On controller or method                             | Quick testing or small projects    | ✅ Yes                | Simple, localized, easy to use          | Hard to maintain in large apps                     |
| `WebMvcConfigurer` (Java Config)                                  | Global               | In `@Configuration` implementing `WebMvcConfigurer` | Centralized app-wide CORS settings | ✅ Yes                | Clean separation, easy to update        | Needs custom class, doesn’t cover security headers |
| Spring Security `HttpSecurity.cors()` + `CorsConfigurationSource` | Global with Security | Inside `WebSecurityConfigurerAdapter`               | Apps using Spring Security         | ✅ Yes                | Security-compliant, highly configurable | Slightly more verbose configuration                |

#### 19. What is `@RestController` vs `@Controller`?

**Answer:**

- `@RestController` = `@Controller` + `@ResponseBody`
    
| Feature                  | `@RestController`                 | `@Controller`                           |
| ------------------------ | --------------------------------- | --------------------------------------- |
| Purpose                  | REST APIs (JSON/XML responses)    | Web MVC (return HTML views/templates)   |
| Implicit `@ResponseBody` | ✅ Yes                             | ❌ No (must be added manually)           |
| Returns                  | Response body (object → JSON/XML) | View name or HTTP redirect              |
| Common Use               | APIs for mobile/web clients       | Web apps using Thymeleaf, JSP, etc.     |
| Example Return           | `return List<User>` → JSON array  | `return "index"` → renders `index.html` |

Note --> If we change the @RestController with the @Controller then in order to work we have to add the @ResponseBody to make it work as the @Controller does not provide the reponse and it tries to return the view of html file..

```
@Controller
public class UserApiController {
   
	@GetMapping("/users")
    @ResponseBody --> Have to explicitly add this annotations..
    public List<User> getUsers() {
        return userService.findAll();
    }
}

```


#### 20. What is the difference between `@Component`, `@Service`, `@Repository`, `@Controller`?

**Answer:** All are Spring stereotypes.

- `@Component`: Generic bean
    
- `@Service`: Business logic
    
- `@Repository`: DB layer
    
- `@Controller`: MVC web handler

#### 22. How to load external configuration?

Spring Boot supports external configuration from multiple sources such as:

## **Spring Boot Configuration Loading Order (Highest to Lowest Priority)**

| Priority | Source                                                 | Example                         |
| -------- | ------------------------------------------------------ | ------------------------------- |
| 1        | Command-line arguments                                 | `--server.port=8081`            |
| 2        | `application.properties` or `.yml` in `config/` folder | `config/application.yml`        |
| 3        | `application.properties` in `src/main/resources`       | `server.port=8080`              |
| 4        | Environment variables                                  | `SERVER_PORT=8090`              |
| 5        | Default values in code                                 | `@Value("${my.value:default}")` |


#### 25. How does asynchronous processing work in Spring Boot?

> Asynchronous processing in Spring Boot is enabled using `@EnableAsync` and `@Async`.  
> 
> When a method is annotated with `@Async`, it is executed in a **separate thread**, allowing the main thread to proceed without waiting for the method's result.

| Concept         | Description                                                        |
| --------------- | ------------------------------------------------------------------ |
| `@EnableAsync`  | Enables async method execution at the application level            |
| `@Async`        | Marks a method to run asynchronously (in a new thread)             |
| Thread Executor | Uses Spring’s default or custom `TaskExecutor`                     |
| Return Types    | `void`, `Future<T>`, `CompletableFuture<T>`, `ListenableFuture<T>` |
Step 1: Enabling the Async support with the @EnableAsync at the application level..

@SpringBootApplication
@EnableAsync
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}

Step 2: Annotate a method with @Async.. --> Which will run the seperate thread allowing to run it in asynchronous manner..

@Service
public class NotificationService {

    @Async
    public void sendEmail(String to, String subject) {
        // Simulate time-consuming task
        try {
            Thread.sleep(3000);
            System.out.println("Email sent to " + to);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}


#### 24. How to customize embedded Tomcat?

1. Using the application.properties to update the server.port or server releated configurations.
2. Using the `TomcatServletWebServerFactory` Bean 
		Use this when you need full control, like changing access logs, connectors, or enabling compression



```
Use this when you need full control, like changing access logs, connectors, or enabling compression.

```


How to do the validation in the spring boot applications?

Using @Valid and @Validated annotations

@Valid --> Is simple and allows the nested validations

@Validated -> is the spring validation allows the group wise and method level validations.

Valid Vs Validated..

| Feature                  | `@Valid`(javax)       | `@Validated` (Spring)               |
| ------------------------ | --------------------- | ----------------------------------- |
| Standard Java validation | ✅                     | ✅                                   |
| Nested object validation | ✅ (`@Valid` on field) | ✅                                   |
| Validation groups        | ❌                     | ✅ (`@Validated(Group.class)`)       |
| Method-level validation  | ❌                     | ✅ (used with `@Validated` on class) |
| Config properties        | ❌                     | ✅                                   |
Example

public class UserDTO {

    @NotBlank(message = "Username is required", groups = Create.class)
    private String username;

    @Email(message = "Email must be valid")
    private String email;

    @Min(value = 18, message = "Must be 18 or older", groups =     Create.class)
    private int age;

    @Valid // triggers validation on nested AddressDTO
    private AddressDTO address;

    public interface Create {}  // Validation group

    // Getters and setters
}

![[Pasted image 20250525171357.png]]

![[Pasted image 20250525171457.png]]