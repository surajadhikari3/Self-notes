

Perfect — here’s a lean, **working end-to-end** setup where:

- **Angular UI** sends: `topicA`, `topicB`, `joinType`, and columns.
    
- **Spring Boot** posts that as a JSON message to a **Kafka control topic**.
    
- **Deephaven** listens to that control topic, **consumes from the two dynamic topics** using **fixed schemas**, performs the join, and exposes:
    
    - `a_ui`, `b_ui`, `joined_ui` tables for your UI to read via JS API.
        

No registry; schemas are fixed in Deephaven for simplicity.

---

# 1) Spring Boot — REST → Kafka

## `pom.xml`

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId><artifactId>dh-control</artifactId><version>1.0.0</version>
  <properties><java.version>17</java.version></properties>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.kafka</groupId><artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId><artifactId>jackson-databind</artifactId>
    </dependency>
    <dependency>
      <groupId>jakarta.validation</groupId><artifactId>jakarta.validation-api</artifactId>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId><artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

## `application.yml`

```yaml
server:
  port: 8080

dh:
  control-topic: dh.config.commands   # Kafka topic Deephaven listens to

spring:
  kafka:
    bootstrap-servers: pkc-...:9092   # your cluster
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
```

## DTO & Controller

```java
// JoinCommand.java
package com.example.dhcontrol.dto;

import jakarta.validation.constraints.*;
import java.util.List;

public record JoinCommand(
  @NotBlank String commandId,
  @NotBlank String topicA,            // dynamic topic A name
  @NotBlank String topicB,            // dynamic topic B name
  @NotBlank String joinType,          // left | right | inner | exact
  @NotEmpty List<String> onCols,      // ["userId"]
  List<String> joinCols,              // ["balance","accountType"]
  List<String> selectCols             // final projection (optional)
) {}
```

```java
// DhCommandController.java
package com.example.dhcontrol.web;

import com.example.dhcontrol.dto.JoinCommand;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.validation.Valid;
import org.springframework.core.env.Environment;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/dh")
public class DhCommandController {

  private final KafkaTemplate<String,String> kafka;
  private final ObjectMapper mapper;
  private final String controlTopic;

  public DhCommandController(KafkaTemplate<String,String> kafka, ObjectMapper mapper, Environment env) {
    this.kafka = kafka;
    this.mapper = mapper;
    this.controlTopic = env.getProperty("dh.control-topic", "dh.config.commands");
  }

  @PostMapping("/join")
  public String sendJoin(@Valid @RequestBody JoinCommand cmd) throws Exception {
    String payload = mapper.writeValueAsString(cmd);
    kafka.send(controlTopic, cmd.commandId(), payload);
    return "sent:" + cmd.commandId();
  }
}
```

---

# 2) Deephaven — consume control messages → build consumers → join

- **Assumption**: You want **fixed schemas** (simple), but **dynamic topic names** per command.
    
- Edit the two schema maps below to match your payloads.
    

