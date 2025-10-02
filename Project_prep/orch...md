

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

-------------------------------------


Awesome—let’s wire the **minimal, end-to-end** flow you asked for:

- one **control topic** message: `{ topicA, topicB, joinType }`
    
- **Deephaven** listens to control, (re)spins two `kc.consume(...)`, builds the join, and exposes three live tables
    
- tiny **Spring Boot** producer to publish the control message
    
- a simple **Angular** service + component to show Topic A, Topic B, and the **joined** table live
    

---

# 1) Spring Boot — publish control command

```java
// build.gradle (or pom.xml equivalents)
implementation "org.springframework.boot:spring-boot-starter-web"
implementation "org.springframework.kafka:spring-kafka"
implementation "com.fasterxml.jackson.core:jackson-databind"
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: your-broker:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer

app:
  control-topic: dh.control
```

```java
// ControlCommand.java
public record ControlCommand(String topicA, String topicB, String joinType) {}
```

```java
// ControlController.java
@RestController
@RequiredArgsConstructor
class ControlController {
  private final KafkaTemplate<String,String> kafka;
  @Value("${app.control-topic}") String controlTopic;
  private final ObjectMapper om = new ObjectMapper();

  @PostMapping("/control")
  public Map<String,String> post(@RequestBody ControlCommand cmd) throws Exception {
    kafka.send(controlTopic, om.writeValueAsString(cmd));
    return Map.of("status","sent","topic",controlTopic);
  }
}
```

Send:

```bash
curl -X POST localhost:8080/control \
  -H 'Content-Type: application/json' \
  -d '{"topicA":"ccd01_sb_..._raw","topicB":"ccd01_sb_..._curated","joinType":"LEFT_OUTER"}'
```

---

# 2) Deephaven script — minimal control + dynamic spin-up

> Assumption (to keep it simple): **Topic A** has the “users” schema; **Topic B** has the “accounts” schema you showed. Adjust the two `*_VALUE_SPEC` maps if your fields differ.

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt, query_scope
from deephaven.experimental.outer_joins import left_outer_join, natural_join, exact_join
import os, json

# ---- Kafka client config (yours) ----
KAFKA_CONFIG = {
    "bootstrap.servers": os.getenv("BOOTSTRAP", "pkc-...:9092"),
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.oauthbearer.token.endpoint.url": os.getenv("TOKEN_URL", "https://.../token.oauth2"),
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": os.getenv("CLIENT_ID","TestScopeClient"),
    "sasl.oauthbearer.client.secret": os.getenv("CLIENT_SECRET","2Federate"),
    "sasl.oauthbearer.extensions.logicalCluster": os.getenv("LOGICAL_CLUSTER","lkc-ygvwwp"),
    "sasl.oauthbearer.extensions.identityPoolId": os.getenv("IDENTITY_POOL","pool-NRk1"),
    "sasl.endpoint.identification.algorithm": "https",
}

CONTROL_TOPIC = os.getenv("CONTROL_TOPIC","dh.control")

# ---- control message schema: {topicA, topicB, joinType} ----
CONTROL_SPEC = kc.json_spec({
    "topicA": dt.string,
    "topicB": dt.string,
    "joinType": dt.string           # LEFT_OUTER | NATURAL | EXACT
})

# ---- fixed value specs for simplicity (adjust if needed) ----
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string, "name": dt.string, "email": dt.string, "age": dt.int64
})
ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string, "accountType": dt.string, "balance": dt.double
})

# ---- keep current live tables here ----
state = {"A": None, "B": None, "J": None}

def _join_by_type(jtype, lhs, rhs):
    jtype = (jtype or "LEFT_OUTER").upper()
    if jtype == "LEFT_OUTER":
        return left_outer_join(lhs, rhs, on=["userId"], joins=["accountType","balance"])
    if jtype == "NATURAL":
        return natural_join(lhs, rhs, on=["userId"], joins=["accountType","balance"])
    if jtype == "EXACT":
        return exact_join(lhs, rhs, on=["userId"], joins=["accountType","balance"])
    raise ValueError(f"Unknown joinType: {jtype}")

def _apply(topicA, topicB, joinType):
    # (re)create two consumers and the join
    a = kc.consume(KAFKA_CONFIG, topicA, key_spec=kc.KeyValueSpec.IGNORE,
                   value_spec=USER_VALUE_SPEC, table_type=kc.TableType.append())
    b = kc.consume(KAFKA_CONFIG, topicB, key_spec=kc.KeyValueSpec.IGNORE,
                   value_spec=ACCOUNT_VALUE_SPEC, table_type=kc.TableType.append())

    a_ui = a.view(["userId","name","email","age"])
    b_ui = b.view(["userId","accountType","balance"])

    j = _join_by_type(joinType, a_ui, b_ui)
    j_ui = j.view(["userId","accountType","balance","name","email","age"])

    state["A"], state["B"], state["J"] = a_ui, b_ui, j_ui

    # expose stable names Angular can query
    query_scope.expose("topicA_ui", a_ui)
    query_scope.expose("topicB_ui", b_ui)
    query_scope.expose("joined_ui", j_ui)
    return "OK"

# ---- consume control topic and apply latest command ----
control_raw = kc.consume(KAFKA_CONFIG, CONTROL_TOPIC,
                         key_spec=kc.KeyValueSpec.IGNORE,
                         value_spec=CONTROL_SPEC,
                         table_type=kc.TableType.append())

def _dispatch(topicA, topicB, joinType):
    try:
        return _apply(topicA, topicB, joinType)
    except Exception as e:
        return f"ERR: {e}"

control_status = control_raw.update(
    "status = _dispatch(topicA, topicB, joinType)"
)

def exposeToAngular():
    # returns TopicA, TopicB, and Joined tables (in that order)
    return (state["A"], state["B"], state["J"])
```

- Post a new control message → Deephaven immediately switches to the two topics and rebuilds the join.
    
- Angular will always find three tables: `topicA_ui`, `topicB_ui`, `joined_ui`.
    

---

# 3) Angular (v20) — display the three live tables

**Install (one time):**

```bash
npm i @deephaven/jsapi-bootstrap @deephaven/jsapi-types ag-grid-community ag-grid-angular
```

**`src/environments/environment.ts`**

```ts
export const environment = {
  production: false,
  dhUrl: 'http://localhost:10000' // your DH IDE/server URL
};
```

**`deephaven.service.ts`**

```ts
import { Injectable } from '@angular/core';
import { ensureDhConnected } from '@deephaven/jsapi-bootstrap';
import type { dh as DhNS } from '@deephaven/jsapi-types';

@Injectable({ providedIn: 'root' })
export class DeephavenService {
  private dh?: typeof DhNS;
  private client?: DhNS.Client;

  async connect(): Promise<void> {
    if (this.client) return;
    const { dhClient, dh } = await ensureDhConnected({ baseUrl: (window as any).env?.dhUrl || '/'}); // base URL auto
    this.dh = dh;
    this.client = dhClient;
  }

  async getTable(name: string) {
    await this.connect();
    if (!this.client || !this.dh) throw new Error('DH not connected');
    const session = await this.client.getAsSession();
    return session.getTable({ name }); // expects query_scope.expose(name,...)
  }
}
```

**`live-join.component.ts`**

```ts
import { Component, OnDestroy, OnInit } from '@angular/core';
import { DeephavenService } from './deephaven.service';
import { ColDef } from 'ag-grid-community';

@Component({
  selector: 'app-live-join',
  template: `
  <div class="grid-wrap">
    <h3>Topic A</h3>
    <ag-grid-angular class="ag-theme-quartz" style="height:250px"
      [rowData]="rowsA" [columnDefs]="colsA"></ag-grid-angular>

    <h3>Topic B</h3>
    <ag-grid-angular class="ag-theme-quartz" style="height:250px"
      [rowData]="rowsB" [columnDefs]="colsB"></ag-grid-angular>

    <h3>Joined</h3>
    <ag-grid-angular class="ag-theme-quartz" style="height:300px"
      [rowData]="rowsJ" [columnDefs]="colsJ"></ag-grid-angular>
  </div>
  `
})
export class LiveJoinComponent implements OnInit, OnDestroy {
  rowsA:any[]=[]; rowsB:any[]=[]; rowsJ:any[]=[];
  colsA:ColDef[]=[{field:'userId'},{field:'name'},{field:'email'},{field:'age'}];
  colsB:ColDef[]=[{field:'userId'},{field:'accountType'},{field:'balance'}];
  colsJ:ColDef[]=[{field:'userId'},{field:'accountType'},{field:'balance'},{field:'name'},{field:'email'},{field:'age'}];

  private subs:any[]=[];

  constructor(private dhs:DeephavenService){}

