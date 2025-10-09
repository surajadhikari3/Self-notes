
You’re _almost_ there. Two things are causing the 503 and the “emitters = 0”:

1. Your SSE endpoint isn’t announcing itself as an SSE stream (so the browser tears it down and Spring returns 503 before any client is registered).
    
2. You only send to emitters when the Kafka value is a `LinkedHashMap`. If your deserializer isn’t the JSON one, the value will be a `String` and you skip sending—so Angular never sees anything.
    

Here’s a tight, working setup.

# 1) Spring Boot – SSE endpoint (fix 503 + keep-alive)

```java
@RestController
@RequestMapping("/api/kafka")
@RequiredArgsConstructor
@CrossOrigin(origins = "*")
public class KafkaController {

  private final KafkaListenerService kafkaListenerService;

  // Tell Spring/Tomcat this is a server-sent event stream
  @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
  public SseEmitter streamEvents() {
    return kafkaListenerService.streamEvents();
  }
}
```

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class KafkaListenerService {

  // thread-safe for concurrent iteration
  private final CopyOnWriteArrayList<SseEmitter> emitters = new CopyOnWriteArrayList<>();

  // Never timeout (or choose a large value like 30 minutes)
  public SseEmitter streamEvents() {
    SseEmitter emitter = new SseEmitter(0L);
    emitters.add(emitter);

    emitter.onCompletion(() -> emitters.remove(emitter));
    emitter.onTimeout(() -> emitters.remove(emitter));
    emitter.onError(e -> {
      log.warn("SSE emitter error: {}", e.getMessage());
      emitters.remove(emitter);
    });

    try {
      // send a first 'hello' so the connection is established and Angular knows it’s open
      emitter.send(SseEmitter.event().name("init").data("connected"));
    } catch (Exception e) {
      emitters.remove(emitter);
      emitter.completeWithError(e);
    }
    return emitter;
  }

  @KafkaListener(
      topics = "${spring.kafka.consumer.properties.topic}",
      containerFactory = "kafkaListenerContainerFactory" // if you’ve customized it
  )
  public void onMessage(ConsumerRecord<String, Object> record, Acknowledgment ack) {
    try {
      if (record.value() == null) {
        log.warn("Null value for key {}", record.key());
        ack.acknowledge();
        return;
      }

      // Normalize payload to String so Angular can display it
      String payload;
      if (record.value() instanceof String s) {
        payload = s;
      } else if (record.value() instanceof Map<?, ?> m) {
        payload = new ObjectMapper().writeValueAsString(m); // or inject/reuse a single ObjectMapper
      } else {
        payload = String.valueOf(record.value());
      }

      // fan-out
      for (SseEmitter emitter : emitters) {
        try {
          emitter.send(SseEmitter.event().name("message").data(payload));
        } catch (Exception e) {
          // drop dead emitters
          emitters.remove(emitter);
          log.warn("Removing dead emitter: {}", e.getMessage());
        }
      }

      ack.acknowledge();
    } catch (Exception e) {
      log.error("Error handling Kafka message key={}, value={}, ex={}",
          record.key(), record.value(), e.getMessage(), e);
    }
  }
}
```

**application.properties (important)**

```
# Do not time out async requests (prevents 503 on long-lived SSE)
spring.mvc.async.request-timeout=0
# If you use manual acks:
spring.kafka.listener.ack-mode=MANUAL_IMMEDIATE
```

> If you expect JSON values from Kafka, make sure your consumer uses the JSON deserializer (or Confluent Avro/JSON as appropriate). Otherwise your values will be plain strings.

# 2) Angular – EventSource usage (works once SSE is correct)

Service:

```ts
@Injectable({ providedIn: 'root' })
export class KafkaEventService {
  private eventSource: EventSource | null = null;

