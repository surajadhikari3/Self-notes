
there is open-api-generator cli which can genrate the controller and model from the open-api spec you provide based on the open-api.yaml file

1. Generate the code from the spec

--> It generates the delegates which is interface. You as a developer have to implement the interface in your service and provide the business implementation..............................


------------------------

2. generate the spec from code:

add the dependecy name 'springdoc-openapi-starter-webmvc-ui' in your pom.xml and you can see the swagger in the ui at localhost:8080/api/swagger-ui.html


Got it 👍 — let’s switch gears.

Now you want the **reverse**: instead of _spec → code_ (contract-first), you want _code → spec_ (code-first).  
That means: you write Spring Boot controllers, and **Swagger/OpenAPI spec is generated automatically**.

---

## ✅ Dependency you need in `pom.xml`

If you’re on **Spring Boot 3** (Jakarta), use **springdoc-openapi**:

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.5.0</version>
</dependency>
```

👉 If you’re using **WebFlux** instead of MVC:

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
  <version>2.5.0</version>
</dependency>
```

That’s it — **just this one dependency**.

---

## ✅ Example controller

```java
@RestController
@RequestMapping("/api/todos")
public class TodoController {

    @Operation(summary = "Get all todos", description = "Returns the list of todos")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Successful retrieval"),
        @ApiResponse(responseCode = "401", description = "Not authorized")
    })
    @GetMapping
    public List<Todo> getTodos() {
        return List.of(new Todo(1L, "Buy milk", false));
    }

    @Operation(summary = "Create a new todo")
    @PostMapping
    public Todo createTodo(@Valid @RequestBody Todo todo) {
        todo.setId(2L);
        return todo;
    }
}
```

---

## ✅ Where to see the generated docs

Run your Spring Boot app and open:

- **Swagger UI:**  
    👉 `http://localhost:8080/swagger-ui.html`
    
- **OpenAPI JSON:**  
    👉 `http://localhost:8080/v3/api-docs`
    
- **OpenAPI YAML:**  
    👉 `http://localhost:8080/v3/api-docs.yaml`
    

---

## ✅ Summary in simple words

1. Add **springdoc-openapi** dependency.
    
2. Annotate controllers with `@Operation`, `@ApiResponse`, etc. (optional; it works even without them, but adds nice docs).
    
3. Run your app → Springdoc generates Swagger UI + OpenAPI spec for you automatically.
    

---

Do you want me to also give you the **Gradle version** of the dependency, in case you’re using Gradle instead of Maven?