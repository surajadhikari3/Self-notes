

Gotcha—let’s wire your **Spring endpoint** so it takes the JSON payload and executes the Python that calls `ingestion.create_live_table(...)` on the Deephaven server.

Below is a compact, production-ready slice:

---

### 1) Request models

```java
// Column one-to-one with your FastAPI model
public record ColumnDto(String name, String dtype) {}

// Table open request (what your controller receives)
public record OpenTopicReq(
    String topic,
    String alias,
    List<ColumnDto> columns,
    String bootstrap,           // e.g. "127.0.0.1:19092"
    String tableType,           // "append" | "blink"
    Boolean ignoreKey           // default true if null
) {}
```

---

### 2) Controller (uses your existing path)

```java
@RestController
@RequestMapping("/api/dh")
public class DeephavenController {

  private final DeephavenControlService dh;

  public DeephavenController(DeephavenControlService dh) { this.dh = dh; }

  @PostMapping("/topics")
  public ResponseEntity<?> openTopic(@Valid @RequestBody OpenTopicReq req) {
    dh.openTopic(req);
    return ResponseEntity.ok(Map.of(
        "status", "ok",
        "topic",  req.topic(),
        "alias",  req.alias() != null ? req.alias() : req.topic().replace('.', '_').replace('-', '_')
    ));
  }
}
```

---

### 3) Service: connect via PSK and run Python

This version:

- Reads host/port/psk from Spring config or env.
    
- Tries common PSK header variants (some deployments need the FQCN; most accept `psk <token>`).
    
- Builds the Python that imports `ingestion.create_live_table(...)` and executes it.
    