  async ngOnInit() {
    const [tA,tB,tJ] = await Promise.all([
      this.dhs.getTable('topicA_ui'),
      this.dhs.getTable('topicB_ui'),
      this.dhs.getTable('joined_ui')
    ]);

    // Minimal live viewport loop
    const attach = async (t:any, sink:(rows:any[])=>void) => {
      const vp = await t.setViewport(0, 500); // first 501 rows; adjust as needed
      const sub = t.addEventListener(t.EVENT_UPDATED, () => {
        const r: any[] = [];
        const sz = vp.rows;
        for (let i=0;i<sz;i++){
          r.push(vp.getRow(i));  // plain object per row
        }
        sink(r);
      });
      this.subs.push(()=> t.removeEventListener(t.EVENT_UPDATED, sub));
      // trigger once
      const init: any[] = [];
      for (let i=0;i<vp.rows;i++) init.push(vp.getRow(i));
      sink(init);
    };

    await attach(tA, rows => this.rowsA = rows);
    await attach(tB, rows => this.rowsB = rows);
    await attach(tJ, rows => this.rowsJ = rows);
  }

  ngOnDestroy(){ this.subs.forEach(fn=>fn()); }
}
```

> The component simply binds to three table names you exposed from Deephaven. When you POST a new control message, Deephaven swaps the sources and the grid updates live.

---

## How to use

1. Start Deephaven with the script above.
    
2. Start Spring Boot and POST:
    
    - `{"topicA":"<your users topic>","topicB":"<your accounts topic>","joinType":"LEFT_OUTER"}`
        
3. Open the Angular page → you’ll see **Topic A**, **Topic B**, and **Joined**.  
    Re-POST with different topics or a different `joinType` → the tables update live, no app restarts.
    

If you want to also change schemas later, we can re-introduce a `value_spec_json` per topic—but for now this keeps it dead simple while giving you dynamic topics + join type.

---------------------------------------------

Awesome — let’s wire an **end-to-end flow** where:

- **Spring Boot** pushes the _two topic names_ (and, for now, hard-codes `LEFT_OUTER`).
    
- **Deephaven** listens on a **control topic**, (re)creates two Kafka consumers and the join whenever a new control message arrives, and exposes three tables (`A_ui`, `B_ui`, `J_ui`) at top-level (no `query_scope`).
    
- **Angular (later)** can switch the join type by sending another control message; you won’t need to touch the Deephaven script.
    

Below is everything you need: message schema, a working DH script, a tiny Spring Boot producer, and the (later) Angular call.

---

# 1) Control message shape (single topic to drive everything)

We’ll keep it primitive so it works cleanly with `kc.json_spec`:

```json
{
  "topicA": "ccd01_sb_its_esp_tap3507_bishowocaseraw",
  "topicB": "ccd01_sb_its_esp_tap3507_bishowcasecurated",
  "joinType": "LEFT_OUTER",   // later: NATURAL | EXACT
  "ts": 1730412345123         // epoch millis (optional)
}
```

- For now, Spring Boot will always send `LEFT_OUTER`.
    
- Later, Angular can send the same payload with a different `joinType`.
    

---

# 2) Deephaven script (Option-2: no query_scope, imports fixed)

> Drop this into a single Python script in the DH IDE.  
> It listens to the control topic and (re)builds A, B, and the joined table whenever a new control row arrives.

```python
# --- imports
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt
from deephaven.experimental.outer_joins import left_outer_join, natural_join, exact_join

# --- config
CONTROL_TOPIC = "ccd01_sb_its_esp_tap3567_metadata"   # <— your control topic

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
    "auto.offset.reset": "latest",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;",
    "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.extensions.logicalCluster": "lkc-ygvwwp",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.endpoint.identification.algorithm": "https",
}

# --- value specs for your two data topics
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

# --- control topic spec (primitive only)
CONTROL_SPEC = kc.json_spec({
    "topicA": dt.string,
    "topicB": dt.string,
    "joinType": dt.string,   # LEFT_OUTER | NATURAL | EXACT
    "ts": dt.int64,
})

# ----------------------------
# helpers
# ----------------------------
def _join_by_type(jtype, lhs, rhs):
    j = (jtype or "LEFT_OUTER").upper()
    if j == "LEFT_OUTER":
        return left_outer_join(lhs, rhs, on=["userId"], joins=["accountType", "balance"])
    if j == "NATURAL":
        return natural_join(lhs, rhs, on=["userId"], joins=["accountType", "balance"])
    if j == "EXACT":
        return exact_join(lhs, rhs, on=["userId"], joins=["accountType", "balance"])
    raise ValueError(f"Unknown joinType: {jtype}")

def _apply(topA, topB, joinType):
    """
    (Re)create two consumers and the join.
    Exposes A_ui, B_ui, and J_ui as top-level variables (no query_scope).
    """
    # Recreate consumers (append mode for streaming)
    left_raw = kc.consume(
        KAFKA_CONFIG, topA,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=USER_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )
    right_raw = kc.consume(
        KAFKA_CONFIG, topB,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=ACCOUNT_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )

    # Projections that your UI uses
    left_ui = left_raw.view(["userId", "name", "email", "age"])
    right_ui = right_raw.view(["userId", "accountType", "balance"])

    # Join by requested type (default LEFT_OUTER)
    joined = _join_by_type(joinType, left_ui, right_ui)
    joined_ui = joined.view(["userId", "accountType", "balance", "name", "email", "age"])

    # --- IMPORTANT: bind at top level so Angular/JS can get them directly
    global A_ui, B_ui, J_ui
    A_ui, B_ui, J_ui = left_ui, right_ui, joined_ui
    return "OK"

# ----------------------------
# control stream + dispatcher
# ----------------------------
control_raw = kc.consume(
    KAFKA_CONFIG, CONTROL_TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=CONTROL_SPEC,
    table_type=kc.TableType.append(),
)

# Only react to *changes* (dedupe) and keep the latest row (by ts if present).
# If your control topic is compacted, last_by on the three fields works well.
control_latest = control_raw.last_by(["topicA", "topicB", "joinType"])

# Trigger side-effect to apply config;
# The new status will simply show you the last applied config in a column.
control_status = control_latest.update([
    'status = _apply(topicA, topicB, joinType)'
])

# Optional: a function your Angular can call to fetch the three tables
def exposeToAngular():
    return A_ui, B_ui, J_ui
```

**Why this works well**

- Variables `A_ui`, `B_ui`, `J_ui` are **global**, so they’re directly visible to the JS API and your `exposeToAngular()` without any `query_scope`.
    
- Re-applying the config creates new live consumers; we don’t call `.close()` on the old ones (avoids “table already closed” errors you saw earlier). DH will GC unused references.
    

---

# 3) Spring Boot — send control command

Minimal Spring Boot producer using Spring for Apache Kafka. If you’re on Confluent Cloud OAuth, keep your existing client props; I’ll show core bits only.

**`build.gradle` (or Maven equivalents)**

```groovy
implementation 'org.springframework.boot:spring-boot-starter-web'
implementation 'org.springframework.kafka:spring-kafka'
implementation 'com.fasterxml.jackson.core:jackson-databind'
```

**`application.yml`** (adapt your OAuth props)

```yaml
spring:
  kafka:
    bootstrap-servers: pkc-k13op.canadacentral.azure.confluent.cloud:9092
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: OAUTHBEARER
      sasl.login.callback.handler.class: org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler
      sasl.jaas.config: org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;
      sasl.oauthbearer.token.endpoint.url: https://fedsit.rastest.tdbank.ca/as/token.oauth2
      sasl.oauthbearer.sub.claim.name: client_id
      sasl.oauthbearer.client.id: TestScopeClient
      sasl.oauthbearer.client.secret: 2Federate
      sasl.oauthbearer.extensions.logicalCluster: lkc-ygvwwp
      sasl.oauthbearer.extensions.identityPoolId: pool-NRk1
      ssl.endpoint.identification.algorithm: https
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

app:
  controlTopic: ccd01_sb_its_esp_tap3567_metadata
```

**DTO + Controller**

```java
// ControlCommand.java
public record ControlCommand(String topicA, String topicB, String joinType, Long ts) {}

// ControlController.java
@RestController
@RequiredArgsConstructor
public class ControlController {

  private final KafkaTemplate<String, ControlCommand> kafka;
  @Value("${app.controlTopic}") private String controlTopic;