```python
# --- dh_control_join.py (run inside Deephaven) ---
from deephaven.stream.kafka import consumer as kc
from deephaven.experimental.outer_joins import left_outer_join, natural_join, exact_join
from deephaven import dtypes as dt, query_scope, update_graph
import json, os

# ------------- Fixed schemas (edit to your formats) -----------------
# Use these for any topic name provided by the control message
SCHEMA_A = {  # e.g., "users-like" events
    "userId": "string",
    "name": "string",
    "email": "string",
    "age": "int64",
}
SCHEMA_B = {  # e.g., "accounts-like" events
    "userId": "string",
    "accountType": "string",
    "balance": "double",
}
# -------------------------------------------------------------------

KAFKA_CFG = {
    "bootstrap.servers": os.getenv("BOOTSTRAP","pkc-...:9092"),
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.oauthbearer.token.endpoint.url": os.getenv("TOKEN_URL","https://.../token.oauth2"),
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": os.getenv("CLIENT_ID","TestScopeClient"),
    "sasl.oauthbearer.client.secret": os.getenv("CLIENT_SECRET","2Federate"),
    "sasl.oauthbearer.extensions.logicalCluster": os.getenv("LOGICAL_CLUSTER","lkc-ygvwwp"),
    "sasl.oauthbearer.extensions.identityPoolId": os.getenv("IDENTITY_POOL","pool-NRk1"),
    "sasl.endpoint.identification.algorithm": "https",
}

CONTROL_TOPIC = os.getenv("DH_CONTROL_TOPIC","dh.config.commands")
CONTROL = kc.consume(
    {"bootstrap.servers": os.getenv("CTRL_BOOTSTRAP", KAFKA_CFG["bootstrap.servers"])},
    CONTROL_TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=kc.SimpleSpec.STRING,        # Spring sends String JSON
    table_type=kc.TableType.append()
)

def _dtype(t):
    return {
        "string": dt.string, "double": dt.double, "float": dt.float64,
        "int": dt.int32, "int32": dt.int32, "int64": dt.int64, "long": dt.int64,
        "bool": dt.bool_
    }[t.lower()]

def _json_spec(map_spec):
    return kc.json_spec({k: _dtype(v) for k, v in map_spec.items()})

def _consume(topic: str, schema: dict, select=None):
    tbl = kc.consume(
        KAFKA_CFG,
        topic,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=_json_spec(schema),
        table_type=kc.TableType.append()
    )
    return tbl.view(select) if select else tbl

def _do_join(join_type: str, lhs, rhs, on_cols, join_cols):
    jt = join_type.lower()
    if jt == "left":
        return left_outer_join(lhs, rhs, on=on_cols, joins=join_cols or [])
    if jt == "right":
        # simulate right join by swapping sides
        return left_outer_join(rhs, lhs, on=on_cols, joins=join_cols or [])
    if jt == "inner":
        return natural_join(lhs, rhs, on=on_cols, joins=join_cols or [])
    if jt == "exact":
        return exact_join(lhs, rhs, on=on_cols, joins=join_cols or [])
    raise ValueError(f"Unsupported joinType: {join_type}")

_state = {"a": None, "b": None, "joined": None, "lastCommandId": None}

def apply_join_command(cmd_json: str):
    """
    Expected JSON from Spring:
    {
      "commandId":"cmd-123",
      "topicA":"your.dynamic.topicA",
      "topicB":"your.dynamic.topicB",
      "joinType":"left",
      "onCols":["userId"],
      "joinCols":["accountType","balance"],
      "selectCols":["userId","accountType","balance","name","email","age"]
    }
    """
    cmd = json.loads(cmd_json)

    # Consume with fixed schemas, dynamic topic names:
    a = _consume(cmd["topicA"], SCHEMA_A)
    b = _consume(cmd["topicB"], SCHEMA_B)

    j = _do_join(cmd["joinType"], a, b, cmd["onCols"], cmd.get("joinCols"))
    if cmd.get("selectCols"):
        j = j.view(cmd["selectCols"])

    def _swap():
        _state["a"], _state["b"], _state["joined"] = a, b, j
        _state["lastCommandId"] = cmd["commandId"]
        query_scope.expose("a_ui", a)
        query_scope.expose("b_ui", b)
        query_scope.expose("joined_ui", j)
    update_graph.run_later(_swap)

def _on_added(t, added):
    for row in added.to_rows():
        apply_join_command(row["Value"])

LISTENER = CONTROL.update_view("ts = i").last_by("ts").listen(_on_added)
```

> Notes  
> • If _both_ topics share the same schema, you can just use `SCHEMA_A` for both.  
> • If you want to support multiple shapes later, you can add a `schemaSide: "A"|"B"` choice in the command or a small server-side registry.

---

# 3) Angular — send the minimal command

## Service