  connectToKafkaStream(): Observable<string> {
    return new Observable<string>((observer) => {
      this.eventSource = new EventSource('http://localhost:8080/api/kafka/stream');

      this.eventSource.onmessage = (event) => {
        observer.next(event.data);        // receive payload string
      };

      this.eventSource.onerror = (err) => {
        observer.error(err);
        this.eventSource?.close();
      };

      return () => this.eventSource?.close();
    });
  }
}
```

Component (your code is fine):

```ts
this.eventSubscription = this.kafkaService.connectToKafkaStream().subscribe({
  next: (event: string) => {
    console.log('Kafka Event Received:', event);
    this.events.push(event);
  },
  error: (err) => console.error('Error receiving Kafka events:', err),
});
```

Template (your code is fine):

```html
<ng-container *ngFor="let event of events">
  <p>{{ event }}</p>
</ng-container>
```

# Quick checklist

-  `/stream` has `produces = MediaType.TEXT_EVENT_STREAM_VALUE`.
    
-  `spring.mvc.async.request-timeout=0` is set.
    
-  On connect, you `send()` an initial event (e.g., `init`) so the browser keeps the stream open.
    
-  In the listener, always convert the payload to a string and send; don’t gate on `LinkedHashMap` only.
    
-  If you use manual acks, set `ack-mode=MANUAL_IMMEDIATE`.
    
-  CORS is enabled (you already have `@CrossOrigin("*")`).
    

After these tweaks, Angular’s first load should return HTTP 200 and your emitter list will contain at least one emitter; when Kafka messages arrive, they’ll be fanned out to the browser immediately.



----------------------------------


Yep—SSE is “live only.” On a browser refresh you lose in-memory state unless you (a) replay from the server or (b) hydrate from client storage. Do both for a great UX:

## A) Server-side replay (authoritative)

Keep a small ring buffer of the last N events; send that backlog to any new SSE subscriber before streaming live updates.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class KafkaListenerService {

  private final CopyOnWriteArrayList<SseEmitter> emitters = new CopyOnWriteArrayList<>();

  // ring buffer
  private static final int MAX_RECENT = 500;
  private final Deque<String> recent = new ConcurrentLinkedDeque<>();

  public SseEmitter streamEvents() {
    SseEmitter emitter = new SseEmitter(0L);
    emitters.add(emitter);

    emitter.onCompletion(() -> emitters.remove(emitter));
    emitter.onTimeout(() -> emitters.remove(emitter));
    emitter.onError(e -> { emitters.remove(emitter); });

    try {
      // 1) tell client we're on
      emitter.send(SseEmitter.event().name("init").data("connected"));

      // 2) REPLAY recent events
      for (String msg : recent) {
        emitter.send(SseEmitter.event().name("replay").data(msg));
      }
    } catch (Exception e) {
      emitters.remove(emitter);
      emitter.completeWithError(e);
    }
    return emitter;
  }

  @KafkaListener(topics = "${spring.kafka.consumer.properties.topic}")
  public void onMessage(ConsumerRecord<String, Object> record, Acknowledgment ack) {
    try {
      String payload = (record.value() instanceof String s)
          ? s
          : new ObjectMapper().writeValueAsString(record.value());

      // push into ring buffer
      recent.addLast(payload);
      while (recent.size() > MAX_RECENT) recent.pollFirst();

      // fan out
      for (SseEmitter emitter : emitters) {
        try {
          emitter.send(SseEmitter.event().name("message").data(payload));
        } catch (Exception e) {
          emitters.remove(emitter);
        }
      }
      ack.acknowledge();

    } catch (Exception e) {
      log.error("Kafka handle error", e);
    }
  }

  // Optional REST to fetch recent manually
  public List<String> getRecent(int limit) {
    return recent.stream().skip(Math.max(0, recent.size() - limit)).toList();
  }
}
```

Controller (optional REST):

```java
@GetMapping("/recent")
public List<String> recent(@RequestParam(defaultValue = "100") int limit) {
  return kafkaListenerService.getRecent(limit);
}
```

## B) Client-side hydration (fast UX)

Seed the UI from localStorage, then connect to SSE; keep caching as new events arrive so a refresh is instant.