  @PostMapping("/control")
  public ResponseEntity<?> send(@RequestBody ControlCommand cmd) {
    ControlCommand out = new ControlCommand(
        cmd.topicA(),
        cmd.topicB(),
        cmd.joinType() == null ? "LEFT_OUTER" : cmd.joinType(),
        cmd.ts() == null ? System.currentTimeMillis() : cmd.ts()
    );
    kafka.send(controlTopic, "cfg", out);
    return ResponseEntity.ok(Map.of("status", "sent", "payload", out));
  }
}
```

**How you’ll use it now**

```
POST /control
{
  "topicA": "ccd01_sb_its_esp_tap3507_bishowocaseraw",
  "topicB": "ccd01_sb_its_esp_tap3507_bishowcasecurated"
  // joinType omitted => LEFT_OUTER
}
```

DH will pick it up, build both consumers, and expose `A_ui`, `B_ui`, `J_ui`.

---

# 4) Angular (later) — change join type

When you’re ready to make the join type dynamic, don’t touch DH. Just have Angular call the same endpoint with a different `joinType`:

```ts
// angular service (pseudo)
changeJoinType(joinType: 'LEFT_OUTER' | 'NATURAL' | 'EXACT') {
  return this.http.post('/control', {
    topicA: this.currentTopicA,
    topicB: this.currentTopicB,
    joinType
  });
}
```

The Spring endpoint publishes a new control message; DH receives it and rebuilds the join accordingly.

---

## Notes & gotchas

- **Debouncing control events**: `last_by(["topicA","topicB","joinType"])` ensures we only apply the newest unique combination.
    
- **Schema safety**: topics A and B must match the `USER_VALUE_SPEC` and `ACCOUNT_VALUE_SPEC`. If you switch to different columns later, either (a) add specs to DH, or (b) send a richer control message with a types map and parse it (we can extend easily).
    
- **Resource churn**: The script replaces references without closing old tables, which avoids the “table closed” errors you’ve seen. If you expect hundreds of reconfigs, we can add explicit `.close()` on older tables once the new ones are live.
    
- **Angular data hookup**: keep using your existing `exposeToAngular()` call to fetch the three tables. No extra imports needed.
    

If you want, I can also add **validation + error status** in `control_status` so you can see failures (bad topic names, etc.) right in the DH IDE.

------------------------------------

Here’s a **ready-to-paste** Deephaven script that:

- Listens to a **control topic** for `{ topicA, topicB, joinType }`
    
- Consumes the two data topics
    
- Builds the join using **table methods**: `left_join`, `natural_join`, or `exact_join`
    
- Exposes `A_ui`, `B_ui`, `J_ui` as top-level variables (no `query_scope`)
    
- Provides `exposeToAngular()` for your Angular bridge
    

```python
# =========================
# Deephaven: dynamic 2-topic join (no query_scope)
# =========================
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt

# -------- Kafka & topics --------
CONTROL_TOPIC = "ccd01_sb_its_esp_tap3567_metadata"   # control stream with {topicA, topicB, joinType}

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
    "auto.offset.reset": "latest",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;",
    "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.extensions.logicalCluster": "lkc-ygvwwp",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.endpoint.identification.algorithm": "https",
}

# -------- Value specs (match your data) --------
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

# Control message is primitive-only for easy json_spec
CONTROL_SPEC = kc.json_spec({
    "topicA": dt.string,      # users topic
    "topicB": dt.string,      # accounts topic
    "joinType": dt.string,    # LEFT_OUTER | NATURAL | EXACT
    "ts": dt.int64,           # optional
})

# -------- Helpers --------
def _join_by_type(jtype: str, lhs, rhs, on_cols, join_cols):
    jt = (jtype or "LEFT_OUTER").upper()
    if jt in ("LEFT_OUTER", "LEFT"):
        return lhs.left_join(rhs, on=on_cols, joins=join_cols)
    if jt == "NATURAL":
        return lhs.natural_join(rhs, on=on_cols, joins=join_cols)
    if jt == "EXACT":
        return lhs.exact_join(rhs, on=on_cols, joins=join_cols)
    raise ValueError(f"Unknown joinType: {jtype}")

def _apply(topicA: str, topicB: str, joinType: str):
    """
    (Re)create two consumers and the join.
    Binds A_ui, B_ui, J_ui at module top-level (no query_scope needed).
    """
    # Consumers (append for streaming)
    left_raw = kc.consume(
        KAFKA_CONFIG, topicA,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=USER_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )
    right_raw = kc.consume(
        KAFKA_CONFIG, topicB,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=ACCOUNT_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )

    # Projections
    left_ui  = left_raw.view(["userId", "name", "email", "age"])
    right_ui = right_raw.view(["userId", "accountType", "balance"])

    # Join
    joined   = _join_by_type(joinType, left_ui, right_ui,
                             on_cols=["userId"],
                             join_cols=["accountType", "balance"])

    joined_ui = joined.view(["userId", "accountType", "balance", "name", "email", "age"])

    # Expose as top-level vars
    global A_ui, B_ui, J_ui
    A_ui, B_ui, J_ui = left_ui, right_ui, joined_ui
    return "OK"

# -------- Control stream & dispatcher --------
control_raw = kc.consume(
    KAFKA_CONFIG, CONTROL_TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=CONTROL_SPEC,
    table_type=kc.TableType.append(),
)

# Keep the latest unique configuration
control_latest = control_raw.last_by(["topicA", "topicB", "joinType"])

# Apply on update; 'status' column shows the last apply result
control_status = control_latest.update(['status = _apply(topicA, topicB, joinType)'])

# Optional: for Angular/JS API to fetch all three
def exposeToAngular():
    return A_ui, B_ui, J_ui
```

**Control message example** (from Spring Boot now, Angular later):

```json
{
  "topicA": "ccd01_sb_its_esp_tap3507_bishowocaseraw",
  "topicB": "ccd01_sb_its_esp_tap3507_bishowcasecurated",
  "joinType": "LEFT_OUTER"
}
```

Paste the script, publish the control message, and you’ll get live `A_ui`, `B_ui`, and `J_ui`.

----------------------------------


Got it—here’s a **ready-to-run** Deephaven script using your requirement:

- **LEFT_OUTER** uses the function from `deephaven.experimental.outer_joins`
    
- **NATURAL** and **EXACT** use the **table methods** (`lhs.natural_join(...)`, `lhs.exact_join(...)`)
    
- No `query_scope` needed (Option 2). It also includes control-topic debugging so you can see what DH received.
    

```python
# ================== Imports ==================
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt
from deephaven.experimental.outer_joins import left_outer_join  # only LEFT_OUTER from experimental

# ================== Config ===================
CONTROL_TOPIC = "ccd01_sb_its_esp_tap3567_metadata"  # <— your control topic carrying topicA/topicB/joinType

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
    "auto.offset.reset": "latest",  # for data topics; control uses 'earliest' below
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;",
    "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.extensions.logicalCluster": "lkc-ygvwwp",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.endpoint.identification.algorithm": "https",
}

# Value specs for your two data topics
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

# Control topic schema (primitive fields only)
CONTROL_SPEC = kc.json_spec({
    "topicA": dt.string,
    "topicB": dt.string,
    "joinType": dt.string,   # LEFT_OUTER | NATURAL | EXACT
    "ts": dt.int64,
})

# ================== Join helper ==================
def _join_by_type(jtype: str, lhs, rhs, on_cols, join_cols):
    """
    LEFT_OUTER -> experimental function
    NATURAL / EXACT -> table methods on lhs
    """
    jt = (jtype or "LEFT_OUTER").upper()
    if jt == "LEFT_OUTER":
        return left_outer_join(lhs, rhs, on=on_cols, joins=join_cols)
    if jt == "NATURAL":
        # table method
        return lhs.natural_join(rhs, on=on_cols, joins=join_cols)
    if jt == "EXACT":
        # table method
        return lhs.exact_join(rhs, on=on_cols, joins=join_cols)
    raise ValueError(f"Unknown joinType: {jtype}")

# ================== Apply current config ==================
def _apply(topicA: str, topicB: str, joinType: str):
    """
    (Re)create two consumers and the join.
    Binds A_ui, B_ui, J_ui at module top-level (no query_scope needed).
    """
    # Consumers (append mode for streaming)
    left_raw = kc.consume(
        KAFKA_CONFIG, topicA,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=USER_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )
    right_raw = kc.consume(
        KAFKA_CONFIG, topicB,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=ACCOUNT_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )

    # UI projections
    left_ui = left_raw.view(["userId", "name", "email", "age"])
    right_ui = right_raw.view(["userId", "accountType", "balance"])

    # Join using required API per type
    joined = _join_by_type(joinType, left_ui, right_ui,
                           on_cols=["userId"], join_cols=["accountType", "balance"])
    joined_ui = joined.view(["userId", "accountType", "balance", "name", "email", "age"])

    # Expose as top-level vars (visible to IDE / JS API)
    global A_ui, B_ui, J_ui
    A_ui, B_ui, J_ui = left_ui, right_ui, joined_ui
    return "OK"

