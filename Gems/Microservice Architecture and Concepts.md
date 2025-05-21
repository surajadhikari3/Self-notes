
![[Pasted image 20250508133412.png]]


It breaks down a large monolithic application into independent, loosely coupled services where:
	Each service is focused on a single business capability.
	 Services communicate over lightweight protocols (typically HTTP/REST or messaging).
	 Each service can be developed, deployed, and scaled independently.


### ⚙️ **Key Components of Microservices Architecture**

| Component                        | Description                                                                 |
| -------------------------------- | --------------------------------------------------------------------------- |
| **API Gateway**                  | Entry point for all client requests. Handles routing, rate limiting, auth.  |
| **Service Registry & Discovery** | Allows services to find each other. E.g., Eureka, Consul.                   |
| **Microservices**                | Each service is independently deployable and loosely coupled.               |
| **Database per service**         | Each microservice manages its own database to ensure decoupling.            |
| **Centralized Configuration**    | Externalized config management. E.g., Spring Cloud Config.                  |
| **Service Communication**        | Synchronous (REST, gRPC) or Asynchronous (Kafka, RabbitMQ).                 |
| **Monitoring/Logging**           | Centralized logging and monitoring. E.g., ELK, Prometheus + Grafana.        |
| **Security**                     | AuthN/AuthZ, usually with OAuth2/JWT at API Gateway or individual services. |



Database per service pattern

API gateway pattern

SAGA Pattern

CQRS Pattern

### ✅ **Key Benefits of Microservice Architecture**

| Benefit                                     | Description                                                                                                                                  |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Scalability**                          | Each microservice can be scaled independently based on its own resource needs, improving performance and resource utilization.               |
| **2. Faster Development & Deployment**      | Teams can build, test, and deploy services independently, allowing **parallel development** and faster release cycles (CI/CD friendly).      |
| **3. Technology Diversity**                 | Teams can choose the best technology for each service (Java, Node.js, Go, etc.) without being forced to use a single stack.                  |
| **4. Fault Isolation**                      | Failure in one microservice (e.g., payment) doesn’t necessarily bring down the entire application (e.g., product listing continues working). |
| **5. Easier Maintenance & Updates**         | Smaller codebases are easier to understand, test, and maintain. Changes in one service don’t impact others directly.                         |
| **6. Better Alignment with DevOps & Agile** | Encourages autonomous, cross-functional teams who own their services end-to-end — from development to deployment.                            |
| **7. Reusability & Composability**          | Services can be reused across different applications (e.g., authentication or email service).                                                |
| **8. Granular Security**                    | Different services can be isolated with firewalls, roles, and tokens, improving security boundaries.                                         |
| **9. Faster Innovation**                    | Teams can experiment and iterate faster without waiting for a centralized release train.                                                     |
| **10. Continuous Delivery Friendly**        | Supports zero-downtime deployments and rollback of individual components.                                                                    |

APi gateway

we can register the diff service in the api gateway. It can

Orchestrate the request --> Means it can make diffe req under the hood and aggregate and provide the response..

If the service is internet facing instead of exposing it directly we register it to the api gateway and access via api gateway

Serve as a entrypoint
Can transform the response
Logging
Authentication and authorization

Example Are

Netflix Zuul api gateway
Nginix api gateway
Apigee API Management (Google):**


Term 
Ingress --> Request is coming in........
Egress --> Request is going out 


### 🧠 **What’s Inside an API Gateway?**

Think of it as a smart router combined with security, monitoring, and protocol translation. Here’s what goes on under the hood:

|Feature / Component|What It Does|Example|
|---|---|---|
|🔄 **Routing**|Forwards requests to the appropriate microservice.|`/users → UserService`|
|🔐 **Authentication & Authorization**|Verifies identity and permissions (OAuth2, JWT, API Keys).|Validates JWT token|
|📉 **Rate Limiting & Throttling**|Prevents abuse by limiting how often users can hit APIs.|100 requests/min|
|🛡 **Security**|Protects against threats (e.g., SQLi, XSS, DoS) via filters and headers.|Adds `X-Frame-Options`|
|📦 **Aggregation**|Combines data from multiple services into one response.|`/order` → User + Product + Payment|
|🔄 **Load Balancing**|Distributes traffic across instances of services.|Round-robin to `UserService` replicas|
|📊 **Monitoring & Logging**|Captures metrics, request logs, and traces for observability.|Integrated with Prometheus / ELK|
|🔁 **Caching**|Stores responses temporarily to reduce backend load.|Cache `/products` for 1 hour|
|🔄 **Protocol Translation**|Converts protocols (e.g., HTTP → WebSocket, REST → gRPC).|REST API call → gRPC internal call|
|🧪 **Request/Response Transformation**|Modifies request/response formats if needed.|Rename `userId` → `id`|
|🧰 **DevOps Integration**|Supports CI/CD, versioning, health checks, blue-green deployments.|`/v1/users`, `/v2/users`|

---

### 🏗 Common Implementations of API Gateways

| Tool / Product           | Platform            | Notable Features                         |
| ------------------------ | ------------------- | ---------------------------------------- |
| **Spring Cloud Gateway** | Java / Spring Boot  | Reactive, filter-based                   |
| **Amazon API Gateway**   | AWS                 | Fully managed, supports REST + WebSocket |
| **Kong**                 | Open-source / Cloud | Lua-based plugins, high performance      |
| **NGINX**                | Open-source         | Reverse proxy, custom configs            |
| **Apigee**               | Google Cloud        | Enterprise-level control & analytics     |



### . **Circuit Breaker Pattern**

- **What it does**: Prevents a service from making requests to a failing service after a threshold of failures is reached.
    
- **Why use it**: Avoids wasting resources and speeds up recovery by failing fast.
    
- **States**:
    
    - **Closed**: Requests flow normally.
        
    - **Open**: Requests are blocked.
        
    - **Half-open**: A few requests are allowed through to test if the service has recovered.


Resilience4j is library that can be used for circuit breaker, and many more...
Now they declare the resilience pattern for the following

| Resilience Pattern       | Description                                                       | Benefit / Purpose                                         |
| ------------------------ | ----------------------------------------------------------------- | --------------------------------------------------------- |
| Retry                    | Automatically retries failed operations after a delay.            | Handles transient failures (e.g., network issues).        |
| Circuit Breaker          | Stops requests to a failing service after a threshold is reached. | Prevents system overload and cascading failures.          |
| Timeout                  | Limits the maximum time to wait for a response.                   | Avoids hanging calls and resource exhaustion.             |
| Bulkhead                 | Isolates service resources into separate pools.                   | Contains failure to a small part of the system.           |
| Fallback                 | Provides an alternative (e.g., cached data) when a service fails. | Maintains partial functionality under failure.            |
| Rate Limiting / Throttle | Controls the number of requests to a service over time.           | Prevents abuse and overload, ensures fair resource usage. |
| Fail Fast                | Immediately fails if a known failure condition exists.            | Saves resources and time by avoiding futile operations.   |
| Cache-Aside              | Retrieves from cache first; fetches and stores if not found.      | Improves performance and reduces backend load.            |

Externalized configuration
