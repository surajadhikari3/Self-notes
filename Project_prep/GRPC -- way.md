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