# ================== Control stream & dispatcher ==================
# For control only, read from earliest so you catch messages sent before this script started.
CONTROL_KAFKA_CONFIG = dict(KAFKA_CONFIG)
CONTROL_KAFKA_CONFIG["auto.offset.reset"] = "earliest"

control_raw = kc.consume(
    CONTROL_KAFKA_CONFIG, CONTROL_TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=CONTROL_SPEC,
    table_type=kc.TableType.append(),
)

# Debug: tail of control messages so you can verify the payload DH received
control_debug = control_raw.tail(10)

# Keep latest unique config and apply
control_latest = control_raw.last_by(["topicA", "topicB", "joinType"])

def _dispatch(topicA, topicB, joinType):
    try:
        return _apply(topicA, topicB, joinType)
    except Exception as e:
        return f"ERR: {e!s}"

control_status = control_latest.update(['status = _dispatch(topicA, topicB, joinType)'])

# ================== Optional: counts & heads to eyeball data ==================
from deephaven import empty_table
try:
    A_ui
except NameError:
    A_ui = empty_table(0).update_view("userId=(String)null", "name=(String)null",
                                      "email=(String)null", "age=(long)NULL_LONG")
    B_ui = empty_table(0).update_view("userId=(String)null", "accountType=(String)null",
                                      "balance=(double)NaN")
    J_ui = empty_table(0).update_view("userId=(String)null", "accountType=(String)null",
                                      "balance=(double)NaN", "name=(String)null",
                                      "email=(String)null", "age=(long)NULL_LONG")

A_count = A_ui.count_by("rows")
B_count = B_ui.count_by("rows")
J_count = J_ui.count_by("rows")

A_head = A_ui.head(5)
B_head = B_ui.head(5)
J_head = J_ui.head(5)

# Angular/JS bridge
def exposeToAngular():
    return A_ui, B_ui, J_ui
```

### How to verify quickly

1. **Publish a proper JSON object** (not a quoted string) to the control topic:
    
    ```json
    {"topicA":"ccd01_sb_its_esp_tap3507_bishowocaseraw","topicB":"ccd01_sb_its_esp_tap3507_bishowcasecurated","joinType":"LEFT_OUTER","ts":1730412345123}
    ```
    
2. In the DH IDE open:
    
    - `control_debug` → should show your row with non-null `topicA/topicB/joinType`.
        
    - `control_status` → `status` should be `OK` (or an error message if schemas/columns mismatch).
        
    - `A_head`, `B_head`, `J_head` & `A_count/B_count/J_count` to see live data and counts.
        

If you still see an error, it’ll now surface in `control_status.status`. Share that one line and I’ll pinpoint the fix.

--------------------------

Got it—here’s a **ready-to-run** Deephaven script using your requirement:

- **LEFT_OUTER** uses the function from `deephaven.experimental.outer_joins`
    
- **NATURAL** and **EXACT** use the **table methods** (`lhs.natural_join(...)`, `lhs.exact_join(...)`)
    
- No `query_scope` needed (Option 2). It also includes control-topic debugging so you can see what DH received.
    

```python
# ================== Imports ==================
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt
from deephaven.experimental.outer_joins import left_outer_join  # only LEFT_OUTER from experimental

# ================== Config ===================
CONTROL_TOPIC = "ccd01_sb_its_esp_tap3567_metadata"  # <— your control topic carrying topicA/topicB/joinType

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
    "auto.offset.reset": "latest",  # for data topics; control uses 'earliest' below
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;",
    "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.extensions.logicalCluster": "lkc-ygvwwp",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.endpoint.identification.algorithm": "https",
}

# Value specs for your two data topics
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

# Control topic schema (primitive fields only)
CONTROL_SPEC = kc.json_spec({
    "topicA": dt.string,
    "topicB": dt.string,
    "joinType": dt.string,   # LEFT_OUTER | NATURAL | EXACT
    "ts": dt.int64,
})

# ================== Join helper ==================
def _join_by_type(jtype: str, lhs, rhs, on_cols, join_cols):
    """
    LEFT_OUTER -> experimental function
    NATURAL / EXACT -> table methods on lhs
    """
    jt = (jtype or "LEFT_OUTER").upper()
    if jt == "LEFT_OUTER":
        return left_outer_join(lhs, rhs, on=on_cols, joins=join_cols)
    if jt == "NATURAL":
        # table method
        return lhs.natural_join(rhs, on=on_cols, joins=join_cols)
    if jt == "EXACT":
        # table method
        return lhs.exact_join(rhs, on=on_cols, joins=join_cols)
    raise ValueError(f"Unknown joinType: {jtype}")

# ================== Apply current config ==================
def _apply(topicA: str, topicB: str, joinType: str):
    """
    (Re)create two consumers and the join.
    Binds A_ui, B_ui, J_ui at module top-level (no query_scope needed).
    """
    # Consumers (append mode for streaming)
    left_raw = kc.consume(
        KAFKA_CONFIG, topicA,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=USER_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )
    right_raw = kc.consume(
        KAFKA_CONFIG, topicB,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=ACCOUNT_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )

    # UI projections
    left_ui = left_raw.view(["userId", "name", "email", "age"])
    right_ui = right_raw.view(["userId", "accountType", "balance"])

    # Join using required API per type
    joined = _join_by_type(joinType, left_ui, right_ui,
                           on_cols=["userId"], join_cols=["accountType", "balance"])
    joined_ui = joined.view(["userId", "accountType", "balance", "name", "email", "age"])

    # Expose as top-level vars (visible to IDE / JS API)
    global A_ui, B_ui, J_ui
    A_ui, B_ui, J_ui = left_ui, right_ui, joined_ui
    return "OK"

# ================== Control stream & dispatcher ==================
# For control only, read from earliest so you catch messages sent before this script started.
CONTROL_KAFKA_CONFIG = dict(KAFKA_CONFIG)
CONTROL_KAFKA_CONFIG["auto.offset.reset"] = "earliest"

control_raw = kc.consume(
    CONTROL_KAFKA_CONFIG, CONTROL_TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=CONTROL_SPEC,
    table_type=kc.TableType.append(),
)

# Debug: tail of control messages so you can verify the payload DH received
control_debug = control_raw.tail(10)

# Keep latest unique config and apply
control_latest = control_raw.last_by(["topicA", "topicB", "joinType"])

def _dispatch(topicA, topicB, joinType):
    try:
        return _apply(topicA, topicB, joinType)
    except Exception as e:
        return f"ERR: {e!s}"

control_status = control_latest.update(['status = _dispatch(topicA, topicB, joinType)'])

# ================== Optional: counts & heads to eyeball data ==================
from deephaven import empty_table
try:
    A_ui
except NameError:
    A_ui = empty_table(0).update_view("userId=(String)null", "name=(String)null",
                                      "email=(String)null", "age=(long)NULL_LONG")
    B_ui = empty_table(0).update_view("userId=(String)null", "accountType=(String)null",
                                      "balance=(double)NaN")
    J_ui = empty_table(0).update_view("userId=(String)null", "accountType=(String)null",
                                      "balance=(double)NaN", "name=(String)null",
                                      "email=(String)null", "age=(long)NULL_LONG")

A_count = A_ui.count_by("rows")
B_count = B_ui.count_by("rows")
J_count = J_ui.count_by("rows")

A_head = A_ui.head(5)
B_head = B_ui.head(5)
J_head = J_ui.head(5)

# Angular/JS bridge
def exposeToAngular():
    return A_ui, B_ui, J_ui
