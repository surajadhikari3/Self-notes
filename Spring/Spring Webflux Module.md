
Webflux is a reactive web framework in Spring 5 for building non-blocking(asynchronous), event-driven web applications....

Core concepts.....


### 🔑 Core Concepts

| Concept       | Description                                            | Return Type                     |
| ------------- | ------------------------------------------------------ | ------------------------------- |
| **Mono**      | 0 or 1 element                                         | `Mono<T>`                       |
| **Flux**      | 0 to many elements (stream)                            | `Flux<T>`                       |
| **WebClient** | Reactive client to call services in aynchronous manner | `Mono<T>`/`Flux<T>`             |
| **WebFlux**   | Framework to build reactive APIs                       | `@RestController` + `Mono/Flux` |


🌐 What is **WebClient**?

WebClient is non-blocking, reactive HTTP client introduced in the spring 5 as a part of the WebFlux module. 

Key Features

| Feature             | Description                                                            |
| ------------------- | ---------------------------------------------------------------------- |
| 🧵 **Non-blocking** | Doesn't block the thread while waiting for HTTP response.              |
| ⚡ **Reactive**      | Works with reactive types like `Mono` and `Flux`.                      |
| ☁️ **Asynchronous** | Great for calling remote services (e.g., other microservices or APIs). |
| 💡 **Replacement**  | Replaces `RestTemplate`, which is synchronous and blocking.            |

```
WebClient client = WebClient.create();

Mono<String> result = client.get()
    .uri("http://example.com/data")
    .retrieve()
    .bodyToMono(String.class);


```