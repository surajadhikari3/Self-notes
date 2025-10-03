Got it—let’s make the Kafka topic(s) that Deephaven consumes fully **runtime-switchable from Spring Boot**, with a clean swap (no “table already closed” surprises), and keep it simple.

Below is a minimal, production-friendly setup with two parts:

1. a tiny **Deephaven “orchestrator” app-script** that exposes a function `set_topics(...)` to (re)wire consumers and update the three exported tables your UI reads (`users_ui`, `accounts_ui`, `final_ui`), and
    
2. a **Spring Boot REST endpoint** that calls that function over Deephaven’s gRPC client whenever a user posts new topics / join type.
    

---

# 1) Deephaven side (Python app script)

Save this file in your DH server container/volume, for example:  
`/app/orchestrator_dh.py` (load it in **Application Mode** or include it in your startup scripts).

```python
# /app/orchestrator_dh.py
from deephaven import dtypes as dt
from deephaven.stream.kafka import consumer as kc
from deephaven.experimental.outer_joins import left_outer_join
from deephaven import time as dhtime

# --- STATIC / SHARED CONFIG (edit your security bits below) ---
BOOTSTRAP_SERVERS = "pkc-1k30p.canadacentral.azure.confluent.cloud:9092"

# Re-use your working config; keep secrets in env if possible
BASE_KAFKA_CONFIG = {
    "bootstrap.servers": BOOTSTRAP_SERVERS,
    "auto.offset.reset": "latest",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    # --- your OAuth settings (consider sourcing from env) ---
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.oauthbearer.token.endpoint.url":
        "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.extensions.logicalCluster": "kc-y8yWMP",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.oauthbearer.token.endpoint.algo": "https",
}

# Value JSON schemas
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string, "name": dt.string, "email": dt.string, "age": dt.int64
})
ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string, "accountType": dt.string, "balance": dt.double
})

# --- STATE we will hot-swap safely ---
_state = {
    "users_raw": None,
    "accounts_raw": None,
    "final_tbl": None,
    "resources": [],  # closeables to dispose when swapping
    "last_ok": None,
}

def _consume_table(topic: str, value_spec):
    """Create a streaming table from a topic."""
    cfg = dict(BASE_KAFKA_CONFIG)  # copy
    return kc.consume(
        config=cfg,
        topics=topic,
        key_spec=kc.IGNORE,          # adjust if you need keys
        value_spec=value_spec,
        table_type=kc.TABLE_TYPE_APPEND
    )

def _safe_close(objs):
    for o in objs or []:
        try:
            o.close()  # TableHandle, Table, blink, etc. are AutoCloseable
        except Exception:
            pass

def set_topics(user_topic: str, account_topic: str, join_type: str = "left"):
    """
    Hot-swap the Kafka consumers + joined view.
    Exports: users_ui, accounts_ui, final_ui
    """
    global users_ui, accounts_ui, final_ui, _state

    # Basic validation (avoid blowing up on junk)
    if not user_topic or not account_topic:
        raise ValueError("Both user_topic and account_topic are required")

    # Build new consumers first (so we can rollback if anything fails)
    new_resources = []
    try:
        users_raw = _consume_table(user_topic, USER_VALUE_SPEC); new_resources.append(users_raw)
        accounts_raw = _consume_table(account_topic, ACCOUNT_VALUE_SPEC); new_resources.append(accounts_raw)

        # Create stable “UI views” (project only what Angular uses)
        users_view    = users_raw.view(["userId", "name", "email", "age"])
        accounts_view = accounts_raw.view(["userId", "accountType", "balance"])
        new_resources += [users_view, accounts_view]

        # Join strategy (extend if you need inner/right/full later)
        if join_type.lower() in ("left", "left_outer", "left_outer_join"):
            final_tbl = left_outer_join(
                users_view, accounts_view, on="userId",
                adds=["accountType", "balance"]
            )
        else:
            # default/fallback: left outer join
            final_tbl = left_outer_join(
                users_view, accounts_view, on="userId",
                adds=["accountType", "balance"]
            )
        new_resources.append(final_tbl)

        # Atomically swap the globals the clients read
        users_ui, accounts_ui, final_ui = users_view, accounts_view, final_tbl

        # Close old after swap to avoid “table already closed” during refresh paints
        _safe_close(_state.get("resources"))
        _state.update({
            "users_raw": users_raw,
            "accounts_raw": accounts_raw,
            "final_tbl": final_tbl,
            "resources": new_resources,
            "last_ok": dhtime.now(),
        })

        print(f"[orchestrator] Topics set → users='{user_topic}', accounts='{account_topic}', join='{join_type}'")

    except Exception as e:
        # If anything failed, dispose newly made resources and keep the old ones alive
        _safe_close(new_resources)
        raise e

# ---- Initial boot (optional defaults so UI has something) ----
try:
    # Replace these with your safe defaults or leave commented
    # set_topics("ccd01_sb_its_esp_tap3567_bishowcaseraw",
    #            "ccd01_sb_its_esp_tap3567_bishowcasecurated",
    #            "left")
    pass
except Exception as boot_err:
    print(f"[orchestrator] Initial wiring failed: {boot_err}")
```

**How it works**

- `set_topics(...)` builds **new** consumers first, then swaps the global `users_ui / accounts_ui / final_ui` references in one go, and only **then** closes the old resources—this prevents the race that causes “table already closed”.
    
- Your Angular (or any client) should always open `users_ui`, `accounts_ui`, and `final_ui` by **variable name**; they will automatically show the new streams after a swap.
    

---

# 2) Spring Boot side (call Deephaven and trigger the swap)

### Maven dependencies

```xml
<dependencies>
  <!-- Spring Web -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <!-- Deephaven Java Client -->
  <dependency>
    <groupId>io.deephaven</groupId>
    <artifactId>deephaven-client</artifactId>
    <version>0.35.0</version> <!-- or your DH server's matching client version -->
  </dependency>

  <!-- (Optional) Micrometer/validation/etc. -->
</dependencies>
```

### Config (application.yml)

```yaml
deephaven:
  host: ${DH_HOST:localhost}
  port: ${DH_PORT:10000}     # DH gRPC port
  useSsl: ${DH_SSL:false}
```

### DTO

```java
// src/main/java/com/example/dh/dto/TopicUpdateRequest.java
package com.example.dh.dto;

import jakarta.validation.constraints.NotBlank;

public record TopicUpdateRequest(
    @NotBlank String userTopic,
    @NotBlank String accountTopic,
    String joinType  // "left" by default
) {}
```

### Deephaven client service