```java
import io.deephaven.client.impl.*;
import io.deephaven.client.impl.script.ConsoleSession;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.Metadata;
import io.grpc.stub.MetadataUtils;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.List;
import java.util.Map;

@Service
public class DeephavenControlService {

  private final String host;
  private final int port;
  private final String psk;
  private final boolean useTls;

  public DeephavenControlService(
      @Value("${dh.host:127.0.0.1}") String host,
      @Value("${dh.port:10000}") int port,
      @Value("${dh.psk}") String psk,
      @Value("${dh.tls:false}") boolean useTls) {
    this.host = host;
    this.port = port;
    this.psk = psk;
    this.useTls = useTls;
  }

  public void openTopic(OpenTopicReq req) {
    // Basic validation
    if (req.topic() == null || req.topic().isBlank())
      throw new IllegalArgumentException("topic is required");
    if (req.columns() == null || req.columns().isEmpty())
      throw new IllegalArgumentException("columns must contain at least one item");

    // Build Python
    String code = buildPython(req);

    // Connect and execute on the "python" console
    runOnPythonConsole(code);
  }

  /* -------- internals -------- */

  private void runOnPythonConsole(String code) {
    // 3 auth header strategies that cover most PSK deployments
    String[] authHeaderValues = new String[] {
        "io.deephaven.authentication.psk.PskAuthenticationHandler " + psk, // FQCN + token
        "psk " + psk,                                                       // short form
        "Bearer " + psk                                                     // some older stacks accept this
    };

    StatusRuntimeException last = null;

    for (String auth : authHeaderValues) {
      ManagedChannelBuilder<?> base = ManagedChannelBuilder.forAddress(host, port);
      if (useTls) base.useTransportSecurity(); else base.usePlaintext();
      ManagedChannel channel = base.build();

      try {
        // attach Authorization header
        Metadata.Key<String> AUTH = Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
        Metadata md = new Metadata();
        md.put(AUTH, auth);

        ManagedChannel intercepted =
            ManagedChannelBuilder.forAddress(host, port)
                .intercept(MetadataUtils.newAttachHeadersInterceptor(md))
                .usePlaintext(!useTls)
                .useTransportSecurity(useTls)
                .build();

        // session root bound to the intercepted channel
        DeephavenSessionRoot root = DaggerDeephavenSessionRoot.create();
        try (io.deephaven.client.impl.Session sub =
                 root.factoryBuilder()
                     .managedChannel(intercepted)
                     .scheduler(root.newScheduler("dh-client-scheduler"))
                     .authenticationTypeAndValue(auth)      // helps some servers
                     .build()
                     .newSession()) {

          java.util.concurrent.CompletableFuture<? extends ConsoleSession> fut = sub.console("python");
          try (ConsoleSession py = fut.get(15, java.util.concurrent.TimeUnit.SECONDS)) {
            py.executeCode(code).get(30, java.util.concurrent.TimeUnit.SECONDS);
          }
          // success
          return;
        }
      } catch (io.grpc.StatusRuntimeException sre) {
        last = sre;
        if (sre.getStatus().getCode() != io.grpc.Status.Code.UNAUTHENTICATED) {
          throw sre; // different error -> bubble up
        }
        // else try next variant
      } catch (Exception e) {
        throw new RuntimeException("Error executing code on DH console", e);
      }
    }

    if (last != null) {
      throw new IllegalStateException("Authentication failed: " + last.getStatus() +
          " (check DH host/port, PSK value, and TLS/plaintext)");
    } else {
      throw new IllegalStateException("Authentication failed (unknown)");
    }
  }

  private static String pystr(String s) {
    if (s == null) return "None";
    return "'" + s.replace("\\", "\\\\").replace("'", "\\'") + "'";
  }

  private static String pyBool(Boolean b) {
    return (b != null && b) ? "True" : "False";
  }

  private static String dhDtype(String dtype) {
    switch (dtype.toLowerCase()) {
      case "int32":  return "dht.int32";
      case "int64":  return "dht.int64";
      case "float":  return "dht.float32";
      case "double": return "dht.double";
      case "string": return "dht.string";
      case "bool":   return "dht.bool_";
      default: throw new IllegalArgumentException("Unsupported dtype: " + dtype);
    }
  }

  private static String buildSchemaBody(List<ColumnDto> cols) {
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < cols.size(); i++) {
      ColumnDto c = cols.get(i);
      sb.append("    ").append(pystr(c.name())).append(": ").append(dhDtype(c.dtype())).append(",");
      if (i + 1 < cols.size()) sb.append("\n");
    }
    return sb.toString();
  }

  private static String safeAlias(OpenTopicReq r) {
    String alias = r.alias();
    if (alias == null || alias.isBlank()) {
      alias = r.topic().replace(".", "_").replace("-", "_");
    }
    return alias;
  }

  /** Build the exact Python you want to run on the DH server. */
  private static String buildPython(OpenTopicReq r) {
    String alias = safeAlias(r);
    String bootstrap = (r.bootstrap() == null || r.bootstrap().isBlank())
        ? "127.0.0.1:19092" : r.bootstrap();
    String tableType = (r.tableType() == null ? "append" : r.tableType().toLowerCase());

    return ""
        + "from deephaven import dtypes as dht\n"
        + "from ingestion import create_live_table\n"
        + "\n"
        + "_TOPIC = " + pystr(r.topic()) + "\n"
        + "_ALIAS = " + pystr(alias) + "\n"
        + "_BOOTSTRAP = " + pystr(bootstrap) + "\n"
        + "_IGNORE_KEY = " + pyBool(r.ignoreKey() == null ? true : r.ignoreKey()) + "\n"
        + "_TABLE_TYPE = " + pystr(tableType) + "\n"
        + "\n"
        + "_SCHEMA = {\n" + buildSchemaBody(r.columns()) + "\n}\n"
        + "\n"
        + "_tbl_name = create_live_table(\n"
        + "    _TOPIC,\n"
        + "    schema=_SCHEMA,\n"
        + "    alias=_ALIAS,\n"
        + "    bootstrap=_BOOTSTRAP,\n"
        + "    table_type=_TABLE_TYPE,\n"
        + "    ignore_key=_IGNORE_KEY,\n"
        + ")\n"
        + "print('[SPRING] created', _ALIAS, 'from topic', _TOPIC, 'via', _BOOTSTRAP)\n";
  }
}
```

> Ensure your server process has `PYTHONPATH` including `%USERPROFILE%\.deephaven\startup.d\` so `from ingestion import create_live_table` succeeds (you already showed you set that).

---

### 4) Example request payload

```json
POST /api/dh/topics
Content-Type: application/json