```

### How to verify quickly

1. **Publish a proper JSON object** (not a quoted string) to the control topic:
    
    ```json
    {"topicA":"ccd01_sb_its_esp_tap3507_bishowocaseraw","topicB":"ccd01_sb_its_esp_tap3507_bishowcasecurated","joinType":"LEFT_OUTER","ts":1730412345123}
    ```
    
2. In the DH IDE open:
    
    - `control_debug` → should show your row with non-null `topicA/topicB/joinType`.
        
    - `control_status` → `status` should be `OK` (or an error message if schemas/columns mismatch).
        
    - `A_head`, `B_head`, `J_head` & `A_count/B_count/J_count` to see live data and counts.
        

If you still see an error, it’ll now surface in `control_status.status`. Share that one line and I’ll pinpoint the fix.
-----------------------


Great questions. Here’s how to **verify** your globals / `exposeToAngular()` on the Deephaven side, and how to **fetch the dynamic tables from Angular** using the supported JS API (via `@deephaven/jsapi-bootstrap`). No `dh-core.js` paths needed.

---

# 1) Verify in Deephaven IDE (server side)

After your Python script runs, you should see these **tables** in the IDE’s Variables pane:

- `A_ui`, `B_ui`, `J_ui`
    
- `control_debug`, `control_status`, etc.
    

Also add this at the very end of your script (purely for sanity):

```python
A_exists = str('A_ui' in globals())
B_exists = str('B_ui' in globals())
J_exists = str('J_ui' in globals())
```

Open `A_exists/B_exists/J_exists` → should be `"True"` once the control message applies.

You can also execute inside the session:

```python
# quick check inside DH:
A_ui.head(5)
B_ui.head(5)
J_ui.head(5)
```

If those open with rows, you’re golden.

---

# 2) Call `exposeToAngular()` from JS and bind results to named variables

When you call a Python function from JS, the easiest way to receive the tables with **stable names** is to run a tiny Python snippet that assigns the tuple to variables. Do this from your Angular app.

---

# 3) Angular: connect & fetch tables (using `@deephaven/jsapi-bootstrap`)

> This works with current Deephaven Web. It does **not** use `/jsapi/dh-core`. Use the bootstrap package.

```ts
// dh.service.ts
import { Injectable } from '@angular/core';
import { getDeephaven } from '@deephaven/jsapi-bootstrap';

@Injectable({ providedIn: 'root' })
export class DhService {
  private dh: any;
  private client: any;
  private session: any;

  async connect(url = 'http://localhost:10000'): Promise<void> {
    this.dh = await getDeephaven();                         // loads Deephaven JS API
    this.client = await this.dh.Client.connect({ url });    // opens WS connection
    // Start a Python console session (or reuse if already created)
    this.session = await this.client.getConsole('python');  // some builds: await this.client.getConsole({ language: 'python' })
  }

  /** Optional: list variables to verify globals exist */
  async listVariables(): Promise<any[]> {
    const vars = await this.session.getVariables();
    // Each entry typically has fields like `title`, `type`, `id`.
    return vars;
  }

  /** Bind exposeToAngular() results to *named* globals in the session */
  async bindAngularTuple(): Promise<void> {
    // This line runs in the server Python session:
    // it calls your function and assigns the tuple to named variables.
    await this.session.runCode('A_ui, B_ui, J_ui = exposeToAngular()');
  }

  /** Get a table handle by variable name */
  async getTableByName(varName: 'A_ui' | 'B_ui' | 'J_ui') {
    // Two common APIs across versions:
    // 1) getObject + asTable()
    // 2) getTable directly (newer)
    try {
      // newer:
      return await this.session.getTable(varName);
    } catch {
      // fallback:
      const obj = await this.session.getObject(varName);
      return obj.asTable(); // returns Table
    }
  }
}
```

Usage (e.g., in a component):

```ts
// example.component.ts
import { Component, OnInit } from '@angular/core';
import { DhService } from './dh.service';

@Component({
  selector: 'app-example',
  template: `<div>Loaded: {{loaded}}</div>`
})
export class ExampleComponent implements OnInit {
  loaded = false;

  constructor(private dh: DhService) {}

  async ngOnInit() {
    await this.dh.connect('http://localhost:10000');

    // (1) See variables (A_ui, B_ui, J_ui should appear after control message)
    const vars = await this.dh.listVariables();
    console.log('DH variables:', vars.map(v => ({ title: v.title, type: v.type })));

    // (2) Ensure the tuple is bound to names
    await this.dh.bindAngularTuple();

    // (3) Fetch tables
    const A = await this.dh.getTableByName('A_ui');
    const B = await this.dh.getTableByName('B_ui');
    const J = await this.dh.getTableByName('J_ui');

    console.log('A table', A);
    console.log('B table', B);
    console.log('J table', J);

    // Now you can adaptors to AG Grid or snapshot for debug:
    const AData = await A.snapshot(); // small snapshot for debug
    console.log('A snapshot', AData);

    this.loaded = true;
  }
}
```

> If your build doesn’t have `session.getTable(name)`, the fallback using `getObject(name).asTable()` will work.

---

# 4) Common pitfalls & fixes

1. **You can’t see vars / function**
    
    - Make sure your Python script is **running** in the same session your client connects to. With default setup, there’s one shared session; the code above connects to it.
        
    - Ensure `exposeToAngular()` is defined at the **top level** (not inside another function).
        
2. **`/jsapi/dh-core` 404**
    
    - Don’t call this path. It’s from the old, direct bundle. Use `@deephaven/jsapi-bootstrap` as above; it fetches the correct assets automatically.
        
3. **Control topic applied, but J_ui empty**
    
    - Open `control_status` table in DH IDE; check the `status` column (it will show `ERR: ...` if your join failed).
        
    - Verify `A_head` & `B_head` have rows and share `userId`.
        
    - Confirm column names/types match the `USER_VALUE_SPEC` / `ACCOUNT_VALUE_SPEC`.
        
4. **Multiple applies**
    
    - The script uses `last_by([topicA, topicB, joinType])`, so only the latest unique combo applies. Send a fresh message to re-apply when testing.
        

---

# 5) Minimal “smoke test” from the browser console (no Angular)

If you want to quickly test outside Angular:

```js
(async () => {
  const dh = await window.deephaven.loadModule('@deephaven/jsapi-bootstrap').then(m => m.getDeephaven());
  const client = await dh.Client.connect({ url: 'http://localhost:10000' });
  const session = await client.getConsole('python');
  await session.runCode('A_ui, B_ui, J_ui = exposeToAngular()');
  const A = await (session.getTable ? session.getTable('A_ui') : session.getObject('A_ui').then(o => o.asTable()));
  console.log('A rows snapshot', await A.snapshot());
})();
```

(If `window.deephaven.loadModule` isn’t available, just use the Angular/TypeScript version above.)

---

## TL;DR

- Don’t use `/jsapi/dh-core`; use `@deephaven/jsapi-bootstrap`.
    
- In JS, call `session.runCode('A_ui, B_ui, J_ui = exposeToAngular()')`, then fetch each by name with `getTable('A_ui')` or `getObject('A_ui').asTable()`.
    
- Verify on the server by opening `control_debug`, `control_status`, and `A_head/B_head/J_head` in the IDE.
    

If you paste your current package versions (`@deephaven/jsapi-bootstrap`, `@deephaven/jsapi-types`) I can tailor the `getTable` vs `getObject().asTable()` line to match exactly.


-----------------------------


You’re running into a **session boundary**.

- The Deephaven script that builds `first_scope_ui`, `second_scope_ui`, and `final_ui` is running in **one Python console session**.
    
- Your Angular code creates a **new** session with `startSession('python')`. New sessions can’t “see” variables from other sessions, so `getTable('first_scope_ui')` fails even if you just called `exposeToAngular()`—because that function lives in the other session.
    

You have three solid ways to fix this. Pick one.

---

## ✅ Fix 1 (best): Attach to the same IDE console instead of starting a new one

Change your connection code to attach to the existing IDE Python console:

```ts
// before:
// const ideConn = await client.getAsIdeConnection();
// const ide = await ideConn.startSession('python');

// after:
let ide: any;
try {
  // Works on most JSAPI builds: attaches to the IDE's running Python console
  ide = await this.client.getConsole('python');
} catch {
  // Fallback if your build doesn’t have getConsole()
  const ideConn = await this.client.getAsIdeConnection();
  const consoles = await ideConn.listConsoles?.(); // if available
  const py = consoles?.find((c: any) => c.type === 'python');
  ide = py ? await ideConn.getConsole(py.id) : await ideConn.startSession('python');
}
```

Now you’re in the **same session** as the script. You can just:

```ts
await ide.runCode('first_scope_ui, second_scope_ui, final_ui = exposeToAngular()');
const t1 = await ide.getTable('first_scope_ui');
const t2 = await ide.getTable('second_scope_ui');
const t3 = await ide.getTable('final_ui');
```

---

## ✅ Fix 2: Publish the tables via App Mode (session-agnostic)

In your DH Python script (the one that builds the tables), add:

```python
from deephaven.appmode import Application

app = Application("angular_bridge") \
    .with_table("first_scope_ui", first_scope_ui) \
    .with_table("second_scope_ui", second_scope_ui) \
    .with_table("final_ui", final_ui)