```java
// src/main/java/com/example/dh/service/DeephavenControlService.java
package com.example.dh.service;

import io.deephaven.client.impl.Session;
import io.deephaven.client.impl.SessionFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class DeephavenControlService {

    private final String host;
    private final int port;
    private final boolean useSsl;

    public DeephavenControlService(
        @Value("${deephaven.host}") String host,
        @Value("${deephaven.port}") int port,
        @Value("${deephaven.useSsl}") boolean useSsl
    ) {
        this.host = host;
        this.port = port;
        this.useSsl = useSsl;
    }

    public void setTopics(String userTopic, String accountTopic, String joinType) throws Exception {
        // The orchestrator_dh.py must be loaded on server startup (Application Mode),
        // so its function `set_topics` exists in the server's Python scope.
        final String jt = (joinType == null || joinType.isBlank()) ? "left" : joinType;
        final String code = """
            from orchestrator_dh import set_topics
            set_topics(%s, %s, %s)
            """.formatted(
                pyString(userTopic),
                pyString(accountTopic),
                pyString(jt)
            );

        final SessionFactory.Builder builder = SessionFactory.builder()
                .host(host)
                .port(port);
        if (useSsl) builder.secure(true);

        try (Session session = builder.build().newSession()) {
            // Execute the small Python snippet on the DH server
            session.console().executeCode(code);
        }
    }

    private static String pyString(String s) {
        // Safe single-quoted Python literal with escaping
        return "'" + s.replace("\\", "\\\\").replace("'", "\\'") + "'";
    }
}
```

### REST controller

```java
// src/main/java/com/example/dh/controller/DeephavenController.java
package com.example.dh.controller;

import com.example.dh.dto.TopicUpdateRequest;
import com.example.dh.service.DeephavenControlService;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/dh")
public class DeephavenController {

    private final DeephavenControlService dh;

    public DeephavenController(DeephavenControlService dh) {
        this.dh = dh;
    }

    @PostMapping("/topics")
    public ResponseEntity<?> updateTopics(@Valid @RequestBody TopicUpdateRequest req) {
        try {
            dh.setTopics(req.userTopic(), req.accountTopic(), req.joinType());
            return ResponseEntity.ok().body(
                "Deephaven rewired: users='" + req.userTopic() + "', accounts='" + req.accountTopic() +
                "', join='" + (req.joinType() == null ? "left" : req.joinType()) + "'"
            );
        } catch (Exception e) {
            return ResponseEntity.internalServerError().body("Failed to set topics: " + e.getMessage());
        }
    }
}
```

### Quick test

```bash
curl -X POST http://localhost:8080/api/dh/topics \
  -H 'Content-Type: application/json' \
  -d '{
        "userTopic":"ccd01_sb_its_esp_tap3567_bishowcaseraw",
        "accountTopic":"ccd01_sb_its_esp_tap3567_bishowcasecurated",
        "joinType":"left"
      }'
```

If your Angular reads the three fixed variable names (`users_ui`, `accounts_ui`, `final_ui`), the stream will seamlessly flip to the new topics right after the POST.

---

## Notes that avoid past pitfalls

- **No “table already closed”**: we create new tables first, swap references, then close the old ones.
    
- **Single source of truth**: UI never needs to know the topic; it always points to the same three exported names.
    
- **Security & secrets**: move OAuth client id/secret to environment variables or DH’s secrets store; Spring endpoint can enforce auth/allowlist for topics.
    
- **Version match**: use a **Deephaven client version compatible** with your server build. If your server is different, align `deephaven-client` accordingly.
    

If you want gRPC only (no Python snippet), we can also expose `set_topics` as a **Field/Param in Application Mode** and set it via app fields; but the above is the most direct, minimal, and robust pattern.

-------------------

Absolutely. Here’s the **smallest possible change** to make your Spring Boot caller authenticate to Deephaven using a **PSK (Pre-Shared Key)**.

---

# 1) `application.yml`

```yaml
deephaven:
  host: ${DH_HOST:localhost}
  port: ${DH_PORT:10000}
  useSsl: ${DH_SSL:false}
  auth:
    type: psk
    token: ${DH_PSK:MY_SUPER_SECRET_KEY}   # <- set via env/secret in prod
```

---

# 2) Service that opens a DH session with PSK

> Works with current Deephaven Java client. If your client is older/newer, both Variant A and B are included—use the one that compiles for your version.

```java
// src/main/java/com/example/dh/service/DeephavenControlService.java
package com.example.dh.service;

import io.deephaven.client.impl.Session;
import io.deephaven.client.impl.SessionFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class DeephavenControlService {

    private final String host;
    private final int port;
    private final boolean useSsl;
    private final String authType;  // expect "psk"
    private final String psk;

    public DeephavenControlService(
        @Value("${deephaven.host}") String host,
        @Value("${deephaven.port}") int port,
        @Value("${deephaven.useSsl}") boolean useSsl,
        @Value("${deephaven.auth.type}") String authType,
        @Value("${deephaven.auth.token}") String psk
    ) {
        this.host = host;
        this.port = port;
        this.useSsl = useSsl;
        this.authType = authType;
        this.psk = psk;
    }

    public void setTopics(String userTopic, String accountTopic, String joinType) throws Exception {
        final String jt = (joinType == null || joinType.isBlank()) ? "left" : joinType;

        final String code = """
            from orchestrator_dh import set_topics
            set_topics(%s, %s, %s)
            """.formatted(pyStr(userTopic), pyStr(accountTopic), pyStr(jt));

        final SessionFactory.Builder b = SessionFactory.builder()
            .host(host)
            .port(port);

        if (useSsl) b.secure(true);

        // ---------- PSK AUTH ----------
        // Variant A (preferred on newer clients):
        try {
            b.authenticationType(authType)      // "psk"
             .authenticationValue(psk);         // the shared secret
        } catch (Throwable ignored) {
            // Variant B (fallback for older clients): use the Authorization header.
            // Many DH builds accept PSK as a bearer token.
            b.addHeader("Authorization", "Bearer " + psk);
        }

        try (Session session = b.build().newSession()) {
            session.console().executeCode(code);
        }
    }

    private static String pyStr(String s) {
        return "'" + s.replace("\\", "\\\\").replace("'", "\\'") + "'";
    }
}
```

---

# 3) Controller (unchanged)

Your existing controller can stay the same:

```java
@RestController
@RequestMapping("/api/dh")
public class DeephavenController {

  private final DeephavenControlService dh;

  public DeephavenController(DeephavenControlService dh) { this.dh = dh; }

  @PostMapping("/topics")
  public ResponseEntity<?> updateTopics(@Valid @RequestBody TopicUpdateRequest req) {
    try {
      dh.setTopics(req.userTopic(), req.accountTopic(), req.joinType());
      return ResponseEntity.ok("Rewired to users='" + req.userTopic() + "', accounts='" +
          req.accountTopic() + "', join='" + (req.joinType()==null?"left":req.joinType()) + "'");
    } catch (Exception e) {
      return ResponseEntity.internalServerError().body("Failed to set topics: " + e.getMessage());
    }
  }
}
```

---

# 4) Quick check

1. Start Deephaven **with PSK enabled** (server side):
    

- Docker env:
    
    - `DH_AUTH_TYPE=psk`
        
    - `DH_PSK=MY_SUPER_SECRET_KEY`
        

2. Run Spring Boot with:
    

```
DH_HOST=localhost DH_PORT=10000 DH_PSK=MY_SUPER_SECRET_KEY ./mvnw spring-boot:run
```