{
  "topic": "ccd01_sb_its_esp_tap3507_bishoowcasecurated",
  "alias": "account_data",
  "bootstrap": "127.0.0.1:19092",
  "tableType": "append",
  "ignoreKey": true,
  "columns": [
    { "name": "userId",      "dtype": "string" },
    { "name": "accountType", "dtype": "string" },
    { "name": "balance",     "dtype": "double" }
  ]
}
```

---

### 5) Quick checklist

- DH server is up on the host/port in `application.yml` (or env), **PSK** matches.
    
- `ingestion.py` lives in `~/.deephaven/startup.d/` and DH was launched with that dir on `PYTHONPATH`.
    
- Your Java service uses the **“python”** console (`sub.console("python")`).
    
- Dtype strings in the payload are one of: `int32,int64,float,double,string,bool`.
    

That’s all you need to accept the endpoint input and create the live table from Java via `ingestion.create_live_table(...)`.

------------------------------

Here’s the **Lombok-based equivalent** for the `record` types we discussed earlier — these are useful if you’re using older Spring versions that don’t support Java 16+ `record` syntax or want more control (e.g. validation, Jackson customization):

---

### ✅ Equivalent Lombok POJOs for `ColumnDto` and `OpenTopicReq`

```java
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ColumnDto {
    @NotBlank
    private String name;

    @NotBlank
    private String dtype;
}
```

And:

```java
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OpenTopicReq {

    @NotBlank
    private String topic;

    private String alias;

    @NotEmpty
    private List<ColumnDto> columns;

    private String bootstrap;   // e.g. "127.0.0.1:19092"
    private String tableType;   // "append" or "blink"
    private Boolean ignoreKey;  // defaults true if null
}
```

---

### 💡Notes:

- `@Data` → generates getters/setters, equals/hashCode, and toString.
    
- `@Builder` → lets you easily construct from your controller or tests.
    
- `@NoArgsConstructor` + `@AllArgsConstructor` → required for JSON deserialization by Jackson.
    
- `@NotBlank` and `@NotEmpty` keep your validation constraints consistent with the record version.
    

---

### Example JSON (same as before)

```json
{
  "topic": "ccd01_sb_its_esp_tap3507_bishoowcasecurated",
  "alias": "account_data",
  "bootstrap": "127.0.0.1:19092",
  "tableType": "append",
  "ignoreKey": true,
  "columns": [
    { "name": "userId", "dtype": "string" },
    { "name": "accountType", "dtype": "string" },
    { "name": "balance", "dtype": "double" }
  ]
}
```

---

Would you like me to show the **`application.yml`** snippet that declares `dh.host`, `dh.port`, and `dh.psk` for this setup (so your `@Value` injections in the service auto-populate)?

-------------------------------

Perfect 👍 — here’s the clean, production-ready **`application.yml`** setup for your Spring Boot project so your  
`DeephavenControlService` can automatically pick up host, port, PSK, and TLS configuration:

---

### 🧩 `src/main/resources/application.yml`

```yaml
server:
  port: 8080
  servlet:
    context-path: /

spring:
  application:
    name: deephaven-ingestion-api

logging:
  level:
    root: INFO
    com.td.espfc: DEBUG     # <-- your package, optional for local debugging
    io.deephaven: INFO
    org.springframework.web: INFO

# ----------------------------------------------------------------------
# Deephaven connection configuration
# ----------------------------------------------------------------------
dh:
  host: 127.0.0.1         # or "localhost"
  port: 10000             # matches your Deephaven server port
  psk: ttg5gyi79tna       # PSK secret printed when DH starts
  tls: false              # set to true if you enable TLS in Deephaven

# Optional advanced timeout/retry tuning (if you want)
dh.client:
  connect-timeout-seconds: 15
  request-timeout-seconds: 30
```

---

### ⚙️ How it ties into your service

Your `DeephavenControlService` already includes:

```java
public DeephavenControlService(
    @Value("${dh.host:127.0.0.1}") String host,
    @Value("${dh.port:10000}") int port,
    @Value("${dh.psk}") String psk,
    @Value("${dh.tls:false}") boolean useTls) {
    ...
}
```

✅ This automatically resolves values from the YAML above.  
✅ You can also override them dynamically via environment variables:

```bash
# Linux / Mac
export DH_HOST=10.0.0.5
export DH_PORT=10000
export DH_PSK=your-secret-here
export DH_TLS=false

# Windows PowerShell
$env:DH_HOST="10.0.0.5"
$env:DH_PORT="10000"
$env:DH_PSK="your-secret-here"
$env:DH_TLS="false"
```

---

### 🧪 Quick sanity check

Run:

```bash
curl -s -o NUL -w "HTTP %{http_code}\n" -H "Authorization: psk ttg5gyi79tna" -L "http://127.0.0.1:10000/"
```

→ Should return `HTTP 200`

Then launch your Spring Boot app and POST this:

```bash
curl -X POST http://localhost:8080/api/dh/topics \
  -H "Content-Type: application/json" \
  -d @payload.json
```

---

Would you like me to include a **`application-dev.yml`** and **`application-prod.yml`** variant too (so you can separate localhost vs deployed Deephaven clusters)?