```

Then fetch by name from JS without worrying about the original session (exact JS call varies slightly by JSAPI version; if `client.getObject` is present):

```ts
const t1 = await this.client.getObject({ name: 'first_scope_ui' });
const t2 = await this.client.getObject({ name: 'second_scope_ui' });
const t3 = await this.client.getObject({ name: 'final_ui' });
```

If your JSAPI doesn’t expose `getObject`, stick with **Fix 1**.

---

## ✅ Fix 3: Build a globals index in your session (robust fallback)

Run once in your Angular session to create a table listing all global tables **in that session**:

```ts
await ide.runCode(`
from deephaven.table import Table
from deephaven import new_table, Column
def _globals_table_index():
    names, kinds = [], []
    from deephaven.table import Table as _T
    for n, v in globals().items():
        try:
            if isinstance(v, _T):
                names.append(n); kinds.append("Table")
        except: pass
    return new_table([Column("name", names), Column("kind", kinds)])
__dh_globals__ = _globals_table_index()
`);

const idx = await ide.getTable('__dh_globals__');
const vp = await idx.getViewportData();
const names: string[] = [];
for (let i = 0; i < vp.rows.length; i++) names.push(vp.rows[i].get('name'));
```

…but remember, if you started a new session, those names won’t include the tables from the original script. Combine with **Fix 1** so you introspect the right session.

---

## Quick check you can add right now

Right after you call `exposeToAngular()` in your Angular code, ask Python to confirm the variables exist in **that** session:

```ts
await ide.runCode(`
__exists__ = {
  "first": "first_scope_ui" in globals(),
  "second": "second_scope_ui" in globals(),
  "final": "final_ui" in globals(),
}
`);
const exists = await ide.getTable('__exists__'); // If your JSAPI renders dicts as tables; otherwise log in Python
```

If any are `false`, you’re in the wrong session → apply **Fix 1**.

---

### Why your current code still fails

You’re doing:

```ts
const ide = await ideConn.startSession('python');
await ide.getTable(tableName)   // fails
catch { await ide.runCode('first_scope_ui, second_scope_ui, final_ui = exposeToAngular()') }
```

`exposeToAngular()` is defined in the **other** session. Calling it here won’t work unless you attach to that original session. That’s exactly what **Fix 1** does.

---

If you share which JSAPI methods are available on your `CoreClient` (e.g., whether `getConsole`, `listConsoles`, or `getObject` exist), I can drop in the exact minimal code for your version.

------------------

Here’s a **single Deephaven IDE script** you can paste and run.  
It creates the three globals `first_ui`, `second_ui`, `final_ui` **up-front** (as empty placeholders), then replaces them when your control message arrives. It also defines `exposeToAngular()` so your Angular code can bind them into its session.

> No App Mode. No query_scope. Just globals in the IDE session.

```python
# ================== Imports ==================
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt, new_table, Column
from deephaven.experimental.outer_joins import left_outer_join  # only LEFT_OUTER from experimental

# ================== Config ===================
CONTROL_TOPIC = "ccd01_sb_its_esp_tap3507_metadata"  # control topic that sends topicA/topicB/joinType

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
    "auto.offset.reset": "latest",  # for data topics; control will use 'earliest' below
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;",
    "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.extensions.logicalCluster": "lkc-ygvwwp",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.endpoint.identification.algorithm": "https",
}

# Value specs for your two data topics
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

# Control topic schema (primitive fields only)
CONTROL_SPEC = kc.json_spec({
    "topicA": dt.string,
    "topicB": dt.string,
    "joinType": dt.string,   # LEFT_OUTER | NATURAL | EXACT
    "ts": dt.int64,
})

# ================== Placeholders (so variables ALWAYS exist) ==================
def _empty_users():
    return new_table([Column("userId", []), Column("name", []), Column("email", []), Column("age", [])])

def _empty_accounts():
    return new_table([Column("userId", []), Column("accountType", []), Column("balance", [])])

def _empty_joined():
    return new_table([
        Column("userId", []), Column("accountType", []), Column("balance", []),
        Column("name", []), Column("email", []), Column("age", []),
    ])

# Create the three globals immediately
first_ui  = _empty_users()
second_ui = _empty_accounts()
final_ui  = _empty_joined()

# Helper Angular/JS will call to bind these names into its session
def exposeToAngular():
    return first_ui, second_ui, final_ui

# ================== Join helper ==================
def _join_by_type(jtype: str, lhs, rhs, on_cols, join_cols):
    """
    LEFT_OUTER -> experimental function
    NATURAL / EXACT -> table methods on lhs
    """
    jt = (jtype or "LEFT_OUTER").upper()
    if jt == "LEFT_OUTER":
        return left_outer_join(lhs, rhs, on=on_cols, joins=join_cols)
    if jt == "NATURAL":
        return lhs.natural_join(rhs, on=on_cols, joins=join_cols)
    if jt == "EXACT":
        return lhs.exact_join(rhs, on=on_cols, joins=join_cols)
    raise ValueError(f"Unknown joinType: {jtype}")

# ================== Apply current config ==================
def _apply(topicA: str, topicB: str, joinType: str):
    """
    1) Create/replace the two consumers
    2) Build projections
    3) Build the join
    4) Update the three GLOBALS (first_ui, second_ui, final_ui)
    """
    # 1) Consumers (append mode for streaming)
    left_raw = kc.consume(
        KAFKA_CONFIG, topicA,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=USER_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )
    right_raw = kc.consume(
        KAFKA_CONFIG, topicB,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=ACCOUNT_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )

    # 2) Projections
    left_ui = left_raw.view(["userId", "name", "email", "age"])
    right_ui = right_raw.view(["userId", "accountType", "balance"])

    # 3) Join (default LEFT_OUTER; NATURAL/EXACT use table methods)
    joined = _join_by_type(joinType, left_ui, right_ui,
                           on_cols=["userId"], join_cols=["accountType", "balance"])
    joined_ui = joined.view(["userId", "accountType", "balance", "name", "email", "age"])

    # 4) Update GLOBALS so Angular can fetch them
    global first_ui, second_ui, final_ui
    first_ui, second_ui, final_ui = left_ui, right_ui, joined_ui

    return "OK"

# ================== Control stream & dispatcher ==================
# For control only, read from earliest so you catch messages sent before this script started.
CONTROL_KAFKA_CONFIG = dict(KAFKA_CONFIG)
CONTROL_KAFKA_CONFIG["auto.offset.reset"] = "earliest"

control_raw = kc.consume(
    CONTROL_KAFKA_CONFIG, CONTROL_TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=CONTROL_SPEC,
    table_type=kc.TableType.append(),
)

# Debug: tail of control messages so you can verify the payload DH received
control_debug = control_raw.tail(10)

# Keep latest unique config and apply
control_latest = control_raw.last_by(["topicA", "topicB", "joinType"])

def _dispatch(topicA, topicB, joinType):
    try:
        return _apply(topicA, topicB, joinType)
    except Exception as e:
        return f"ERR: {e!s}"

control_status = control_latest.update(['status = _dispatch(topicA, topicB, joinType)'])

# Optional quick samples/counters, useful to eyeball in IDE
A_count = first_ui.count_by("rows")
B_count = second_ui.count_by("rows")
J_count = final_ui.count_by("rows")

A_head = first_ui.head(5)
B_head = second_ui.head(5)
J_head = final_ui.head(5)
```

### How Angular should fetch them

Attach to the **existing IDE Python console**, then bind & get:

```ts
// attach to IDE python (don’t create a new session)
const ide = await (client as any).getConsole?.('python')
  ?? (await client.getAsIdeConnection()).startSession('python'); // fallback

// bind the three names into THIS session from the IDE globals
await ide.runCode('first_ui, second_ui, final_ui = exposeToAngular()');

// fetch
const [A, B, J] = await Promise.all([
  ide.getTable('first_ui'),
  ide.getTable('second_ui'),
  ide.getTable('final_ui'),
]);
```

### Why this works

- The three names **always exist** (placeholders first).
    
- When your control message arrives, `_apply(...)` **replaces the globals** with live streaming tables.
    
- `exposeToAngular()` lets your client rebind the names inside its own session and call `getTable(...)` successfully.




-------------------------------

Here you go — **complete, ready-to-run Deephaven IDE script** with the fixed, **typed placeholder columns** (using `empty_table().update_view(...)`). It exposes the three globals `first_ui`, `second_ui`, `final_ui` immediately, then replaces them when a control message arrives. No App Mode, no `query_scope`.

```python
# ================== Imports ==================
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt, empty_table
from deephaven.experimental.outer_joins import left_outer_join  # LEFT_OUTER from experimental

# ================== Config ===================
CONTROL_TOPIC = "ccd01_sb_its_esp_tap3507_metadata"  # control topic with topicA/topicB/joinType

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
    "auto.offset.reset": "latest",  # data topics use latest; control uses 'earliest' below
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;",
    "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.extensions.logicalCluster": "lkc-ygvwwp",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.endpoint.identification.algorithm": "https",
}

# Value specs for your two data topics (adjust field names if your JSON differs)
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