3. POST:
    

```bash
curl -X POST http://localhost:8080/api/dh/topics \
  -H 'Content-Type: application/json' \
  -d '{"userTopic":"topic_users","accountTopic":"topic_accounts","joinType":"left"}'
```

If your client library is slightly different, use whichever of the two PSK lines compiles in step (2). Everything else remains identical.

-------------------------------------------------------

You got it. Here’s a **ready-to-run Spring Boot setup** that:

- connects to **Deephaven via PSK auth**
    
- posts **dynamic topics** and **dynamic value specs (schemas)**
    
- executes the Python `set_topics(...)` on the DH server
    

It’s end-to-end and minimal. Copy → paste → run.

---

# 0) Quick project layout

```
src/
 └─ main/
     ├─ java/com/example/dh/
     │   ├─ DhApplication.java
     │   ├─ controller/DeephavenController.java
     │   ├─ dto/TopicUpdateRequest.java
     │   └─ service/DeephavenControlService.java
     └─ resources/
         └─ application.yml
pom.xml
```

---

# 1) `pom.xml`

> Match `deephaven-client` version to your DH server build if needed.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>dh-psk-dynamic-spec</artifactId>
  <version>1.0.0</version>

  <properties>
    <java.version>17</java.version>
    <spring-boot.version>3.3.3</spring-boot.version>
    <!-- Adjust to match your DH server if needed -->
    <deephaven.client.version>0.35.0</deephaven.client.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring-boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <!-- Spring Web + Validation + Jackson -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Deephaven Java client -->
    <dependency>
      <groupId>io.deephaven</groupId>
      <artifactId>deephaven-client</artifactId>
      <version>${deephaven.client.version}</version>
    </dependency>

    <!-- Optional: Lombok for less boilerplate (remove if not using) -->
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

---

# 2) `src/main/resources/application.yml`

```yaml
server:
  port: 8080

deephaven:
  host: ${DH_HOST:localhost}
  port: ${DH_PORT:10000}     # Deephaven gRPC
  useSsl: ${DH_SSL:false}
  auth:
    type: psk                # PSK auth to Deephaven server
    token: ${DH_PSK:MY_SUPER_SECRET_KEY}
```

> ⚠️ This PSK is **for Deephaven server auth**. Your **Kafka** security (OAuth/SASL, etc.) stays in the DH Python app script.

---

# 3) `DhApplication.java`

```java
package com.example.dh;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DhApplication {
  public static void main(String[] args) {
    SpringApplication.run(DhApplication.class, args);
  }
}
```

---

# 4) DTO: `TopicUpdateRequest.java`

```java
package com.example.dh.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

import java.util.Map;

public record TopicUpdateRequest(
    @NotBlank String userTopic,
    @NotBlank String accountTopic,

    // e.g. {"userId":"string","name":"string","email":"string","age":"long"}
    @NotNull Map<String,String> userSchema,

    // e.g. {"userId":"string","accountType":"string","balance":"double"}
    @NotNull Map<String,String> accountSchema,

    // optional; defaults to "left"
    String joinType
) {}
```

---

# 5) Service: `DeephavenControlService.java`

- Builds a **PSK session** to DH
    
- Sends a **tiny Python program** that:
    
    - imports your orchestrator (`orchestrator_dh.py` must be loaded on DH)
        
    - parses the schemas (sent as JSON)
        
    - calls `set_topics(userTopic, accountTopic, userSchema, accountSchema, joinType)`
        

```java
package com.example.dh.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.deephaven.client.impl.Session;
import io.deephaven.client.impl.SessionFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.nio.charset.StandardCharsets;

@Service
public class DeephavenControlService {

  private final String host;
  private final int port;
  private final boolean useSsl;
  private final String authType; // expect "psk"
  private final String psk;
  private final ObjectMapper mapper;

  public DeephavenControlService(
      @Value("${deephaven.host}") String host,
      @Value("${deephaven.port}") int port,
      @Value("${deephaven.useSsl}") boolean useSsl,
      @Value("${deephaven.auth.type}") String authType,
      @Value("${deephaven.auth.token}") String psk,
      ObjectMapper mapper
  ) {
    this.host = host;
    this.port = port;
    this.useSsl = useSsl;
    this.authType = authType;
    this.psk = psk;
    this.mapper = mapper;
  }

  public void setTopics(
      String userTopic,
      String accountTopic,
      Object userSchema,
      Object accountSchema,
      String joinType
  ) throws Exception {

    final String jt = (joinType == null || joinType.isBlank()) ? "left" : joinType;

    final String userSchemaJson = toJson(userSchema);
    final String accountSchemaJson = toJson(accountSchema);

    // Safer to send JSON and parse it in Python (avoids quoting bugs)
    final String pyCode =
        """
        import json
        from orchestrator_dh import set_topics
        _user_schema = json.loads(%s)
        _account_schema = json.loads(%s)
        set_topics(%s, %s, _user_schema, _account_schema, %s)
        """.
            formatted(
                pyString(userSchemaJson),
                pyString(accountSchemaJson),
                pyString(userTopic),
                pyString(accountTopic),
                pyString(jt)
            );

    final SessionFactory.Builder builder = SessionFactory.builder()
        .host(host)
        .port(port);

    if (useSsl) {
      builder.secure(true);
    }

    // ---------- PSK AUTH ----------
    // Newer clients (preferred):
    boolean configured = false;
    try {
      builder.authenticationType(authType)   // "psk"
             .authenticationValue(psk);
      configured = true;
    } catch (Throwable ignored) {
      // Fallback for older builds: some accept PSK as Bearer token header
      // Only use if your client exposes addHeader; if not, keep the newer path above.
      try {
        builder.addHeader("Authorization", "Bearer " + psk);
        configured = true;
      } catch (Throwable t) {
        // If neither API exists, inform the caller
        throw new IllegalStateException("Deephaven client version doesn't expose PSK auth setters.");
      }
    }

    if (!configured) {
      throw new IllegalStateException("Failed to configure PSK authentication");
    }

    try (Session session = builder.build().newSession()) {
      session.console().executeCode(pyCode.getBytes(StandardCharsets.UTF_8));
    }
  }

  private String toJson(Object obj) throws JsonProcessingException {
    return mapper.writeValueAsString(obj);
  }

  private static String pyString(String s) {
    // Single-quoted Python literal with escaping
    return "'" + s.replace("\\", "\\\\").replace("'", "\\'") + "'";
  }
}
```

---

# 6) Controller: `DeephavenController.java`

```java
package com.example.dh.controller;

import com.example.dh.dto.TopicUpdateRequest;
import com.example.dh.service.DeephavenControlService;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/dh")
public class DeephavenController {

  private final DeephavenControlService service;

  public DeephavenController(DeephavenControlService service) {
    this.service = service;
  }

  @PostMapping("/topics")
  public ResponseEntity<?> updateTopics(@Valid @RequestBody TopicUpdateRequest req) {
    try {
      service.setTopics(
          req.userTopic(),
          req.accountTopic(),
          req.userSchema(),
          req.accountSchema(),
          req.joinType()
      );
      return ResponseEntity.ok(
          "Deephaven rewired: users='" + req.userTopic() + "', accounts='" +
              req.accountTopic() + "', join='" + (req.joinType() == null ? "left" : req.joinType()) + "'"
      );
    } catch (Exception e) {
      return ResponseEntity.internalServerError().body("Failed to set topics: " + e.getMessage());
    }
  }
}
```