```ts
@Injectable({ providedIn: 'root' })
export class KafkaEventService {
  private readonly KEY = 'kafka_events_v1';
  private eventSource: EventSource | null = null;

  loadCached(): string[] {
    const raw = localStorage.getItem(this.KEY);
    return raw ? JSON.parse(raw) : [];
  }
  saveCached(events: string[]) {
    localStorage.setItem(this.KEY, JSON.stringify(events.slice(-500)));
  }

  connect(): Observable<string> {
    return new Observable<string>((observer) => {
      this.eventSource = new EventSource('http://localhost:8080/api/kafka/stream');

      this.eventSource.onmessage = (e) => observer.next(e.data);
      this.eventSource.onerror = (err) => { observer.error(err); this.eventSource?.close(); };

      return () => this.eventSource?.close();
    });
  }
}
```

Component:

```ts
events: string[] = [];

ngOnInit() {
  // 1) hydrate from cache
  this.events = this.kafkaService.loadCached();

  // 2) (optional) pull server backlog if you exposed /recent
  // this.http.get<string[]>('/api/kafka/recent?limit=200').subscribe(list => {
  //   this.events = list;
  //   this.kafkaService.saveCached(this.events);
  // });

  // 3) live stream
  this.eventSubscription = this.kafkaService.connect().subscribe({
    next: (ev) => {
      this.events.push(ev);
      this.kafkaService.saveCached(this.events);
    },
    error: (err) => console.error(err)
  });
}
```

**Result:**

- New tab/refresh immediately shows cached events.
    
- Server also replays the last N events so every client can catch up accurately.
    
- Live messages continue as before.

---------------

here’s a clean controller you can drop in:

```java
package com.tdsecurities.bishowcase2025.controller;

import com.tdsecurities.bishowcase2025.service.KafkaListenerService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.util.List;

@RestController
@RequestMapping("/api/kafka")
@RequiredArgsConstructor
@CrossOrigin(origins = "*") // adjust for your domains in prod
public class KafkaController {

    private final KafkaListenerService kafkaListenerService;

    // SSE stream endpoint (Angular EventSource connects here)
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamEvents() {
        return kafkaListenerService.streamEvents();
    }

    // Optional: fetch recent (server-side replay) to hydrate UI on refresh
    @GetMapping("/recent")
    public List<String> recent(@RequestParam(defaultValue = "100") int limit) {
        return kafkaListenerService.getRecent(limit);
    }

    // Optional: simple health check (useful for debugging 503 vs 200)
    @GetMapping("/health")
    public String health() {
        return "OK";
    }
}
```

If you also want a manual way to clear the buffer during testing:

```java
    @DeleteMapping("/recent")
    public void clearRecent() { kafkaListenerService.clearRecent(); }
```

(Then add `public void clearRecent(){ recent.clear(); }` to the service.)


-------------------------------

gotcha—here’s a drop-in Angular setup that:

- hydrates from the backend `/recent` (authoritative),
    
- keeps a rolling cache in `localStorage` so a refresh/tab reopen paints instantly,
    
- stays live via `/stream`,
    
- dedupes and caps to the last N events,
    
- needs no extra clicks after page refresh.
    

---

# 1) app.module.ts

```ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, HttpClientModule],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

# 2) environment.ts

```ts
export const environment = {
  production: false,
  apiBase: 'http://localhost:8080/api/kafka'
};
```

# 3) kafka-event.service.ts

```ts
import { Injectable, NgZone } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, firstValueFrom } from 'rxjs';
import { environment } from '../../environments/environment';

@Injectable({ providedIn: 'root' })
export class KafkaEventService {
  private readonly KEY = 'kafka_events_v1';
  private readonly MAX = 500;                 // keep last N
  private eventSource: EventSource | null = null;
  private saving = false;

  private readonly eventsSubject = new BehaviorSubject<string[]>([]);
  public readonly events$ = this.eventsSubject.asObservable();

  constructor(private http: HttpClient, private zone: NgZone) {}