```ts
// dh-control.service.ts (Angular 16+ / 20.x)
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';

export type JoinType = 'left'|'right'|'inner'|'exact';

@Injectable({ providedIn: 'root' })
export class DhControlService {
  constructor(private http: HttpClient) {}
  sendJoin(cmd: {
    commandId: string;
    topicA: string;   // dynamic topic name
    topicB: string;   // dynamic topic name
    joinType: JoinType;
    onCols: string[];
    joinCols?: string[];
    selectCols?: string[];
  }) {
    return this.http.post<string>('/api/dh/join', cmd);
  }
}
```

## Component (bare-bones UI)

```ts
// dh-join.component.ts
import { Component } from '@angular/core';
import { FormBuilder, Validators } from '@angular/forms';
import { DhControlService } from './dh-control.service';

@Component({
  selector: 'app-dh-join',
  template: `
  <form [formGroup]="f" (ngSubmit)="submit()">
    <input formControlName="commandId" placeholder="commandId">

    <input formControlName="topicA" placeholder="topicA (full Kafka topic)">
    <input formControlName="topicB" placeholder="topicB (full Kafka topic)">

    <select formControlName="joinType">
      <option>left</option><option>right</option><option>inner</option><option>exact</option>
    </select>

    <input formControlName="onCsv"    placeholder="on cols (CSV), e.g. userId">
    <input formControlName="joinCsv"  placeholder="join cols (CSV), e.g. accountType,balance">
    <input formControlName="selectCsv" placeholder="final select (CSV, optional)">

    <button type="submit">Apply</button>
  </form>
  <pre>{{res}}</pre>
  `
})
export class DhJoinComponent {
  res = '';
  f = this.fb.group({
    commandId: ['cmd-1', Validators.required],
    topicA: ['', Validators.required],
    topicB: ['', Validators.required],
    joinType: ['left', Validators.required],
    onCsv: ['userId', Validators.required],
    joinCsv: ['accountType,balance'],
    selectCsv: ['userId,accountType,balance,name,email,age']
  });

  constructor(private fb: FormBuilder, private api: DhControlService) {}

  submit() {
    const v: any = this.f.value;
    const cmd = {
      commandId: v.commandId,
      topicA: v.topicA,
      topicB: v.topicB,
      joinType: v.joinType,
      onCols: (v.onCsv || '').split(',').map((s:string)=>s.trim()).filter(Boolean),
      joinCols: (v.joinCsv || '').split(',').map((s:string)=>s.trim()).filter(Boolean),
      selectCols: (v.selectCsv || '').split(',').map((s:string)=>s.trim()).filter(Boolean),
    };
    this.api.sendJoin(cmd).subscribe({
      next: r => this.res = r,
      error: e => this.res = 'ERROR: ' + (e?.message ?? e)
    });
  }
}
```

## (Optional) Angular subscribe via JS API

```ts
// Minimal idea; adjust to your auth/CORS setup
import { ensureDhClient } from '@deephaven/jsapi-bootstrap';

async function subscribeJoined() {
  const dh = await ensureDhClient({ url: 'http://<dh-host>:10000' });
  const { object } = await dh.getObject({ name: 'joined_ui' }); // exposed by DH
  const table = object; // bind to your table viewer / ag-Grid adapter
}
```

---

## Quick sanity checklist

- **Kafka auth/env** in DH: set `BOOTSTRAP`, `TOKEN_URL`, `CLIENT_ID`, `CLIENT_SECRET`, etc.
    
- **Control topic** consistent in both Spring & DH (`dh.config.commands` by default).
    
- **Schemas** (`SCHEMA_A`, `SCHEMA_B`) match the actual message structure on the two topics you’ll point to.
    
- **CORS/PSK** for JS API: ensure your DH server allows your Angular origin (or use a dev proxy).
    

If you want, paste your two actual topic names and the exact JSON payloads — I’ll tailor `SCHEMA_A` / `SCHEMA_B` precisely so you don’t hit parsing or join type issues.