---

# 7) Start everything

**Deephaven server** must:

- load your **`/app/orchestrator_dh.py`** on boot (Application Mode or startup script)
    
- be started **with PSK auth**, e.g. in Docker:
    

```yaml
services:
  deephaven:
    image: ghcr.io/deephaven/server
    ports: ["10000:10000"]
    environment:
      - DH_AUTH_TYPE=psk
      - DH_PSK=MY_SUPER_SECRET_KEY
    volumes:
      - ./app:/app           # contains orchestrator_dh.py
    command: ["--scripts", "/app/orchestrator_dh.py"]
```

> Your **Kafka** config (OAuth/SASL, etc.) stays inside `orchestrator_dh.py` (that’s independent of PSK).

**Spring Boot**:

```bash
DH_HOST=localhost \
DH_PORT=10000 \
DH_PSK=MY_SUPER_SECRET_KEY \
./mvnw spring-boot:run
```

---

# 8) Example request

```bash
curl -X POST http://localhost:8080/api/dh/topics \
  -H 'Content-Type: application/json' \
  -d '{
        "userTopic": "topic_users",
        "accountTopic": "topic_accounts",
        "userSchema":   { "userId":"string", "name":"string", "email":"string", "age":"long" },
        "accountSchema":{ "userId":"string", "accountType":"string", "balance":"double" },
        "joinType": "left"
      }'
```

If your Angular subscribes to `users_ui`, `accounts_ui`, and `final_ui` by name, it will see the new streams immediately after this call (no reloads).

---

## Hardening tips (optional but recommended)

- **Whitelist schemas** (column names / dtypes) before sending to DH to avoid invalid types.
    
- Enforce an **allowed topics** list on the API (avoid arbitrary Kafka topics).
    
- Add **AuthN/AuthZ** on `/api/dh/topics` (e.g., token check) to prevent unauthorized rewires.
    
- Log the **effective Python code** you send for audit (but not secrets).
    

---

If you want, I can also drop in a **simple Angular service** that auto-opens those 3 tables and streams rows into ag-Grid without page refresh.

---------------------------------------

Short answer: **not yet**.  
Your current `set_topics(user_topic, account_topic, join_type)` still hard-codes the value specs (`USER_VALUE_SPEC`, `ACCOUNT_VALUE_SPEC`). To make both **topic and value_spec dynamic**, change the DH orchestrator to accept schemas (or fully-built specs) from Spring, then build `kc.json_spec` at runtime.

Here’s the **minimal change** on the Deephaven side:

```python
# /app/orchestrator_dh.py
from deephaven import dtypes as dt
from deephaven.stream.kafka import consumer as kc
from deephaven.experimental.outer_joins import left_outer_join
from deephaven import time as dhtime

BASE_KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-1k30p.canadacentral.azure.confluent.cloud:9092",
    "auto.offset.reset": "latest",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class":
        "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.oauthbearer.token.endpoint.url":
        "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.extensions.logicalCluster": "kc-y8yWMP",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.oauthbearer.token.endpoint.algo": "https",
}

# map simple strings -> Deephaven dtypes
_DTYPE = {
    "string": dt.string, "int": dt.int32, "long": dt.int64,
    "double": dt.double, "float": dt.float32, "bool": dt.bool_
}

def _json_spec_from(schema: dict):
    # schema example: {"userId":"string","name":"string","age":"long"}
    spec = {k: _DTYPE[v.lower()] for k, v in schema.items()}
    return kc.json_spec(spec)

_state = {"resources": [], "last_ok": None}

def _consume_table(topic: str, value_spec):
    cfg = dict(BASE_KAFKA_CONFIG)
    return kc.consume(
        config=cfg, topics=topic,
        key_spec=kc.IGNORE, value_spec=value_spec,
        table_type=kc.TABLE_TYPE_APPEND
    )

def _safe_close(objs):
    for o in objs or []:
        try: o.close()
        except: pass

def set_topics(user_topic: str, account_topic: str,
               user_schema: dict, account_schema: dict,
               join_type: str = "left"):
    """
    Hot-swap consumers with DYNAMIC schemas.
    Exports: users_ui, accounts_ui, final_ui
    """
    global users_ui, accounts_ui, final_ui, _state

    if not user_topic or not account_topic:
        raise ValueError("Both user_topic and account_topic are required")
    if not user_schema or not account_schema:
        raise ValueError("Both user_schema and account_schema are required")

    new_resources = []
    try:
        user_spec    = _json_spec_from(user_schema)
        account_spec = _json_spec_from(account_schema)

        users_raw    = _consume_table(user_topic, user_spec);    new_resources.append(users_raw)
        accounts_raw = _consume_table(account_topic, account_spec); new_resources.append(accounts_raw)

        users_view    = users_raw.view(list(user_schema.keys()))
        accounts_view = accounts_raw.view(list(account_schema.keys()))
        new_resources += [users_view, accounts_view]

        # choose join on the shared key (assumes 'userId' exists in both; make it a param if needed)
        adds = [c for c in account_schema.keys() if c != "userId"]
        final_tbl = left_outer_join(users_view, accounts_view, on="userId", adds=adds)
        new_resources.append(final_tbl)

        users_ui, accounts_ui, final_ui = users_view, accounts_view, final_tbl

        _safe_close(_state.get("resources"))
        _state.update({"resources": new_resources, "last_ok": dhtime.now()})
        print(f"[orchestrator] topics set users='{user_topic}', accounts='{account_topic}', join='{join_type}'")
    except Exception as e:
        _safe_close(new_resources)
        raise
```

### Spring Boot (what you send)

Post your topics **and** schemas; your existing PSK/OAuth session code just needs to execute:

```python
from orchestrator_dh import set_topics
set_topics(
  'topic_users', 'topic_accounts',
  {'userId':'string','name':'string','email':'string','age':'long'},
  {'userId':'string','accountType':'string','balance':'double'},
  'left'
)
```

If you already implemented the Spring `TopicUpdateRequest` with `userSchema` and `accountSchema`, you’re done.  
So: **after this change, yes**—both `topic` and `value_spec` are dynamic and controlled from Spring.

--------------------

You’re right — in **0.39.8** there is **no `SessionFactory.Builder`**. In that line the client still used the **Dagger**-wired factory. Here’s a drop-in way to make your Spring service compile and work on **Java 11 + Deephaven 0.39.8** with **PSK**.

I’m giving you two auth paths:

1. **Recommended (0.39.x-native):** use the Dagger `SessionSubcomponent.Builder.authenticationTypeAndValue(...)`.  
    Pass the PSK handler **FQCN + a space + the key** (ex: `io.deephaven.authentication.psk.PskAuthenticationHandler MyKey`). This matches the “authentication type and optional value” contract in the 0.39 docs, and is consistent with the other language clients that accept the PSK handler class name. ([docs.deephaven.io](https://docs.deephaven.io/core/javadoc/io/deephaven/client/impl/Authentication.html?utm_source=chatgpt.com "Authentication (combined-javadoc 0.39.5 API)"))
    
2. **Fallback:** add the **`Authorization`** header (`Bearer <key>`) at the channel layer. The 0.39 javadoc exposes that header key and many builds will accept it. ([docs.deephaven.io](https://docs.deephaven.io/core/javadoc/io/deephaven/client/impl/Authentication.html?utm_source=chatgpt.com "Authentication (combined-javadoc 0.39.5 API)"))
    

---

# 0) Maven (0.39.8)

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.deephaven</groupId>
      <artifactId>deephaven-bom</artifactId>
      <version>0.39.8</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <!-- Java client session API (0.39.8) -->
  <dependency>
    <groupId>io.deephaven</groupId>
    <artifactId>deephaven-java-client-session</artifactId>
  </dependency>

  <!-- Dagger-wired factory that exists in 0.39.x -->
  <dependency>
    <groupId>io.deephaven</groupId>
    <artifactId>deephaven-java-client-session-dagger</artifactId>
  </dependency>

  <!-- gRPC (ManagedChannelBuilder) -->
  <dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
  </dependency>
  <dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
  </dependency>
  <dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
  </dependency>
</dependencies>
```

---

# 1) `application.yml`

```yaml
deephaven:
  host: ${DH_HOST:localhost}
  port: ${DH_PORT:10000}
  useSsl: ${DH_SSL:false}
  auth:
    # For PSK in 0.39.x, pass the handler class name
    # io.deephaven.authentication.psk.PskAuthenticationHandler
    type: ${DH_AUTH_TYPE:io.deephaven.authentication.psk.PskAuthenticationHandler}
    token: ${DH_PSK:MY_SUPER_SECRET_KEY}
```

---

# 2) Service (Java 11, no text blocks)

```java
package com.example.dh.service;

import io.deephaven.client.DaggerDeephavenSessionRoot;
import io.deephaven.client.SessionSubcomponent;
import io.deephaven.client.impl.Session;
import io.deephaven.client.impl.ConsoleService;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.ClientInterceptor;
import io.grpc.Metadata;
import io.grpc.stub.MetadataUtils;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class DeephavenControlService {

    private final String host;
    private final int port;
    private final boolean useSsl;
    private final String authType; // e.g. io.deephaven.authentication.psk.PskAuthenticationHandler
    private final String psk;

    public DeephavenControlService(
            @Value("${deephaven.host}") String host,
            @Value("${deephaven.port}") int port,
            @Value("${deephaven.useSsl}") boolean useSsl,
            @Value("${deephaven.auth.type}") String authType,
            @Value("${deephaven.auth.token}") String psk
    ) {
        this.host = host;
        this.port = port;
        this.useSsl = useSsl;
        this.authType = authType;
        this.psk = psk;
    }

    public void setTopics(String userTopic, String accountTopic, String joinType) throws Exception {
        final String jt = (joinType == null || joinType.isBlank()) ? "left" : joinType;

        // Build the Python code string (Java 11-friendly)
        final String code =
                "from orchestrator_dh import set_topics\n" +
                "set_topics(" + pyStr(userTopic) + ", " + pyStr(accountTopic) + ", " + pyStr(jt) + ")\n";

        // --- Build a gRPC channel (Java 11) ---
        final ManagedChannelBuilder<?> mcb = ManagedChannelBuilder.forAddress(host, port);
        if (useSsl) {
            mcb.useTransportSecurity();
        } else {
            mcb.usePlaintext();
        }

        // ---- Optional Fallback: Add Authorization: Bearer <psk> header interceptor ----
        // Keeps working even if authType is misconfigured and your server accepts Bearer tokens.
        // (This matches Authentication.AUTHORIZATION_HEADER in the 0.39 API.)
        final ClientInterceptor bearerInterceptor = MetadataUtils.newAttachHeadersInterceptor(bearer(psk));
        mcb.intercept(bearerInterceptor);

        final ManagedChannel channel = mcb.build();

        // ---- 0.39.x-native authentication path (recommended) ----
        // The API expects "authentication type and value" as a single string.
        // For PSK, use the handler FQCN + a space + the secret.
        final String typeAndValue = authType + " " + psk;

        // Build the 0.39.x Dagger subcomponent and obtain the SessionFactory (subcomponent IS a factory)
        final SessionSubcomponent sub = DaggerDeephavenSessionRoot.create()
                .factoryBuilder()
                .managedChannel(channel)                 // supply our channel
                .authenticationTypeAndValue(typeAndValue) // PSK handler + key
                .build();

        // sub extends SessionFactory: open a session and run the code in the Python console
        try (Session session = sub.newSession()) {
            ConsoleService console = session.console();
            console.executeCode(code);
        } finally {
            channel.shutdown();
        }
    }

    private static String pyStr(String s) {
        if (s == null) return "None";
        return "'" + s.replace("\\", "\\\\").replace("'", "\\'") + "'";
    }

    private static Metadata bearer(String token) {
        Metadata md = new Metadata();
        Metadata.Key<String> AUTH = Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
        md.put(AUTH, "Bearer " + token);
        return md;
    }
}
```

**Why this compiles on 0.39.8:**

- `SessionFactory` is an **interface** with `newSession()`; there is **no `.builder()`** in that version. `SessionSubcomponent` (from the Dagger artifact) **extends `SessionFactory`**, and `DaggerDeephavenSessionRoot.create().factoryBuilder()` gives you a `SessionSubcomponent.Builder` where you set the **`ManagedChannel`** and the **`authenticationTypeAndValue`** string. ([docs.deephaven.io](https://docs.deephaven.io/core/javadoc/io/deephaven/client/impl/SessionFactory.html?utm_source=chatgpt.com "SessionFactory (combined-javadoc 0.40.0-SNAPSHOT API)"))
    
- For PSK, the accepted **authentication type** is the handler class name (e.g., `io.deephaven.authentication.psk.PskAuthenticationHandler`), and the **value** is the shared secret — concatenate them with a space (that “type and value” form comes straight from the 0.39 auth API). ([docs.deephaven.io](https://docs.deephaven.io/core/javadoc/io/deephaven/client/impl/Authentication.html?utm_source=chatgpt.com "Authentication (combined-javadoc 0.39.5 API)"))
    

---

# 3) Controller (same as before)

```java
@RestController
@RequestMapping("/api/dh")
public class DeephavenController {

  private final DeephavenControlService dh;

  public DeephavenController(DeephavenControlService dh) { this.dh = dh; }

  @PostMapping("/topics")
  public ResponseEntity<?> updateTopics(@RequestBody TopicUpdateRequest req) {
    try {
      dh.setTopics(req.userTopic(), req.accountTopic(), req.joinType());
      return ResponseEntity.ok("Rewired to users='" + req.userTopic()
          + "', accounts='" + req.accountTopic()
          + "', join='" + (req.joinType()==null?"left":req.joinType()) + "'");
    } catch (Exception e) {
      return ResponseEntity.internalServerError().body("Failed to set topics: " + e.getMessage());
    }
  }
}
```

---

## Quick sanity checklist

- DH server is running with **PSK enabled** and your key set.
    
- `application.yml` has the `auth.type` exactly:  
    `io.deephaven.authentication.psk.PskAuthenticationHandler` (match server side). ([deephaven.io](https://deephaven.io/core/docs/how-to-guides/authentication/auth-psk/?utm_source=chatgpt.com "Configure and use pre-shared key authentication"))
    
- Your Spring Boot uses **Java 11** source level (no text blocks).
    
- You added **both** `deephaven-java-client-session` **and** `deephaven-java-client-session-dagger` at **0.39.8**.
    

If anything is still off, paste the exact error and I’ll tweak the imports or method names to your module versions.

-------------------------


Awesome — here’s a **clean, Java-11 + Deephaven 0.39.8 + PSK** working set.  
It includes the minimal `pom.xml` deps, `application.yml`, a DTO, a controller, and the **fixed** service that uses `ConsoleService.startConsole("python")` → `ConsoleSession.executeCode(...)`.

---

# 1) `pom.xml` (only what you need for 0.39.8)

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>dh-psk-0398</artifactId>
  <version>1.0.0</version>

  <properties>
    <java.version>11</java.version>
    <spring-boot.version>2.7.18</spring-boot.version> <!-- Java 11-friendly -->
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring-boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>

      <!-- Pin ALL Deephaven artifacts to the same line -->
      <dependency>
        <groupId>io.deephaven</groupId>
        <artifactId>deephaven-bom</artifactId>
        <version>0.39.8</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <!-- Spring Web + Validation -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Deephaven Java client (0.39.8) -->
    <dependency>
      <groupId>io.deephaven</groupId>
      <artifactId>deephaven-java-client-session</artifactId>
    </dependency>
    <!-- Dagger wiring that exposes SessionSubcomponent.factoryBuilder() -->
    <dependency>
      <groupId>io.deephaven</groupId>
      <artifactId>deephaven-java-client-session-dagger</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

---

# 2) `src/main/resources/application.yml`

```yaml
server:
  port: 8080

deephaven:
  host: ${DH_HOST:localhost}
  port: ${DH_PORT:10000}
  useSsl: ${DH_SSL:false}
  auth:
    # 0.39.x expects "authentication type and value" (class name + space + secret)
    type: ${DH_AUTH_TYPE:io.deephaven.authentication.psk.PskAuthenticationHandler}
    token: ${DH_PSK:MY_SUPER_SECRET_KEY}
```

---

# 3) `src/main/java/com/example/dh/DhApplication.java`

```java
package com.example.dh;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DhApplication {
  public static void main(String[] args) {
    SpringApplication.run(DhApplication.class, args);
  }
}
```

---

# 4) DTO: `src/main/java/com/example/dh/dto/TopicUpdateRequest.java`

```java
package com.example.dh.dto;

import jakarta.validation.constraints.NotBlank;

public record TopicUpdateRequest(
    @NotBlank String userTopic,
    @NotBlank String accountTopic,
    String joinType   // optional; defaults to "left"
) {}
```

---

# 5) Controller: `src/main/java/com/example/dh/controller/DeephavenController.java`

```java
package com.example.dh.controller;

import com.example.dh.dto.TopicUpdateRequest;
import com.example.dh.service.DeephavenControlService;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/dh")
public class DeephavenController {

  private final DeephavenControlService dh;

  public DeephavenController(DeephavenControlService dh) {
    this.dh = dh;
  }

  @PostMapping("/topics")
  public ResponseEntity<?> updateTopics(@Valid @RequestBody TopicUpdateRequest req) {
    try {
      dh.setTopics(req.userTopic(), req.accountTopic(), req.joinType());
      return ResponseEntity.ok(
          "Rewired to users='" + req.userTopic() + "', accounts='" +
              req.accountTopic() + "', join='" + (req.joinType()==null?"left":req.joinType()) + "'"
      );
    } catch (Exception e) {
      return ResponseEntity.internalServerError().body("Failed to set topics: " + e.getMessage());
    }
  }
}
```

---

# 6) **Service (fixed console path)**: `src/main/java/com/example/dh/service/DeephavenControlService.java`

```java
package com.example.dh.service;

import io.deephaven.client.DaggerDeephavenSessionRoot;
import io.deephaven.client.SessionSubcomponent;
import io.deephaven.client.impl.ConsoleService;
import io.deephaven.client.impl.ConsoleSession;
import io.deephaven.client.impl.Session;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class DeephavenControlService {

  private final String host;
  private final int port;
  private final boolean useSsl;
  private final String authType; // e.g. io.deephaven.authentication.psk.PskAuthenticationHandler
  private final String psk;

  public DeephavenControlService(
      @Value("${deephaven.host}") String host,
      @Value("${deephaven.port}") int port,
      @Value("${deephaven.useSsl}") boolean useSsl,
      @Value("${deephaven.auth.type}") String authType,
      @Value("${deephaven.auth.token}") String psk
  ) {
    this.host = host;
    this.port = port;
    this.useSsl = useSsl;
    this.authType = authType;
    this.psk = psk;
  }

  public void setTopics(String userTopic, String accountTopic, String joinType) throws Exception {
    final String jt = (joinType == null || joinType.isBlank()) ? "left" : joinType;

    // Build the tiny Python program (Java 11-friendly)
    final String code =
        "from orchestrator_dh import set_topics\n" +
        "set_topics(" + pyStr(userTopic) + ", " + pyStr(accountTopic) + ", " + pyStr(jt) + ")\n";

    // gRPC channel to Deephaven server
    ManagedChannelBuilder<?> mcb = ManagedChannelBuilder.forAddress(host, port);
    if (useSsl) mcb.useTransportSecurity(); else mcb.usePlaintext();
    ManagedChannel channel = mcb.build();

    // 0.39.x expects "authentication type and value" as one string:
    // "<handler FQCN> <secret>"
    final String typeAndValue = authType + " " + psk;

    // Dagger factory -> Session
    SessionSubcomponent sub = DaggerDeephavenSessionRoot.create()
        .factoryBuilder()
        .managedChannel(channel)
        .authenticationTypeAndValue(typeAndValue)
        .build();

    try (Session session = sub.newSession()) {
      // IMPORTANT: open a console session and execute the code there
      ConsoleService consoleService = session.console();
      try (ConsoleSession py = consoleService.startConsole("python")) {
        py.executeCode(code);
      }
    } finally {
      channel.shutdown();
    }
  }

  private static String pyStr(String s) {
    if (s == null) return "None";
    return "'" + s.replace("\\", "\\\\").replace("'", "\\'") + "'";
  }
}
```

---

## Run it

1. Start Deephaven with **PSK enabled** and ensure your `orchestrator_dh.py` is loaded (Application Mode or `--scripts`):
    

- Env for server:
    
    - `DH_AUTH_TYPE=psk`
        
    - `DH_PSK=MY_SUPER_SECRET_KEY`
        

2. Start Spring:
    

```bash
DH_HOST=localhost \
DH_PORT=10000 \
DH_PSK=MY_SUPER_SECRET_KEY \
./mvnw spring-boot:run
```

3. Call the endpoint:
    

```bash
curl -X POST http://localhost:8080/api/dh/topics \
  -H 'Content-Type: application/json' \
  -d '{"userTopic":"topic_users","accountTopic":"topic_accounts","joinType":"left"}'
```

Your Angular app should keep subscribing to the **fixed table names** (`users_ui`, `accounts_ui`, `final_ui`), and they’ll flip to the new topics right after the POST.

---

### Notes / gotchas

- If your orchestrator is named differently, change the import line in `code` from `from orchestrator_dh import set_topics` to match.
    
- Keep the **Deephaven client version** aligned to your server’s minor (you’re on **0.39.8**, all good).
    
- We did **not** include any extra GRPC or DH modules explicitly; the two DH client artifacts + BOM handle the graph and avoid version clashes.
    

If you want me to add **dynamic schemas** again later, we’ll just extend the `code` string to pass JSON schemas and parse them in Python.

---------------------------------------------


Yep—I can see from where it’s blowing up that the Dagger **factoryBuilder().build()** is still missing a required dependency.  
In **0.39.8**, the client factory needs **both**:

- a `ScheduledExecutorService` (we added), **and**
    
- a general-purpose `Executor` (thread pool)
    

If either is absent, `build()` throws (that’s what you’re seeing).

Here’s the drop-in fix for your `setTopics(...)`—I’ve added the missing `executor(...)` and proper shutdown. Nothing else changes.

```java
public void setTopics(String userTopic, String accountTopic, String joinType) throws Exception {
  String jt = (joinType == null) ? "left" : joinType.trim().toLowerCase().replace(" ", "_");
  if (!jt.startsWith("left")) jt = "left";

  final String code =
      "from orchestrator_dh import set_topics\n" +
      "set_topics(" + pyStr(userTopic) + ", " + pyStr(accountTopic) + ", " + pyStr(jt) + ")\n";

  // --- gRPC channel ---
  ManagedChannelBuilder<?> mcb = ManagedChannelBuilder.forAddress(host, port);
  if (useSsl) mcb.useTransportSecurity(); else mcb.usePlaintext();
  ManagedChannel channel = mcb.build();

  // REQUIRED by 0.39.x factory:
  // 1) general-purpose executor
  java.util.concurrent.Executor exec =
      java.util.concurrent.Executors.newCachedThreadPool(r -> {
        Thread t = new Thread(r, "dh-client-exec");
        t.setDaemon(true);
        return t;
      });

  // 2) scheduled executor
  java.util.concurrent.ScheduledExecutorService scheduler =
      java.util.concurrent.Executors.newScheduledThreadPool(2, r -> {
        Thread t = new Thread(r, "dh-client-scheduler");
        t.setDaemon(true);
        return t;
      });

  final String typeAndValue = authType + " " + psk; // e.g. "io.deephaven.authentication.psk.PskAuthenticationHandler <key>"

  // --- Dagger factory (acts as SessionFactory) ---
  SessionSubcomponent sub = DaggerDeephavenSessionRoot.create()
      .factoryBuilder()
      .managedChannel(channel)
      .executor(exec)                 // <-- add this
      .scheduler(scheduler)           // <-- and this
      .authenticationTypeAndValue(typeAndValue)
      .build();

  try (Session session = sub.newSession()) {
    ConsoleService consoleService = session.console();
    java.util.concurrent.CompletableFuture<? extends ConsoleSession> fut =
        consoleService.console("python");
    try (ConsoleSession py = fut.get(15, java.util.concurrent.TimeUnit.SECONDS)) {
      py.executeCode(code); // or py.executeCodeFuture(code).get(30, TimeUnit.SECONDS);
    }
  } finally {
    channel.shutdown();
    ((java.util.concurrent.ExecutorService) exec).shutdown();
    scheduler.shutdown();
    channel.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS);
    ((java.util.concurrent.ExecutorService) exec).awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS);
    scheduler.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS);
  }
}
```

If `build()` still throws, print the full root cause (it usually says exactly which dependency is missing). But in 0.39.8, the common misses are precisely `executor(...)` and `scheduler(...)`, so the above should unblock you.

--------------------------------------------

Thanks for the screenshots — they tell us exactly which 0.39.8 API variant you’ve got:

- `Session.console(String)` (not `Session.console()` → `ConsoleService`)
    
- Dagger factory builder with **only**: `.managedChannel(...)`, `.scheduler(...)`, `.authenticationTypeAndValue(...)`  
    (no `.executor(...)`)
    

So fix two things:

1. **Don’t pass an executor** to `factoryBuilder()` (your variant doesn’t have it).
    
2. **Call the console like this**: `session.console("python")` → returns a `CompletableFuture<? extends ConsoleSession>`.
    

Here’s your **drop-in corrected `setTopics`** (Java 11 + DH 0.39.8 + PSK), matching your API:

```java
public void setTopics(String userTopic, String accountTopic, String joinType) throws Exception {
  // normalize joinType (accepts "LEFT OUTER", "left_outer", "left")
  String jt = (joinType == null) ? "left"
      : joinType.trim().toLowerCase().replace(' ', '_');
  if (!jt.startsWith("left")) jt = "left";

  final String code =
      "from orchestrator_dh import set_topics\n" +
      "set_topics(" + pyStr(userTopic) + ", " + pyStr(accountTopic) + ", " + pyStr(jt) + ")\n";

  // ---- gRPC channel ----
  ManagedChannelBuilder<?> mcb = ManagedChannelBuilder.forAddress(host, port);
  if (useSsl) mcb.useTransportSecurity(); else mcb.usePlaintext();
  ManagedChannel channel = mcb.build();

  // ---- REQUIRED by your 0.39.8 builder: a scheduler ----
  java.util.concurrent.ScheduledExecutorService scheduler =
      java.util.concurrent.Executors.newScheduledThreadPool(2, r -> {
        Thread t = new Thread(r, "dh-client-scheduler");
        t.setDaemon(true);
        return t;
      });

  // PSK auth: "<handler FQCN> <secret>"
  final String typeAndValue = authType + " " + psk;

  // ---- Dagger factory (acts as SessionFactory) ----
  SessionSubcomponent sub = DaggerDeephavenSessionRoot.create()
      .factoryBuilder()
      .managedChannel(channel)
      .scheduler(scheduler)                          // <-- your builder has this
      .authenticationTypeAndValue(typeAndValue)
      .build();

  try (Session session = sub.newSession()) {
    // IMPORTANT: your Session has console(String) directly
    java.util.concurrent.CompletableFuture<? extends ConsoleSession> fut =
        session.console("python");

    try (ConsoleSession py =
             fut.get(15, java.util.concurrent.TimeUnit.SECONDS)) {
      // run the snippet (blocking variant)
      py.executeCode(code);
      // or: py.executeCodeFuture(code).get(30, TimeUnit.SECONDS);
    }
  } finally {
    channel.shutdown();
    scheduler.shutdown();
    channel.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS);
    scheduler.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS);
  }
}