  /** Call once (e.g., AppComponent ngOnInit) */
  async init(): Promise<void> {
    // 1) seed from local cache (instant paint)
    const cached = this.loadCache();
    this.eventsSubject.next(cached);

    // 2) get authoritative backlog from server
    try {
      const recent = await firstValueFrom(
        this.http.get<string[]>(`${environment.apiBase}/recent?limit=${this.MAX}`)
      );
      this.setAndPersist(this.mergeDedupe(cached, recent));
    } catch (e) {
      // backend might be starting; keep cached UI and continue
      console.warn('recent fetch failed, using cache only for now', e);
    }

    // 3) open live stream (SSE auto-reconnects)
    this.openStream();
  }

  private openStream() {
    // Use NgZone to avoid Angular change detection storms
    this.zone.runOutsideAngular(() => {
      this.eventSource = new EventSource(`${environment.apiBase}/stream`);

      this.eventSource.onmessage = (evt: MessageEvent) => {
        const next = this.addEvent(evt.data);
        // jump back into Angular zone to update UI
        this.zone.run(() => this.eventsSubject.next(next));
      };

      this.eventSource.onerror = (err) => {
        console.error('SSE error', err);
        // Let browser auto-reconnect; do not close() here
      };
    });
  }

  /** Current list accessor if needed */
  getSnapshot(): string[] {
    return this.eventsSubject.getValue();
  }

  /** Cleanup (call on app destroy if you wish) */
  close() {
    this.eventSource?.close();
    this.eventSource = null;
  }

  /** ---------- helpers ---------- */

  private addEvent(e: string): string[] {
    const curr = this.eventsSubject.getValue();
    const next = [...curr, e].slice(-this.MAX);
    this.persist(next);
    return next;
  }

  private setAndPersist(list: string[]) {
    const next = list.slice(-this.MAX);
    this.eventsSubject.next(next);
    this.persist(next);
  }

  private mergeDedupe(a: string[], b: string[]): string[] {
    // simple string dedupe; if you later add IDs, dedupe by id
    const seen = new Set<string>();
    const merged: string[] = [];
    for (const v of [...a, ...b]) {
      if (!seen.has(v)) { seen.add(v); merged.push(v); }
    }
    return merged.slice(-this.MAX);
  }

  private loadCache(): string[] {
    try {
      const raw = localStorage.getItem(this.KEY);
      return raw ? JSON.parse(raw) : [];
    } catch { return []; }
  }

  private persist(list: string[]) {
    // throttle writes so we don’t hammer localStorage
    if (this.saving) return;
    this.saving = true;
    queueMicrotask(() => {
      try { localStorage.setItem(this.KEY, JSON.stringify(list)); }
      finally { this.saving = false; }
    });
  }
}
```

# 4) component usage

## entity-list.component.ts (or any component showing events)

```ts
import { Component, OnDestroy, OnInit } from '@angular/core';
import { Subscription } from 'rxjs';
import { KafkaEventService } from '../services/kafka-event.service';

@Component({
  selector: 'app-entity-list',
  templateUrl: './entity-list.component.html'
})
export class EntityListComponent implements OnInit, OnDestroy {
  events$ = this.kafkaSvc.events$;    // async stream for template
  private sub: Subscription | null = null;

  constructor(private kafkaSvc: KafkaEventService) {}

  async ngOnInit() {
    await this.kafkaSvc.init(); // hydrate + connect SSE
    // If you need imperative access, you can also subscribe:
    // this.sub = this.events$.subscribe(list => console.log('events size', list.length));
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();
    // optional: do NOT close SSE here if other components also use it.
    // If this is a global/singleton stream, leave it open.
  }
}
```

## entity-list.component.html

```html
<div class="container">
  <ng-container *ngIf="events$ | async as events">
    <ng-container *ngFor="let e of events">
      <p>{{ e }}</p>
    </ng-container>
  </ng-container>