# Control topic schema (primitive fields only)
CONTROL_SPEC = kc.json_spec({
    "topicA": dt.string,
    "topicB": dt.string,
    "joinType": dt.string,   # LEFT_OUTER | NATURAL | EXACT
    "ts": dt.int64,
})

# ================== Typed placeholder tables ==================
def _empty_users():
    # Defines columns & types even with 0 rows
    return empty_table(0).update_view(
        "userId=(String)null",
        "name=(String)null",
        "email=(String)null",
        "age=(long)NULL_LONG",
    )

def _empty_accounts():
    return empty_table(0).update_view(
        "userId=(String)null",
        "accountType=(String)null",
        "balance=(double)NaN",
    )

def _empty_joined():
    return empty_table(0).update_view(
        "userId=(String)null",
        "accountType=(String)null",
        "balance=(double)NaN",
        "name=(String)null",
        "email=(String)null",
        "age=(long)NULL_LONG",
    )

# Create the three globals immediately so Angular can always open them
first_ui  = _empty_users()
second_ui = _empty_accounts()
final_ui  = _empty_joined()

# Helper Angular/JS will call to bind these names into its own session
def exposeToAngular():
    return first_ui, second_ui, final_ui

# ================== Join helper ==================
def _join_by_type(jtype: str, lhs, rhs, on_cols, join_cols):
    """
    LEFT_OUTER -> experimental function
    NATURAL / EXACT -> table methods on lhs (as you requested)
    """
    jt = (jtype or "LEFT_OUTER").upper()
    if jt == "LEFT_OUTER":
        return left_outer_join(lhs, rhs, on=on_cols, joins=join_cols)
    if jt == "NATURAL":
        return lhs.natural_join(rhs, on=on_cols, joins=join_cols)
    if jt == "EXACT":
        return lhs.exact_join(rhs, on=on_cols, joins=join_cols)
    raise ValueError(f"Unknown joinType: {jtype}")

# ================== Apply current config ==================
def _apply(topicA: str, topicB: str, joinType: str):
    """
    1) Create/replace the two consumers
    2) Build projections
    3) Build the join
    4) Update the three GLOBALS (first_ui, second_ui, final_ui)
    """
    # 1) Consumers (append mode for streaming)
    left_raw = kc.consume(
        KAFKA_CONFIG, topicA,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=USER_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )
    right_raw = kc.consume(
        KAFKA_CONFIG, topicB,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=ACCOUNT_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )

    # 2) Projections
    left_proj = left_raw.view(["userId", "name", "email", "age"])
    right_proj = right_raw.view(["userId", "accountType", "balance"])

    # 3) Join
    joined = _join_by_type(joinType, left_proj, right_proj,
                           on_cols=["userId"], join_cols=["accountType", "balance"])
    joined_proj = joined.view(["userId", "accountType", "balance", "name", "email", "age"])

    # 4) Update GLOBALS so Angular can fetch them
    global first_ui, second_ui, final_ui
    first_ui, second_ui, final_ui = left_proj, right_proj, joined_proj

    return "OK"

# ================== Control stream & dispatcher ==================
# Control consumer should read from earliest so you catch messages sent before this script started.
CONTROL_KAFKA_CONFIG = dict(KAFKA_CONFIG)
CONTROL_KAFKA_CONFIG["auto.offset.reset"] = "earliest"

control_raw = kc.consume(
    CONTROL_KAFKA_CONFIG, CONTROL_TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=CONTROL_SPEC,
    table_type=kc.TableType.append(),
)

# Debug: last few control messages so you can verify payload DH received
control_debug = control_raw.tail(10)

# Keep latest unique config and apply
control_latest = control_raw.last_by(["topicA", "topicB", "joinType"])

def _dispatch(topicA, topicB, joinType):
    try:
        return _apply(topicA, topicB, joinType)
    except Exception as e:
        return f"ERR: {e!s}"

control_status = control_latest.update(['status = _dispatch(topicA, topicB, joinType)'])

# ================== Optional quick samples/counters ==================
A_count = first_ui.count_by("rows")
B_count = second_ui.count_by("rows")
J_count = final_ui.count_by("rows")

A_head = first_ui.head(5)
B_head = second_ui.head(5)
J_head = final_ui.head(5)
```

### How to consume from Angular

1. **Attach to the existing IDE Python console** (don’t create a fresh one):
    

```ts
const ide = await (client as any).getConsole?.('python')
  ?? (await client.getAsIdeConnection()).startSession('python'); // fallback if .getConsole isn't available
```

2. **Bind and fetch**:
    

```ts
await ide.runCode('first_ui, second_ui, final_ui = exposeToAngular()');
const [A, B, J] = await Promise.all([
  ide.getTable('first_ui'),
  ide.getTable('second_ui'),
  ide.getTable('final_ui'),
]);
```

Now your three tables are guaranteed to have columns from the start (typed placeholders), and they’ll swap to live streaming once your control message is consumed.

---------------------------------------------------------------


Here’s a **clean, copy-paste** App Mode setup that (1) always exports placeholders immediately so Panels/JS never fail, (2) consumes your control topic, (3) builds two Kafka streams + the join, and (4) rebinds the exports safely. The earlier `update_view` error is fixed.

---

# 1) `app/stream_join.app`

```
type=script
scriptType=python
enabled=true
id=stream.join.demo
name=Stream Join Demo
file=stream_join_app.py
```

---

# 2) `app/stream_join_app.py`

```python
# ----------------- Imports -----------------
from typing import Callable
from deephaven.appmode import ApplicationState, initialize
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt, empty_table, new_table, Column
from deephaven.experimental.outer_joins import left_outer_join

# ----------------- Config ------------------
CONTROL_TOPIC = "ccd01_sb_its_esp_tap3507_metadata"

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
    "auto.offset.reset": "latest",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
    "sasl.jaas.config": "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required;",
    "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
    "sasl.oauthbearer.sub.claim.name": "client_id",
    "sasl.oauthbearer.client.id": "TestScopeClient",
    "sasl.oauthbearer.client.secret": "2Federate",
    "sasl.oauthbearer.extensions.logicalCluster": "lkc-ygvwwp",
    "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
    "sasl.endpoint.identification.algorithm": "https",
}

USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

CONTROL_SPEC = kc.json_spec({
    "topicA": dt.string,
    "topicB": dt.string,
    "joinType": dt.string,  # LEFT_OUTER | NATURAL | EXACT
    "ts": dt.int64,
})

# ----------------- Placeholder table builders (fixed update_view) -----------------
def _empty_users():
    return empty_table(0).update_view([
        "userId=(String)null",
        "name=(String)null",
        "email=(String)null",
        "age=(long)NULL_LONG",
    ])

def _empty_accounts():
    return empty_table(0).update_view([
        "userId=(String)null",
        "accountType=(String)null",
        "balance=(double)NaN",
    ])

def _empty_joined():
    return empty_table(0).update_view([
        "userId=(String)null",
        "accountType=(String)null",
        "balance=(double)NaN",
        "name=(String)null",
        "email=(String)null",
        "age=(long)NULL_LONG",
    ])

# ----------------- Utilities -----------------
def _join_by_type(jtype: str, lhs, rhs, on_cols, join_cols):
    """
    LEFT_OUTER uses experimental function; NATURAL/EXACT use table methods.
    """
    jt = (jtype or "LEFT_OUTER").upper()
    if jt == "LEFT_OUTER":
        return left_outer_join(lhs, rhs, on=on_cols, joins=join_cols)
    if jt == "NATURAL":
        return lhs.natural_join(rhs, on=on_cols, joins=join_cols)
    if jt == "EXACT":
        return lhs.exact_join(rhs, on=on_cols, joins=join_cols)
    raise ValueError(f"Unknown joinType: {jtype}")

def _export_tables(app: ApplicationState, A, B, J):
    """Idempotently (re)bind the three exports."""
    app["first_scope_ui"]  = A
    app["second_scope_ui"] = B
    app["final_ui"]        = J

# ----------------- Build & Apply -----------------
def _apply(topA: str, topB: str, joinType: str, app: ApplicationState):
    """
    Create/replace the two consumers and the join, then re-export the three UI tables.
    On failure, keep previous exports and surface the error in control_status/last_error.
    """
    try:
        left_raw = kc.consume(
            KAFKA_CONFIG, topA,
            key_spec=kc.KeyValueSpec.IGNORE,
            value_spec=USER_VALUE_SPEC,
            table_type=kc.TableType.append(),
        )
        right_raw = kc.consume(
            KAFKA_CONFIG, topB,
            key_spec=kc.KeyValueSpec.IGNORE,
            value_spec=ACCOUNT_VALUE_SPEC,
            table_type=kc.TableType.append(),
        )

        A = left_raw.view(["userId", "name", "email", "age"])
        B = right_raw.view(["userId", "accountType", "balance"])

        J = _join_by_type(joinType, A, B, on_cols=["userId"], join_cols=["accountType", "balance"])
        J = J.view(["userId", "accountType", "balance", "name", "email", "age"])

        _export_tables(app, A, B, J)
        app["control_status"] = new_table([Column("status", ["OK"])])
    except Exception as e:
        app["control_status"] = new_table([Column("status", [f"ERR: {e}"])])
        app["last_error"] = new_table([Column("error", [str(e)])])