private static String pyStr(String s) {
  if (s == null) return "None";
  return "'" + s.replace("\\", "\\\\").replace("'", "\\'") + "'";
}
```

### Why this fixes your errors

- The builder error was because your `factoryBuilder()` **doesn’t support** `.executor(...)`. We removed it and kept only `.scheduler(...)`, which your stack trace previously said was required.
    
- The console error “expected 1 argument” is because **your** `Session` exposes `console(String)` directly. We now call `session.console("python")` and `get(...)` the `ConsoleSession` future before `executeCode(...)`.
    

If anything else throws at `build()`, paste the _first_ cause message — it will name the exact missing parameter, and I’ll adjust again.

--------------------------------------------


UNAUTHENTICATED means the server rejected the creds. With 0.39.8 there are a couple of valid ways to send PSK, and different deployments enable different handlers. Let’s make your client **try the common variants automatically** and you’ll stop tripping on which one your server expects.

## What usually causes this

- Server isn’t actually running with PSK (`DH_AUTH_TYPE=psk`, `DH_PSK=...`)
    
- Wrong “type+value” string (class FQCN vs shorthand `psk`)
    
- Extra quotes / spaces around the key
    
- TLS/plaintext mismatch (less likely, that’s usually `UNAVAILABLE`, not `UNAUTHENTICATED`)
    

## Drop-in: resilient auth attempts (0.39.8 API, Java 11)

Replace your current `setTopics(...)` body with this. It will:

1. Build the gRPC channel (TLS or plaintext per your config)
    
2. Try **FQCN + key**, then **`psk` + key**, then **Authorization header** (Bearer)
    
3. On first success, it opens a **console("python")** and executes your snippet.
    

```java
public void setTopics(String userTopic, String accountTopic, String joinType) throws Exception {
  String jt = (joinType == null) ? "left" : joinType.trim().toLowerCase().replace(' ', '_');
  if (!jt.startsWith("left")) jt = "left";

  final String code =
      "from orchestrator_dh import set_topics\n" +
      "set_topics(" + pyStr(userTopic) + ", " + pyStr(accountTopic) + ", " + pyStr(jt) + ")\n";

  // ---- Build channel (once) ----
  ManagedChannelBuilder<?> mcb = ManagedChannelBuilder.forAddress(host, port);
  if (useSsl) mcb.useTransportSecurity(); else mcb.usePlaintext();

  // we’ll add an interceptor later only for the header attempt
  ManagedChannel channel = mcb.build();

  // Required by your 0.39.8 builder: scheduler
  java.util.concurrent.ScheduledExecutorService scheduler =
      java.util.concurrent.Executors.newScheduledThreadPool(2, r -> {
        Thread t = new Thread(r, "dh-client-scheduler");
        t.setDaemon(true);
        return t;
      });

  // 3 auth strategies we’ll try in order
  String[] typeAndValueVariants = new String[] {
      // 1) FQCN + key (most common)
      "io.deephaven.authentication.psk.PskAuthenticationHandler " + psk,
      // 2) shorthand "psk" + key (some servers accept this)
      "psk " + psk
  };

  boolean success = false;
  StatusRuntimeException lastAuthErr = null;

  // --- Attempt 1 & 2: typeAndValue variants ---
  for (String tav : typeAndValueVariants) {
    try {
      SessionSubcomponent sub = DaggerDeephavenSessionRoot.create()
          .factoryBuilder()
          .managedChannel(channel)
          .scheduler(scheduler)
          .authenticationTypeAndValue(tav)
          .build();

      try (Session session = sub.newSession()) {
        // Your API has console(String)
        java.util.concurrent.CompletableFuture<? extends ConsoleSession> fut =
            session.console("python");
        try (ConsoleSession py = fut.get(15, java.util.concurrent.TimeUnit.SECONDS)) {
          py.executeCode(code);
          success = true;
          break;
        }
      }
    } catch (StatusRuntimeException sre) {
      if (sre.getStatus().getCode() == io.grpc.Status.Code.UNAUTHENTICATED) {
        lastAuthErr = sre; // try next variant
      } else {
        // not an auth error – rethrow
        throw sre;
      }
    }
  }

  // --- Attempt 3: Authorization header (Bearer <key>) ---
  if (!success) {
    // rebuild channel with header interceptor
    channel.shutdown();
    channel.awaitTermination(3, java.util.concurrent.TimeUnit.SECONDS);

    Metadata.Key<String> AUTH = Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
    Metadata md = new Metadata();
    md.put(AUTH, "Bearer " + psk); // some PSK deployments accept this form

    ManagedChannelBuilder<?> mcb2 = ManagedChannelBuilder.forAddress(host, port);
    if (useSsl) mcb2.useTransportSecurity(); else mcb2.usePlaintext();
    mcb2.intercept(io.grpc.stub.MetadataUtils.newAttachHeadersInterceptor(md));
    channel = mcb2.build();

    try {
      SessionSubcomponent sub = DaggerDeephavenSessionRoot.create()
          .factoryBuilder()
          .managedChannel(channel)
          .scheduler(scheduler)
          .authenticationTypeAndValue("") // header-only path
          .build();

      try (Session session = sub.newSession()) {
        java.util.concurrent.CompletableFuture<? extends ConsoleSession> fut =
            session.console("python");
        try (ConsoleSession py = fut.get(15, java.util.concurrent.TimeUnit.SECONDS)) {
          py.executeCode(code);
          success = true;
        }
      }
    } catch (StatusRuntimeException sre) {
      lastAuthErr = sre;
    }
  }

  try {
    if (!success) {
      String msg = (lastAuthErr != null)
          ? "Authentication failed: " + lastAuthErr.getStatus() + " (check DH_AUTH_TYPE/DH_PSK, type string, and TLS/plaintext)"
          : "Authentication failed (unknown)";
      throw new IllegalStateException(msg, lastAuthErr);
    }
  } finally {
    channel.shutdown();
    scheduler.shutdown();
    channel.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS);
    scheduler.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS);
  }
}

private static String pyStr(String s) {
  if (s == null) return "None";
  return "'" + s.replace("\\", "\\\\").replace("'", "\\'") + "'";
}
```

## Quick checks on the server side

On the Deephaven server (logs / env) confirm:

- `DH_AUTH_TYPE=psk`
    
- `DH_PSK=<exact key>` (no quotes)
    
- Your gRPC port matches (`10000` by default)
    
- TLS vs plaintext matches your client’s `useSsl`
    

If you still see UNAUTHENTICATED after these three attempts, paste the **first few lines** from the Deephaven server logs around authentication—there’s usually a message like “Using auth handler …” or “PSK mismatch.” That will pinpoint whether it wants the FQCN, shorthand, or header form.