</div>
```

---

## how this keeps state across refresh/close

- While running, every new event is pushed into a **BehaviorSubject** and **persisted to `localStorage`** (last 500 items).
    
- On a **refresh or tab reopen**, `init()`:
    
    1. paints the cached list immediately (fast UI),
        
    2. fetches `/recent` from your Spring controller to get the authoritative last N,
        
    3. reopens `/stream` for live updates (server also replays the ring buffer at connect).
        
- Because we persist on every event, closing the tab is safe—on next open, you still have a cache to paint instantly, and `/recent` reconciles any gap.
    

You can tune:

- `MAX` (client) and `MAX_RECENT` (server) to your retention needs,
    
- switch to an object shape `{id, ts, payload}` later for perfect dedupe via `id`,
    
- scope the service as a true singleton (providedIn root) so the SSE stream is shared across pages in the same SPA session.
----------------------------


You got it. Here’s a **drop-in, 0.40.2-safe app** with all fixes baked in:

- Control topic supports **JSON object** _or_ **single JSON string** payload.
    
- Correct listener signature `(update, is_replay)`.
    
- Right-side dedupe (`last_by("userId")`) + drop null keys to prevent join failure.
    
- **Graceful hot-swap** (keeps the previous generation alive ~2s so the IDE/Angular don’t see “dependency cancelled by user”).
    
- Exposes `control_raw`, `users_ui`, `accounts_ui`, `final_ui`, and `orchestrator_status`.
    

---

## 1) Descriptor — `C:\Deephaven\app.d\orchestrator.app`

```properties
id=kafka-orchestrator
name=Kafka Orchestrator
type=script
scriptType=python
enabled=true
file_0=orchestrator.py
```

## 2) App — `C:\Deephaven\app.d\orchestrator.py`

```python
# Deephaven 0.40.2 — Kafka orchestrator with graceful hot-swap

from threading import Lock, Thread
import time, json, math
import pandas as pd

import deephaven.dtypes as dt
from deephaven.appmode import get_app_state, ApplicationState
from deephaven.table_listener import listen
from deephaven.stream.kafka import consumer as kc
from deephaven.pandas import to_pandas
from deephaven import agg as agg
from deephaven import new_table
from deephaven.column import string_col
from deephaven import time as dhtime

# --------------------------- CONFIG ---------------------------
# Initial topics (can be changed at runtime via control topic)
DEFAULT_USERS_TOPIC    = "ccd01_sb_its_esp_tap3507_bishowcaseraw"
DEFAULT_ACCOUNTS_TOPIC = "ccd01_sb_its_esp_tap3507_bishowcasecurated"
CONTROL_TOPIC          = "ccd01_sb_its_esp_tap3507_metadata"

# Kafka client props (edit to match your cluster)
DATA_KAFKA = {
    "bootstrap.servers": "localhost:9092",
    "auto.offset.reset": "latest",
    # Example for Confluent Cloud:
    # "security.protocol": "SASL_SSL",
    # "sasl.mechanism": "PLAIN",
    # "sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required "
    #                     'username="<API_KEY>" password="<API_SECRET>";'
}
# Control consumer uses earliest so we always have a last row
CONTROL_KAFKA = dict(DATA_KAFKA)
CONTROL_KAFKA["auto.offset.reset"] = "earliest"

# Value specs
USER_SPEC = kc.json_spec({
    "userId": dt.string, "name": dt.string, "email": dt.string, "age": dt.int64
})
ACCOUNT_SPEC = kc.json_spec({
    "userId": dt.string, "accountType": dt.string, "balance": dt.double
})
CONTROL_JSON_SPEC   = kc.json_spec({"usersTopic": dt.string, "accountsTopic": dt.string, "joinType": dt.string})
CONTROL_SIMPLE_SPEC = kc.simple_spec(dt.string)  # value is a JSON string
# --------------------------------------------------------------

def consume_append(topic: str, spec, cfg):
    return kc.consume(
        dict(cfg),
        topic,
        key_spec=kc.KeyValueSpec.IGNORE,
        value_spec=spec,
        table_type=kc.TableType.append(),
    )

# ---- helpers (module-level; do NOT put inside the class) ----
def _scalar_str(val, default=""):
    """Return a clean string for scalar-like values; handle NaN / pd.NA."""
    try:
        if val is None:
            return default
        try:
            if pd.isna(val):
                return default
        except Exception:
            pass
        if isinstance(val, float) and math.isnan(val):
            return default
        return str(val).strip()
    except Exception:
        return default