# ----------------- Control stream wiring -----------------
def _wire_control(app: ApplicationState):
    ctrl_cfg = dict(KAFKA_CONFIG)
    ctrl_cfg["auto.offset.reset"] = "earliest"  # catch messages sent before app start

    control_raw = kc.consume(
        ctrl_cfg, CONTROL_TOPIC,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=CONTROL_SPEC,
        table_type=kc.TableType.append(),
    )
    app["control_raw"]   = control_raw
    app["control_debug"] = control_raw.tail(10)

    control_latest = control_raw.last_by(["topicA", "topicB", "joinType"])
    app["control_latest"] = control_latest

    # call _apply on each change; return a small status string
    def _dispatch(topicA, topicB, joinType):
        _apply(topicA, topicB, joinType, app)
        return "applied"

    app["control_status"] = control_latest.update(['status = _dispatch(topicA, topicB, joinType)'])

# ----------------- App entrypoint -----------------
def start(app: ApplicationState):
    # 1) Export placeholders immediately so Panels/JS can always open them (no DEPENDENCY_FAILED)
    _export_tables(app, _empty_users(), _empty_accounts(), _empty_joined())
    app["last_error"] = new_table([Column("error", [])])  # visible slot for any future exception

    # 2) Wire the control stream; exports will be rebound when a valid control row arrives
    _wire_control(app)

# Register with App Mode
initialize(start)
```

---

## How to use / verify

1. Put both files in the `app/` folder and (re)start Deephaven.
    
2. Open Panels → you should see `first_scope_ui`, `second_scope_ui`, `final_ui`, `control_raw`, `control_debug`, `control_latest`, `control_status`, `last_error`.
    
    - The three `*_ui` tables will be empty but **open without errors** (placeholders).
        
3. Publish a control message (JSON object, not a quoted string) to the control topic:
    
    ```json
    {
      "topicA": "ccd01_sb_its_esp_tap3507_bishowocaseraw",
      "topicB": "ccd01_sb_its_esp_tap3507_bishowcasecurated",
      "joinType": "LEFT_OUTER",
      "ts": 1730412345123
    }
    ```
    
4. In Panels:
    
    - `control_debug` shows your row with non-null fields.
        
    - `control_status` becomes `OK`.
        
    - `first_scope_ui`, `second_scope_ui`, `final_ui` populate with live data.
        
    - If anything fails, `control_status` shows `ERR: …` and `last_error` contains the exception text.
        

This pattern avoids the dependency and session pitfalls, and it’s the standard way to expose dynamic content in **Application Mode**.


--------------------------------------------

Got it 👍 — you want to stay in **IDE mode** (not full app mode), but still make your joined/consumed tables visible to Angular (via JS API or Panels). The right way in Deephaven IDE mode is to use `get_app_state()` and put the tables into it. That registers them as **exports** that Angular (or any client) can subscribe to.

Here’s a **full updated working script** you can drop in, with `get_app_state` used correctly:

```python
from deephaven.stream.kafka import consumer as kc
from deephaven import dtypes as dt, empty_table
from deephaven.experimental.outer_joins import left_outer_join
from deephaven.appmode import get_app_state

# -------------------------------
# Kafka Topics and Config
# -------------------------------
TOPIC_USERS = "ccd01_sb_its_esp_tap3507_bishowocaseraw"
TOPIC_ACCTS = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC = "ccd01_sb_its_esp_tap3507_metadata"

KAFKA_CONFIG = {
    "bootstrap.servers": "pkc-...:9092",
    "auto.offset.reset": "latest",
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "OAUTHBEARER",
    # ... your existing auth configs here ...
}

# -------------------------------
# JSON Schemas
# -------------------------------
USER_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "name": dt.string,
    "email": dt.string,
    "age": dt.int64,
})

ACCOUNT_VALUE_SPEC = kc.json_spec({
    "userId": dt.string,
    "accountType": dt.string,
    "balance": dt.double,
})

CONTROL_SPEC = kc.json_spec({
    "topicA": dt.string,
    "topicB": dt.string,
    "joinType": dt.string,   # LEFT_OUTER | NATURAL | EXACT
    "ts": dt.int64,
})

# -------------------------------
# Helpers: placeholder tables
# -------------------------------
def _empty_users():
    return empty_table(0).update_view([
        "userId=(String)null",
        "name=(String)null",
        "email=(String)null",
        "age=(long)NULL_LONG",
    ])

def _empty_accounts():
    return empty_table(0).update_view([
        "userId=(String)null",
        "accountType=(String)null",
        "balance=(double)NaN",
    ])

def _empty_joined():
    return empty_table(0).update_view([
        "userId=(String)null",
        "accountType=(String)null",
        "balance=(double)NaN",
        "name=(String)null",
        "email=(String)null",
        "age=(long)NULL_LONG",
    ])

# -------------------------------
# Join helper
# -------------------------------
def _join_by_type(jtype: str, lhs, rhs, on_cols, join_cols):
    jt = (jtype or "LEFT_OUTER").upper()
    if jt == "LEFT_OUTER":
        return left_outer_join(lhs, rhs, on=on_cols, joins=join_cols)
    if jt == "NATURAL":
        return lhs.natural_join(rhs, on=on_cols, joins=join_cols)
    if jt == "EXACT":
        return lhs.exact_join(rhs, on=on_cols, joins=join_cols)
    raise ValueError(f"Unknown joinType: {jtype}")

# -------------------------------
# Apply a join from control topic
# -------------------------------
def _apply(topicA: str, topicB: str, joinType: str):
    left_raw = kc.consume(
        KAFKA_CONFIG, topicA,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=USER_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )
    right_raw = kc.consume(
        KAFKA_CONFIG, topicB,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=ACCOUNT_VALUE_SPEC,
        table_type=kc.TableType.append(),
    )

    left_ui = left_raw.view(["userId","name","email","age"])
    right_ui = right_raw.view(["userId","accountType","balance"])

    joined = _join_by_type(joinType, left_ui, right_ui,
                           on_cols=["userId"],
                           join_cols=["accountType","balance"])

    joined_ui = joined.view(["userId","accountType","balance","name","email","age"])

    # Expose to Angular / JS clients
    app = get_app_state()
    app["first_scope_ui"] = left_ui
    app["second_scope_ui"] = right_ui
    app["final_ui"] = joined_ui

    return "OK"

# -------------------------------
# Control stream
# -------------------------------
control_raw = kc.consume(
    KAFKA_CONFIG,
    CONTROL_TOPIC,
    key_spec=kc.KeyValueSpec.IGNORE,
    value_spec=CONTROL_SPEC,
    table_type=kc.TableType.append(),
)

control_debug = control_raw.tail(10)

control_latest = control_raw.last_by(["topicA","topicB","joinType"])

# when a new control msg arrives, apply it
control_status = control_latest.update([
    "status = _apply(topicA, topicB, joinType)"
])

# -------------------------------
# Initialize placeholders so Angular doesn’t fail before first control msg
# -------------------------------
app = get_app_state()
app["first_scope_ui"]  = _empty_users()
app["second_scope_ui"] = _empty_accounts()
app["final_ui"]        = _empty_joined()
app["control_raw"]     = control_raw
app["control_debug"]   = control_debug
app["control_status"]  = control_status
```

---

✅ With this script in IDE mode:

- Before any control message: Angular/JS sees three empty tables with correct schema.
    
- When Spring Boot (or producer) sends a control JSON like:
    
    ```json
    {"topicA":"ccd01_sb_its_esp_tap3507_bishowocaseraw","topicB":"ccd01_sb_its_esp_tap3507_bishowcasecurated","joinType":"LEFT_OUTER","ts":1730493241}
    ```
    
    → `_apply` is called, and the exports (`first_scope_ui`, `second_scope_ui`, `final_ui`) are rebound to the live joined tables.
    
- Angular (your `DeephavenService`) can `getTable("first_scope_ui")` etc and subscribe with no export failure.
    

---

Do you also want me to show you the **Angular-side snippet** that waits for these exports (`first_scope_ui`, etc.) to exist before setting the viewport, so you don’t hit the “DEPENDENCY_FAILED” race?