# -------------------------------------------------------------

class Orchestrator:
    def __init__(self, app: ApplicationState):
        self.app = app
        self.lock = Lock()
        self.resources = []                 # current generation
        self._last_topics = (None, None, None)

    def _close_list(self, xs):
        for x in xs or []:
            try:
                x.close()
            except Exception:
                pass

    # ----------------- hot-swap with grace period -----------------
    def set_topics(self, users_topic: str, accounts_topic: str, join_type: str = "left"):
        if not users_topic or not accounts_topic:
            raise ValueError("Both users_topic and accounts_topic are required")

        with self.lock:
            old_resources = list(self.resources)  # close after swap

            new_res = []
            try:
                # sources
                users_raw = consume_append(users_topic, USER_SPEC, DATA_KAFKA);         new_res.append(users_raw)
                accts_raw = consume_append(accounts_topic, ACCOUNT_SPEC, DATA_KAFKA);   new_res.append(accts_raw)

                # projections
                users = users_raw.view(["userId","name","email","age"]);                new_res.append(users)
                accts = accts_raw.view(["userId","accountType","balance"]);             new_res.append(accts)

                # ensure unique right side & drop null keys for natural_join
                accts_non_null = accts.where("userId != null")
                try:
                    accts_one = accts_non_null.last_by("userId")
                except AttributeError:
                    accts_one = accts_non_null.agg_by(
                        [agg.last("accountType"), agg.last("balance")], by=["userId"]
                    )
                new_res.append(accts_one)

                jt = (join_type or "left").lower()
                if jt.startswith("inner"):
                    final = users.join(accts, on=["userId"], joins=["accountType","balance"])
                else:
                    final = users.natural_join(accts_one, on=["userId"], joins=["accountType","balance"])
                new_res.append(final)

                # export (same keys; Angular/IDE keep reading them)
                self.app["users_ui"]    = users
                self.app["accounts_ui"] = accts
                self.app["final_ui"]    = final

                # status row (handy in the Apps panel)
                self.app["orchestrator_status"] = new_table([
                    string_col("usersTopic",   [users_topic]),
                    string_col("accountsTopic",[accounts_topic]),
                    string_col("joinType",     [jt]),
                    string_col("lastApplied",  [str(dhtime.dh_now())]),
                ])

                # publish new generation; remember topics
                self.resources = new_res
                self._last_topics = (users_topic, accounts_topic, jt)
                print(f"[orchestrator] topics set users='{users_topic}' accounts='{accounts_topic}' join='{jt}'")

                # delayed close of previous generation (prevents UI “dependency cancelled”)
                def _close_after_delay(res_list):
                    try:
                        time.sleep(2.0)  # tune between 1–3s if needed
                        self._close_list(res_list)
                    except Exception:
                        pass
                Thread(target=_close_after_delay, args=(old_resources,), daemon=True).start()

            except Exception as e:
                # if building the new generation failed, close it and keep old live
                self._close_list(new_res)
                print("[orchestrator] set_topics error:", e)
                raise
    # --------------------------------------------------------------

    # -------- control handling --------
    def _parse_control_df(self, df: pd.DataFrame):
        cols = set(df.columns)
        if {"usersTopic","accountsTopic","joinType"}.issubset(cols):
            row = df.iloc[0]
            return (
                _scalar_str(row.get("usersTopic")),
                _scalar_str(row.get("accountsTopic")),
                _scalar_str(row.get("joinType"), "left") or "left",
            )
        if "Value" in cols:  # single string payload
            try:
                payload = json.loads(_scalar_str(df["Value"].iloc[0]))
                return (
                    _scalar_str(payload.get("usersTopic")),
                    _scalar_str(payload.get("accountsTopic")),
                    _scalar_str(payload.get("joinType"), "left") or "left",
                )
            except Exception as e:
                print("[orchestrator] control parse error:", e)
                return None
        print("[orchestrator] control row had unexpected columns:", list(cols))
        return None

    def _apply_last_control(self, ctrl_tbl):
        snap = ctrl_tbl.tail(1).snapshot()
        if snap.size == 0:
            return
        parsed = self._parse_control_df(to_pandas(snap))
        if not parsed:
            return
        users, accts, join = parsed
        if users and accts and (users, accts, join) != self._last_topics:
            print(f"[orchestrator] applying control: users='{users}', accounts='{accts}', join='{join}'")
            self.set_topics(users, accts, join)

    def start_control_listener(self, control_topic: str):
        # start with JSON spec; if it looks wrong, rebuild with simple
        ctrl = consume_append(control_topic, CONTROL_JSON_SPEC, CONTROL_KAFKA)

        def rebuild_simple():
            try: ctrl.close()
            except Exception: pass
            return consume_append(control_topic, CONTROL_SIMPLE_SPEC, CONTROL_KAFKA)

        def on_update(_update, _is_replay):
            try:
                self._apply_last_control(self.app["control_raw"])
            except Exception as e:
                print("[orchestrator] control listener err:", e)

        self.app["control_raw"] = ctrl
        disp = listen(ctrl, on_update)

        # initial apply & shape check
        try:
            snap = ctrl.tail(1).snapshot()
            df = to_pandas(snap) if snap.size > 0 else pd.DataFrame()
            if snap.size > 0 and not ({"usersTopic","accountsTopic"} & set(df.columns)):
                ctrl2 = rebuild_simple()
                self.app["control_raw"] = ctrl2
                try: disp.close()
                except Exception: pass
                disp2 = listen(ctrl2, on_update)
                self.resources.extend([ctrl2, disp2])
                self._apply_last_control(ctrl2)
            else:
                self._apply_last_control(ctrl)
                self.resources.extend([ctrl, disp])
        except Exception as e:
            print("[orchestrator] initial apply err:", e)
            self.resources.extend([ctrl, disp])

        print(f"[orchestrator] control listener on '{control_topic}'")

# --------------------------- BOOT ---------------------------
_app = get_app_state()
_orc = Orchestrator(_app)
try:
    _orc.set_topics(DEFAULT_USERS_TOPIC, DEFAULT_ACCOUNTS_TOPIC, "left")
except Exception as boot_err:
    print("[orchestrator] initial wiring failed:", boot_err)
_orc.start_control_listener(CONTROL_TOPIC)
print("[orchestrator] ready")
```

---

## Run & verify

1. Start Deephaven:
    

```bash
source /c/Users/suraj/source/apps-testing/venv/Scripts/activate
deephaven server --host localhost --port 10000
```

2. In the IDE → **Applications → Kafka Orchestrator**. Open:
    
    - `orchestrator_status` (shows current topics & last applied time)
        
    - `control_raw` (you’ll see control messages arrive)
        
    - `users_ui`, `accounts_ui`, `final_ui`
        
3. Produce a control message (either format works):
    

**Object form (preferred):**

```json
{"usersTopic":"<NEW_USERS>", "accountsTopic":"<NEW_ACCTS>", "joinType":"left"}
```

**Single-string form (value is JSON text):**

```
{"usersTopic":"<NEW_USERS>", "accountsTopic":"<NEW_ACCTS>", "joinType":"left"}
```

You should see in the console:

```
[orchestrator] applying control: users='<NEW_USERS>' accounts='<NEW_ACCTS>' join='left'
[orchestrator] topics set users='<NEW_USERS>' accounts='<NEW_ACCTS>' join='left'
```

Panels shouldn’t fail; Angular (reading the same exports via the IDE session) keeps working without any session churn.

---

### If anything still acts up

- “Exported table already failed” on `final_ui` → schema/key mismatch on the right side; we already de-dup & remove null `userId`. Confirm both topics have a `userId` string field.
    
- No rows in `control_raw` → control consumer not receiving (bootstrap/security/ACL/topic name).
    
- Share the first 10 lines of any new stack trace and I’ll pinpoint the exact fix.